# 活动节点说明

## ACT-01：参数校验与商户身份验证

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-010 |
| **入口方法** | `F2FPayController.initPay()` |
| **前置条件** | 商户已注册并获取payKey |
| **后置条件** | 获取到有效的RpUserPayConfig |
| **输入数据** | F2FPayRequestBo(payKey, authCode, productName, orderNo, orderPrice, orderIp, orderDate, orderTime, payType, sign) |
| **输出数据** | RpUserPayConfig(商户支付配置) |
| **执行步骤** | 1. 接收F2FPayRequestBo参数<br/>2. 调用CnpPayService.checkParamAndGetUserPayConfig()<br/>  2.1 验证payKey长度(16-32位)<br/>  2.2 验证authCode长度(16-20位)<br/>  2.3 验证orderNo长度(5-20位)<br/>  2.4 验证orderPrice格式<br/>  2.5 验证payType非空<br/>  2.6 验证签名sign = MD5(排序参数 + paySecret)<br/>  2.7 通过payKey查询RpUserPayConfig<br/>  2.8 验证商户状态是否正常<br/>3. 返回RpUserPayConfig |
| **异常处理** | 参数校验失败 → BizException<br/>签名验证失败 → BizException<br/>商户配置不存在 → UserBizException |
| **关联方法** | CnpPayService.checkParamAndGetUserPayConfig() |

---

## ACT-02：订单管理

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-011 |
| **入口方法** | `RpTradePaymentManagerServiceImpl.f2fPay()` |
| **前置条件** | 已通过参数校验，获取到RpUserPayConfig |
| **后置条件** | 获取到有效的RpTradePaymentOrder（新建或已存在） |
| **输入数据** | RpUserPayConfig, F2FPayRequestBo |
| **输出数据** | RpTradePaymentOrder |
| **数据读取** | rp_trade_payment_order, rp_user_info, rp_pay_way |
| **数据写入** | rp_trade_payment_order (订单不存在时insert) |
| **执行步骤** | 1. 验证payType为MICRO_PAY或F2F_PAY<br/>2. 根据payType获取PayWayEnum(WEIXIN/ALIPAY)<br/>3. 调用rpPayWayService.getByPayWayTypeCode()获取费率<br/>4. 调用rpUserInfoService.getDataByMerchentNo()获取商户信息<br/>5. 查询订单: selectByMerchantNoAndMerchantOrderNo()<br/>6. 分支: 不存在→创建订单; 已存在→校验金额和状态 |
| **异常处理** | 交易类型不支持 → PayBizException<br/>费率配置不存在 → UserBizException<br/>商户不存在 → UserBizException<br/>订单金额不匹配 → TradeBizException<br/>订单已支付成功 → TradeBizException |
| **关联方法** | sealF2FRpTradePaymentOrder(), selectByMerchantNoAndMerchantOrderNo(), insert() |

---

## ACT-03：执行支付

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-012 |
| **入口方法** | `RpTradePaymentManagerServiceImpl.getF2FPayResultVo()` |
| **前置条件** | 订单已创建/查询成功 |
| **后置条件** | 支付结果确定（成功/失败/未知） |
| **输入数据** | RpTradePaymentOrder, RpPayWay, authCode |
| **输出数据** | F2FPayResultVo |
| **数据读取** | rp_user_pay_info |
| **数据写入** | rp_trade_payment_record (insert) |
| **执行步骤** | 1. 设置订单payTypeCode/payWayCode/payWayName<br/>2. 更新订单<br/>3. 封装支付记录(sealRpTradePaymentRecord)<br/>4. insert支付记录<br/>5. 分支: 微信→micropay; 支付宝→tradePay<br/>6. 根据返回结果调用completeSuccessOrder/completeFailOrder或发起轮询 |
| **异常处理** | 商户支付渠道配置不存在 → UserBizException<br/>微信API异常 → 结果未知，发起轮询<br/>支付宝API异常 → TradeBizException |
| **关联方法** | sealRpTradePaymentRecord(), WeiXinPayUtil.micropay(), AliPayUtil.tradePay(), completeSuccessOrder(), completeFailOrder() |

---

## ACT-04：结果处理

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-013 |
| **入口方法** | `completeSuccessOrder()` / `completeFailOrder()` |
| **前置条件** | 支付结果已确定 |
| **后置条件** | 订单和记录状态已更新，商户已通知 |
| **输入数据** | 支付结果(成功/失败) |
| **输出数据** | F2FPayResultVo(含签名) |
| **数据读取** | rp_trade_payment_record, rp_trade_payment_order |
| **数据写入** | rp_trade_payment_record, rp_trade_payment_order |
| **事务属性** | completeSuccessOrder: @Transactional(rollbackFor = Exception.class) |
| **执行步骤** | 成功场景: 1.设置支付时间 2.设置银行流水号 3.更新记录=SUCCESS 4.更新订单=SUCCESS 5.账户入账(平台收款时) 6.构建通知URL 7.notifySend通知<br/>失败场景: 1.更新记录=FAILED 2.更新订单=FAILED 3.构建通知URL 4.notifySend通知 |
| **关联方法** | getMerchantNotifyUrl(), creditToAccount(), notifySend() |

---

## ACT-05：订单查询

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-014 |
| **入口方法** | `F2FPayController.orderQuery()` |
| **前置条件** | 已知交易流水号trxNo |
| **后置条件** | 返回支付记录JSON |
| **输入数据** | String trxNo |
| **输出数据** | RpTradePaymentRecord (JSON) |
| **数据读取** | rp_trade_payment_record |
| **数据写入** | 无 |
| **执行步骤** | 1. 接收trxNo参数<br/>2. 调用queryService.getRecordByTrxNo(trxNo)<br/>3. 序列化为JSON返回 |
| **异常处理** | 无 |
| **关联方法** | RpTradePaymentQueryService.getRecordByTrxNo() |
