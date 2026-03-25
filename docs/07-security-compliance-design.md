# 07 — 安全与合规设计

## 1. 认证授权

### 1.1 认证方案

```
┌── 认证架构 ────────────────────────────────────────────┐
│                                                        │
│  客户端 → Gateway → auth-service                       │
│                                                        │
│  认证方式：                                             │
│  ├─ 账号密码登录 (BCrypt 哈希)                           │
│  ├─ 手机验证码登录                                      │
│  ├─ 企业 SSO 对接 (SAML/OIDC)                          │
│  └─ API Key 认证 (系统对接场景)                          │
│                                                        │
│  Token 方案：JWT + Redis                                │
│  ├─ Access Token: 有效期 2h，存放用户基本信息             │
│  ├─ Refresh Token: 有效期 7d，存 Redis，支持主动失效      │
│  ├─ Token 黑名单（登出、修改密码时加入）                   │
│  └─ 多端登录控制（可配置：允许/互踢/限制N个设备）          │
│                                                        │
│  网关鉴权：                                              │
│  ├─ Gateway Filter 统一解析 JWT                         │
│  ├─ 白名单路径跳过（/login, /callback/*）                │
│  ├─ 解析后将用户信息放入请求 Header 传递到下游服务          │
│  └─ 下游服务通过 ThreadLocal 获取当前用户                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 1.2 权限模型（RBAC）

```
角色体系：
├─ 客户端角色
│   ├─ CUSTOMER_ADMIN (客户管理员) — 管理开票抬头、查看所有申请
│   ├─ CUSTOMER_OPERATOR (客户操作员) — 提交开票申请、下载发票
│   └─ CUSTOMER_VIEWER (客户查看员) — 只读
│
└─ 管理端角色
    ├─ FINANCE_ADMIN (财务管理员) — 全部权限
    ├─ FINANCE_OPERATOR (财务操作员) — 开票处理、人工调整
    ├─ FINANCE_REVIEWER (财务审核员) — 审批调整、审核合同
    ├─ TAX_MANAGER (税务经理) — 税负配置、进项管理
    └─ SYSTEM_ADMIN (系统管理员) — 客户配置、系统配置
```

### 1.3 数据权限隔离

```
┌── 数据隔离策略 ────────────────────────────────────────┐
│                                                        │
│  租户隔离（SaaS 多钢贸企业场景）                          │
│  ├─ 所有数据表包含 tenant_id 字段                        │
│  ├─ MyBatis 拦截器自动注入 tenant_id 条件                │
│  └─ 防止跨租户数据泄露                                   │
│                                                        │
│  客户隔离（客户只看自己的数据）                            │
│  ├─ 客户查询自动追加 customer_id = 当前用户.customerId    │
│  ├─ 接口层校验资源归属                                   │
│  └─ 防止横向越权                                        │
│                                                        │
│  操作员隔离（管理端按负责客户分配）                        │
│  ├─ 操作员 ↔ 客户绑定关系                                │
│  └─ 只能操作分配给自己的客户                              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 2. 数据安全

### 2.1 敏感数据加密

| 数据类型 | 存储加密 | 传输加密 | 脱敏展示 |
|---------|---------|---------|---------|
| 银行账号 | AES-256 | TLS 1.2+ | 只显示后4位 |
| 纳税人识别号 | AES-256 | TLS 1.2+ | 显示前4后4位 |
| 手机号 | AES-256 | TLS 1.2+ | 138****5678 |
| 身份证号 | AES-256 | TLS 1.2+ | 31****0521 |
| 密码 | BCrypt hash | TLS 1.2+ | 不可逆 |
| API Secret | AES-256 | TLS 1.2+ | 不展示 |

加密存储使用 MyBatis TypeHandler 自动加解密，业务代码无感知。

### 2.2 日志脱敏

```
日志中自动脱敏以下字段：
├─ 请求/响应中的敏感字段（基于注解 @SensitiveField）
├─ SQL 日志中的参数值（配置敏感字段列表）
├─ 异常堆栈中的敏感变量值
└─ 使用正则替换：手机号、身份证、银行卡号模式
```

### 2.3 SQL 注入防护

- MyBatis 全面使用 `#{}` 预编译参数
- 禁止使用 `${}` 拼接 SQL
- 开启 SQL 防火墙（Druid WallFilter）
- 定期安全扫描（OWASP ZAP）

### 2.4 XSS/CSRF 防护

- XSS：全局输出编码（Spring Security + Content-Security-Policy Header）
- CSRF：前后端分离使用 JWT，不需要传统 CSRF Token
- 文件上传：校验文件类型（Magic Number 校验，非仅后缀名）

## 3. 接口安全

### 3.1 防重放攻击

```
关键接口（开票提交、人工调整）使用时间戳 + Nonce + 签名：

1. 客户端生成：
   timestamp = 当前时间戳（5分钟有效窗口）
   nonce = UUID（一次性，Redis 存储去重，TTL=5min）
   signature = HMAC-SHA256(timestamp + nonce + body, secret)

2. 服务端校验：
   ├─ timestamp 在 ±5 分钟内
   ├─ nonce 未使用过（Redis SETNX）
   └─ signature 验签通过
```

### 3.2 业务级幂等

```
开票申请幂等设计：

idempotentKey 生成规则：
  MD5(customerId + orderIds排序拼接 + titleId + amount + 日期)

校验流程：
1. Redis SETNX idempotent:{key} → 成功则继续
2. 失败则查询已有申请单 → 返回已有申请信息
3. 开票成功后 idempotent key 保留 24h
```

### 3.3 防刷策略

| 维度 | 策略 | 说明 |
|------|------|------|
| 用户级 | 1 分钟内最多 10 次开票申请 | 滑动窗口 |
| IP 级 | 1 分钟内最多 100 次请求 | 令牌桶 |
| 全局 | 总 QPS 不超过 Gateway 限流 | 令牌桶 |
| 验证码 | 连续 3 次操作失败后要求验证码 | 行为验证码 |

## 4. 审计追踪

### 4.1 操作日志

所有关键操作记录审计日志：

```sql
CREATE TABLE t_audit_log (
    id              BIGINT          NOT NULL,
    tenant_id       BIGINT          NOT NULL,
    user_id         BIGINT          NOT NULL,
    user_name       VARCHAR(64)     NOT NULL,
    user_type       VARCHAR(16)     NOT NULL COMMENT 'CUSTOMER/ADMIN',
    module          VARCHAR(32)     NOT NULL COMMENT '模块: INVOICE/CONTRACT/INPUT/PAYMENT',
    action          VARCHAR(32)     NOT NULL COMMENT '操作: CREATE/UPDATE/DELETE/APPROVE/REJECT',
    resource_type   VARCHAR(32)     NOT NULL COMMENT '资源类型',
    resource_id     VARCHAR(64)     NOT NULL COMMENT '资源ID',
    detail          JSON            COMMENT '操作详情（变更前后对比）',
    ip_address      VARCHAR(64)     COMMENT '操作IP',
    user_agent      VARCHAR(512)    COMMENT '浏览器信息',
    request_id      VARCHAR(64)     COMMENT '请求链路ID',
    created_time    DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    PRIMARY KEY (id),
    KEY idx_tenant_user (tenant_id, user_id, created_time),
    KEY idx_resource (resource_type, resource_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='审计日志表';
```

### 4.2 记录范围

| 操作类型 | 记录内容 |
|---------|---------|
| 开票申请提交 | 申请人、单据、金额、抬头 |
| 人工调整 | 调整人、调整前后对比、调整原因 |
| 审批操作 | 审批人、审批结果、审批意见 |
| 合同签署 | 签署方、签署时间、签署方式 |
| 配置变更 | 变更人、变更项、变更前后值 |
| 发票红冲 | 操作人、红冲原因、原发票信息 |
| 系统异常 | 异常类型、影响范围、处理结果 |

### 4.3 审计日志不可篡改

- 审计日志表设为 **只追加**（INSERT ONLY）
- 应用层无 UPDATE/DELETE 权限
- 定期导出归档到独立存储
- 关键日志同步写入区块链/可信时间戳（可选）

## 5. 合规要求

### 5.1 税务合规

| 合规项 | 措施 |
|--------|------|
| 发票真实性 | 所有发票通过正规渠道（诺诺网→电子税务局）开具 |
| 金额准确性 | 发票金额 ≤ 实际交易金额，系统强制校验 |
| 进销匹配 | 销项必须有对应进项支撑，系统自动匹配 |
| 税负率合理 | 税负率在行业合理区间内，系统自动调控 |
| 数据留痕 | 所有开票过程完整记录，可追溯 |
| 发票保管 | 电子发票永久保存，满足存档要求 |

### 5.2 数据保护合规

- 遵循《个人信息保护法》，用户数据最小化采集
- 数据出境限制（如涉及，需评估）
- 用户注销后数据保留策略（税务数据依法保留，个人信息匿名化）
- 数据泄露应急预案

### 5.3 电子合同合规

- 使用持有 CA 牌照的电子签章服务
- 符合《电子签名法》要求
- 签署过程完整留痕（签署意愿确认、身份验证、时间戳）
- 签署后的文件带有可验证的数字签名

## 6. 灾备方案

### 6.1 数据备份

| 备份对象 | 备份策略 | 恢复 RPO | 恢复 RTO |
|---------|---------|---------|---------|
| MySQL | 每日全量 + 实时 Binlog | < 1 分钟 | < 30 分钟 |
| Redis | RDB 每小时 + AOF 实时 | < 1 秒 | < 10 分钟 |
| MinIO/OSS | 跨区域复制 | 0 | < 5 分钟 |
| RocketMQ | 同步双写 | 0 | < 5 分钟 |
| ES | 每日快照 | < 24 小时 | < 1 小时 |

### 6.2 容灾架构

```
主数据中心 (Region-A)              备数据中心 (Region-B)
├─ 全量服务部署                     ├─ 数据库从库（异步复制）
├─ 全量中间件                       ├─ MinIO 跨区域同步
└─ 正常承担所有流量                  └─ 可快速升主切换（手动决策）

切换流程：
1. 监控发现主中心不可用
2. 运维团队确认并决策
3. DNS 切换到备中心
4. 备中心数据库升主
5. 启动备中心全部服务
6. 恢复服务（RTO 目标 < 30 分钟）
```
