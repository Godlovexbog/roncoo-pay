# 场景分析

## 场景清单

### 主场景（正常流程）

| 场景ID | 场景名称 | 触发条件 | 预期结果 | 覆盖规则 |
|--------|---------|---------|---------|---------|
| SC-001 | 微信条码支付-成功 | 有效payKey + 有效authCode + 新订单 + 微信返回SUCCESS | 订单状态=SUCCESS，支付记录=SUCCESS，账户入账(平台收款时)，商户收到通知 | RULE-01, RULE-03, RULE-05, RULE-08 |
| SC-002 | 支付宝条码支付-成功 | 有效payKey + 有效authCode + 新订单 + 支付宝返回成功 | 发起订单轮询，轮询确认后订单状态=SUCCESS，商户收到通知 | RULE-01, RULE-03, RULE-07 |
| SC-003 | 重复订单-正常支付 | 已存在但未支付的订单 + 相同金额 | 复用已有订单，执行支付，返回结果 | RULE-01, RULE-02 |
| SC-004 | 订单查询 | 已知trxNo | 返回支付记录JSON | - |

### 异常场景（业务异常）

| 场景ID | 场景名称 | 触发条件 | 预期结果 | 覆盖规则 |
|--------|---------|---------|---------|---------|
| SC-101 | 重复支付 | 已支付成功的订单再次提交 | 抛出TradeBizException("订单已支付成功,无需重复支付") | RULE-03 |
| SC-102 | 金额不匹配 | 传入金额与已有订单金额不一致 | 抛出TradeBizException("错误的订单") | RULE-02 |
| SC-103 | 交易类型不支持 | payType不是MICRO_PAY或F2F_PAY | 抛出PayBizException("交易类型有误，不支持该交易") | RULE-04 |
| SC-104 | 商户配置不存在 | payKey对应的配置不存在 | 返回参数校验异常 | - |
| SC-105 | 商户不存在 | merchantNo对应的商户信息不存在 | 抛出UserBizException("用户不存在") | - |
| SC-106 | 费率配置缺失 | productCode对应的费率配置不存在 | 抛出UserBizException("用户支付配置有误") | - |
| SC-107 | 支付方式错误 | payWayCode不是WEIXIN或ALIPAY | 抛出TradeBizException("错误的支付方式") | - |

### 支付通道异常场景

| 场景ID | 场景名称 | 触发条件 | 预期结果 | 覆盖规则 |
|--------|---------|---------|---------|---------|
| SC-201 | 微信返回空结果 | micropay返回null或空Map | 状态保持WAITING_PAYMENT，发起orderSend轮询 | RULE-06 |
| SC-202 | 微信验签失败 | 微信返回结果的签名不匹配 | completeFailOrder("签名校验失败")，订单状态=FAILED | RULE-05 |
| SC-203 | 微信通讯失败 | return_code != SUCCESS | completeFailOrder("通讯失败")，订单状态=FAILED | RULE-06 |
| SC-204 | 微信用户正在输入密码 | err_code = USERPAYING | 状态保持WAITING_PAYMENT，发起orderSend轮询 | RULE-06 |
| SC-205 | 微信银行系统异常 | err_code = BANKERROR | 状态保持WAITING_PAYMENT，发起orderSend轮询 | RULE-06 |
| SC-206 | 微信系统异常 | err_code = SYSTEMERROR | 状态保持WAITING_PAYMENT，发起orderSend轮询 | RULE-06 |
| SC-207 | 微信业务失败 | 其他err_code（如余额不足、卡号错误） | completeFailOrder(err_code_des)，订单状态=FAILED | RULE-06 |
| SC-208 | 支付宝API调用异常 | tradePay抛出AlipayApiException | 抛出PayBizException("请求支付宝异常") | RULE-07 |

### 外部依赖故障场景

| 场景ID | 场景名称 | 触发条件 | 预期结果 | 影响范围 |
|--------|---------|---------|---------|---------|
| SC-301 | user模块不可用 | RpUserPayConfigService查询超时 | 请求失败，返回系统异常 | 所有请求 |
| SC-302 | notify模块不可用 | RpNotifyService.notifySend超时 | 支付成功但商户未收到通知，商户可主动查询 | 支付成功场景 |
| SC-303 | account模块不可用 | RpAccountTransactionService.creditToAccount异常 | 事务回滚，订单状态不更新 | 平台收款场景 |
| SC-304 | 微信API网络超时 | micropay调用超时 | 结果未知，发起orderSend轮询 | 微信支付场景 |
| SC-305 | 支付宝API网络超时 | tradePay调用超时 | 抛出异常，订单状态不变 | 支付宝支付场景 |

### 边界场景

| 场景ID | 场景名称 | 触发条件 | 预期结果 | 覆盖规则 |
|--------|---------|---------|---------|---------|
| SC-401 | 最小金额支付 | orderPrice = 0.01 | 正常支付流程 | - |
| SC-402 | 最大金额支付 | orderPrice = 9999999999.99 | 正常支付流程 | - |
| SC-403 | 最小长度payKey | payKey = 16位 | 正常通过校验 | - |
| SC-404 | 最大长度payKey | payKey = 32位 | 正常通过校验 | - |
| SC-405 | 最小长度authCode | authCode = 16位 | 正常通过校验 | - |
| SC-406 | 最大长度authCode | authCode = 20位 | 正常通过校验 | - |
| SC-407 | 平台收款场景 | fundIntoType = PLAT_RECEIVES | 支付成功后执行creditToAccount入账 | RULE-08 |
| SC-408 | 商户收款场景 | fundIntoType = MERCHANT_RECEIVES | 支付成功后跳过入账 | RULE-08 |
