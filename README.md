# AiInvoiceClerk — 钢贸企业财务开票智能体

## 项目简介

AiInvoiceClerk 是一套面向钢贸企业的 **全自动智能开票系统**，支持客户自助申请开票、进项库存智能匹配、税负率/毛利区间自动调控、合同在线签署、诺诺网电子发票开具等全链路能力。

## 技术栈

| 层级 | 技术选型 |
|------|---------|
| 前端 | Vue 3 + TypeScript + Element Plus + Vite |
| 后端 | Java 17 + Spring Boot 3 + Spring Cloud Alibaba |
| 数据库 | MySQL 8 (分库分表) + Redis 7 Cluster |
| 消息队列 | Apache RocketMQ 5 |
| 搜索 | Elasticsearch 8 |
| 网关 | Spring Cloud Gateway |
| 注册/配置中心 | Nacos 2.x |
| 分布式事务 | Seata |
| 分布式锁 | Redisson |
| 对象存储 | MinIO / 阿里云 OSS |
| 电子签章 | e签宝 / 法大大 |
| 开票接口 | 诺诺网 (电子税务局) |
| 容器化 | Docker + Kubernetes |

## 文档目录

- [系统架构设计](docs/01-architecture-design.md)
- [业务流程设计](docs/02-business-process-design.md)
- [数据库设计](docs/03-database-design.md)
- [API接口设计](docs/04-api-design.md)
- [高并发与性能设计](docs/05-high-concurrency-design.md)
- [第三方集成设计](docs/06-integration-design.md)
- [安全与合规设计](docs/07-security-compliance-design.md)
- [前端设计 (Vue 3)](docs/08-frontend-design.md)

## 设计目标

- 支持 **1 万并发用户** 同时在线操作
- 全自动开票流程，减少人工干预
- 灵活适配不同客户开票策略（先款后票 / 先票后款）
- 智能进项库存匹配与税负率调控
- 在线合同签署与线下盖章上传双通道
- 完整的审计追踪与合规保障
