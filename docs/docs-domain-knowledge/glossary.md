# 领域术语表

## 业务实体术语

| 术语 | 英文 | 定义 | 相关代码位置 | 关联能力 |
|------|------|------|-------------|---------|
| 条码支付 | F2F Payment (Face-to-Face) | 商户通过扫码枪扫描用户付款码完成支付 | F2FPayController | CAP-02 |
| 支付授权码 | AuthCode | 用户付款码，16-20位数字 | F2FPayRequestBo.authCode | CAP-02 |
| 商户 | Merchant | 接入支付平台的商家用户 | RpUserInfo | CAP-02 |
| 商户订单号 | Merchant Order No | 商户系统生成的唯一订单标识 | RpTradePaymentOrder.merchantOrderNo | CAP-02 |
| 交易流水号 | Transaction No (TrxNo) | 支付平台生成的唯一交易标识 | RpTradePaymentRecord.trxNo | CAP-02 |
| 银行订单号 | Bank Order No | 支付平台发给银行/支付通道的订单号 | RpTradePaymentRecord.bankOrderNo | CAP-02 |
| 银行流水号 | Bank Transaction No | 银行/支付通道返回的交易流水号 | RpTradePaymentRecord.bankTrxNo | CAP-02 |
| 支付记录 | Payment Record | 每次支付尝试的详细记录 | RpTradePaymentRecord | CAP-02 |
| 支付订单 | Payment Order | 商户发起的支付请求订单 | RpTradePaymentOrder | CAP-02 |
| 支付KEY | PayKey | 商户身份标识，用于API鉴权 | RpUserPayConfig.payKey | CAP-02 |
| 商户通知URL | Notify URL | 支付完成后异步通知商户的回调地址 | RpTradePaymentRecord.notifyUrl | CAP-02 |
| 平台收入 | Platform Income | 支付平台从交易中获得的收入 | RpTradePaymentRecord.platIncome | CAP-02 |
| 平台成本 | Platform Cost | 支付平台为交易付出的通道成本 | RpTradePaymentRecord.platCost | CAP-02 |
| 平台利润 | Platform Profit | 平台收入减去平台成本 | RpTradePaymentRecord.platProfit | CAP-02 |

## 状态术语

| 术语 | 英文 | 枚举类 | 值 | 含义 | 关联能力 |
|------|------|--------|----|------|---------|
| 支付成功 | Success | TradeStatusEnum | SUCCESS | 订单支付成功，资金已到账 | CAP-02 |
| 支付失败 | Failed | TradeStatusEnum | FAILED | 订单支付失败 | CAP-02 |
| 等待支付 | Waiting | TradeStatusEnum | WAITING_PAYMENT | 订单已创建，等待用户付款 | CAP-02 |
| 订单已创建 | Created | TradeStatusEnum | CREATED | 订单已创建但未发起支付 | CAP-02 |
| 订单已取消 | Canceled | TradeStatusEnum | CANCELED | 订单已被取消 | CAP-02 |
| 微信刷卡 | Micro Pay | PayTypeEnum | MICRO_PAY | 微信条码支付（用户被扫） | CAP-02 |
| 支付宝条码 | F2F Pay | PayTypeEnum | F2F_PAY | 支付宝条码支付（用户被扫） | CAP-02 |
| 微信支付 | WeChat Pay | PayWayEnum | WEIXIN | 微信支付通道 | CAP-02 |
| 支付宝 | Alipay | PayWayEnum | ALIPAY | 支付宝支付通道 | CAP-02 |

## 业务操作术语

| 术语 | 英文 | 含义 | 相关方法 | 关联能力 |
|------|------|------|---------|---------|
| 初始化支付 | Init Pay | 接收商户支付请求，校验参数 | F2FPayController.initPay() | CAP-02 |
| 执行条码支付 | F2F Pay | 调用微信/支付宝API完成支付 | RpTradePaymentManagerServiceImpl.f2fPay() | CAP-02 |
| 获取支付结果 | Get F2F Result | 生成支付结果VO，含签名 | getF2FPayResultVo() | CAP-02 |
| 完成成功订单 | Complete Success | 更新状态、入账、通知商户 | completeSuccessOrder() | CAP-02 |
| 完成失败订单 | Complete Fail | 更新状态为失败、通知商户 | completeFailOrder() | CAP-02 |
| 封装订单 | Seal Order | 创建RpTradePaymentOrder实体 | sealF2FRpTradePaymentOrder() | CAP-02 |
| 封装支付记录 | Seal Record | 创建RpTradePaymentRecord实体 | sealRpTradePaymentRecord() | CAP-02 |
| 订单查询 | Order Query | 按交易流水号查询支付记录 | F2FPayController.orderQuery() | CAP-02 |
| 订单轮询 | Order Polling | 主动查询上游支付状态 | RpNotifyService.orderSend() | CAP-02 |
| 商户通知 | Merchant Notify | 异步通知商户支付结果 | RpNotifyService.notifySend() | CAP-02 |

## 外部系统术语

| 术语 | 英文 | 含义 | 相关代码 | 关联能力 |
|------|------|------|---------|---------|
| 微信刷卡支付API | WeChat Micropay API | 微信条码支付接口 | WeiXinPayUtil.micropay() | CAP-02 |
| 支付宝条码支付API | Alipay TradePay API | 支付宝条码支付接口 | AliPayUtil.tradePay() | CAP-02 |
| 商户API签名 | Merchant API Sign | 商户通知URL的MD5签名 | MerchantApiUtil.getSign() | CAP-02 |
| 账户入账 | Account Credit | 平台收款时向商户账户加款 | RpAccountTransactionService.creditToAccount() | CAP-02 |
