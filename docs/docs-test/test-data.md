# 测试数据

## F2FPayRequestBo 测试数据

### 字段定义

| 字段 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|---------|------|
| payKey | String | 是 | @Size(min=16, max=32), @NotNull | 商户支付KEY |
| authCode | String | 是 | @Size(min=16, max=20), @NotNull | 用户支付授权码 |
| productName | String | 是 | @Size(max=200), @NotNull | 商品名称 |
| orderNo | String | 是 | @Size(min=5, max=20), @NotNull | 商户订单号 |
| orderPrice | BigDecimal | 是 | @Digits(integer=12, fraction=2), @NotNull | 订单金额 |
| orderIp | String | 是 | @Size(min=1, max=20), @NotNull | 下单IP |
| orderDate | String | 是 | @Size(min=1, max=8), @NotNull | 订单日期 |
| orderTime | String | 是 | @Size(min=1, max=14), @NotNull | 订单时间 |
| payType | String | 是 | @Size(min=1, max=14), @NotNull | 交易类型 |
| sign | String | 是 | @NotNull | 数据签名 |
| remark | String | 否 | - | 支付备注 |

### 正常测试数据

| 用例 | payKey | authCode | productName | orderNo | orderPrice | orderIp | orderDate | orderTime | payType |
|------|--------|----------|-------------|---------|------------|---------|-----------|-----------|---------|
| 正常-微信 | "test1234567890123456" | "1234567890123456" | "测试商品" | "ORDER001" | 100.00 | "192.168.1.1" | "20260405" | "20260405120000" | "MICRO_PAY" |
| 正常-支付宝 | "test1234567890123456" | "12345678901234567890" | "测试商品" | "ORDER002" | 0.01 | "10.0.0.1" | "20260405" | "20260405120000" | "F2F_PAY" |
| 正常-大额 | "test1234567890123456" | "1234567890123456" | "大额商品" | "ORDER003" | 9999999999.99 | "172.16.0.1" | "20260405" | "20260405120000" | "MICRO_PAY" |
| 正常-带备注 | "test1234567890123456" | "1234567890123456" | "测试商品" | "ORDER004" | 50.50 | "192.168.1.1" | "20260405" | "20260405120000" | "MICRO_PAY" |

### 边界测试数据

| 用例 | 字段 | 测试值 | 预期 | 说明 |
|------|------|--------|------|------|
| 边界-payKey最小 | payKey | 16位字符串 | 通过 | @Size min=16 |
| 边界-payKey最大 | payKey | 32位字符串 | 通过 | @Size max=32 |
| 边界-authCode最小 | authCode | 16位数字 | 通过 | @Size min=16 |
| 边界-authCode最大 | authCode | 20位数字 | 通过 | @Size max=20 |
| 边界-orderNo最小 | orderNo | 5位字符串 | 通过 | @Size min=5 |
| 边界-orderNo最大 | orderNo | 20位字符串 | 通过 | @Size max=20 |
| 边界-orderPrice最小 | orderPrice | 0.01 | 通过 | 最小正金额 |
| 边界-orderPrice最大 | orderPrice | 9999999999.99 | 通过 | @Digits integer=12 |
| 边界-orderIp最小 | orderIp | "1.1.1.1" | 通过 | 最短有效IP |
| 边界-orderDate最小 | orderDate | "20260405" | 通过 | 8位日期 |
| 边界-orderTime最小 | orderTime | "20260405120000" | 通过 | 14位时间 |

### 异常测试数据

| 用例 | 字段 | 测试值 | 预期 | 说明 |
|------|------|--------|------|------|
| 异常-payKey空 | payKey | null | 校验失败 | @NotNull |
| 异常-payKey过短 | payKey | 15位字符串 | 校验失败 | @Size min=16 |
| 异常-payKey过长 | payKey | 33位字符串 | 校验失败 | @Size max=32 |
| 异常-authCode空 | authCode | null | 校验失败 | @NotNull |
| 异常-authCode过短 | authCode | 15位数字 | 校验失败 | @Size min=16 |
| 异常-authCode过长 | authCode | 21位数字 | 校验失败 | @Size max=20 |
| 异常-orderPrice负数 | orderPrice | -1.00 | 校验失败 | @Digits不允许负数 |
| 异常-orderPrice超限 | orderPrice | 10000000000.00 | 校验失败 | @Digits integer=12 |
| 异常-orderPrice精度超限 | orderPrice | 1.999 | 校验失败 | @Digits fraction=2 |
| 异常-orderPrice空 | orderPrice | null | 校验失败 | @NotNull |
| 异常-orderNo空 | orderNo | null | 校验失败 | @NotNull |
| 异常-orderNo过短 | orderNo | "ABCD" | 校验失败 | @Size min=5 |
| 异常-payType空 | payType | null | 校验失败 | @NotNull |
| 异常-payType不支持 | payType | "SCANPAY" | 业务异常 | 不是MICRO_PAY/F2F_PAY |
| 异常-sign空 | sign | null | 校验失败 | @NotNull |
| 异常-sign错误 | sign | "wrong_sign" | 业务异常 | 签名验证失败 |

## TradeStatusEnum 测试数据

| 枚举值 | 用途 | 测试场景 |
|--------|------|---------|
| WAITING_PAYMENT | 订单初始状态 | TC-001, TC-002, TC-003, TC-004 |
| SUCCESS | 支付成功 | TC-001, TC-101, TC-301 |
| FAILED | 支付失败 | TC-101, TC-202, TC-205, TC-302, TC-303 |
| CREATED | 订单已创建 | 预留状态，当前f2fPay流程不直接使用 |
| CANCELED | 订单已取消 | 预留状态，当前f2fPay流程不直接使用 |

## PayTypeEnum 测试数据

| 枚举值 | 用途 | 测试场景 |
|--------|------|---------|
| MICRO_PAY | 微信刷卡支付 | TC-001, TC-003, TC-004 |
| F2F_PAY | 支付宝条码支付 | TC-002 |
| DIRECT_PAY | 支付宝即时到账 | TC-103 (应拒绝) |
| SCANPAY | 扫码支付 | TC-103 (应拒绝) |

## PayWayEnum 测试数据

| 枚举值 | 用途 | 测试场景 |
|--------|------|---------|
| WEIXIN | 微信支付通道 | TC-001, TC-201~TC-205 |
| ALIPAY | 支付宝通道 | TC-002, TC-206 |

## RpTradePaymentOrder 测试数据

| 字段 | 正常值 | 边界值 | 异常值 |
|------|--------|--------|--------|
| merchantOrderNo | "ORDER20260405001" | 5位最小值 | null |
| orderAmount | 100.00 | 0.01 / 9999999999.99 | null / 负数 |
| status | "WAITING_PAYMENT" | 各枚举值 | null |
| merchantNo | "M100001" | - | null |
| productName | "测试商品" | 200位最大值 | null |

## RpTradePaymentRecord 测试数据

| 字段 | 正常值 | 边界值 | 异常值 |
|------|--------|--------|--------|
| trxNo | "TRX2026040500001" | - | null |
| bankOrderNo | "BNK2026040500001" | - | null |
| orderAmount | 100.00 | 0.01 / 9999999999.99 | null |
| platIncome | 0.60 (0.6%费率) | - | null |
| platCost | 0.20 | - | null |
| platProfit | 0.40 | - | null |
| status | "WAITING_PAYMENT" | SUCCESS / FAILED | null |

## 签名计算测试数据

### 签名算法: MD5(排序参数拼接 + paySecret)

| 参数 | 值 |
|------|-----|
| payKey | "test1234567890123456" |
| productName | "测试商品" |
| orderNo | "ORDER001" |
| orderPrice | 100.00 |
| payWayCode | "WEIXIN" |
| tradeStatus | "SUCCESS" |
| orderDate | "20260405" |
| orderTime | "20260405120000" |
| remark | "" |
| trxNo | "TRX2026040500001" |
| paySecret | "secret1234567890123456" |

**签名步骤**:
1. 按参数名ASCII排序: orderDate, orderNo, orderPrice, orderTime, payKey, payWayCode, productName, remark, tradeStatus, trxNo
2. 拼接为 key=value&key=value 格式
3. 末尾追加 &key=paySecret
4. MD5加密，转大写
