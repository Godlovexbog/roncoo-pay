# 业务规则清单：条码支付流程

## RULE-01：订单幂等控制

| 属性 | 内容 |
|------|------|
| **规则描述** | 同一(商户号 + 商户订单号)只能有一个活跃订单 |
| **实现位置** | RpTradePaymentManagerServiceImpl.f2fPay() L197-L211 |
| **判断逻辑** | `rpTradePaymentOrderDao.selectByMerchantNoAndMerchantOrderNo(merchantNo, orderNo)` 查询订单是否存在，不存在则创建，存在则复用 |
| **违规处理** | 订单已存在时不创建新订单，复用已有订单 |

---

## RULE-02：订单金额一致性

| 属性 | 内容 |
|------|------|
| **规则描述** | 已存在订单的金额必须与传入金额完全一致 |
| **实现位置** | RpTradePaymentManagerServiceImpl.f2fPay() L204-L206 |
| **判断逻辑** | `rpTradePaymentOrder.getOrderAmount().compareTo(f2FPayRequestBo.getOrderPrice()) != 0` |
| **违规处理** | 抛出 TradeBizException("错误的订单") |

---

## RULE-03：防重复支付

| 属性 | 内容 |
|------|------|
| **规则描述** | 已支付成功的订单不能重复支付 |
| **实现位置** | RpTradePaymentManagerServiceImpl.f2fPay() L208-L210 |
| **判断逻辑** | `TradeStatusEnum.SUCCESS.name().equals(rpTradePaymentOrder.getStatus())` |
| **违规处理** | 抛出 TradeBizException("订单已支付成功,无需重复支付") |

---

## RULE-04：交易类型限制

| 属性 | 内容 |
|------|------|
| **规则描述** | 条码支付只支持微信MICRO_PAY和支付宝F2F_PAY两种交易类型 |
| **实现位置** | RpTradePaymentManagerServiceImpl.f2fPay() L175-L178 |
| **判断逻辑** | `!PayTypeEnum.F2F_PAY.name().equals(payType) && !PayTypeEnum.MICRO_PAY.name().equals(payType)` |
| **违规处理** | 抛出 PayBizException("交易类型有误，不支持该交易") |

---

## RULE-05：微信支付结果验签

| 属性 | 内容 |
|------|------|
| **规则描述** | 微信返回的支付结果必须通过签名验证 |
| **实现位置** | RpTradePaymentManagerServiceImpl.getF2FPayResultVo() L254 |
| **判断逻辑** | `"YES".equals(wxResultMap.get("verify"))` |
| **违规处理** | 调用 completeFailOrder("签名校验失败!") |

---

## RULE-06：微信错误码分类处理

| 属性 | 内容 |
|------|------|
| **规则描述** | 根据微信返回的错误码决定是失败还是轮询 |
| **实现位置** | RpTradePaymentManagerServiceImpl.getF2FPayResultVo() L259 |
| **判断逻辑** | 以下错误码视为"结果未知"，需轮询确认:<br/>- BANKERROR: 银行系统异常<br/>- USERPAYING: 用户正在输入密码<br/>- SYSTEMERROR: 微信系统异常 |
| **其他错误码** | 视为支付失败，直接调用 completeFailOrder() |

---

## RULE-07：支付宝统一轮询

| 属性 | 内容 |
|------|------|
| **规则描述** | 支付宝条码支付不实时处理结果，统一通过订单轮询确认 |
| **实现位置** | RpTradePaymentManagerServiceImpl.getF2FPayResultVo() L279 |
| **判断逻辑** | 调用 AliPayUtil.tradePay() 后，无论返回什么，都调用 `rpNotifyService.orderSend()` |
| **原因** | 支付宝条码支付可能存在延迟，轮询机制确保结果准确性 |

---

## RULE-08：平台收款入账

| 属性 | 内容 |
|------|------|
| **规则描述** | 仅当资金流入类型为"平台收款"时，才执行账户入账操作 |
| **实现位置** | RpTradePaymentManagerServiceImpl.completeSuccessOrder() L331-L333 |
| **判断逻辑** | `FundInfoTypeEnum.PLAT_RECEIVES.name().equals(rpTradePaymentRecord.getFundIntoType())` |
| **入账金额** | `orderAmount - platIncome` (订单金额减去平台收入) |
| **商户收款** | 资金直接到商户账户，平台不执行入账 |

---

## RULE-09：商户通知签名

| 属性 | 内容 |
|------|------|
| **规则描述** | 发送给商户的异步通知URL必须携带签名 |
| **实现位置** | RpTradePaymentManagerServiceImpl.getMerchantNotifyUrl() L378-L380 |
| **签名算法** | `sign = MD5(排序参数拼接 + paySecret)` |
| **通知参数** | payKey, productName, orderNo, orderPrice, payWayCode, tradeStatus, orderDate, orderTime, remark, trxNo |

---

## RULE-10：事务一致性

| 属性 | 内容 |
|------|------|
| **规则描述** | 支付成功时的状态更新和入账操作必须在同一事务中 |
| **实现位置** | RpTradePaymentManagerServiceImpl.completeSuccessOrder() 方法注解 |
| **事务注解** | `@Transactional(rollbackFor = Exception.class)` |
| **事务范围** | 更新支付记录 → 更新订单 → 账户入账 → 构建通知URL |
| **回滚条件** | 任何 Exception 及其子类都会触发回滚 |

---

## RULE-11：IP白名单校验

| 属性 | 内容 |
|------|------|
| **规则描述** | 当商户安全等级为MD5_IP时，必须校验请求IP是否在白名单中 |
| **实现位置** | CnpPayService.checkIp() L41-L62 |
| **判断逻辑** | `SecurityRatingEnum.MD5_IP.name().equals(rpUserPayConfig.getSecurityRating())` 且 `merchantServerIp.indexOf(ip) < 0` |
| **违规处理** | 抛出 TradeBizException("非法IP请求") |

---

## RULE-12：请求参数签名验证

| 属性 | 内容 |
|------|------|
| **规则描述** | 所有请求必须携带MD5签名，服务端验证签名正确性 |
| **实现位置** | CnpPayService.checkParamAndGetUserPayConfig() L96-L99 |
| **判断逻辑** | `!MerchantApiUtil.isRightSign(jsonParamMap, rpUserPayConfig.getPaySecret(), sign)` |
| **违规处理** | 抛出 TradeBizException("订单签名异常") |
