# 03 — 数据库设计

## 1. 数据库架构策略

### 1.1 分库分表方案

| 库 | 分片键 | 分片策略 | 说明 |
|----|--------|---------|------|
| db_customer | customer_id | hash(customer_id) % 4 | 客户库 4 片 |
| db_order | customer_id | hash(customer_id) % 8 | 单据库 8 片 |
| db_invoice | customer_id | hash(customer_id) % 8 | 开票库 8 片（核心高并发写入） |
| db_input | company_id | hash(company_id) % 4 | 进项库 4 片（按钢贸企业分） |
| db_contract | customer_id | hash(customer_id) % 4 | 合同库 4 片 |
| db_payment | customer_id | hash(customer_id) % 4 | 付款库 4 片 |

分片中间件使用 **ShardingSphere-JDBC 5.x**，路由层直连，无代理层性能损耗。

### 1.2 公共字段约定

所有业务表均包含以下公共字段：

```sql
-- 公共字段
id              BIGINT          NOT NULL AUTO_INCREMENT COMMENT '主键ID (雪花算法)',
created_by      BIGINT          COMMENT '创建人ID',
created_time    DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
updated_by      BIGINT          COMMENT '更新人ID',
updated_time    DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
deleted         TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '逻辑删除 0-未删除 1-已删除',
version         INT             NOT NULL DEFAULT 1 COMMENT '乐观锁版本号',
tenant_id       BIGINT          NOT NULL COMMENT '租户ID（支持多钢贸企业SaaS化）'
```

---

## 2. 核心表结构设计

### 2.1 客户域 (db_customer)

#### t_customer — 客户主表

```sql
CREATE TABLE t_customer (
    id                  BIGINT          NOT NULL COMMENT '客户ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    customer_code       VARCHAR(32)     NOT NULL COMMENT '客户编号',
    customer_name       VARCHAR(128)    NOT NULL COMMENT '客户名称',
    customer_type       VARCHAR(16)     NOT NULL COMMENT '客户类型: VIP-大客户, DEALER-经销商, END_USER-用料企业',
    contact_name        VARCHAR(64)     COMMENT '联系人',
    contact_phone       VARCHAR(20)     COMMENT '联系电话',
    contact_email       VARCHAR(128)    COMMENT '联系邮箱',
    status              VARCHAR(16)     NOT NULL DEFAULT 'ACTIVE' COMMENT '状态: ACTIVE, INACTIVE, FROZEN',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant_code (tenant_id, customer_code),
    KEY idx_customer_name (customer_name),
    KEY idx_customer_type (tenant_id, customer_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='客户主表';
```

#### t_customer_invoice_config — 客户开票配置表

```sql
CREATE TABLE t_customer_invoice_config (
    id                      BIGINT          NOT NULL COMMENT '配置ID',
    tenant_id               BIGINT          NOT NULL COMMENT '租户ID',
    customer_id             BIGINT          NOT NULL COMMENT '客户ID',
    invoice_policy          VARCHAR(16)     NOT NULL COMMENT '开票策略: PAYMENT_FIRST-先款后票, INVOICE_FIRST-先票后款',
    credit_limit            DECIMAL(18,2)   DEFAULT 0 COMMENT '授信额度(先票后款时有效)',
    credit_used             DECIMAL(18,2)   DEFAULT 0 COMMENT '已用授信额度',
    allow_partial_invoice   TINYINT(1)      NOT NULL DEFAULT 1 COMMENT '是否允许部分开票',
    allow_merge_invoice     TINYINT(1)      NOT NULL DEFAULT 1 COMMENT '是否允许合并开票',
    match_strategy          VARCHAR(16)     NOT NULL DEFAULT 'FLEXIBLE' COMMENT '进项匹配策略: EXACT_MATCH-精确匹配, FLEXIBLE-灵活调整',
    price_adjustment        VARCHAR(16)     NOT NULL DEFAULT 'ADJUST_WEIGHT' COMMENT '价格调整方式: ADJUST_WEIGHT-调重量, ADJUST_PRICE-调单价, NONE-不调',
    target_tax_rate_min     DECIMAL(6,4)    COMMENT '目标税负率下限(%)',
    target_tax_rate_max     DECIMAL(6,4)    COMMENT '目标税负率上限(%)',
    target_margin_min       DECIMAL(6,4)    COMMENT '目标毛利率下限(%)',
    target_margin_max       DECIMAL(6,4)    COMMENT '目标毛利率上限(%)',
    -- 公共字段
    created_by              BIGINT,
    created_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by              BIGINT,
    updated_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                 TINYINT(1)      NOT NULL DEFAULT 0,
    version                 INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant_customer (tenant_id, customer_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='客户开票配置表';
```

#### t_customer_invoice_title — 客户开票抬头表

```sql
CREATE TABLE t_customer_invoice_title (
    id                  BIGINT          NOT NULL COMMENT '抬头ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    customer_id         BIGINT          NOT NULL COMMENT '客户ID',
    title_name          VARCHAR(256)    NOT NULL COMMENT '开票抬头名称（购买方名称）',
    tax_no              VARCHAR(32)     NOT NULL COMMENT '纳税人识别号',
    address             VARCHAR(256)    COMMENT '地址',
    phone               VARCHAR(32)     COMMENT '电话',
    bank_name           VARCHAR(128)    COMMENT '开户行',
    bank_account        VARCHAR(64)     COMMENT '银行账号',
    invoice_type        VARCHAR(16)     NOT NULL DEFAULT 'SPECIAL' COMMENT '发票类型: SPECIAL-专票, NORMAL-普票, ELECTRONIC-电子票',
    is_default          TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '是否默认抬头',
    status              VARCHAR(16)     NOT NULL DEFAULT 'ACTIVE' COMMENT '状态',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_customer_id (tenant_id, customer_id),
    KEY idx_tax_no (tax_no)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='客户开票抬头表';
```

### 2.2 单据域 (db_order)

#### t_delivery_order — 提货单主表

```sql
CREATE TABLE t_delivery_order (
    id                  BIGINT          NOT NULL COMMENT '提货单ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    order_no            VARCHAR(32)     NOT NULL COMMENT '提货单号',
    customer_id         BIGINT          NOT NULL COMMENT '客户ID',
    delivery_company    VARCHAR(256)    NOT NULL COMMENT '提货方公司名称（提货抬头）',
    total_amount        DECIMAL(18,2)   NOT NULL COMMENT '单据总金额',
    total_weight        DECIMAL(18,4)   NOT NULL COMMENT '总重量(吨)',
    invoiced_amount     DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '已开票金额',
    invoice_status      VARCHAR(16)     NOT NULL DEFAULT 'UN_INVOICED' COMMENT '开票状态: UN_INVOICED-未开票, PARTIAL-部分开票, FULLY-全部开票',
    order_date          DATE            NOT NULL COMMENT '提货日期',
    status              VARCHAR(16)     NOT NULL DEFAULT 'ACTIVE' COMMENT '单据状态',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant_order_no (tenant_id, order_no),
    KEY idx_customer_id (tenant_id, customer_id),
    KEY idx_invoice_status (tenant_id, invoice_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='提货单主表';
```

#### t_delivery_order_item — 提货单明细表

```sql
CREATE TABLE t_delivery_order_item (
    id                  BIGINT          NOT NULL COMMENT '明细ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    order_id            BIGINT          NOT NULL COMMENT '提货单ID',
    item_no             INT             NOT NULL COMMENT '行号',
    product_name        VARCHAR(128)    NOT NULL COMMENT '品名',
    specification       VARCHAR(128)    NOT NULL COMMENT '规格型号',
    unit                VARCHAR(16)     NOT NULL DEFAULT '吨' COMMENT '单位',
    quantity            DECIMAL(18,4)   NOT NULL COMMENT '数量（重量）',
    unit_price          DECIMAL(18,2)   NOT NULL COMMENT '单价（含税）',
    amount              DECIMAL(18,2)   NOT NULL COMMENT '金额（含税）',
    tax_rate            DECIMAL(4,2)    NOT NULL DEFAULT 13.00 COMMENT '税率(%)',
    invoiced_quantity   DECIMAL(18,4)   NOT NULL DEFAULT 0 COMMENT '已开票数量',
    invoiced_amount     DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '已开票金额',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_order_id (order_id),
    KEY idx_product (tenant_id, product_name, specification)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='提货单明细表';
```

### 2.3 付款域 (db_payment)

#### t_payment_record — 付款记录表

```sql
CREATE TABLE t_payment_record (
    id                  BIGINT          NOT NULL COMMENT '付款记录ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    payment_no          VARCHAR(32)     NOT NULL COMMENT '付款流水号',
    customer_id         BIGINT          NOT NULL COMMENT '客户ID',
    payer_name          VARCHAR(256)    NOT NULL COMMENT '付款方名称',
    payer_account       VARCHAR(64)     COMMENT '付款方账号',
    payer_bank          VARCHAR(128)    COMMENT '付款方开户行',
    payment_type        VARCHAR(16)     NOT NULL COMMENT '付款类型: CORPORATE-对公, PERSONAL-对私',
    is_third_party      TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '是否第三方代付',
    third_party_auth_id BIGINT          COMMENT '第三方付款授权书ID',
    amount              DECIMAL(18,2)   NOT NULL COMMENT '付款金额',
    payment_date        DATETIME(3)     NOT NULL COMMENT '付款到账时间',
    allocated_amount    DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '已分配至单据金额',
    status              VARCHAR(16)     NOT NULL DEFAULT 'CONFIRMED' COMMENT '状态: PENDING, CONFIRMED, ALLOCATED',
    remark              VARCHAR(512)    COMMENT '备注',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant_payment_no (tenant_id, payment_no),
    KEY idx_customer_id (tenant_id, customer_id),
    KEY idx_payer_name (payer_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='付款记录表';
```

#### t_payment_order_rel — 付款与单据关联表

```sql
CREATE TABLE t_payment_order_rel (
    id                  BIGINT          NOT NULL COMMENT 'ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    payment_id          BIGINT          NOT NULL COMMENT '付款记录ID',
    order_id            BIGINT          NOT NULL COMMENT '提货单ID',
    allocated_amount    DECIMAL(18,2)   NOT NULL COMMENT '分配金额',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_payment_id (payment_id),
    KEY idx_order_id (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='付款与单据关联表';
```

#### t_third_party_auth — 第三方付款授权书

```sql
CREATE TABLE t_third_party_auth (
    id                  BIGINT          NOT NULL COMMENT '授权书ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    customer_id         BIGINT          NOT NULL COMMENT '客户ID',
    payment_id          BIGINT          NOT NULL COMMENT '关联付款记录ID',
    delegator_name      VARCHAR(256)    NOT NULL COMMENT '委托方名称',
    agent_name          VARCHAR(256)    NOT NULL COMMENT '受托方（实际付款方）名称',
    auth_amount         DECIMAL(18,2)   NOT NULL COMMENT '授权金额',
    auth_file_url       VARCHAR(512)    NOT NULL COMMENT '授权书文件URL',
    ocr_result          JSON            COMMENT 'OCR识别结果JSON',
    review_status       VARCHAR(16)     NOT NULL DEFAULT 'PENDING' COMMENT '审核状态: PENDING, APPROVED, REJECTED',
    reviewer_id         BIGINT          COMMENT '审核人ID',
    review_time         DATETIME(3)     COMMENT '审核时间',
    review_remark       VARCHAR(512)    COMMENT '审核备注',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_customer_id (tenant_id, customer_id),
    KEY idx_payment_id (payment_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='第三方付款授权书';
```

### 2.4 开票域 (db_invoice) — 核心

#### t_invoice_apply — 开票申请单

```sql
CREATE TABLE t_invoice_apply (
    id                      BIGINT          NOT NULL COMMENT '申请单ID',
    tenant_id               BIGINT          NOT NULL COMMENT '租户ID',
    apply_no                VARCHAR(32)     NOT NULL COMMENT '申请单号',
    customer_id             BIGINT          NOT NULL COMMENT '客户ID',
    -- 开票抬头信息（冗余快照，防止后续抬头变更影响历史记录）
    title_id                BIGINT          NOT NULL COMMENT '抬头ID',
    buyer_name              VARCHAR(256)    NOT NULL COMMENT '购买方名称',
    buyer_tax_no            VARCHAR(32)     NOT NULL COMMENT '购买方税号',
    buyer_address           VARCHAR(256)    COMMENT '购买方地址',
    buyer_phone             VARCHAR(32)     COMMENT '购买方电话',
    buyer_bank_name         VARCHAR(128)    COMMENT '购买方开户行',
    buyer_bank_account      VARCHAR(64)     COMMENT '购买方银行账号',
    -- 发票类型
    invoice_type            VARCHAR(16)     NOT NULL COMMENT '发票类型: SPECIAL-专票, NORMAL-普票, ELECTRONIC-电子票',
    -- 金额信息
    total_amount            DECIMAL(18,2)   NOT NULL COMMENT '申请开票总金额（含税）',
    total_tax               DECIMAL(18,2)   NOT NULL COMMENT '总税额',
    total_amount_without_tax DECIMAL(18,2)  NOT NULL COMMENT '总金额（不含税）',
    -- 状态信息
    apply_status            VARCHAR(16)     NOT NULL DEFAULT 'SUBMITTED' COMMENT '申请状态（见状态机）',
    -- 抬头匹配标记
    title_match_delivery    TINYINT(1)      NOT NULL DEFAULT 1 COMMENT '开票抬头是否与提货抬头一致',
    contract_id             BIGINT          COMMENT '关联补签合同ID（抬头不一致时）',
    -- 进项匹配信息
    input_match_status      VARCHAR(16)     DEFAULT 'PENDING' COMMENT '进项匹配状态: PENDING, MATCHED, PARTIAL, FAILED',
    -- 人工调整标记
    is_manual_adjust        TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '是否经过人工调整',
    manual_adjust_reason    VARCHAR(512)    COMMENT '人工调整原因',
    manual_adjuster_id      BIGINT          COMMENT '人工调整操作人',
    -- 审批信息
    approval_status         VARCHAR(16)     COMMENT '审批状态: PENDING, APPROVED, REJECTED',
    approver_id             BIGINT          COMMENT '审批人ID',
    approval_time           DATETIME(3)     COMMENT '审批时间',
    -- 诺诺网信息
    nuonuo_serial_no        VARCHAR(64)     COMMENT '诺诺网流水号',
    nuonuo_order_no         VARCHAR(64)     COMMENT '诺诺网订单号',
    -- 发票结果
    invoice_code            VARCHAR(32)     COMMENT '发票代码',
    invoice_number          VARCHAR(32)     COMMENT '发票号码',
    invoice_date            DATE            COMMENT '开票日期',
    invoice_pdf_url         VARCHAR(512)    COMMENT '发票PDF文件URL',
    invoice_ofd_url         VARCHAR(512)    COMMENT '发票OFD文件URL',
    -- 错误信息
    error_code              VARCHAR(32)     COMMENT '错误码',
    error_message           VARCHAR(1024)   COMMENT '错误信息',
    retry_count             INT             NOT NULL DEFAULT 0 COMMENT '重试次数',
    -- 幂等键
    idempotent_key          VARCHAR(64)     NOT NULL COMMENT '幂等键（防重复提交）',
    -- 公共字段
    created_by              BIGINT,
    created_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by              BIGINT,
    updated_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                 TINYINT(1)      NOT NULL DEFAULT 0,
    version                 INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_apply_no (tenant_id, apply_no),
    UNIQUE KEY uk_idempotent (idempotent_key),
    KEY idx_customer_status (tenant_id, customer_id, apply_status),
    KEY idx_apply_status (tenant_id, apply_status),
    KEY idx_nuonuo_serial (nuonuo_serial_no),
    KEY idx_invoice_code_num (invoice_code, invoice_number)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='开票申请单';
```

#### t_invoice_apply_order_rel — 申请单与单据关联表

```sql
CREATE TABLE t_invoice_apply_order_rel (
    id                  BIGINT          NOT NULL COMMENT 'ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    apply_id            BIGINT          NOT NULL COMMENT '申请单ID',
    order_id            BIGINT          NOT NULL COMMENT '提货单ID',
    order_amount        DECIMAL(18,2)   NOT NULL COMMENT '本次从该单据开票金额',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_apply_id (apply_id),
    KEY idx_order_id (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='开票申请单与单据关联表';
```

#### t_invoice_apply_item — 开票申请明细行

```sql
CREATE TABLE t_invoice_apply_item (
    id                      BIGINT          NOT NULL COMMENT '明细ID',
    tenant_id               BIGINT          NOT NULL COMMENT '租户ID',
    apply_id                BIGINT          NOT NULL COMMENT '申请单ID',
    item_no                 INT             NOT NULL COMMENT '行号',
    -- 商品信息
    product_name            VARCHAR(128)    NOT NULL COMMENT '品名（发票上显示的商品名称）',
    specification           VARCHAR(128)    COMMENT '规格型号',
    unit                    VARCHAR(16)     NOT NULL DEFAULT '吨' COMMENT '单位',
    quantity                DECIMAL(18,4)   NOT NULL COMMENT '数量',
    unit_price              DECIMAL(18,6)   NOT NULL COMMENT '单价（不含税）',
    amount                  DECIMAL(18,2)   NOT NULL COMMENT '金额（不含税）',
    tax_rate                DECIMAL(4,2)    NOT NULL COMMENT '税率(%)',
    tax_amount              DECIMAL(18,2)   NOT NULL COMMENT '税额',
    total_amount            DECIMAL(18,2)   NOT NULL COMMENT '含税金额',
    -- 税收分类编码
    tax_category_code       VARCHAR(32)     COMMENT '税收分类编码',
    tax_category_name       VARCHAR(128)    COMMENT '税收分类名称',
    -- 来源信息
    source_order_id         BIGINT          COMMENT '来源提货单ID',
    source_order_item_id    BIGINT          COMMENT '来源提货单明细ID',
    -- 进项匹配信息
    matched_input_id        BIGINT          COMMENT '匹配的进项库存ID',
    matched_input_quantity  DECIMAL(18,4)   COMMENT '匹配的进项数量',
    -- 调整标记
    is_adjusted             TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '是否经过调整（0-原始 1-已调整）',
    adjust_type             VARCHAR(16)     COMMENT '调整类型: WEIGHT-调重量, PRICE-调单价, MANUAL-人工',
    original_quantity       DECIMAL(18,4)   COMMENT '调整前原始数量',
    original_unit_price     DECIMAL(18,6)   COMMENT '调整前原始单价',
    -- 公共字段
    created_by              BIGINT,
    created_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by              BIGINT,
    updated_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                 TINYINT(1)      NOT NULL DEFAULT 0,
    version                 INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_apply_id (apply_id),
    KEY idx_matched_input (matched_input_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='开票申请明细行';
```

#### t_invoice_adjust_log — 人工调整日志

```sql
CREATE TABLE t_invoice_adjust_log (
    id                  BIGINT          NOT NULL COMMENT 'ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    apply_id            BIGINT          NOT NULL COMMENT '申请单ID',
    operator_id         BIGINT          NOT NULL COMMENT '操作人ID',
    operator_name       VARCHAR(64)     NOT NULL COMMENT '操作人名称',
    adjust_type         VARCHAR(32)     NOT NULL COMMENT '调整类型: ADD_ITEM, REMOVE_ITEM, MODIFY_ITEM, CHANGE_TITLE, AMOUNT_ADJUST',
    before_snapshot     JSON            NOT NULL COMMENT '调整前快照（JSON）',
    after_snapshot      JSON            NOT NULL COMMENT '调整后快照（JSON）',
    amount_diff         DECIMAL(18,2)   COMMENT '金额差异',
    reason              VARCHAR(512)    NOT NULL COMMENT '调整原因',
    approval_status     VARCHAR(16)     NOT NULL DEFAULT 'PENDING' COMMENT '审批状态',
    approver_id         BIGINT          COMMENT '审批人',
    approval_time       DATETIME(3)     COMMENT '审批时间',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_apply_id (apply_id),
    KEY idx_operator (tenant_id, operator_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='人工调整日志表';
```

### 2.5 进项域 (db_input)

#### t_input_inventory — 进项库存表

```sql
CREATE TABLE t_input_inventory (
    id                  BIGINT          NOT NULL COMMENT '进项库存ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    input_invoice_id    BIGINT          COMMENT '来源进项发票ID',
    product_name        VARCHAR(128)    NOT NULL COMMENT '品名',
    product_category    VARCHAR(64)     NOT NULL COMMENT '品类大类（用于模糊匹配）',
    specification       VARCHAR(128)    NOT NULL COMMENT '规格型号',
    unit                VARCHAR(16)     NOT NULL DEFAULT '吨' COMMENT '单位',
    tax_rate            DECIMAL(4,2)    NOT NULL COMMENT '税率(%)',
    unit_price          DECIMAL(18,6)   NOT NULL COMMENT '进项单价（不含税）',
    unit_price_with_tax DECIMAL(18,6)   NOT NULL COMMENT '进项单价（含税）',
    total_quantity      DECIMAL(18,4)   NOT NULL COMMENT '总数量',
    available_quantity  DECIMAL(18,4)   NOT NULL COMMENT '可用数量（= 总数量 - 已锁定 - 已消耗）',
    locked_quantity     DECIMAL(18,4)   NOT NULL DEFAULT 0 COMMENT '已锁定数量（开票中）',
    consumed_quantity   DECIMAL(18,4)   NOT NULL DEFAULT 0 COMMENT '已消耗数量（已开票）',
    status              VARCHAR(16)     NOT NULL DEFAULT 'AVAILABLE' COMMENT '状态: AVAILABLE, EXHAUSTED',
    input_date          DATE            NOT NULL COMMENT '进项日期',
    supplier_name       VARCHAR(256)    COMMENT '供应商名称',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_product (tenant_id, product_name, specification),
    KEY idx_category (tenant_id, product_category),
    KEY idx_status (tenant_id, status),
    KEY idx_available (tenant_id, product_name, available_quantity)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='进项库存表';
```

#### t_input_lock_record — 进项锁定记录表

```sql
CREATE TABLE t_input_lock_record (
    id                  BIGINT          NOT NULL COMMENT 'ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    input_inventory_id  BIGINT          NOT NULL COMMENT '进项库存ID',
    apply_id            BIGINT          NOT NULL COMMENT '开票申请单ID',
    apply_item_id       BIGINT          NOT NULL COMMENT '开票申请明细ID',
    locked_quantity     DECIMAL(18,4)   NOT NULL COMMENT '锁定数量',
    lock_status         VARCHAR(16)     NOT NULL DEFAULT 'LOCKED' COMMENT '状态: LOCKED-已锁定, CONFIRMED-已确认, RELEASED-已释放',
    lock_time           DATETIME(3)     NOT NULL COMMENT '锁定时间',
    lock_expire_time    DATETIME(3)     NOT NULL COMMENT '锁定过期时间',
    confirm_time        DATETIME(3)     COMMENT '确认时间',
    release_time        DATETIME(3)     COMMENT '释放时间',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_inventory_id (input_inventory_id),
    KEY idx_apply_id (apply_id),
    KEY idx_status_expire (lock_status, lock_expire_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='进项锁定记录表';
```

#### t_tax_account — 税务账户（税负率追踪）

```sql
CREATE TABLE t_tax_account (
    id                          BIGINT          NOT NULL COMMENT 'ID',
    tenant_id                   BIGINT          NOT NULL COMMENT '租户ID',
    period                      VARCHAR(8)      NOT NULL COMMENT '税务期间(YYYYMM)',
    cumulative_sales_amount     DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '累计销售额（不含税）',
    cumulative_output_tax       DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '累计销项税额',
    cumulative_input_tax        DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '累计进项税额',
    cumulative_tax_payable      DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '累计应纳税额',
    tax_burden_rate             DECIMAL(6,4)    NOT NULL DEFAULT 0 COMMENT '当期税负率(%)',
    -- 公共字段
    created_by                  BIGINT,
    created_time                DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by                  BIGINT,
    updated_time                DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                     TINYINT(1)      NOT NULL DEFAULT 0,
    version                     INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant_period (tenant_id, period)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='税务账户表';
```

### 2.6 合同域 (db_contract)

#### t_contract — 合同/补充协议表

```sql
CREATE TABLE t_contract (
    id                  BIGINT          NOT NULL COMMENT '合同ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    contract_no         VARCHAR(32)     NOT NULL COMMENT '合同编号',
    contract_type       VARCHAR(16)     NOT NULL COMMENT '合同类型: SUPPLEMENT-补充协议, THIRD_PARTY_AUTH-第三方授权',
    customer_id         BIGINT          NOT NULL COMMENT '客户ID',
    apply_id            BIGINT          COMMENT '关联开票申请单ID',
    -- 合同双方
    party_a_name        VARCHAR(256)    NOT NULL COMMENT '甲方（我方企业名称）',
    party_b_name        VARCHAR(256)    NOT NULL COMMENT '乙方（客户/开票方名称）',
    -- 原提货方/新开票方
    original_delivery_company VARCHAR(256)  COMMENT '原提货方名称',
    new_invoice_company       VARCHAR(256)  COMMENT '新开票方名称',
    -- 合同文件
    template_id         BIGINT          COMMENT '合同模板ID',
    draft_file_url      VARCHAR(512)    COMMENT '合同草稿PDF URL',
    signed_file_url     VARCHAR(512)    COMMENT '已签署合同PDF URL',
    -- 签署方式
    sign_method         VARCHAR(16)     COMMENT '签署方式: ONLINE-在线电子签, OFFLINE-线下签署',
    -- 在线签署信息
    esign_flow_id       VARCHAR(64)     COMMENT '电子签章平台流程ID',
    esign_sign_url      VARCHAR(512)    COMMENT '电子签章签署链接',
    party_a_signed      TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '甲方是否已签',
    party_a_sign_time   DATETIME(3)     COMMENT '甲方签署时间',
    party_b_signed      TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '乙方是否已签',
    party_b_sign_time   DATETIME(3)     COMMENT '乙方签署时间',
    -- 线下签署信息
    uploaded_file_url   VARCHAR(512)    COMMENT '上传的盖章合同URL',
    ocr_result          JSON            COMMENT '印章OCR识别结果',
    -- 状态与审核
    contract_status     VARCHAR(16)     NOT NULL DEFAULT 'DRAFT' COMMENT '合同状态: DRAFT, PENDING_SIGN, SIGNING, SIGNED, UPLOADED, REVIEWING, EFFECTIVE, REJECTED, EXPIRED',
    reviewer_id         BIGINT          COMMENT '审核人ID',
    review_time         DATETIME(3)     COMMENT '审核时间',
    review_remark       VARCHAR(512)    COMMENT '审核备注',
    -- 有效期
    effective_date      DATE            COMMENT '生效日期',
    expire_date         DATE            COMMENT '过期日期',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_contract_no (tenant_id, contract_no),
    KEY idx_customer_id (tenant_id, customer_id),
    KEY idx_apply_id (apply_id),
    KEY idx_status (tenant_id, contract_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='合同/补充协议表';
```

#### t_contract_template — 合同模板表

```sql
CREATE TABLE t_contract_template (
    id                  BIGINT          NOT NULL COMMENT '模板ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    template_name       VARCHAR(128)    NOT NULL COMMENT '模板名称',
    template_type       VARCHAR(16)     NOT NULL COMMENT '模板类型: SUPPLEMENT, THIRD_PARTY_AUTH',
    template_file_url   VARCHAR(512)    NOT NULL COMMENT '模板文件URL（Word/PDF）',
    placeholder_config  JSON            NOT NULL COMMENT '占位符配置（JSON: 变量名→描述）',
    status              VARCHAR(16)     NOT NULL DEFAULT 'ACTIVE' COMMENT '状态',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_type (tenant_id, template_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='合同模板表';
```

---

## 3. 索引设计要点

### 3.1 核心查询场景与索引匹配

| 查询场景 | 使用表 | 索引 |
|---------|--------|------|
| 客户查询自己的开票申请列表 | t_invoice_apply | idx_customer_status (tenant_id, customer_id, apply_status) |
| 后台查询待处理的开票申请 | t_invoice_apply | idx_apply_status (tenant_id, apply_status) |
| 根据诺诺网回调查询申请单 | t_invoice_apply | idx_nuonuo_serial (nuonuo_serial_no) |
| 查询可用进项库存 | t_input_inventory | idx_available (tenant_id, product_name, available_quantity) |
| 扫描过期锁定记录 | t_input_lock_record | idx_status_expire (lock_status, lock_expire_time) |
| 查询付款方名称校验抬头 | t_payment_record | idx_payer_name (payer_name) |

### 3.2 ES 索引（CQRS 查询端）

开票申请的列表查询走 Elasticsearch，减轻 MySQL 压力：

```json
{
  "invoice_apply_index": {
    "mappings": {
      "properties": {
        "applyId":          { "type": "long" },
        "applyNo":          { "type": "keyword" },
        "tenantId":         { "type": "long" },
        "customerId":       { "type": "long" },
        "customerName":     { "type": "text", "analyzer": "ik_smart" },
        "buyerName":        { "type": "text", "analyzer": "ik_smart" },
        "buyerTaxNo":       { "type": "keyword" },
        "invoiceType":      { "type": "keyword" },
        "totalAmount":      { "type": "double" },
        "applyStatus":      { "type": "keyword" },
        "invoiceCode":      { "type": "keyword" },
        "invoiceNumber":    { "type": "keyword" },
        "invoiceDate":      { "type": "date" },
        "createdTime":      { "type": "date" },
        "updatedTime":      { "type": "date" }
      }
    }
  }
}
```

---

## 4. Redis 数据结构设计

| Key 模式 | 类型 | 用途 | TTL |
|---------|------|------|-----|
| `input:available:{tenantId}:{productName}:{spec}` | String (数值) | 进项可用库存计数器 | 永久 |
| `input:lock:{inputId}:{applyId}` | Hash | 进项锁定详情 | 30min |
| `invoice:idempotent:{idempotentKey}` | String | 开票幂等键 | 24h |
| `customer:config:{customerId}` | Hash | 客户开票配置缓存 | 10min |
| `customer:title:{customerId}` | List<Hash> | 客户开票抬头缓存 | 10min |
| `tax:account:{tenantId}:{period}` | Hash | 税务账户缓存 | 5min |
| `invoice:apply:status:{applyId}` | String | 开票进度状态（WebSocket推送用）| 1h |
| `rate:limit:invoice:{customerId}` | String (计数器) | 开票频率限制 | 滑动窗口 |

---

## 5. ER 关系图（核心表）

```
t_customer ─┬─< t_customer_invoice_config
            ├─< t_customer_invoice_title
            ├─< t_delivery_order ──< t_delivery_order_item
            ├─< t_payment_record ──< t_payment_order_rel >── t_delivery_order
            │                    └─< t_third_party_auth
            ├─< t_invoice_apply ─┬─< t_invoice_apply_item ──> t_input_inventory
            │                    ├─< t_invoice_apply_order_rel >── t_delivery_order
            │                    ├─< t_invoice_adjust_log
            │                    └──> t_contract
            └─< t_contract

t_input_inventory ──< t_input_lock_record >── t_invoice_apply

t_tax_account (独立按租户+期间)
t_contract_template (独立模板管理)
```
