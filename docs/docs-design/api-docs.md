# 接口文档

## POST /f2fPay/doPay

**所属能力**: CAP-02 条码支付能力
**Controller**: F2FPayController.initPay()
**JavaDoc**: 条码支付,商户通过前置设备获取到用户支付授权码后,请求支付网关支付.

### 请求参数

| 参数 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|---------|------|
| payKey | String | 是 | @Size(min=16, max=32) | 商户支付KEY |
| authCode | String | 是 | @Size(min=16, max=20) | 用户支付授权码（付款码） |
| productName | String | 是 | @Size(max=200) | 商品名称 |
| orderNo | String | 是 | @Size(min=5, max=20) | 商户订单号 |
| orderPrice | BigDecimal | 是 | @Digits(integer=12, fraction=2) | 订单金额（元） |
| orderIp | String | 是 | @Size(min=1, max=20) | 下单IP地址 |
| orderDate | String | 是 | @Size(min=1, max=8) | 订单日期（yyyyMMdd） |
| orderTime | String | 是 | @Size(min=1, max=14) | 订单时间（yyyyMMddHHmmss） |
| payType | String | 是 | @Size(min=1, max=14) | 交易类型：MICRO_PAY / F2F_PAY |
| sign | String | 是 | @NotNull | 数据签名（MD5） |
| remark | String | 否 | - | 支付备注 |

### 响应

| 类型 | 说明 |
|------|------|
| 成功 | 视图 `/f2fAffirmPay`，modelMap中包含 `result` (F2FPayResultVo) |
| 业务异常 | 视图 `exception/exception`，modelMap中包含 `errorMsg` |
| 系统异常 | 视图 `exception/exception`，modelMap中包含 `errorMsg` = "系统异常" |

### F2FPayResultVo 响应结构

| 字段 | 类型 | 说明 |
|------|------|------|
| status | String | 交易状态: SUCCESS / FAILED / WAITING_PAYMENT |
| trxNo | String | 交易流水号 |
| orderNo | String | 商户订单号 |
| payKey | String | 支付KEY |
| productName | String | 产品名称 |
| remark | String | 支付备注 |
| orderIp | String | 下单IP |
| sign | String | 签名数据 |
| field1-5 | String | 备注字段 |

### 错误码

| 错误场景 | 异常类型 | 错误信息 | 触发条件 |
|---------|---------|---------|---------|
| 参数校验失败 | BizException | 字段校验错误 | payKey/authCode/orderNo长度不合法，orderPrice格式错误 |
| 交易类型错误 | PayBizException | 交易类型有误，不支持该交易 | payType不是MICRO_PAY或F2F_PAY |
| 商户配置异常 | UserBizException | 用户支付配置有误 | payKey对应的商户配置不存在或费率配置缺失 |
| 商户不存在 | UserBizException | 用户不存在 | merchantNo对应的商户信息不存在 |
| 订单金额不匹配 | TradeBizException | 错误的订单 | 已存在订单的金额与传入金额不一致 |
| 重复支付 | TradeBizException | 订单已支付成功,无需重复支付 | 已存在订单状态为SUCCESS |
| 支付方式错误 | TradeBizException | 错误的支付方式 | payWayCode不是WEIXIN或ALIPAY |

### 调用示例

```
POST /f2fPay/doPay
Content-Type: application/x-www-form-urlencoded

payKey=test1234567890123456
&authCode=1234567890123456
&productName=测试商品
&orderNo=ORDER20260405001
&orderPrice=100.00
&orderIp=192.168.1.100
&orderDate=20260405
&orderTime=20260405120000
&payType=MICRO_PAY
&sign=md5签名值
```

---

## GET /f2fPay/order/query

**所属能力**: CAP-02 条码支付能力
**Controller**: F2FPayController.orderQuery()

### 请求参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| trxNo | String | 是 | 交易流水号 |

### 响应

| 类型 | 说明 |
|------|------|
| 成功 | JSON — RpTradePaymentRecord 序列化结果 |

### 调用示例

```
GET /f2fPay/order/query?trxNo=TRX2026040500001
```
