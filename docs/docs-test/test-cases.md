# 测试用例

## 功能测试用例

| 用例ID | 场景 | 前置条件 | 测试步骤 | 预期结果 | 优先级 | 覆盖方法 |
|--------|------|---------|---------|---------|--------|---------|
| TC-001 | 微信条码支付-成功 | 商户已注册，payKey有效，微信配置完整 | 1. POST /f2fPay/doPay<br/>2. 传入有效payKey(16位)+authCode(18位)+orderNo+orderPrice=100.00+payType=MICRO_PAY+正确签名<br/>3. 微信micropay返回SUCCESS | 1. 返回视图/f2fAffirmPay<br/>2. result.status=SUCCESS<br/>3. result.trxNo非空<br/>4. 订单状态=SUCCESS<br/>5. 支付记录状态=SUCCESS | P0 | initPay, f2fPay, getF2FPayResultVo, completeSuccessOrder |
| TC-002 | 支付宝条码支付-成功 | 商户已注册，支付宝配置完整 | 1. POST /f2fPay/doPay<br/>2. 传入payType=F2F_PAY<br/>3. 支付宝tradePay返回成功 | 1. 发起orderSend轮询<br/>2. 轮询确认后订单状态=SUCCESS | P0 | initPay, f2fPay, getF2FPayResultVo |
| TC-003 | 订单查询 | 已有支付成功的订单 | 1. GET /f2fPay/order/query?trxNo=xxx | 返回RpTradePaymentRecord JSON，status=SUCCESS | P1 | orderQuery, getRecordByTrxNo |
| TC-004 | 重复订单正常支付 | 已存在WAITING_PAYMENT状态的订单，金额一致 | 1. 使用相同orderNo再次请求<br/>2. 金额与已有订单一致 | 1. 复用已有订单<br/>2. 执行支付<br/>3. 不创建新订单 | P1 | f2fPay |

## 异常测试用例

| 用例ID | 场景 | 前置条件 | 测试步骤 | 预期结果 | 优先级 | 覆盖方法 |
|--------|------|---------|---------|---------|--------|---------|
| TC-101 | 重复支付 | 订单状态=SUCCESS | 1. 使用已支付成功的orderNo再次请求 | 抛出TradeBizException("订单已支付成功,无需重复支付") | P0 | f2fPay |
| TC-102 | 金额不匹配 | 已存在订单，金额=100.00 | 1. 使用相同orderNo但orderPrice=200.00 | 抛出TradeBizException("错误的订单") | P0 | f2fPay |
| TC-103 | 交易类型不支持 | - | 1. 传入payType=SCANPAY | 抛出PayBizException("交易类型有误，不支持该交易") | P0 | f2fPay |
| TC-104 | payKey过短 | - | 1. 传入payKey=15位 | 参数校验失败，返回异常页面 | P1 | initPay |
| TC-105 | payKey过长 | - | 1. 传入payKey=33位 | 参数校验失败，返回异常页面 | P1 | initPay |
| TC-106 | authCode过短 | - | 1. 传入authCode=15位 | 参数校验失败，返回异常页面 | P1 | initPay |
| TC-107 | authCode过长 | - | 1. 传入authCode=21位 | 参数校验失败，返回异常页面 | P1 | initPay |
| TC-108 | 签名错误 | - | 1. 传入错误的sign值 | 参数校验失败，返回异常页面 | P0 | initPay |
| TC-109 | 商户不存在 | payKey有效但商户已注销 | 1. 正常请求 | 抛出UserBizException("用户不存在") | P1 | f2fPay |
| TC-110 | 费率配置缺失 | 商户存在但未配置费率 | 1. 正常请求 | 抛出UserBizException("用户支付配置有误") | P1 | f2fPay |

## 支付通道异常测试用例

| 用例ID | 场景 | 前置条件 | 测试步骤 | 预期结果 | 优先级 | 覆盖方法 |
|--------|------|---------|---------|---------|--------|---------|
| TC-201 | 微信返回空结果 | 微信API超时或无响应 | 1. 模拟micropay返回null | 1. 状态=WAITING_PAYMENT<br/>2. 调用orderSend发起轮询 | P1 | getF2FPayResultVo |
| TC-202 | 微信验签失败 | 微信返回数据被篡改 | 1. 模拟micropay返回错误签名 | 1. completeFailOrder("签名校验失败")<br/>2. 订单状态=FAILED | P0 | getF2FPayResultVo, completeFailOrder |
| TC-203 | 微信USERPAYING | 用户正在输入密码 | 1. 模拟micropay返回err_code=USERPAYING | 1. 状态=WAITING_PAYMENT<br/>2. 调用orderSend发起轮询 | P1 | getF2FPayResultVo |
| TC-204 | 微信BANKERROR | 银行系统异常 | 1. 模拟micropay返回err_code=BANKERROR | 1. 状态=WAITING_PAYMENT<br/>2. 调用orderSend发起轮询 | P1 | getF2FPayResultVo |
| TC-205 | 微信业务失败 | 余额不足 | 1. 模拟micropay返回err_code=NOTENOUGH | 1. completeFailOrder("余额不足")<br/>2. 订单状态=FAILED | P1 | getF2FPayResultVo, completeFailOrder |
| TC-206 | 支付宝API异常 | 支付宝服务不可用 | 1. 模拟tradePay抛出AlipayApiException | 抛出PayBizException("请求支付宝异常") | P1 | getF2FPayResultVo |

## 状态转换测试用例

| 用例ID | 场景 | 初始状态 | 触发事件 | 预期状态 | 优先级 | 覆盖方法 |
|--------|------|---------|---------|---------|--------|---------|
| TC-301 | 支付成功 | WAITING_PAYMENT | 微信返回SUCCESS | SUCCESS | P0 | completeSuccessOrder |
| TC-302 | 支付失败-签名错误 | WAITING_PAYMENT | 微信验签失败 | FAILED | P0 | completeFailOrder |
| TC-303 | 支付失败-业务错误 | WAITING_PAYMENT | 微信返回NOTENOUGH | FAILED | P1 | completeFailOrder |
| TC-304 | 保持等待-用户输入中 | WAITING_PAYMENT | 微信返回USERPAYING | WAITING_PAYMENT | P1 | getF2FPayResultVo |
| TC-305 | 保持等待-结果未知 | WAITING_PAYMENT | 微信返回空 | WAITING_PAYMENT | P1 | getF2FPayResultVo |

## 外部依赖故障测试用例

| 用例ID | 场景 | 模拟故障 | 预期结果 | 优先级 |
|--------|------|---------|---------|--------|
| TC-401 | user模块超时 | RpUserPayConfigService.getByPayKey超时 | 请求失败，返回系统异常页面 | P2 |
| TC-402 | notify模块超时 | RpNotifyService.notifySend超时 | 支付成功，事务不回滚（通知在事务外） | P2 |
| TC-403 | account模块异常 | RpAccountTransactionService.creditToAccount抛异常 | 事务回滚，订单状态不变 | P1 |
