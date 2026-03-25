# 11 — 收票智能体 API 接口设计

## 1. 接口分组

| 分组 | 前缀 | 使用者 | 说明 |
|------|------|--------|------|
| 收票管理 | /api/v1/admin/receive-invoice | 财务/采购人员 | 进项发票管理、分配、确认 |
| 供应商管理 | /api/v1/admin/supplier | 管理人员 | 供应商档案及别名管理 |
| 进货单管理 | /api/v1/admin/purchase-order | 采购人员 | 进货单及收票状态管理 |
| 进项台账 | /api/v1/admin/input-ledger | 财务人员 | 进项发票台账查询统计 |
| 进项库存配置 | /api/v1/admin/input-config | 管理人员 | 库存粒度、品类映射配置 |
| 内部服务 | /internal/receive-invoice | 微服务间 | 收票引擎内部调用 |
| 回调 | /callback/nuonuo/input-invoice | 诺诺网回调 | 进项发票推送回调 |

---

## 2. 收票管理接口

### 2.1 进项发票列表

#### GET /api/v1/admin/receive-invoice/list

查询收到的进项发票列表

**请求参数 (Query)**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNum | int | 否 | 页码，默认 1 |
| pageSize | int | 否 | 每页条数，默认 20 |
| processStatus | string | 否 | 处理状态筛选 |
| supplierName | string | 否 | 供应商名称模糊搜索 |
| invoiceCode | string | 否 | 发票代码 |
| invoiceNumber | string | 否 | 发票号码 |
| startDate | date | 否 | 开票起始日期 |
| endDate | date | 否 | 开票截止日期 |
| orderMatchStatus | string | 否 | 进货单匹配状态 |
| accountingPeriod | string | 否 | 入账期间 YYYYMM |

**响应示例**

```json
{
  "code": 200,
  "data": {
    "total": 128,
    "list": [
      {
        "receivedInvoiceId": 90001,
        "invoiceCode": "3100222222",
        "invoiceNumber": "88888888",
        "invoiceDate": "2025-03-24",
        "invoiceType": "SPECIAL",
        "sellerName": "河北XX钢铁集团有限公司",
        "sellerTaxNo": "91130000XXXXXXXXXX",
        "totalAmount": 442477.88,
        "totalTax": 57522.12,
        "totalAmountWithTax": 500000.00,
        "supplierId": 2001,
        "supplierMatchStatus": "MATCHED",
        "supplierMatchScore": 100.00,
        "orderMatchStatus": "AUTO_MATCHED",
        "processStatus": "WRITE_OFF_COMPLETE",
        "erpInvoiceNo": "ERP-INV-20250324001",
        "accountingPeriod": "202503",
        "deductionStatus": "NOT_DEDUCTED",
        "source": "AUTO_PULL",
        "createdTime": "2025-03-24T14:30:00",
        "items": [
          {
            "itemId": 91001,
            "productName": "热轧卷板",
            "specification": "Q235B 5.75*1500*C",
            "quantity": 100.0000,
            "unitPrice": 3716.81,
            "amount": 371681.00,
            "taxRate": 13.00,
            "taxAmount": 48318.53,
            "amountWithTax": 420000.00,
            "allocationStatus": "FULLY"
          },
          {
            "itemId": 91002,
            "productName": "冷轧卷板",
            "specification": "SPCC 1.0*1250*C",
            "quantity": 20.0000,
            "unitPrice": 3539.82,
            "amount": 70796.88,
            "taxRate": 13.00,
            "taxAmount": 9203.59,
            "amountWithTax": 80000.00,
            "allocationStatus": "FULLY"
          }
        ],
        "allocations": [
          {
            "purchaseOrderNo": "PO20250320001",
            "allocatedAmountWithTax": 420000.00,
            "matchType": "AUTO",
            "matchScore": 92.50
          },
          {
            "purchaseOrderNo": "PO20250321003",
            "allocatedAmountWithTax": 80000.00,
            "matchType": "AUTO",
            "matchScore": 88.00
          }
        ]
      }
    ]
  }
}
```

### 2.2 手动同步进项发票

#### POST /api/v1/admin/receive-invoice/sync

手动触发从诺诺网同步进项发票

**请求体**

```json
{
  "syncType": "DATE_RANGE",
  "startDate": "2025-03-20",
  "endDate": "2025-03-25"
}
```

或按具体发票同步：

```json
{
  "syncType": "SPECIFIC",
  "invoiceCode": "3100222222",
  "invoiceNumber": "88888888"
}
```

**响应示例**

```json
{
  "code": 200,
  "data": {
    "syncTaskId": "SYNC20250325001",
    "syncType": "DATE_RANGE",
    "expectedCount": 15,
    "status": "PROCESSING",
    "message": "同步任务已提交，预计处理 15 张发票"
  }
}
```

### 2.3 进项发票详情

#### GET /api/v1/admin/receive-invoice/{invoiceId}

查询单张进项发票的完整详情，含分配结果、匹配过程等。

**响应示例**

```json
{
  "code": 200,
  "data": {
    "invoiceInfo": {
      "receivedInvoiceId": 90001,
      "invoiceCode": "3100222222",
      "invoiceNumber": "88888888",
      "invoiceDate": "2025-03-24",
      "invoiceType": "SPECIAL",
      "sellerName": "河北XX钢铁集团有限公司",
      "sellerTaxNo": "91130000XXXXXXXXXX",
      "buyerName": "某某钢贸有限公司",
      "buyerTaxNo": "91310000YYYYYYYY",
      "totalAmount": 442477.88,
      "totalTax": 57522.12,
      "totalAmountWithTax": 500000.00,
      "processStatus": "WRITE_OFF_COMPLETE"
    },
    "items": [
      {
        "itemId": 91001,
        "productName": "热轧卷板",
        "specification": "Q235B 5.75*1500*C",
        "quantity": 100.0000,
        "unitPrice": 3716.81,
        "amount": 371681.00,
        "taxRate": 13.00,
        "taxAmount": 48318.53,
        "productCategory": "热轧卷板",
        "allocationStatus": "FULLY"
      }
    ],
    "supplierMatch": {
      "supplierId": 2001,
      "supplierName": "河北XX钢铁集团有限公司",
      "matchMethod": "TAX_NO_EXACT",
      "matchScore": 100.00,
      "matchTime": "2025-03-24T14:30:02"
    },
    "orderMatch": {
      "matchStatus": "AUTO_MATCHED",
      "matchedOrders": [
        {
          "purchaseOrderId": 60001,
          "purchaseOrderNo": "PO20250320001",
          "totalAmount": 420000.00,
          "matchScore": 92.50,
          "matchDetails": {
            "amountScore": 38,
            "productScore": 25,
            "timeScore": 15,
            "specScore": 10,
            "quantityScore": 9
          }
        }
      ]
    },
    "allocation": {
      "allocations": [
        {
          "invoiceItemId": 91001,
          "purchaseOrderId": 60001,
          "purchaseOrderItemId": 70001,
          "allocatedQuantity": 100.0000,
          "allocatedAmount": 371681.00,
          "allocatedTax": 48318.53,
          "matchType": "AUTO"
        }
      ]
    },
    "erpInvoice": {
      "erpInvoiceId": 80001,
      "erpInvoiceNo": "ERP-INV-20250324001",
      "erpStatus": "POSTED",
      "accountingPeriod": "202503"
    },
    "ledger": {
      "ledgerId": 85001,
      "certificationStatus": "NOT_CERTIFIED",
      "deductionStatus": "NOT_DEDUCTED"
    },
    "processTimeline": [
      { "status": "RECEIVED", "time": "2025-03-24T14:30:00", "desc": "诺诺网同步获取" },
      { "status": "SUPPLIER_MATCHED", "time": "2025-03-24T14:30:02", "desc": "供应商匹配成功: 河北XX钢铁集团" },
      { "status": "ALLOCATED", "time": "2025-03-24T14:30:05", "desc": "自动分配至进货单 PO20250320001" },
      { "status": "ERP_GENERATED", "time": "2025-03-24T14:30:08", "desc": "ERP进项发票已生成" },
      { "status": "WRITE_OFF_COMPLETE", "time": "2025-03-24T14:30:10", "desc": "待收票核销完成, 台账已写入" }
    ]
  }
}
```

### 2.4 人工确认供应商

#### POST /api/v1/admin/receive-invoice/{invoiceId}/confirm-supplier

对未自动匹配的发票人工确认供应商

**请求体**

```json
{
  "supplierId": 2001,
  "saveAsAlias": true
}
```

或创建新供应商：

```json
{
  "createNewSupplier": true,
  "supplierInfo": {
    "supplierCode": "SUP20250325001",
    "supplierName": "河北XX钢铁集团有限公司",
    "taxNo": "91130000XXXXXXXXXX",
    "address": "河北省唐山市...",
    "supplierType": "STEEL_MILL"
  },
  "saveAsAlias": true
}
```

**响应示例**

```json
{
  "code": 200,
  "data": {
    "supplierId": 2001,
    "supplierName": "河北XX钢铁集团有限公司",
    "aliasSaved": true,
    "nextStep": "ORDER_MATCHING",
    "candidateOrders": [
      {
        "purchaseOrderId": 60001,
        "purchaseOrderNo": "PO20250320001",
        "totalAmount": 420000.00,
        "pendingAmount": 420000.00,
        "purchaseDate": "2025-03-20",
        "matchScore": 72.50
      }
    ]
  }
}
```

### 2.5 人工分配发票到进货单

#### POST /api/v1/admin/receive-invoice/{invoiceId}/manual-allocate

人工将发票明细分配到指定进货单

**请求体**

```json
{
  "allocations": [
    {
      "invoiceItemId": 91001,
      "purchaseOrderId": 60001,
      "purchaseOrderItemId": 70001,
      "allocatedQuantity": 100.0000,
      "allocatedAmount": 371681.00
    },
    {
      "invoiceItemId": 91002,
      "purchaseOrderId": 60002,
      "purchaseOrderItemId": 70005,
      "allocatedQuantity": 20.0000,
      "allocatedAmount": 70796.88
    }
  ],
  "remark": "人工分配：发票品名与进货单略有差异，确认为同一批货"
}
```

**响应示例**

```json
{
  "code": 200,
  "data": {
    "allocateResult": "SUCCESS",
    "totalAllocatedAmount": 442477.88,
    "invoiceTotalAmount": 442477.88,
    "unallocatedAmount": 0.00,
    "validationResult": {
      "amountMatch": true,
      "allItemsAllocated": true,
      "noOverAllocation": true
    },
    "nextStep": "ERP_GENERATE",
    "message": "分配成功，即将自动生成 ERP 进项发票并核销"
  }
}
```

### 2.6 确认自动匹配结果

#### POST /api/v1/admin/receive-invoice/{invoiceId}/confirm-match

对推荐匹配的结果进行确认

**请求体**

```json
{
  "confirmed": true,
  "adjustments": []
}
```

或调整后确认：

```json
{
  "confirmed": true,
  "adjustments": [
    {
      "allocationId": 95001,
      "newPurchaseOrderId": 60003,
      "newPurchaseOrderItemId": 70010,
      "adjustedQuantity": 50.0000,
      "adjustedAmount": 185840.50
    }
  ]
}
```

### 2.7 暂存未分配发票

#### POST /api/v1/admin/receive-invoice/{invoiceId}/park

将无法匹配的发票暂存（等待后续进货单录入后再匹配）

**请求体**

```json
{
  "parkReason": "供应商确认，但目前无对应的进货单，等待采购部门录入",
  "expectOrderDate": "2025-03-28"
}
```

### 2.8 发票红冲回退

#### POST /api/v1/admin/receive-invoice/{invoiceId}/void

处理进项发票红冲/作废

**请求体**

```json
{
  "voidReason": "发票信息有误，供应商已红冲重开",
  "autoRollback": true
}
```

**响应示例**

```json
{
  "code": 200,
  "data": {
    "voidResult": "SUCCESS",
    "rollbackActions": [
      "进项库存已回退: 热轧卷板 Q235B 5.75*1500*C -100.0000吨",
      "进货单 PO20250320001 收票状态已回退为: PENDING_INVOICE",
      "ERP进项发票 ERP-INV-20250324001 已取消",
      "进项台账已标记红冲",
      "税务账户已更新: 累计进项税额 -57522.12"
    ]
  }
}
```

---

## 3. 供应商管理接口

### 3.1 供应商列表

#### GET /api/v1/admin/supplier/list

### 3.2 供应商详情

#### GET /api/v1/admin/supplier/{supplierId}

### 3.3 新增供应商

#### POST /api/v1/admin/supplier

**请求体**

```json
{
  "supplierCode": "SUP001",
  "supplierName": "河北XX钢铁集团有限公司",
  "taxNo": "91130000XXXXXXXXXX",
  "address": "河北省唐山市...",
  "phone": "0315-XXXXXXXX",
  "bankName": "中国银行唐山分行",
  "bankAccount": "0000XXXXXXXXXXXX",
  "contactName": "张经理",
  "contactPhone": "138XXXXXXXX",
  "supplierType": "STEEL_MILL"
}
```

### 3.4 管理供应商别名

#### GET /api/v1/admin/supplier/{supplierId}/aliases

#### POST /api/v1/admin/supplier/{supplierId}/aliases

**请求体**

```json
{
  "aliasName": "河北XX集团",
  "aliasTaxNo": "91130000XXXXXXXXXX"
}
```

#### DELETE /api/v1/admin/supplier/{supplierId}/aliases/{aliasId}

---

## 4. 进货单管理接口

### 4.1 进货单列表（收票视角）

#### GET /api/v1/admin/purchase-order/invoice-view

以收票状态为主要维度查询进货单

**请求参数 (Query)**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNum | int | 否 | 页码 |
| pageSize | int | 否 | 每页条数 |
| invoiceStatus | string | 否 | 收票状态: PENDING_INVOICE / PARTIAL_INVOICED / FULLY_INVOICED |
| supplierId | long | 否 | 供应商ID |
| orderNo | string | 否 | 进货单号 |
| startDate | date | 否 | 起始日期 |
| endDate | date | 否 | 截止日期 |

**响应示例**

```json
{
  "code": 200,
  "data": {
    "total": 45,
    "summary": {
      "pendingCount": 28,
      "pendingAmount": 8500000.00,
      "partialCount": 10,
      "partialAmount": 3200000.00,
      "fullyCount": 7
    },
    "list": [
      {
        "purchaseOrderId": 60001,
        "orderNo": "PO20250320001",
        "supplierId": 2001,
        "supplierName": "河北XX钢铁集团有限公司",
        "totalAmount": 420000.00,
        "invoicedAmount": 420000.00,
        "pendingAmount": 0.00,
        "invoiceStatus": "FULLY_INVOICED",
        "purchaseDate": "2025-03-20",
        "purchaserName": "李采购",
        "items": [
          {
            "productName": "热轧卷板",
            "specification": "Q235B 5.75*1500*C",
            "quantity": 100.0000,
            "unitPrice": 4200.00,
            "amount": 420000.00,
            "invoicedQuantity": 100.0000,
            "invoicedAmount": 420000.00
          }
        ],
        "receivedInvoices": [
          {
            "invoiceCode": "3100222222",
            "invoiceNumber": "88888888",
            "invoiceDate": "2025-03-24",
            "amountWithTax": 420000.00
          }
        ]
      }
    ]
  }
}
```

---

## 5. 进项台账接口

### 5.1 台账查询

#### GET /api/v1/admin/input-ledger/list

**请求参数 (Query)**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNum | int | 否 | 页码 |
| pageSize | int | 否 | 每页条数 |
| accountingPeriod | string | 否 | 会计期间 YYYYMM |
| supplierId | long | 否 | 供应商 |
| deductionStatus | string | 否 | 抵扣状态 |
| certificationStatus | string | 否 | 认证状态 |
| invoiceType | string | 否 | 发票类型 |

**响应示例**

```json
{
  "code": 200,
  "data": {
    "total": 86,
    "periodSummary": {
      "period": "202503",
      "totalInvoiceCount": 86,
      "totalAmount": 12500000.00,
      "totalTax": 1625000.00,
      "certifiedCount": 72,
      "certifiedTax": 1400000.00,
      "deductedCount": 65,
      "deductedTax": 1280000.00,
      "transferOutTax": 50000.00
    },
    "list": [
      {
        "ledgerId": 85001,
        "invoiceCode": "3100222222",
        "invoiceNumber": "88888888",
        "invoiceDate": "2025-03-24",
        "invoiceType": "SPECIAL",
        "supplierName": "河北XX钢铁集团有限公司",
        "totalAmount": 442477.88,
        "totalTax": 57522.12,
        "totalAmountWithTax": 500000.00,
        "certificationStatus": "CERTIFIED",
        "certificationDate": "2025-03-25",
        "deductionStatus": "DEDUCTED",
        "deductionPeriod": "202503",
        "deductionAmount": 57522.12,
        "accountingPeriod": "202503",
        "purchaseOrderNos": ["PO20250320001", "PO20250321003"],
        "productCategories": ["热轧卷板", "冷轧卷板"],
        "ledgerStatus": "ACTIVE"
      }
    ]
  }
}
```

### 5.2 台账统计看板

#### GET /api/v1/admin/input-ledger/dashboard

**响应示例**

```json
{
  "code": 200,
  "data": {
    "currentPeriod": "202503",
    "inputSummary": {
      "totalInvoices": 86,
      "totalAmount": 12500000.00,
      "totalTax": 1625000.00,
      "avgInvoiceAmount": 145348.84
    },
    "certificationSummary": {
      "certified": 72,
      "certifiedRate": 83.72,
      "pendingCertification": 14,
      "pendingTax": 225000.00
    },
    "deductionSummary": {
      "deducted": 65,
      "deductedTax": 1280000.00,
      "notDeducted": 21,
      "notDeductedTax": 345000.00,
      "transferredOut": 3,
      "transferOutTax": 50000.00
    },
    "supplierDistribution": [
      { "supplierName": "河北XX钢铁集团", "invoiceCount": 25, "totalTax": 580000.00 },
      { "supplierName": "山东YY钢铁公司", "invoiceCount": 18, "totalTax": 420000.00 },
      { "supplierName": "江苏ZZ钢管厂", "invoiceCount": 12, "totalTax": 280000.00 }
    ],
    "categoryDistribution": [
      { "category": "热轧卷板", "totalAmount": 5200000.00, "percentage": 41.60 },
      { "category": "冷轧卷板", "totalAmount": 3100000.00, "percentage": 24.80 },
      { "category": "镀锌卷板", "totalAmount": 2500000.00, "percentage": 20.00 },
      { "category": "其他", "totalAmount": 1700000.00, "percentage": 13.60 }
    ],
    "monthlyTrend": [
      { "period": "202501", "invoiceCount": 78, "totalTax": 1450000.00 },
      { "period": "202502", "invoiceCount": 82, "totalTax": 1520000.00 },
      { "period": "202503", "invoiceCount": 86, "totalTax": 1625000.00 }
    ],
    "pendingReceiveOrders": {
      "totalPending": 28,
      "totalPendingAmount": 8500000.00,
      "oldestPendingDate": "2025-02-15",
      "top5Suppliers": [
        { "supplierName": "河北XX钢铁集团", "pendingCount": 8, "pendingAmount": 2400000.00 },
        { "supplierName": "山东YY钢铁公司", "pendingCount": 6, "pendingAmount": 1800000.00 }
      ]
    }
  }
}
```

### 5.3 更新认证状态

#### PUT /api/v1/admin/input-ledger/{ledgerId}/certification

**请求体**

```json
{
  "certificationStatus": "CERTIFIED",
  "certificationDate": "2025-03-25"
}
```

### 5.4 更新抵扣状态

#### PUT /api/v1/admin/input-ledger/{ledgerId}/deduction

**请求体**

```json
{
  "deductionStatus": "DEDUCTED",
  "deductionPeriod": "202503",
  "deductionAmount": 57522.12
}
```

### 5.5 进项转出

#### POST /api/v1/admin/input-ledger/{ledgerId}/transfer-out

**请求体**

```json
{
  "transferOutAmount": 50000.00,
  "transferOutReason": "非正常损失，需作进项税额转出"
}
```

---

## 6. 进项库存配置接口

### 6.1 获取配置

#### GET /api/v1/admin/input-config

**响应示例**

```json
{
  "code": 200,
  "data": {
    "inventoryGranularity": "HYBRID",
    "amountToleranceRate": 0.005,
    "amountToleranceAbs": 100.00,
    "autoMatchScore": 85.00,
    "suggestMatchScore": 60.00,
    "pullIntervalMinutes": 30,
    "pullEnabled": true,
    "callbackEnabled": true,
    "allowOverInvoice": false,
    "overInvoiceTolerance": 0.005,
    "categoryMappings": [
      { "categoryName": "热轧卷板", "keywords": ["热轧卷板", "热轧卷", "热卷", "HRC"] },
      { "categoryName": "冷轧卷板", "keywords": ["冷轧卷板", "冷轧卷", "冷卷", "CRC", "SPCC", "SPCD"] },
      { "categoryName": "镀锌卷板", "keywords": ["镀锌卷板", "镀锌卷", "镀锌", "DX51D"] },
      { "categoryName": "中厚板", "keywords": ["中厚板", "中板", "厚板", "热轧板"] },
      { "categoryName": "型材", "keywords": ["H型钢", "角钢", "槽钢", "工字钢", "型材"] },
      { "categoryName": "线材", "keywords": ["线材", "盘条", "螺纹钢", "圆钢"] },
      { "categoryName": "管材", "keywords": ["无缝管", "焊管", "方管", "镀锌管", "管材"] }
    ]
  }
}
```

### 6.2 更新配置

#### PUT /api/v1/admin/input-config

**请求体**

```json
{
  "inventoryGranularity": "HYBRID",
  "amountToleranceRate": 0.005,
  "amountToleranceAbs": 100.00,
  "autoMatchScore": 85.00,
  "suggestMatchScore": 60.00,
  "pullIntervalMinutes": 30,
  "pullEnabled": true,
  "callbackEnabled": true,
  "allowOverInvoice": false,
  "overInvoiceTolerance": 0.005
}
```

### 6.3 品类映射管理

#### GET /api/v1/admin/input-config/category-mappings

#### POST /api/v1/admin/input-config/category-mappings

#### PUT /api/v1/admin/input-config/category-mappings/{mappingId}

#### DELETE /api/v1/admin/input-config/category-mappings/{mappingId}

---

## 7. 内部服务接口

### 7.1 收票引擎 — 处理单张发票全流程

#### POST /internal/receive-invoice/process

**请求体**

```json
{
  "receivedInvoiceId": 90001,
  "tenantId": 1
}
```

**响应示例**

```json
{
  "code": 200,
  "data": {
    "processResult": "AUTO_COMPLETE",
    "supplierMatch": {
      "matched": true,
      "supplierId": 2001,
      "method": "TAX_NO_EXACT"
    },
    "orderMatch": {
      "matched": true,
      "matchType": "AUTO",
      "matchedOrderCount": 2,
      "totalMatchScore": 90.25
    },
    "inventoryUpdate": {
      "granularity": "SPECIFICATION",
      "itemsAdded": 2,
      "totalQuantityAdded": 120.0000
    },
    "erpInvoice": {
      "erpInvoiceId": 80001,
      "erpInvoiceNo": "ERP-INV-20250324001"
    },
    "writeOff": {
      "ordersWrittenOff": 2,
      "fullyInvoiced": 1,
      "partialInvoiced": 1
    },
    "ledger": {
      "ledgerId": 85001,
      "accountingPeriod": "202503"
    }
  }
}
```

### 7.2 匹配引擎 — 进货单匹配

#### POST /internal/receive-invoice/match-orders

**请求体**

```json
{
  "receivedInvoiceId": 90001,
  "supplierId": 2001,
  "tenantId": 1,
  "invoiceItems": [
    {
      "itemId": 91001,
      "productName": "热轧卷板",
      "specification": "Q235B 5.75*1500*C",
      "quantity": 100.0000,
      "amount": 371681.00,
      "amountWithTax": 420000.00,
      "taxRate": 13.00
    }
  ]
}
```

### 7.3 进项库存入库

#### POST /internal/input-inventory/receive

**请求体**

```json
{
  "tenantId": 1,
  "receivedInvoiceId": 90001,
  "erpInvoiceId": 80001,
  "granularity": "SPECIFICATION",
  "items": [
    {
      "erpInvoiceItemId": 82001,
      "productName": "热轧卷板",
      "productCategory": "热轧卷板",
      "specification": "Q235B 5.75*1500*C",
      "unit": "吨",
      "quantity": 100.0000,
      "unitPrice": 3716.81,
      "unitPriceWithTax": 4200.00,
      "taxRate": 13.00,
      "supplierName": "河北XX钢铁集团有限公司",
      "inputDate": "2025-03-24"
    }
  ]
}
```

---

## 8. 诺诺网回调接口

### 8.1 进项发票推送回调

#### POST /callback/nuonuo/input-invoice

**请求体（诺诺网格式）**

```json
{
  "invoiceList": [
    {
      "invoiceCode": "3100222222",
      "invoiceNo": "88888888",
      "invoiceDate": "2025-03-24",
      "invoiceType": "s",
      "checkCode": "123456",
      "sellerName": "河北XX钢铁集团有限公司",
      "sellerTaxNo": "91130000XXXXXXXXXX",
      "sellerAddressPhone": "河北省唐山市XX路XX号 0315-XXXXXXXX",
      "sellerBankInfo": "中国银行唐山分行 0000XXXXXXXXXXXX",
      "buyerName": "某某钢贸有限公司",
      "buyerTaxNo": "91310000YYYYYYYY",
      "totalAmount": "442477.88",
      "totalTax": "57522.12",
      "amountWithTax": "500000.00",
      "items": [
        {
          "goodsName": "*钢材*热轧卷板",
          "specification": "Q235B 5.75*1500*C",
          "unit": "吨",
          "quantity": "100",
          "unitPrice": "3716.81",
          "amount": "371681.00",
          "taxRate": "0.13",
          "tax": "48318.53"
        }
      ]
    }
  ]
}
```

**响应**

```json
{
  "code": "0000",
  "message": "success"
}
```

---

## 9. MQ 事件定义（收票域新增）

| Topic | Tag | 生产者 | 消费者 | 说明 |
|-------|-----|--------|--------|------|
| RECEIVE_INVOICE | RECEIVED | input-service | input-service (engine) | 新进项发票已入库 |
| RECEIVE_INVOICE | SUPPLIER_MATCHED | input-service (engine) | input-service (matcher) | 供应商匹配完成 |
| RECEIVE_INVOICE | ALLOCATED | input-service (matcher) | input-service (erp-gen) | 分配完成 |
| RECEIVE_INVOICE | ERP_GENERATED | input-service (erp-gen) | input-service, notify-service | ERP发票已生成 |
| RECEIVE_INVOICE | WRITE_OFF_COMPLETE | input-service | notify-service, invoice-service | 核销完成，通知销项开票检查待开票队列 |
| RECEIVE_INVOICE | VOIDED | input-service | input-service, notify-service | 发票红冲/作废 |
| RECEIVE_INVOICE | UNMATCHED | input-service | notify-service | 无法自动匹配，需人工 |
| INPUT_INVENTORY | REPLENISH | input-service | invoice-service | 进项库存新增（联动销项开票） |
