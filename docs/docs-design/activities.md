# 活动节点说明：条码支付流程

## ACT-01：参数校验与商户身份验证

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-010 |
| **入口方法** | F2FPayController.initPay() → CnpPayService.checkParamAndGetUserPayConfig() |
| **前置条件** | 商户已注册并获取payKey |
| **后置条件** | 获取到有效的RpUserPayConfig |
| **输入数据** | F2FPayRequestBo(payKey, authCode, productName, orderNo, orderPrice, orderIp, orderDate, orderTime, payType, sign) |
| **输出数据** | RpUserPayConfig(商户支付配置) |
| **数据读取** | rp_user_pay_config (getByPayKey) |
| **数据写入** | 无 |
| **执行步骤** | 1. Validator.validate(object, bindingResult) 校验请求参数<br/>2. 如果校验失败 → getErrorResponse() 拼凑错误信息 → 抛出 PayBizException<br/>3. JSONObject.toJSON(object) 序列化请求对象<br/>4. JSONObject.parseObject() 反序列化为Map<br/>5. 获取payKey参数<br/>6. RpUserPayConfigService.getByPayKey(payKey) 查询商户配置<br/>7. 如果配置不存在 → 抛出 PayBizException<br/>8. checkIp() IP白名单校验（安全等级=MD5_IP时）<br/>9. MerchantApiUtil.isRightSign() MD5签名验证<br/>10. 如果签名失败 → 抛出 TradeBizException |
| **异常处理** | 参数校验失败 → PayBizException(REQUEST_PARAM_ERR)<br/>商户不存在 → PayBizException(USER_PAY_CONFIG_IS_NOT_EXIST)<br/>IP校验失败 → TradeBizException(TRADE_PARAM_ERROR)<br/>签名验证失败 → TradeBizException(TRADE_ORDER_ERROR) |
| **关联方法** | CnpPayService.checkParamAndGetUserPayConfig(), CnpPayService.checkIp(), CnpPayService.getErrorResponse(), RpUserPayConfigServiceImpl.getByPayKey(), MerchantApiUtil.isRightSign() |

---

## ACT-02：订单管理

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-011 |
| **入口方法** | RpTradePaymentManagerServiceImpl.f2fPay() |
| **前置条件** | 已通过参数校验，获取到RpUserPayConfig |
| **后置条件** | 获取到有效的RpTradePaymentOrder（新建或已存在） |
| **输入数据** | RpUserPayConfig, F2FPayRequestBo |
| **输出数据** | RpTradePaymentOrder |
| **数据读取** | rp_trade_payment_order, rp_user_info, rp_pay_way |
| **数据写入** | rp_trade_payment_order (订单不存在时insert) |
| **执行步骤** | 1. PayTypeEnum.getEnum(payType) 验证交易类型<br/>2. PayWayEnum.getEnum(payType.getWay()) 获取支付方式<br/>3. RpPayWayService.getByPayWayTypeCode() 获取费率配置<br/>4. RpUserInfoService.getDataByMerchentNo() 获取商户信息<br/>5. RpTradePaymentOrderDao.selectByMerchantNoAndMerchantOrderNo() 查询订单<br/>6. 如果订单不存在 → sealF2FRpTradePaymentOrder() 创建订单 → insert()<br/>7. 如果订单已存在 → 校验金额和状态 |
| **异常处理** | 交易类型不支持 → PayBizException(REQUEST_PARAM_ERR)<br/>费率配置不存在 → UserBizException(USER_PAY_CONFIG_ERRPR)<br/>商户不存在 → UserBizException(USER_IS_NULL)<br/>订单金额不匹配 → TradeBizException(TRADE_ORDER_ERROR)<br/>订单已支付成功 → TradeBizException(TRADE_ORDER_ERROR) |
| **关联方法** | RpTradePaymentManagerServiceImpl.f2fPay(), sealF2FRpTradePaymentOrder(), BuildNoServiceImpl.buildBankOrderNo(), BuildNoServiceImpl.buildTrxNo() |

---

## ACT-03：执行支付

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-012 |
| **入口方法** | RpTradePaymentManagerServiceImpl.getF2FPayResultVo() |
| **前置条件** | 订单已创建/查询成功 |
| **后置条件** | 支付结果确定（成功/失败/未知） |
| **输入数据** | RpTradePaymentOrder, RpPayWay, authCode |
| **输出数据** | F2FPayResultVo(含签名) |
| **数据读取** | rp_user_pay_info (获取商户支付渠道配置) |
| **数据写入** | rp_trade_payment_record (insert) |
| **执行步骤** | 1. 根据payWayCode设置订单的payTypeCode和payWayName<br/>2. RpTradePaymentOrderDao.update() 更新订单支付类型<br/>3. sealRpTradePaymentRecord() 封装支付记录<br/>4. RpTradePaymentRecordDao.insert() 保存支付记录<br/>5. 分支判断支付方式：<br/>   a) 微信 → RpUserPayInfoService.getByUserNo() → WeiXinPayUtil.micropay()<br/>   b) 支付宝 → RpUserPayInfoService.getByUserNo() → AliPayUtil.tradePay()<br/>6. 根据返回结果调用completeSuccessOrder/completeFailOrder或发起轮询 |
| **异常处理** | 商户支付渠道配置不存在 → UserBizException(USER_PAY_CONFIG_ERRPR)<br/>微信API异常 → 结果未知，发起轮询<br/>支付宝API异常 → PayBizException(REQUEST_BANK_ERR) |
| **关联方法** | getF2FPayResultVo(), sealRpTradePaymentRecord(), WeiXinPayUtil.micropay(), AliPayUtil.tradePay(), completeSuccessOrder(), completeFailOrder() |

---

## ACT-04：结果处理

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-013 |
| **入口方法** | RpTradePaymentManagerServiceImpl.completeSuccessOrder() / completeFailOrder() |
| **前置条件** | 支付结果已确定 |
| **后置条件** | 订单和记录状态已更新，商户已通知 |
| **输入数据** | 支付结果(成功/失败) |
| **输出数据** | 无（更新数据库，发送通知） |
| **数据读取** | rp_trade_payment_record, rp_trade_payment_order |
| **数据写入** | rp_trade_payment_record, rp_trade_payment_order |
| **执行步骤** | **成功场景**:<br/>1. 设置支付记录的paySuccessTime、bankTrxNo、bankReturnMsg、status=SUCCESS<br/>2. RpTradePaymentRecordDao.update() 更新支付记录<br/>3. 查询关联的订单<br/>4. 设置订单状态=SUCCESS、trxNo<br/>5. RpTradePaymentOrderDao.update() 更新订单<br/>6. 如果资金流入类型=平台收款 → RpAccountTransactionService.creditToAccount() 入账<br/>7. getMerchantNotifyUrl() 构建商户通知URL<br/>8. RpNotifyService.notifySend() 发送商户通知<br/><br/>**失败场景**:<br/>1. 设置支付记录的bankReturnMsg、status=FAILED<br/>2. RpTradePaymentRecordDao.update() 更新支付记录<br/>3. 查询关联的订单<br/>4. 设置订单状态=FAILED<br/>5. RpTradePaymentOrderDao.update() 更新订单<br/>6. getMerchantNotifyUrl() 构建商户通知URL<br/>7. RpNotifyService.notifySend() 发送商户通知 |
| **异常处理** | 事务回滚（completeSuccessOrder标注@Transactional） |
| **关联方法** | completeSuccessOrder(), completeFailOrder(), getMerchantNotifyUrl(), RpAccountTransactionService.creditToAccount(), RpNotifyService.notifySend() |

---

## ACT-05：订单查询

| 属性 | 内容 |
|------|------|
| **需求编号** | REQ-014 |
| **入口方法** | F2FPayController.orderQuery() |
| **前置条件** | 已知交易流水号trxNo |
| **后置条件** | 返回支付记录JSON |
| **输入数据** | String trxNo |
| **输出数据** | RpTradePaymentRecord (JSON) |
| **数据读取** | rp_trade_payment_record |
| **数据写入** | 无 |
| **执行步骤** | 1. 接收trxNo参数<br/>2. 调用queryService.getRecordByTrxNo(trxNo)<br/>3. 序列化为JSON返回 |
| **异常处理** | 无 |
| **关联方法** | F2FPayController.orderQuery(), RpTradePaymentQueryService.getRecordByTrxNo() |
