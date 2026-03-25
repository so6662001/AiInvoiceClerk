# 10 — 收票智能体数据库设计

## 1. 新增分库

| 库 | 分片键 | 分片策略 | 说明 |
|----|--------|---------|------|
| db_purchase | supplier_id | hash(supplier_id) % 4 | 采购/进货库 4 片 |
| db_input（已有，扩展） | company_id | hash(company_id) % 4 | 进项库 4 片 |

---

## 2. 新增/扩展表结构

### 2.1 供应商域 (db_purchase)

#### t_supplier — 供应商主表

```sql
CREATE TABLE t_supplier (
    id                  BIGINT          NOT NULL COMMENT '供应商ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    supplier_code       VARCHAR(32)     NOT NULL COMMENT '供应商编号',
    supplier_name       VARCHAR(256)    NOT NULL COMMENT '供应商名称',
    tax_no              VARCHAR(32)     NOT NULL COMMENT '纳税人识别号',
    address             VARCHAR(256)    COMMENT '地址',
    phone               VARCHAR(32)     COMMENT '电话',
    bank_name           VARCHAR(128)    COMMENT '开户行',
    bank_account        VARCHAR(64)     COMMENT '银行账号',
    contact_name        VARCHAR(64)     COMMENT '联系人',
    contact_phone       VARCHAR(20)     COMMENT '联系人电话',
    supplier_type       VARCHAR(16)     NOT NULL DEFAULT 'NORMAL' COMMENT '供应商类型: STEEL_MILL-钢厂, TRADER-贸易商, NORMAL-普通',
    cooperation_status  VARCHAR(16)     NOT NULL DEFAULT 'ACTIVE' COMMENT '合作状态: ACTIVE-合作中, INACTIVE-暂停合作, BLACKLIST-黑名单',
    status              VARCHAR(16)     NOT NULL DEFAULT 'ACTIVE' COMMENT '状态',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant_code (tenant_id, supplier_code),
    UNIQUE KEY uk_tenant_tax_no (tenant_id, tax_no),
    KEY idx_supplier_name (supplier_name),
    KEY idx_cooperation_status (tenant_id, cooperation_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='供应商主表';
```

#### t_supplier_alias — 供应商别名表（辅助智能识别）

```sql
CREATE TABLE t_supplier_alias (
    id                  BIGINT          NOT NULL COMMENT 'ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    supplier_id         BIGINT          NOT NULL COMMENT '供应商ID',
    alias_name          VARCHAR(256)    NOT NULL COMMENT '别名（来自发票上的销方名称）',
    alias_tax_no        VARCHAR(32)     COMMENT '别名对应的税号',
    source              VARCHAR(16)     NOT NULL DEFAULT 'AUTO' COMMENT '来源: AUTO-系统自动学习, MANUAL-人工添加',
    match_count         INT             NOT NULL DEFAULT 1 COMMENT '匹配次数（用于置信度统计）',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant_alias (tenant_id, alias_name),
    KEY idx_supplier_id (supplier_id),
    KEY idx_alias_tax_no (alias_tax_no)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='供应商别名表';
```

### 2.2 进货单域 (db_purchase)

#### t_purchase_order — 进货单主表

```sql
CREATE TABLE t_purchase_order (
    id                  BIGINT          NOT NULL COMMENT '进货单ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    order_no            VARCHAR(32)     NOT NULL COMMENT '进货单号',
    supplier_id         BIGINT          NOT NULL COMMENT '供应商ID',
    supplier_name       VARCHAR(256)    NOT NULL COMMENT '供应商名称（冗余）',
    buyer_company       VARCHAR(256)    NOT NULL COMMENT '采购方公司名称（本企业）',
    total_amount        DECIMAL(18,2)   NOT NULL COMMENT '进货单总金额（含税）',
    total_amount_without_tax DECIMAL(18,2) NOT NULL COMMENT '总金额（不含税）',
    total_tax           DECIMAL(18,2)   NOT NULL COMMENT '总税额',
    total_weight        DECIMAL(18,4)   NOT NULL COMMENT '总重量(吨)',
    -- 收票核销字段
    invoiced_amount     DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '已收票金额（含税）',
    invoiced_tax        DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '已收票税额',
    pending_amount      DECIMAL(18,2)   AS (total_amount - invoiced_amount) STORED COMMENT '待收票金额',
    invoice_status      VARCHAR(16)     NOT NULL DEFAULT 'PENDING_INVOICE' COMMENT '收票状态: PENDING_INVOICE-待收票, PARTIAL_INVOICED-部分收票, FULLY_INVOICED-全部收票',
    -- 业务字段
    purchase_date       DATE            NOT NULL COMMENT '进货日期',
    warehouse_id        BIGINT          COMMENT '入库仓库ID',
    purchaser_id        BIGINT          COMMENT '采购员ID',
    purchaser_name      VARCHAR(64)     COMMENT '采购员姓名',
    order_status        VARCHAR(16)     NOT NULL DEFAULT 'ACTIVE' COMMENT '单据状态: ACTIVE, CLOSED, CANCELLED',
    remark              VARCHAR(512)    COMMENT '备注',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant_order_no (tenant_id, order_no),
    KEY idx_supplier_id (tenant_id, supplier_id),
    KEY idx_invoice_status (tenant_id, invoice_status),
    KEY idx_purchase_date (tenant_id, purchase_date),
    KEY idx_supplier_pending (tenant_id, supplier_id, invoice_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='进货单主表';
```

#### t_purchase_order_item — 进货单明细表

```sql
CREATE TABLE t_purchase_order_item (
    id                  BIGINT          NOT NULL COMMENT '明细ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    order_id            BIGINT          NOT NULL COMMENT '进货单ID',
    item_no             INT             NOT NULL COMMENT '行号',
    product_name        VARCHAR(128)    NOT NULL COMMENT '品名',
    product_category    VARCHAR(64)     NOT NULL COMMENT '品类大类',
    specification       VARCHAR(128)    NOT NULL COMMENT '规格型号',
    unit                VARCHAR(16)     NOT NULL DEFAULT '吨' COMMENT '单位',
    quantity            DECIMAL(18,4)   NOT NULL COMMENT '数量（重量）',
    unit_price          DECIMAL(18,6)   NOT NULL COMMENT '单价（含税）',
    unit_price_without_tax DECIMAL(18,6) NOT NULL COMMENT '单价（不含税）',
    amount              DECIMAL(18,2)   NOT NULL COMMENT '金额（含税）',
    amount_without_tax  DECIMAL(18,2)   NOT NULL COMMENT '金额（不含税）',
    tax_rate            DECIMAL(4,2)    NOT NULL DEFAULT 13.00 COMMENT '税率(%)',
    tax_amount          DECIMAL(18,2)   NOT NULL COMMENT '税额',
    -- 收票核销字段
    invoiced_quantity   DECIMAL(18,4)   NOT NULL DEFAULT 0 COMMENT '已收票数量',
    invoiced_amount     DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '已收票金额（含税）',
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
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='进货单明细表';
```

### 2.3 收票核心域 (db_input)

#### t_received_invoice — 收到的进项发票主表

```sql
CREATE TABLE t_received_invoice (
    id                      BIGINT          NOT NULL COMMENT '进项发票ID',
    tenant_id               BIGINT          NOT NULL COMMENT '租户ID',
    -- 发票基本信息（从诺诺网获取）
    invoice_code            VARCHAR(32)     NOT NULL COMMENT '发票代码',
    invoice_number          VARCHAR(32)     NOT NULL COMMENT '发票号码',
    invoice_date            DATE            NOT NULL COMMENT '开票日期',
    invoice_type            VARCHAR(16)     NOT NULL COMMENT '发票类型: SPECIAL-专票, NORMAL-普票, ELECTRONIC-电子票',
    check_code              VARCHAR(32)     COMMENT '校验码（后6位）',
    machine_no              VARCHAR(32)     COMMENT '机器编号',
    -- 销方（供应商）信息
    seller_name             VARCHAR(256)    NOT NULL COMMENT '销方名称',
    seller_tax_no           VARCHAR(32)     NOT NULL COMMENT '销方税号',
    seller_address_phone    VARCHAR(256)    COMMENT '销方地址电话',
    seller_bank_info        VARCHAR(256)    COMMENT '销方银行信息',
    -- 购方（本企业）信息
    buyer_name              VARCHAR(256)    NOT NULL COMMENT '购方名称',
    buyer_tax_no            VARCHAR(32)     NOT NULL COMMENT '购方税号',
    -- 金额
    total_amount            DECIMAL(18,2)   NOT NULL COMMENT '合计金额（不含税）',
    total_tax               DECIMAL(18,2)   NOT NULL COMMENT '合计税额',
    total_amount_with_tax   DECIMAL(18,2)   NOT NULL COMMENT '价税合计',
    -- 供应商匹配
    supplier_id             BIGINT          COMMENT '匹配到的供应商ID',
    supplier_match_status   VARCHAR(16)     NOT NULL DEFAULT 'PENDING' COMMENT '供应商匹配状态: PENDING-待匹配, MATCHED-已匹配, UNMATCHED-未匹配, MANUAL_CONFIRMED-人工确认',
    supplier_match_score    DECIMAL(5,2)    COMMENT '匹配分数',
    -- 进货单匹配
    order_match_status      VARCHAR(16)     NOT NULL DEFAULT 'PENDING' COMMENT '进货单匹配状态: PENDING-待匹配, AUTO_MATCHED-自动匹配, SUGGEST_MATCHED-推荐匹配, MANUAL_MATCHED-人工匹配, UNMATCHED-无匹配单据',
    -- 处理状态
    process_status          VARCHAR(20)     NOT NULL DEFAULT 'RECEIVED' COMMENT '处理状态（见状态机）',
    -- ERP相关
    erp_invoice_id          BIGINT          COMMENT 'ERP进项发票ID（生成后回写）',
    erp_voucher_no          VARCHAR(32)     COMMENT 'ERP凭证号',
    -- 台账相关
    ledger_id               BIGINT          COMMENT '进项台账记录ID',
    accounting_period       VARCHAR(8)      COMMENT '入账期间(YYYYMM)',
    deduction_status        VARCHAR(16)     DEFAULT 'NOT_DEDUCTED' COMMENT '抵扣状态: NOT_DEDUCTED-未抵扣, DEDUCTED-已抵扣, TRANSFERRED_OUT-已转出',
    -- 来源
    source                  VARCHAR(16)     NOT NULL DEFAULT 'AUTO_PULL' COMMENT '来源: AUTO_PULL-定时拉取, CALLBACK-回调推送, MANUAL_SYNC-手动同步',
    -- 诺诺网原始数据
    nuonuo_raw_data         JSON            COMMENT '诺诺网原始响应数据（快照备查）',
    -- 发票文件
    invoice_pdf_url         VARCHAR(512)    COMMENT '发票PDF文件URL',
    invoice_ofd_url         VARCHAR(512)    COMMENT '发票OFD文件URL',
    -- 异常信息
    error_code              VARCHAR(32)     COMMENT '错误码',
    error_message           VARCHAR(1024)   COMMENT '错误信息',
    -- 公共字段
    created_by              BIGINT,
    created_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by              BIGINT,
    updated_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                 TINYINT(1)      NOT NULL DEFAULT 0,
    version                 INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_invoice (tenant_id, invoice_code, invoice_number),
    KEY idx_seller_tax_no (seller_tax_no),
    KEY idx_supplier_id (tenant_id, supplier_id),
    KEY idx_process_status (tenant_id, process_status),
    KEY idx_invoice_date (tenant_id, invoice_date),
    KEY idx_order_match (tenant_id, order_match_status),
    KEY idx_accounting_period (tenant_id, accounting_period)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='收到的进项发票主表';
```

#### t_received_invoice_item — 收到的进项发票明细行

```sql
CREATE TABLE t_received_invoice_item (
    id                      BIGINT          NOT NULL COMMENT '明细ID',
    tenant_id               BIGINT          NOT NULL COMMENT '租户ID',
    received_invoice_id     BIGINT          NOT NULL COMMENT '进项发票ID',
    item_no                 INT             NOT NULL COMMENT '行号',
    -- 商品信息
    product_name            VARCHAR(128)    NOT NULL COMMENT '品名（发票上的商品名称）',
    product_category        VARCHAR(64)     COMMENT '品类大类（系统自动映射）',
    specification           VARCHAR(128)    COMMENT '规格型号',
    unit                    VARCHAR(16)     COMMENT '单位',
    quantity                DECIMAL(18,4)   COMMENT '数量',
    unit_price              DECIMAL(18,6)   COMMENT '单价（不含税）',
    amount                  DECIMAL(18,2)   NOT NULL COMMENT '金额（不含税）',
    tax_rate                DECIMAL(4,2)    NOT NULL COMMENT '税率(%)',
    tax_amount              DECIMAL(18,2)   NOT NULL COMMENT '税额',
    amount_with_tax         DECIMAL(18,2)   NOT NULL COMMENT '含税金额',
    -- 税收分类
    tax_category_code       VARCHAR(32)     COMMENT '税收分类编码',
    tax_category_name       VARCHAR(128)    COMMENT '税收分类名称',
    -- 分配状态
    allocated_quantity      DECIMAL(18,4)   NOT NULL DEFAULT 0 COMMENT '已分配数量',
    allocated_amount        DECIMAL(18,2)   NOT NULL DEFAULT 0 COMMENT '已分配金额',
    allocation_status       VARCHAR(16)     NOT NULL DEFAULT 'UNALLOCATED' COMMENT '分配状态: UNALLOCATED-未分配, PARTIAL-部分分配, FULLY-全部分配',
    -- 公共字段
    created_by              BIGINT,
    created_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by              BIGINT,
    updated_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                 TINYINT(1)      NOT NULL DEFAULT 0,
    version                 INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_invoice_id (received_invoice_id),
    KEY idx_product (tenant_id, product_name, specification),
    KEY idx_category (tenant_id, product_category)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='收到的进项发票明细行';
```

#### t_invoice_allocation — 发票与进货单分配关系表

```sql
CREATE TABLE t_invoice_allocation (
    id                          BIGINT          NOT NULL COMMENT 'ID',
    tenant_id                   BIGINT          NOT NULL COMMENT '租户ID',
    received_invoice_id         BIGINT          NOT NULL COMMENT '进项发票ID',
    received_invoice_item_id    BIGINT          NOT NULL COMMENT '进项发票明细行ID',
    purchase_order_id           BIGINT          NOT NULL COMMENT '进货单ID',
    purchase_order_item_id      BIGINT          COMMENT '进货单明细行ID（为空表示分配到单头级）',
    -- 分配金额
    allocated_quantity          DECIMAL(18,4)   NOT NULL COMMENT '分配数量',
    allocated_amount            DECIMAL(18,2)   NOT NULL COMMENT '分配金额（不含税）',
    allocated_tax               DECIMAL(18,2)   NOT NULL COMMENT '分配税额',
    allocated_amount_with_tax   DECIMAL(18,2)   NOT NULL COMMENT '分配含税金额',
    -- 匹配信息
    match_type                  VARCHAR(16)     NOT NULL COMMENT '匹配方式: AUTO-自动匹配, SUGGEST-推荐匹配, MANUAL-人工分配',
    match_score                 DECIMAL(5,2)    COMMENT '匹配分数（自动匹配时记录）',
    -- 状态
    allocation_status           VARCHAR(16)     NOT NULL DEFAULT 'CONFIRMED' COMMENT '分配状态: CONFIRMED-已确认, CANCELLED-已取消',
    -- 公共字段
    created_by                  BIGINT,
    created_time                DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by                  BIGINT,
    updated_time                DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                     TINYINT(1)      NOT NULL DEFAULT 0,
    version                     INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_invoice_id (received_invoice_id),
    KEY idx_invoice_item_id (received_invoice_item_id),
    KEY idx_purchase_order (purchase_order_id),
    KEY idx_purchase_item (purchase_order_item_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='发票与进货单分配关系表';
```

#### t_input_invoice_ledger — 进项发票台账

```sql
CREATE TABLE t_input_invoice_ledger (
    id                      BIGINT          NOT NULL COMMENT '台账ID',
    tenant_id               BIGINT          NOT NULL COMMENT '租户ID',
    received_invoice_id     BIGINT          NOT NULL COMMENT '进项发票ID',
    -- 发票信息冗余（便于台账查询不跨表）
    invoice_code            VARCHAR(32)     NOT NULL COMMENT '发票代码',
    invoice_number          VARCHAR(32)     NOT NULL COMMENT '发票号码',
    invoice_date            DATE            NOT NULL COMMENT '开票日期',
    invoice_type            VARCHAR(16)     NOT NULL COMMENT '发票类型',
    -- 供应商信息
    supplier_id             BIGINT          NOT NULL COMMENT '供应商ID',
    supplier_name           VARCHAR(256)    NOT NULL COMMENT '供应商名称',
    supplier_tax_no         VARCHAR(32)     NOT NULL COMMENT '供应商税号',
    -- 金额信息
    total_amount            DECIMAL(18,2)   NOT NULL COMMENT '合计金额（不含税）',
    total_tax               DECIMAL(18,2)   NOT NULL COMMENT '合计税额',
    total_amount_with_tax   DECIMAL(18,2)   NOT NULL COMMENT '价税合计',
    -- 认证与抵扣
    certification_status    VARCHAR(16)     NOT NULL DEFAULT 'NOT_CERTIFIED' COMMENT '认证状态: NOT_CERTIFIED-未认证, CERTIFIED-已认证, FAILED-认证失败',
    certification_date      DATE            COMMENT '认证日期',
    deduction_status        VARCHAR(16)     NOT NULL DEFAULT 'NOT_DEDUCTED' COMMENT '抵扣状态: NOT_DEDUCTED-未抵扣, DEDUCTED-已抵扣, TRANSFERRED_OUT-进项转出',
    deduction_period        VARCHAR(8)      COMMENT '抵扣所属期间(YYYYMM)',
    deduction_amount        DECIMAL(18,2)   COMMENT '抵扣税额',
    transfer_out_amount     DECIMAL(18,2)   DEFAULT 0 COMMENT '进项转出金额',
    transfer_out_reason     VARCHAR(256)    COMMENT '进项转出原因',
    -- 入账信息
    accounting_period       VARCHAR(8)      NOT NULL COMMENT '入账期间(YYYYMM)',
    erp_invoice_id          BIGINT          COMMENT 'ERP进项发票ID',
    erp_voucher_no          VARCHAR(32)     COMMENT 'ERP凭证号',
    -- 关联进货单
    purchase_order_ids      JSON            NOT NULL COMMENT '关联进货单ID列表 [id1, id2, ...]',
    purchase_order_nos      JSON            NOT NULL COMMENT '关联进货单号列表 ["PO001", "PO002"]',
    -- 品类汇总（便于按品类统计进项）
    product_categories      JSON            COMMENT '涉及品类列表 ["热轧卷板", "冷轧卷板"]',
    -- 发票文件
    invoice_pdf_url         VARCHAR(512)    COMMENT '发票PDF文件URL',
    -- 状态
    ledger_status           VARCHAR(16)     NOT NULL DEFAULT 'ACTIVE' COMMENT '台账状态: ACTIVE-有效, VOIDED-已作废, RED_FLUSHED-已红冲',
    void_reason             VARCHAR(512)    COMMENT '作废/红冲原因',
    void_time               DATETIME(3)     COMMENT '作废/红冲时间',
    -- 公共字段
    created_by              BIGINT,
    created_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by              BIGINT,
    updated_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                 TINYINT(1)      NOT NULL DEFAULT 0,
    version                 INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_invoice (tenant_id, invoice_code, invoice_number),
    KEY idx_supplier (tenant_id, supplier_id),
    KEY idx_accounting_period (tenant_id, accounting_period),
    KEY idx_deduction (tenant_id, deduction_status),
    KEY idx_invoice_date (tenant_id, invoice_date),
    KEY idx_certification (tenant_id, certification_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='进项发票台账';
```

### 2.4 ERP 进项发票域 (db_input)

#### t_erp_input_invoice — ERP 进项发票表

```sql
CREATE TABLE t_erp_input_invoice (
    id                      BIGINT          NOT NULL COMMENT 'ERP进项发票ID',
    tenant_id               BIGINT          NOT NULL COMMENT '租户ID',
    erp_invoice_no          VARCHAR(32)     NOT NULL COMMENT 'ERP进项发票编号（系统生成）',
    received_invoice_id     BIGINT          NOT NULL COMMENT '关联收到的进项发票ID',
    -- 发票信息
    invoice_code            VARCHAR(32)     NOT NULL COMMENT '发票代码',
    invoice_number          VARCHAR(32)     NOT NULL COMMENT '发票号码',
    invoice_date            DATE            NOT NULL COMMENT '开票日期',
    invoice_type            VARCHAR(16)     NOT NULL COMMENT '发票类型',
    -- 供应商信息
    supplier_id             BIGINT          NOT NULL COMMENT '供应商ID',
    supplier_name           VARCHAR(256)    NOT NULL COMMENT '供应商名称',
    supplier_tax_no         VARCHAR(32)     NOT NULL COMMENT '供应商税号',
    -- 金额
    total_amount            DECIMAL(18,2)   NOT NULL COMMENT '合计金额（不含税）',
    total_tax               DECIMAL(18,2)   NOT NULL COMMENT '合计税额',
    total_amount_with_tax   DECIMAL(18,2)   NOT NULL COMMENT '价税合计',
    -- 关联进货单
    purchase_order_count    INT             NOT NULL DEFAULT 0 COMMENT '关联进货单数',
    -- 库存入库
    inventory_status        VARCHAR(16)     NOT NULL DEFAULT 'PENDING' COMMENT '库存入库状态: PENDING-待入库, COMPLETED-已入库, PARTIAL-部分入库',
    inventory_granularity   VARCHAR(16)     NOT NULL COMMENT '入库粒度: SPECIFICATION-具体规格, CATEGORY-大品类, HYBRID-混合',
    -- ERP 状态
    erp_status              VARCHAR(16)     NOT NULL DEFAULT 'CREATED' COMMENT 'ERP状态: CREATED-已创建, POSTED-已过账, CANCELLED-已取消',
    post_date               DATE            COMMENT '过账日期',
    accounting_period       VARCHAR(8)      NOT NULL COMMENT '会计期间(YYYYMM)',
    -- 公共字段
    created_by              BIGINT,
    created_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by              BIGINT,
    updated_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                 TINYINT(1)      NOT NULL DEFAULT 0,
    version                 INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_erp_no (tenant_id, erp_invoice_no),
    KEY idx_received_invoice (received_invoice_id),
    KEY idx_supplier (tenant_id, supplier_id),
    KEY idx_accounting_period (tenant_id, accounting_period),
    KEY idx_erp_status (tenant_id, erp_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='ERP进项发票表';
```

#### t_erp_input_invoice_item — ERP 进项发票明细

```sql
CREATE TABLE t_erp_input_invoice_item (
    id                      BIGINT          NOT NULL COMMENT '明细ID',
    tenant_id               BIGINT          NOT NULL COMMENT '租户ID',
    erp_invoice_id          BIGINT          NOT NULL COMMENT 'ERP进项发票ID',
    item_no                 INT             NOT NULL COMMENT '行号',
    -- 商品信息
    product_name            VARCHAR(128)    NOT NULL COMMENT '品名',
    product_category        VARCHAR(64)     NOT NULL COMMENT '品类大类',
    specification           VARCHAR(128)    COMMENT '规格型号',
    unit                    VARCHAR(16)     NOT NULL DEFAULT '吨' COMMENT '单位',
    quantity                DECIMAL(18,4)   NOT NULL COMMENT '数量',
    unit_price              DECIMAL(18,6)   NOT NULL COMMENT '单价（不含税）',
    amount                  DECIMAL(18,2)   NOT NULL COMMENT '金额（不含税）',
    tax_rate                DECIMAL(4,2)    NOT NULL COMMENT '税率(%)',
    tax_amount              DECIMAL(18,2)   NOT NULL COMMENT '税额',
    amount_with_tax         DECIMAL(18,2)   NOT NULL COMMENT '含税金额',
    -- 来源关联
    received_invoice_item_id BIGINT         NOT NULL COMMENT '来源进项发票明细ID',
    purchase_order_id       BIGINT          COMMENT '关联进货单ID',
    purchase_order_item_id  BIGINT          COMMENT '关联进货单明细ID',
    -- 进项库存
    input_inventory_id      BIGINT          COMMENT '入库的进项库存ID',
    -- 公共字段
    created_by              BIGINT,
    created_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by              BIGINT,
    updated_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                 TINYINT(1)      NOT NULL DEFAULT 0,
    version                 INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_erp_invoice_id (erp_invoice_id),
    KEY idx_received_item (received_invoice_item_id),
    KEY idx_purchase_order (purchase_order_id),
    KEY idx_inventory (input_inventory_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='ERP进项发票明细';
```

### 2.5 配置域 (db_input)

#### t_input_inventory_config — 进项库存管理配置

```sql
CREATE TABLE t_input_inventory_config (
    id                      BIGINT          NOT NULL COMMENT 'ID',
    tenant_id               BIGINT          NOT NULL COMMENT '租户ID',
    -- 库存管理粒度
    inventory_granularity   VARCHAR(16)     NOT NULL DEFAULT 'SPECIFICATION' COMMENT '库存粒度: SPECIFICATION-具体规格, CATEGORY-大品类, HYBRID-混合',
    -- 匹配容差配置
    amount_tolerance_rate   DECIMAL(5,4)    NOT NULL DEFAULT 0.005 COMMENT '金额匹配容差率（默认0.5%）',
    amount_tolerance_abs    DECIMAL(18,2)   NOT NULL DEFAULT 100.00 COMMENT '金额匹配容差绝对值（默认100元）',
    -- 自动匹配阈值
    auto_match_score        DECIMAL(5,2)    NOT NULL DEFAULT 85.00 COMMENT '自动匹配分数阈值',
    suggest_match_score     DECIMAL(5,2)    NOT NULL DEFAULT 60.00 COMMENT '推荐匹配分数阈值',
    -- 拉取配置
    pull_interval_minutes   INT             NOT NULL DEFAULT 30 COMMENT '自动拉取间隔（分钟）',
    pull_enabled            TINYINT(1)      NOT NULL DEFAULT 1 COMMENT '是否启用自动拉取',
    callback_enabled        TINYINT(1)      NOT NULL DEFAULT 1 COMMENT '是否启用回调推送',
    -- 超额收票配置
    allow_over_invoice      TINYINT(1)      NOT NULL DEFAULT 0 COMMENT '是否允许超额收票',
    over_invoice_tolerance  DECIMAL(5,4)    DEFAULT 0.005 COMMENT '超额容差率',
    -- 公共字段
    created_by              BIGINT,
    created_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by              BIGINT,
    updated_time            DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted                 TINYINT(1)      NOT NULL DEFAULT 0,
    version                 INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant (tenant_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='进项库存管理配置';
```

#### t_product_category_mapping — 品名到品类大类映射表

```sql
CREATE TABLE t_product_category_mapping (
    id                  BIGINT          NOT NULL COMMENT 'ID',
    tenant_id           BIGINT          NOT NULL COMMENT '租户ID',
    category_name       VARCHAR(64)     NOT NULL COMMENT '品类大类名称',
    keyword             VARCHAR(64)     NOT NULL COMMENT '品名匹配关键词',
    priority            INT             NOT NULL DEFAULT 0 COMMENT '匹配优先级（数字越大优先级越高）',
    status              VARCHAR(16)     NOT NULL DEFAULT 'ACTIVE' COMMENT '状态',
    -- 公共字段
    created_by          BIGINT,
    created_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_by          BIGINT,
    updated_time        DATETIME(3)     NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted             TINYINT(1)      NOT NULL DEFAULT 0,
    version             INT             NOT NULL DEFAULT 1,
    PRIMARY KEY (id),
    KEY idx_tenant_category (tenant_id, category_name),
    KEY idx_keyword (tenant_id, keyword)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='品名到品类大类映射表';
```

---

## 3. 收票域 Redis 数据结构

| Key 模式 | 类型 | 用途 | TTL |
|---------|------|------|-----|
| `recv_inv_exists:{tenantId}` | Set | 已接收发票去重（发票代码+号码） | 永久 |
| `recv_inv_processing:{invoiceId}` | String | 发票处理中锁（防并发） | 5min |
| `supplier:tax_no:{taxNo}` | String | 税号→供应商ID 缓存 | 1h |
| `supplier:alias:{tenantId}` | Hash (aliasName→supplierId) | 供应商别名缓存 | 30min |
| `purchase:pending:{tenantId}:{supplierId}` | SortedSet (score=金额) | 供应商待收票进货单缓存 | 10min |
| `recv:config:{tenantId}` | Hash | 收票配置缓存 | 10min |
| `category:mapping:{tenantId}` | Hash (keyword→category) | 品类映射缓存 | 30min |
| `recv:pull:lastTime:{tenantId}` | String | 上次拉取时间 | 永久 |
| `recv:stats:{tenantId}:{period}` | Hash | 收票统计（当期收票数/金额） | 24h |

---

## 4. ER 关系图（收票域）

```
t_supplier ──┬──< t_supplier_alias
             ├──< t_purchase_order ──< t_purchase_order_item
             └──< t_received_invoice ──< t_received_invoice_item
                       │                        │
                       │                        ▼
                       │              t_invoice_allocation ──> t_purchase_order_item
                       │
                       ├──> t_erp_input_invoice ──< t_erp_input_invoice_item
                       │                                    │
                       │                                    ▼
                       │                              t_input_inventory (已有表)
                       │
                       └──> t_input_invoice_ledger

t_input_inventory_config (独立配置表)
t_product_category_mapping (品类映射配置)
```

---

## 5. 与已有表的关系

| 已有表 | 关联方式 | 说明 |
|--------|---------|------|
| t_input_inventory | ERP 进项发票明细写入 | 收票确认后，发票明细数据进入进项库存 |
| t_tax_account | 台账写入后更新 | 更新累计进项税额，重新计算税负率 |
| t_input_lock_record | 不直接关联 | 销项开票时才会锁定进项库存 |

收票智能体负责"入"库，销项开票智能体负责"出"库，两者通过 `t_input_inventory` 连接。
