# 04 — API 接口设计

## 1. 接口规范

### 1.1 通用响应格式

```json
{
  "code": 200,
  "message": "success",
  "data": { },
  "traceId": "a1b2c3d4e5f6",
  "timestamp": 1700000000000
}
```

### 1.2 错误码规范

| 错误码 | 说明 |
|--------|------|
| 200 | 成功 |
| 400 | 参数校验失败 |
| 401 | 未认证 |
| 403 | 无权限 |
| 409 | 业务冲突（如重复提交） |
| 422 | 业务规则校验不通过 |
| 500 | 系统内部错误 |
| 10001 | 付款金额不足，不满足开票条件 |
| 10002 | 授信额度不足 |
| 10003 | 对公打款抬头不一致，无法合并开票 |
| 10004 | 进项库存不足 |
| 10005 | 税负率超出目标区间 |
| 10006 | 开票金额超出单据金额 |
| 10007 | 需要补签合同 |
| 10008 | 合同未签署，不允许开票 |
| 10009 | 第三方付款需上传授权书 |
| 10010 | 诺诺网开票失败 |
| 10011 | 归集组成员客户开票资格校验不通过 |
| 10012 | 不属于同一归集组，不允许跨客户合并开票 |
| 10013 | 归集组已停用或过期 |
| 10014 | 归集组框架协议已过期，需续签 |

### 1.3 分页参数

```json
{
  "pageNum": 1,
  "pageSize": 20,
  "orderBy": "created_time",
  "orderDir": "DESC"
}
```

### 1.4 认证方式

所有接口均需在 Header 中携带 JWT Token：

```
Authorization: Bearer <jwt_token>
```

---

## 2. 客户端接口（C端 - 客户使用）

### 2.1 单据查询相关

#### GET /api/v1/customer/orders/invoiceable

查询当前客户可开票的单据列表。如果当前客户属于某个开票归集组的主客户，则同时返回归集组内所有成员客户的可开票单据（按成员客户分组）。

**请求参数 (Query)**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNum | int | 否 | 页码，默认1 |
| pageSize | int | 否 | 每页条数，默认20 |
| orderNo | string | 否 | 单据号模糊搜索 |
| startDate | date | 否 | 起始日期 |
| endDate | date | 否 | 截止日期 |
| memberCustomerId | long | 否 | 归集组成员客户ID筛选（仅归集组主客户可用） |
| includeMembers | boolean | 否 | 是否包含归集组成员单据，默认true |

**响应示例**

```json
{
  "code": 200,
  "data": {
    "total": 50,
    "consolidationGroup": {
      "groupId": 8001,
      "groupName": "XX集团开票组",
      "masterCustomerId": 1001,
      "masterCustomerName": "XX集团有限公司",
      "memberCount": 4,
      "contractStrategy": "FRAMEWORK"
    },
    "ordersByCustomer": [
      {
        "customerId": 1001,
        "customerName": "XX集团有限公司",
        "memberRole": "MASTER",
        "orders": [
          {
            "orderId": 10001,
            "orderNo": "DO20250325001",
            "deliveryCompany": "XX集团有限公司",
            "totalAmount": 158000.00,
            "totalWeight": 25.3200,
            "invoicedAmount": 50000.00,
            "remainAmount": 108000.00,
            "orderDate": "2025-03-20",
            "invoiceStatus": "PARTIAL",
            "items": [
              {
                "itemId": 20001,
                "productName": "热轧卷板",
                "specification": "Q235B 5.75*1500*C",
                "quantity": 12.5000,
                "unitPrice": 4200.00,
                "amount": 52500.00,
                "invoicedAmount": 25000.00,
                "remainAmount": 27500.00
              }
            ],
            "paymentInfo": {
              "totalPaid": 158000.00,
              "payments": [
                {
                  "paymentNo": "PAY20250318001",
                  "payerName": "XX集团有限公司",
                  "paymentType": "CORPORATE",
                  "amount": 158000.00,
                  "paymentDate": "2025-03-18"
                }
              ]
            }
          }
        ]
      },
      {
        "customerId": 1002,
        "customerName": "XX集团上海分公司",
        "memberRole": "MEMBER",
        "orders": [
          {
            "orderId": 10005,
            "orderNo": "DO20250324005",
            "deliveryCompany": "XX集团上海分公司",
            "totalAmount": 200000.00,
            "totalWeight": 48.5000,
            "invoicedAmount": 0,
            "remainAmount": 200000.00,
            "orderDate": "2025-03-24",
            "invoiceStatus": "UN_INVOICED",
            "paymentInfo": {
              "totalPaid": 200000.00,
              "payments": [
                {
                  "paymentNo": "PAY20250322003",
                  "payerName": "XX集团上海分公司",
                  "paymentType": "CORPORATE",
                  "amount": 200000.00,
                  "paymentDate": "2025-03-22"
                }
              ]
            }
          }
        ]
      },
      {
        "customerId": 1003,
        "customerName": "XX集团杭州分公司",
        "memberRole": "MEMBER",
        "orders": [
          {
            "orderId": 10008,
            "orderNo": "DO20250323008",
            "deliveryCompany": "XX集团杭州分公司",
            "totalAmount": 150000.00,
            "remainAmount": 150000.00,
            "orderDate": "2025-03-23",
            "invoiceStatus": "UN_INVOICED"
          }
        ]
      }
    ]
  }
}
```

> 如果客户不属于任何归集组，`consolidationGroup` 为 `null`，`ordersByCustomer` 只含本客户的数据，与此前的 `list` 结构兼容。

### 2.2 开票抬头相关

#### POST /api/v1/customer/invoice/resolve-title

根据选择的单据自动解析开票抬头（核心接口）。支持跨客户归集开票场景：当选择了多个成员客户的单据时，强制使用归集组主客户的开票抬头。

**请求体**

```json
{
  "orderIds": [10001, 10002, 10003]
}
```

**响应示例（正常 - 对公一致）**

```json
{
  "code": 200,
  "data": {
    "resolveResult": "SUCCESS",
    "suggestedTitle": {
      "titleId": 5001,
      "titleName": "上海XX钢铁有限公司",
      "taxNo": "91310000XXXXXXXXXX",
      "address": "上海市宝山区XX路XX号",
      "phone": "021-XXXXXXXX",
      "bankName": "中国工商银行上海分行",
      "bankAccount": "1001XXXXXXXXXXXX"
    },
    "paymentAnalysis": {
      "allCorporate": true,
      "corporatePayerNames": ["上海XX钢铁有限公司"],
      "personalPayments": [],
      "thirdPartyPayments": [],
      "isConsistent": true
    },
    "titleMatchDelivery": true,
    "deliveryCompanies": ["上海XX钢铁有限公司"],
    "needContractSupplement": false,
    "availableTitles": [
      {
        "titleId": 5001,
        "titleName": "上海XX钢铁有限公司",
        "taxNo": "91310000XXXXXXXXXX",
        "isDefault": true
      }
    ]
  }
}
```

**响应示例（异常 - 对公抬头不一致）**

```json
{
  "code": 422,
  "message": "对公打款存在多个不同抬头，无法合并开票",
  "data": {
    "resolveResult": "CORPORATE_MISMATCH",
    "paymentAnalysis": {
      "allCorporate": true,
      "corporatePayerNames": ["上海XX钢铁有限公司", "杭州YY贸易有限公司"],
      "isConsistent": false
    },
    "suggestion": "请拆分单据，相同对公打款抬头的单据分别开票"
  }
}
```

**响应示例（需要第三方付款授权）**

```json
{
  "code": 200,
  "data": {
    "resolveResult": "THIRD_PARTY_DETECTED",
    "suggestedTitle": {
      "titleId": 5001,
      "titleName": "上海XX钢铁有限公司",
      "taxNo": "91310000XXXXXXXXXX"
    },
    "paymentAnalysis": {
      "allCorporate": false,
      "personalPayments": [
        {
          "paymentNo": "PAY20250319001",
          "payerName": "张三",
          "amount": 58000.00,
          "isThirdParty": true,
          "needAuth": true
        }
      ]
    },
    "thirdPartyAuthRequired": true,
    "authTemplateUrl": "/api/v1/file/template/third-party-auth"
  }
}
```

**响应示例（归集开票 — 多提货客户归一抬头）**

```json
{
  "code": 200,
  "data": {
    "resolveResult": "CONSOLIDATED",
    "suggestedTitle": {
      "titleId": 5001,
      "titleName": "XX集团有限公司",
      "taxNo": "91310000XXXXXXXXXX",
      "address": "上海市XX区XX路XX号",
      "phone": "021-XXXXXXXX",
      "bankName": "中国工商银行上海分行",
      "bankAccount": "1001XXXXXXXXXXXX"
    },
    "consolidationInfo": {
      "groupId": 8001,
      "groupName": "XX集团开票组",
      "titleLocked": true,
      "titleLockedReason": "归集开票模式下，开票抬头强制使用归集组主客户的抬头",
      "contractStrategy": "FRAMEWORK",
      "frameworkContractValid": true,
      "frameworkContractExpireDate": "2025-12-31"
    },
    "involvedCustomers": [
      {
        "customerId": 1001,
        "customerName": "XX集团有限公司",
        "deliveryCompany": "XX集团有限公司",
        "orderCount": 1,
        "orderAmount": 108000.00,
        "titleMatchDelivery": true,
        "invoiceQualification": "PASSED"
      },
      {
        "customerId": 1002,
        "customerName": "XX集团上海分公司",
        "deliveryCompany": "XX集团上海分公司",
        "orderCount": 1,
        "orderAmount": 200000.00,
        "titleMatchDelivery": false,
        "needContractSupplement": false,
        "invoiceQualification": "PASSED"
      },
      {
        "customerId": 1003,
        "customerName": "XX集团杭州分公司",
        "deliveryCompany": "XX集团杭州分公司",
        "orderCount": 1,
        "orderAmount": 150000.00,
        "titleMatchDelivery": false,
        "needContractSupplement": false,
        "invoiceQualification": "PASSED"
      }
    ],
    "paymentAnalysis": {
      "perCustomerCheck": true,
      "customerPayments": [
        {
          "customerId": 1001,
          "isConsistent": true,
          "status": "PASSED"
        },
        {
          "customerId": 1002,
          "isConsistent": true,
          "status": "PASSED"
        },
        {
          "customerId": 1003,
          "isConsistent": true,
          "status": "PASSED"
        }
      ]
    },
    "titleMatchDelivery": false,
    "needContractSupplement": false,
    "contractSupplementExemptReason": "归集组已有有效框架补充协议"
  }
}
```

#### GET /api/v1/customer/invoice/titles

查询客户已保存的开票抬头列表

**响应示例**

```json
{
  "code": 200,
  "data": [
    {
      "titleId": 5001,
      "titleName": "上海XX钢铁有限公司",
      "taxNo": "91310000XXXXXXXXXX",
      "address": "上海市宝山区XX路XX号",
      "phone": "021-XXXXXXXX",
      "bankName": "中国工商银行上海分行",
      "bankAccount": "1001XXXXXXXXXXXX",
      "invoiceType": "SPECIAL",
      "isDefault": true
    }
  ]
}
```

### 2.3 开票申请相关

#### POST /api/v1/customer/invoice/apply

提交开票申请。支持归集开票：当 `consolidationGroupId` 不为空时，`orderAmounts` 中的 `orderId` 可以来自归集组内不同成员客户的提货单。

**请求体**

```json
{
  "orderIds": [10001, 10005, 10008],
  "titleId": 5001,
  "invoiceType": "SPECIAL",
  "consolidationGroupId": 8001,
  "orderAmounts": [
    { "orderId": 10001, "customerId": 1001, "amount": 108000.00 },
    { "orderId": 10005, "customerId": 1002, "amount": 200000.00 },
    { "orderId": 10008, "customerId": 1003, "amount": 150000.00 }
  ],
  "remark": "请尽快开具",
  "thirdPartyAuthId": null,
  "idempotentKey": "apply_cust5001_20250325_abc123"
}
```

**响应示例**

```json
{
  "code": 200,
  "data": {
    "applyId": 30001,
    "applyNo": "INV20250325001",
    "status": "SUBMITTED",
    "totalAmount": 183000.00,
    "needContractSupplement": true,
    "contractInfo": {
      "contractId": 40001,
      "contractNo": "SC20250325001",
      "signMethods": ["ONLINE", "OFFLINE"],
      "onlineSignUrl": "https://esign.example.com/sign?flowId=xxx",
      "offlineDownloadUrl": "/api/v1/contract/40001/download"
    }
  }
}
```

#### GET /api/v1/customer/invoice/applies

查询客户的开票申请列表

**请求参数 (Query)**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNum | int | 否 | 页码 |
| pageSize | int | 否 | 每页条数 |
| status | string | 否 | 状态筛选 |
| applyNo | string | 否 | 申请单号 |
| startDate | date | 否 | 起始日期 |
| endDate | date | 否 | 截止日期 |

**响应示例**

```json
{
  "code": 200,
  "data": {
    "total": 15,
    "list": [
      {
        "applyId": 30001,
        "applyNo": "INV20250325001",
        "buyerName": "上海XX钢铁有限公司",
        "invoiceType": "SPECIAL",
        "totalAmount": 183000.00,
        "applyStatus": "SUCCESS",
        "invoiceCode": "3100XXX",
        "invoiceNumber": "XXXXXXXX",
        "invoiceDate": "2025-03-25",
        "invoicePdfUrl": "/api/v1/file/invoice/30001/pdf",
        "createdTime": "2025-03-25T10:30:00",
        "statusHistory": [
          { "status": "SUBMITTED", "time": "2025-03-25T10:30:00" },
          { "status": "PROCESSING", "time": "2025-03-25T10:30:05" },
          { "status": "INVOICING", "time": "2025-03-25T10:30:10" },
          { "status": "SUCCESS", "time": "2025-03-25T10:31:00" }
        ]
      }
    ]
  }
}
```

#### GET /api/v1/customer/invoice/applies/{applyId}

查询开票申请详情

#### GET /api/v1/customer/invoice/applies/{applyId}/pdf

下载/预览发票PDF

**响应**: 直接返回 PDF 文件流，Content-Type: application/pdf

### 2.4 第三方付款授权

#### POST /api/v1/customer/invoice/third-party-auth

上传第三方付款授权书

**请求体 (multipart/form-data)**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| paymentId | long | 是 | 关联的付款记录ID |
| delegatorName | string | 是 | 委托方名称 |
| agentName | string | 是 | 受托方名称 |
| authAmount | decimal | 是 | 授权金额 |
| authFile | file | 是 | 授权书文件（PDF/图片） |

**响应示例**

```json
{
  "code": 200,
  "data": {
    "authId": 6001,
    "reviewStatus": "PENDING",
    "ocrResult": {
      "detectedDelegator": "上海XX钢铁有限公司",
      "detectedAgent": "张三",
      "detectedAmount": "58000.00",
      "confidence": 0.92
    },
    "estimatedReviewTime": "30分钟内"
  }
}
```

### 2.5 合同签署相关

#### POST /api/v1/customer/contract/{contractId}/sign-method

选择合同签署方式

**请求体**

```json
{
  "signMethod": "ONLINE"
}
```

**响应示例（在线签署）**

```json
{
  "code": 200,
  "data": {
    "signMethod": "ONLINE",
    "signUrl": "https://esign.example.com/sign?flowId=xxx&token=yyy",
    "signExpireTime": "2025-03-26T10:30:00",
    "signTips": "请在24小时内完成签署"
  }
}
```

**响应示例（线下签署）**

```json
{
  "code": 200,
  "data": {
    "signMethod": "OFFLINE",
    "downloadUrl": "/api/v1/contract/40001/download",
    "uploadTips": "请下载合同PDF，打印盖章后扫描上传"
  }
}
```

#### GET /api/v1/customer/contract/{contractId}/download

下载合同PDF文件

#### POST /api/v1/customer/contract/{contractId}/upload-signed

上传线下签署的合同

**请求体 (multipart/form-data)**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| signedFile | file | 是 | 已盖章合同文件 |

---

## 3. 管理端接口（B端 - 运营/财务使用）

### 3.1 待处理队列

#### GET /api/v1/admin/invoice/pending-list

查询待处理的开票申请列表（含待进项、待人工处理等）

**请求参数 (Query)**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNum | int | 否 | 页码 |
| pageSize | int | 否 | 每页条数 |
| status | string | 否 | 状态: PENDING_INPUT, MANUAL, FAILED |
| customerId | long | 否 | 客户ID |
| minAmount | decimal | 否 | 最小金额 |
| maxAmount | decimal | 否 | 最大金额 |

### 3.2 人工调整接口

#### POST /api/v1/admin/invoice/{applyId}/manual-adjust

对开票申请进行人工调整

**请求体**

```json
{
  "adjustType": "MODIFY_ITEM",
  "reason": "客户要求调整品名规格",
  "items": [
    {
      "itemId": 50001,
      "action": "MODIFY",
      "productName": "冷轧卷板",
      "specification": "SPCC 1.0*1250*C",
      "quantity": 10.5000,
      "unitPrice": 4500.00,
      "taxRate": 13.00
    },
    {
      "action": "ADD",
      "productName": "镀锌卷板",
      "specification": "DX51D 0.8*1250*C",
      "quantity": 5.0000,
      "unitPrice": 5200.00,
      "taxRate": 13.00
    },
    {
      "itemId": 50003,
      "action": "REMOVE"
    }
  ]
}
```

**响应示例**

```json
{
  "code": 200,
  "data": {
    "adjustLogId": 7001,
    "originalAmount": 183000.00,
    "adjustedAmount": 173250.00,
    "amountDiff": -9750.00,
    "predictedTaxBurden": 2.35,
    "targetTaxRange": [2.0, 3.0],
    "inputMatchResult": {
      "allMatched": true,
      "matchedItems": 2,
      "unmatchedItems": 0
    },
    "needApproval": true,
    "approvalLevel": "财务主管",
    "status": "PENDING_APPROVAL"
  }
}
```

#### POST /api/v1/admin/invoice/{applyId}/manual-adjust/{adjustLogId}/approve

审批人工调整

**请求体**

```json
{
  "approved": true,
  "remark": "同意调整"
}
```

### 3.3 重新触发开票

#### POST /api/v1/admin/invoice/{applyId}/retry

手动重新触发开票（进项到货后或修复问题后）

### 3.4 进项库存管理

#### GET /api/v1/admin/input/inventory

查询进项库存

**请求参数 (Query)**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| productName | string | 否 | 品名 |
| specification | string | 否 | 规格 |
| status | string | 否 | 状态: AVAILABLE, EXHAUSTED |
| pageNum | int | 否 | 页码 |
| pageSize | int | 否 | 每页条数 |

#### POST /api/v1/admin/input/inventory/import

批量导入进项发票（Excel/CSV）

### 3.5 税务看板

#### GET /api/v1/admin/tax/dashboard

获取税务看板数据

**响应示例**

```json
{
  "code": 200,
  "data": {
    "currentPeriod": "202503",
    "cumulativeSales": 12500000.00,
    "cumulativeOutputTax": 1625000.00,
    "cumulativeInputTax": 1312500.00,
    "cumulativeTaxPayable": 312500.00,
    "taxBurdenRate": 2.50,
    "targetTaxRange": [2.0, 3.0],
    "taxBurdenTrend": [
      { "period": "202501", "rate": 2.10 },
      { "period": "202502", "rate": 2.35 },
      { "period": "202503", "rate": 2.50 }
    ],
    "inputInventorySummary": {
      "totalAvailable": 5000.00,
      "totalLocked": 800.00,
      "totalConsumed": 12000.00,
      "topCategories": [
        { "category": "热轧卷板", "available": 2000.00 },
        { "category": "冷轧卷板", "available": 1500.00 },
        { "category": "镀锌卷板", "available": 1500.00 }
      ]
    }
  }
}
```

### 3.6 合同审核

#### GET /api/v1/admin/contract/review-list

查询待审核合同列表

#### POST /api/v1/admin/contract/{contractId}/review

审核线下签署的合同

**请求体**

```json
{
  "approved": true,
  "remark": "印章真实有效"
}
```

### 3.7 客户开票配置

#### PUT /api/v1/admin/customer/{customerId}/invoice-config

更新客户开票配置

**请求体**

```json
{
  "invoicePolicy": "PAYMENT_FIRST",
  "creditLimit": 500000.00,
  "allowPartialInvoice": true,
  "allowMergeInvoice": true,
  "matchStrategy": "FLEXIBLE",
  "priceAdjustment": "ADJUST_WEIGHT",
  "targetTaxRateMin": 2.0,
  "targetTaxRateMax": 3.0,
  "targetMarginMin": 1.5,
  "targetMarginMax": 5.0
}
```

### 3.8 开票归集组管理

#### GET /api/v1/admin/consolidation-group/list

查询开票归集组列表

#### POST /api/v1/admin/consolidation-group

创建开票归集组

**请求体**

```json
{
  "groupName": "XX集团开票组",
  "groupCode": "CG20250325001",
  "masterCustomerId": 1001,
  "masterTitleId": 5001,
  "contractStrategy": "FRAMEWORK",
  "effectiveDate": "2025-01-01",
  "expireDate": "2025-12-31",
  "members": [
    { "customerId": 1002, "allowConsolidation": true },
    { "customerId": 1003, "allowConsolidation": true },
    { "customerId": 1004, "allowConsolidation": true }
  ],
  "remark": "XX集团及其子公司统一开票"
}
```

**响应示例**

```json
{
  "code": 200,
  "data": {
    "groupId": 8001,
    "groupCode": "CG20250325001",
    "groupName": "XX集团开票组",
    "masterCustomerName": "XX集团有限公司",
    "memberCount": 4,
    "status": "ACTIVE"
  }
}
```

#### PUT /api/v1/admin/consolidation-group/{groupId}

更新归集组（修改成员、合同策略等）

#### POST /api/v1/admin/consolidation-group/{groupId}/members

添加成员客户

**请求体**

```json
{
  "customerId": 1005,
  "allowConsolidation": true,
  "authorizationFileUrl": "/files/auth/group_8001_cust_1005.pdf"
}
```

#### DELETE /api/v1/admin/consolidation-group/{groupId}/members/{customerId}

移除成员客户

#### GET /api/v1/admin/consolidation-group/{groupId}

查询归集组详情（含所有成员及其开票统计）

**响应示例**

```json
{
  "code": 200,
  "data": {
    "groupId": 8001,
    "groupName": "XX集团开票组",
    "groupCode": "CG20250325001",
    "masterCustomerId": 1001,
    "masterCustomerName": "XX集团有限公司",
    "masterTitleName": "XX集团有限公司",
    "masterTitleTaxNo": "91310000XXXXXXXXXX",
    "contractStrategy": "FRAMEWORK",
    "frameworkContractStatus": "EFFECTIVE",
    "frameworkContractExpireDate": "2025-12-31",
    "status": "ACTIVE",
    "members": [
      {
        "customerId": 1001,
        "customerName": "XX集团有限公司",
        "memberRole": "MASTER",
        "allowConsolidation": true,
        "totalConsolidatedAmount": 850000.00,
        "lastInvoiceDate": "2025-03-25"
      },
      {
        "customerId": 1002,
        "customerName": "XX集团上海分公司",
        "memberRole": "MEMBER",
        "allowConsolidation": true,
        "totalConsolidatedAmount": 620000.00,
        "lastInvoiceDate": "2025-03-24"
      },
      {
        "customerId": 1003,
        "customerName": "XX集团杭州分公司",
        "memberRole": "MEMBER",
        "allowConsolidation": true,
        "totalConsolidatedAmount": 450000.00,
        "lastInvoiceDate": "2025-03-22"
      }
    ],
    "statistics": {
      "totalInvoiceCount": 15,
      "totalInvoiceAmount": 1920000.00,
      "thisMonthCount": 5,
      "thisMonthAmount": 680000.00
    }
  }
}
```

---

## 4. 内部服务接口（微服务间调用）

### 4.1 开票引擎接口

#### POST /internal/invoice-engine/calculate

开票运算（invoice-service 内部调用）

**请求体**

```json
{
  "applyId": 30001,
  "customerId": 1001,
  "customerType": "DEALER",
  "matchStrategy": "FLEXIBLE",
  "priceAdjustment": "ADJUST_WEIGHT",
  "orderItems": [
    {
      "orderItemId": 20001,
      "productName": "热轧卷板",
      "specification": "Q235B 5.75*1500*C",
      "quantity": 12.5000,
      "unitPrice": 4200.00,
      "amount": 52500.00,
      "taxRate": 13.00
    }
  ],
  "taxControl": {
    "currentTaxBurden": 2.35,
    "targetRange": [2.0, 3.0],
    "currentMargin": 3.2,
    "marginRange": [1.5, 5.0]
  }
}
```

**响应示例**

```json
{
  "code": 200,
  "data": {
    "calculatedItems": [
      {
        "productName": "热轧卷板",
        "specification": "Q235B 5.75*1500*C",
        "quantity": 12.7800,
        "unitPrice": 3634.51,
        "amount": 46472.84,
        "taxRate": 13.00,
        "taxAmount": 6041.47,
        "totalAmount": 52514.31,
        "adjustType": "ADJUST_WEIGHT",
        "originalQuantity": 12.5000,
        "matchedInputId": 8001,
        "matchedInputQuantity": 12.7800
      }
    ],
    "totalAmountWithoutTax": 46472.84,
    "totalTax": 6041.47,
    "totalAmount": 52514.31,
    "predictedTaxBurden": 2.42,
    "predictedMargin": 3.05,
    "inputMatchStatus": "MATCHED"
  }
}
```

### 4.2 进项匹配接口

#### POST /internal/input/match

进项库存匹配与锁定

**请求体**

```json
{
  "applyId": 30001,
  "items": [
    {
      "applyItemId": 50001,
      "productName": "热轧卷板",
      "specification": "Q235B 5.75*1500*C",
      "requiredQuantity": 12.7800,
      "taxRate": 13.00,
      "allowFuzzyMatch": true
    }
  ]
}
```

### 4.3 诺诺网对接接口

#### POST /internal/nuonuo/issue-invoice

调用诺诺网开票

**请求体**

```json
{
  "applyId": 30001,
  "seller": {
    "taxNo": "91310000YYYYYYYY",
    "name": "某某钢贸有限公司",
    "address": "上海市XX区XX路XX号",
    "phone": "021-YYYYYYYY",
    "bankName": "中国建设银行上海分行",
    "bankAccount": "3100XXXXXXXXXXXX"
  },
  "buyer": {
    "taxNo": "91310000XXXXXXXXXX",
    "name": "上海XX钢铁有限公司",
    "address": "上海市宝山区XX路XX号",
    "phone": "021-XXXXXXXX",
    "bankName": "中国工商银行上海分行",
    "bankAccount": "1001XXXXXXXXXXXX"
  },
  "invoiceType": "SPECIAL",
  "items": [
    {
      "goodsName": "热轧卷板",
      "specification": "Q235B 5.75*1500*C",
      "unit": "吨",
      "quantity": 12.7800,
      "unitPrice": 3634.51,
      "amount": 46472.84,
      "taxRate": 0.13,
      "tax": 6041.47,
      "taxCategoryCode": "1080201"
    }
  ],
  "remark": "",
  "callbackUrl": "https://api.example.com/callback/nuonuo/invoice-result"
}
```

---

## 5. WebSocket 接口（实时推送）

### 5.1 开票进度推送

**连接地址**: `wss://api.example.com/ws/invoice-progress`

**认证**: 连接时通过 Query 参数传递 token

**推送消息格式**

```json
{
  "type": "INVOICE_PROGRESS",
  "data": {
    "applyId": 30001,
    "applyNo": "INV20250325001",
    "status": "INVOICING",
    "message": "正在调用电子税务局开票...",
    "progress": 75,
    "timestamp": 1700000000000
  }
}
```

**推送消息类型**

| type | 说明 |
|------|------|
| INVOICE_PROGRESS | 开票进度更新 |
| INVOICE_SUCCESS | 开票成功 |
| INVOICE_FAILED | 开票失败 |
| CONTRACT_SIGN_NOTIFY | 合同签署提醒 |
| CONTRACT_SIGNED | 合同签署完成 |
| AUTH_REVIEW_RESULT | 授权书审核结果 |

---

## 6. 回调接口（第三方回调）

### 6.1 诺诺网开票结果回调

#### POST /callback/nuonuo/invoice-result

**请求体（诺诺网回调格式）**

```json
{
  "serialNo": "NNXXXXXXXXXX",
  "orderNo": "INV20250325001",
  "status": "2",
  "statusMsg": "开票成功",
  "invoiceCode": "3100XXXXXXXX",
  "invoiceNo": "XXXXXXXX",
  "invoiceDate": "2025-03-25",
  "pdfUrl": "https://xxx.nuonuo.com/pdf/xxx",
  "ofdUrl": "https://xxx.nuonuo.com/ofd/xxx"
}
```

### 6.2 电子签章回调

#### POST /callback/esign/sign-result

**请求体**

```json
{
  "flowId": "esign_flow_xxx",
  "signResult": "SUCCESS",
  "signTime": "2025-03-25T15:30:00",
  "signedFileUrl": "https://xxx.esign.cn/file/xxx",
  "signers": [
    {
      "signerName": "上海XX钢铁有限公司",
      "signStatus": "SIGNED",
      "signTime": "2025-03-25T15:30:00"
    }
  ]
}
```

---

## 7. 接口安全与限流

### 7.1 接口限流配置

| 接口分组 | 限流策略 | 说明 |
|---------|---------|------|
| 客户查询类 | 500 QPS / 用户 | 滑动窗口 |
| 开票申请提交 | 10 QPS / 用户 | 令牌桶 |
| 抬头解析 | 100 QPS / 用户 | 滑动窗口 |
| 管理端操作 | 200 QPS / 全局 | 令牌桶 |
| 文件下载 | 50 QPS / 用户 | 漏桶 |
| 诺诺网回调 | 1000 QPS / 全局 | 无限制（内部白名单） |

### 7.2 接口签名（防篡改）

关键接口（开票提交、人工调整）需要额外的请求签名：

```
X-Sign-Timestamp: 1700000000000
X-Sign-Nonce: abc123
X-Sign-Signature: sha256(timestamp + nonce + body + secret)
```
