# 05 — 高并发与性能设计

## 1. 并发目标

| 指标 | 目标值 | 说明 |
|------|--------|------|
| 同时在线用户 | 10,000 | 并发连接 |
| 查询类 QPS | 50,000 | 读多写少，缓存命中率 > 95% |
| 开票提交 TPS | 2,000 | 核心写操作 |
| 开票处理吞吐 | 500/s | 含运算+进项匹配+诺诺网调用 |
| 接口平均响应时间 | < 200ms | P99 < 1s |
| 系统可用性 | 99.95% | 年停机不超过 4.4 小时 |

## 2. 接入层优化

### 2.1 Nginx 负载均衡

```
                        ┌──── Nginx Cluster (Keepalived VIP) ────┐
                        │                                        │
                        │  upstream gateway {                    │
                        │    least_conn;                         │
                        │    server gw-1:8080 weight=5;          │
                        │    server gw-2:8080 weight=5;          │
                        │    server gw-3:8080 weight=5;          │
                        │    keepalive 256;                      │
                        │  }                                     │
                        │                                        │
                        │  # 静态资源 CDN 回源                     │
                        │  # Gzip 压缩                           │
                        │  # HTTP/2 启用                         │
                        │  # 连接池复用                           │
                        │  # WebSocket 升级支持                   │
                        └────────────────────────────────────────┘
```

关键配置：
- `worker_processes auto` — 自动匹配 CPU 核数
- `worker_connections 65535` — 单 Worker 最大连接
- `keepalive_timeout 65` — 长连接超时
- `proxy_connect_timeout 3s` — 后端连接超时
- `proxy_read_timeout 30s` — 后端读超时

### 2.2 Spring Cloud Gateway 限流

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: invoice-apply
          uri: lb://invoice-service
          predicates:
            - Path=/api/v1/customer/invoice/apply
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 2000  # 每秒令牌补充速率
                redis-rate-limiter.burstCapacity: 5000   # 令牌桶容量
                key-resolver: "#{@userKeyResolver}"      # 按用户限流
            - name: CircuitBreaker
              args:
                name: invoiceApplyCB
                fallbackUri: forward:/fallback/invoice
```

使用 Sentinel 做更精细的流控：
- **流量控制**：QPS 阈值 + 预热模式（冷启动 10s 预热到阈值）
- **熔断降级**：慢调用比例 > 50% 触发熔断，5s 后半开探测
- **热点参数**：防止单个客户刷接口

## 3. 应用层优化

### 3.1 线程池设计

```
┌─── invoice-service 线程模型 ─────────────────────────────┐
│                                                          │
│  Tomcat 线程池                                            │
│  ├─ max-threads: 400                                     │
│  ├─ min-spare-threads: 50                                │
│  └─ accept-count: 500                                    │
│                                                          │
│  业务线程池（隔离）                                         │
│  ├─ invoiceCalculatePool (核心20, 最大50, 队列500)         │
│  │   用于：开票运算引擎计算                                 │
│  ├─ inputMatchPool (核心10, 最大30, 队列200)               │
│  │   用于：进项库存匹配                                     │
│  ├─ nuonuoCallPool (核心10, 最大20, 队列100)               │
│  │   用于：调用诺诺网接口（I/O密集型）                       │
│  └─ notifyPool (核心5, 最大20, 队列1000)                   │
│      用于：消息推送                                         │
│                                                          │
│  拒绝策略：CallerRunsPolicy（由调用线程执行，背压）           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 3.2 异步化设计

核心开票流程采用"接收快返回 + 异步处理 + 实时推送"模式：

```
客户提交开票申请
  │
  ├─ 同步阶段 (<100ms)
  │   ├─ 参数校验
  │   ├─ 幂等校验（Redis SETNX）
  │   ├─ 开票资格校验（Redis 缓存）
  │   ├─ 抬头一致性校验
  │   ├─ 写入申请单（MySQL）
  │   ├─ 发送 MQ 消息 → INVOICE_APPLY_SUBMITTED
  │   └─ 返回 applyId + applyNo（立即响应客户）
  │
  └─ 异步阶段 (MQ 消费)
      ├─ 开票运算引擎计算
      ├─ 进项库存匹配 + 锁定
      ├─ 税负率校验 + 调控
      ├─ 调用诺诺网开票
      ├─ 发票回写
      └─ WebSocket 推送进度 → 客户实时看到进度
```

### 3.3 缓存策略

```
┌─────────── 多级缓存架构 ──────────────────────────────────┐
│                                                           │
│  L1: JVM 本地缓存 (Caffeine)                               │
│  ├─ 客户开票配置    TTL=2min  maxSize=5000                  │
│  ├─ 税收分类编码    TTL=1h    maxSize=10000                 │
│  ├─ 合同模板       TTL=30min maxSize=100                   │
│  └─ 命中率目标 > 60%                                       │
│                                                           │
│  L2: Redis Cluster 分布式缓存                               │
│  ├─ 客户开票抬头    TTL=10min                               │
│  ├─ 进项可用库存    TTL=永久(实时更新)                        │
│  ├─ 开票幂等键      TTL=24h                                │
│  ├─ 税务账户数据    TTL=5min                                │
│  ├─ 开票进度状态    TTL=1h                                  │
│  └─ 命中率目标 > 95%                                       │
│                                                           │
│  L3: MySQL (最终数据源)                                     │
│  └─ 读写分离：写主库，读从库                                  │
│                                                           │
│  缓存更新策略：                                              │
│  ├─ 读：Cache-Aside (旁路缓存)                              │
│  ├─ 写：Write-Through + MQ 异步刷新                         │
│  └─ 一致性：Canal 监听 Binlog → 刷新 Redis                   │
│                                                           │
│  缓存穿透防护：                                              │
│  ├─ BloomFilter 拦截不存在的 key                             │
│  └─ 空值缓存 TTL=30s                                       │
│                                                           │
│  缓存雪崩防护：                                              │
│  ├─ TTL 加随机抖动 (±20%)                                   │
│  └─ 预热 + 自动续期                                         │
│                                                           │
│  缓存击穿防护：                                              │
│  └─ 热点 key 使用 Redisson 分布式锁重建                      │
└───────────────────────────────────────────────────────────┘
```

## 4. 数据层优化

### 4.1 MySQL 优化

#### 读写分离

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  MySQL 主库   │────▶│  MySQL 从库1  │────▶│  MySQL 从库2  │
│  (写入)       │     │  (读取)       │     │  (读取)       │
└──────────────┘     └──────────────┘     └──────────────┘
      │
  半同步复制（rpl_semi_sync_master_wait_for_slave_count=1）
  保证至少一个从库同步完成
```

ShardingSphere 读写分离配置：
- 写操作 → 主库
- 读操作 → 从库（权重轮询）
- 同一事务内读写 → 强制走主库

#### 分库分表

- 分片键选择 `customer_id`，保证同一客户的数据在同一分片，减少跨分片查询
- 开票申请表预估单表数据量控制在 500w 以内（8 分片）
- 历史数据按年归档到冷存储

#### SQL 优化要点

- 所有查询走索引，禁止全表扫描
- 避免大事务，单事务控制在 200ms 以内
- 批量操作使用 `INSERT ... ON DUPLICATE KEY UPDATE`
- 进项锁定使用 `SELECT ... FOR UPDATE NOWAIT`（避免长等待）

### 4.2 Redis Cluster

```
┌──── Redis Cluster (6节点, 3主3从) ────────────┐
│                                               │
│  节点配置：                                    │
│  ├─ 单节点内存: 16GB                           │
│  ├─ maxmemory-policy: allkeys-lfu             │
│  ├─ cluster-node-timeout: 5000                │
│  └─ tcp-backlog: 511                          │
│                                               │
│  16384 slot 分配：                              │
│  ├─ Master-1: 0-5460                          │
│  ├─ Master-2: 5461-10922                      │
│  └─ Master-3: 10923-16383                     │
│                                               │
│  Pipeline 批量操作减少 RTT                      │
│  Lua 脚本保证原子性（如库存扣减）                 │
│                                               │
└───────────────────────────────────────────────┘
```

进项库存扣减 Lua 脚本示例：

```lua
-- 进项库存预锁定 Lua 脚本
-- KEYS[1] = available counter key
-- KEYS[2] = lock record key
-- ARGV[1] = required quantity
-- ARGV[2] = lock expire seconds
-- ARGV[3] = lock detail json

local available = tonumber(redis.call('GET', KEYS[1]) or '0')
local required = tonumber(ARGV[1])

if available >= required then
    redis.call('DECRBY', KEYS[1], required)
    redis.call('SET', KEYS[2], ARGV[3], 'EX', ARGV[2])
    return 1  -- 锁定成功
else
    return 0  -- 库存不足
end
```

### 4.3 Elasticsearch 查询优化

- 开票申请列表查询走 ES，MySQL 写入后通过 Canal → MQ → ES 同步
- 索引按月滚动：`invoice_apply_202503`
- 使用 routing 按 `customerId` 路由，客户查询落单分片
- 常用查询场景预聚合

## 5. 消息队列设计

### 5.1 RocketMQ Topic 规划

| Topic | Tag | 生产者 | 消费者 | 说明 |
|-------|-----|--------|--------|------|
| INVOICE_APPLY | SUBMITTED | invoice-service | invoice-service (engine) | 开票申请已提交 |
| INVOICE_APPLY | PROCESSING | invoice-service (engine) | invoice-service (issuer) | 运算完成待开票 |
| INVOICE_RESULT | SUCCESS | invoice-service | notify-service, order-service | 开票成功 |
| INVOICE_RESULT | FAILED | invoice-service | notify-service | 开票失败 |
| INPUT_MATCH | LOCK | input-service | invoice-service | 进项锁定结果 |
| INPUT_MATCH | RELEASE | input-service | — | 进项释放 |
| INPUT_INVENTORY | REPLENISH | input-service | invoice-service | 新进项入库触发待开票重试 |
| CONTRACT | SIGN_COMPLETE | contract-service | invoice-service | 合同签署完成 |
| CONTRACT | REVIEW_COMPLETE | contract-service | invoice-service | 合同审核完成 |

### 5.2 消息可靠性保障

```
生产者 → Broker → 消费者

1. 发送端：
   ├─ 事务消息（开票申请提交 + 发 MQ 的原子性）
   │   ├─ Half Message → 本地事务 → Commit/Rollback
   │   └─ 回查机制：15s 后回查事务状态
   └─ 同步发送 + 重试 3 次

2. Broker 端：
   ├─ 同步刷盘（SYNC_FLUSH）
   ├─ 同步复制（SYNC_MASTER）
   └─ 双主双从架构

3. 消费端：
   ├─ 手动 ACK
   ├─ 消费幂等（Redis 去重 key: msg_consumed:{msgId}）
   ├─ 顺序消费（同一 applyId hash 到同一 Queue）
   └─ 死信队列 → 监控告警 → 人工介入
```

## 6. 分布式事务

### 6.1 开票核心链路事务

```
开票申请提交涉及多服务操作：

┌──────────────────────────────────────────────────────┐
│  使用 RocketMQ 事务消息 保证最终一致性                    │
│                                                      │
│  1. invoice-service 发送 Half Message                │
│  2. 执行本地事务：                                     │
│     ├─ 插入 t_invoice_apply                          │
│     ├─ 插入 t_invoice_apply_order_rel                │
│     └─ 更新 t_delivery_order.invoiced_amount         │
│  3. Commit Message → 触发异步处理                     │
│                                                      │
│  如果本地事务失败 → Rollback Message                   │
│  如果网络异常 → Broker 15s 后回查事务状态               │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  进项锁定 + 开票 使用 TCC 模式                         │
│                                                      │
│  Try:                                                │
│  ├─ Redis 预锁定进项库存                               │
│  └─ 记录锁定流水                                      │
│                                                      │
│  Confirm:                                            │
│  ├─ MySQL 更新进项已消耗数量                           │
│  ├─ 清除 Redis 锁定记录                               │
│  └─ 更新开票状态                                      │
│                                                      │
│  Cancel:                                             │
│  ├─ Redis 恢复可用库存                                │
│  ├─ 删除锁定流水                                      │
│  └─ 更新开票状态为 FAILED                             │
└──────────────────────────────────────────────────────┘
```

### 6.2 兜底对账

每日凌晨跑对账任务：
- 比对 Redis 进项库存计数器 与 MySQL 实际库存数量
- 扫描超时未确认的锁定记录并释放
- 比对诺诺网发票状态与本地状态
- 不一致数据自动修复或告警

## 7. 弹性伸缩

### 7.1 K8s HPA 配置

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: invoice-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: invoice-service
  minReplicas: 5
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: rocketmq_consumer_lag
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 3
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

### 7.2 预热机制

- 大促/月底开票高峰前提前扩容
- 新 Pod 启动后预热缓存（客户配置、税收编码等）
- Readiness Probe 延迟 30s，确保预热完成后才接流量

## 8. 可观测性

### 8.1 监控体系

```
┌────── 可观测性三支柱 ─────────────────────────────────────┐
│                                                          │
│  Metrics (Prometheus + Grafana)                          │
│  ├─ JVM 指标（堆内存、GC、线程）                            │
│  ├─ 接口 QPS / RT / 错误率                                │
│  ├─ 开票成功率 / 处理时长                                   │
│  ├─ 进项库存水位                                           │
│  ├─ 消息队列积压量                                         │
│  ├─ 缓存命中率                                             │
│  └─ 自定义业务指标                                         │
│                                                          │
│  Tracing (SkyWalking / Jaeger)                           │
│  ├─ 全链路追踪：Gateway → Service → DB/Redis/MQ           │
│  ├─ 慢请求分析（P99 > 1s 自动告警）                         │
│  └─ 诺诺网接口调用链路追踪                                   │
│                                                          │
│  Logging (ELK)                                           │
│  ├─ 结构化日志（JSON 格式）                                 │
│  ├─ TraceId 串联全链路                                     │
│  ├─ 敏感信息脱敏（税号、银行账号等）                          │
│  └─ 日志级别动态调整（Nacos 热配置）                         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 8.2 告警规则

| 告警项 | 条件 | 级别 | 通知方式 |
|--------|------|------|---------|
| 接口错误率 | > 5% 持续 1min | P1 Critical | 电话 + 钉钉 |
| 接口 P99 延迟 | > 3s 持续 2min | P2 Warning | 钉钉 |
| 消息队列积压 | > 10000 持续 5min | P2 Warning | 钉钉 |
| 开票成功率 | < 95% 持续 5min | P1 Critical | 电话 + 钉钉 |
| 进项库存水位 | 品类可用量 < 100 吨 | P3 Info | 邮件 |
| Pod CPU 使用率 | > 85% 持续 3min | P2 Warning | 钉钉 |
| MySQL 慢查询 | > 1s 且频次 > 10/min | P2 Warning | 钉钉 |
| Redis 内存使用率 | > 80% | P2 Warning | 钉钉 |

## 9. 容量规划

### 9.1 硬件资源预估（1 万并发）

| 组件 | 节点数 | 单节点配置 | 说明 |
|------|--------|-----------|------|
| Nginx | 2 (主备) | 4C 8G | Keepalived VIP |
| Gateway | 3-10 (HPA) | 4C 8G | 弹性伸缩 |
| invoice-service | 5-20 (HPA) | 8C 16G | 核心服务，最大弹性 |
| customer-service | 3 | 4C 8G | |
| order-service | 3 | 4C 8G | |
| input-service | 3-8 | 4C 8G | 进项匹配计算密集 |
| contract-service | 2 | 4C 8G | |
| payment-service | 3 | 4C 8G | |
| notify-service | 3 | 4C 8G | WebSocket 长连接 |
| file-service | 2 | 4C 8G | |
| auth-service | 3 | 4C 8G | |
| MySQL | 1主2从 | 16C 64G 1T SSD | IOPS > 10000 |
| Redis | 6 (3主3从) | 8C 16G | |
| RocketMQ | 2NS + 4Broker | 8C 16G 500G SSD | |
| ES | 3 | 8C 32G 500G SSD | |
| Nacos | 3 | 4C 8G | |
| MinIO | 4 | 4C 8G 2T | 纠删码模式 |

### 9.2 网络带宽

- 内网带宽：万兆互联
- 公网带宽：200Mbps（主要是 PDF 下载）
- CDN：静态资源 + 发票 PDF 缓存分发

### 9.3 存储预估

| 数据类型 | 月增量 | 年增量 | 保留策略 |
|---------|--------|--------|---------|
| 开票申请（MySQL） | 50w 行 | 600w 行 | 热数据 2 年，冷归档 |
| 进项库存（MySQL） | 10w 行 | 120w 行 | 永久保留 |
| 发票 PDF（OSS） | 10GB | 120GB | 永久保留（合规要求） |
| 合同文件（OSS） | 2GB | 24GB | 永久保留 |
| ES 索引 | 5GB | 60GB | 保留 1 年 |
| 日志（ELK） | 100GB | 1.2TB | 保留 6 个月 |
