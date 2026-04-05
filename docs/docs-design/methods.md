# 方法详细描述：条码支付流程

## 核心方法 (Core)

### F2FPayController.initPay()

| 属性 | 内容 |
|------|------|
| **分类** | entry |
| **签名** | `public String initPay(@ModelAttribute F2FPayRequestBo f2FPayRequestBo, BindingResult bindingResult, HttpServletRequest httpServletRequest, ModelMap modelMap)` |
| **URL映射** | POST /f2fPay/doPay |
| **参数** | f2FPayRequestBo: 条码支付请求参数<br/>bindingResult: 参数校验结果<br/>httpServletRequest: HTTP请求对象<br/>modelMap: 视图数据模型 |
| **返回值** | String — 视图名称 ("/f2fAffirmPay" 或 "exception/exception") |
| **调用链** | → CnpPayService.checkParamAndGetUserPayConfig()<br/>→ RpTradePaymentManagerService.f2fPay()<br/>→ modelMap.put("result", f2FPayResultVo) |
| **数据读写** | 无直接数据库操作 |
| **事务属性** | 无 |
| **异常处理** | try-catch(BizException): 记录业务异常，返回异常页面<br/>try-catch(Exception): 记录系统异常，返回系统异常页面 |
| **代码行号** | L52-L73 |

---

### RpTradePaymentManagerServiceImpl.f2fPay()

| 属性 | 内容 |
|------|------|
| **分类** | core |
| **签名** | `public F2FPayResultVo f2fPay(RpUserPayConfig rpUserPayConfig, F2FPayRequestBo f2FPayRequestBo)` |
| **参数** | rpUserPayConfig: 商户支付配置（已由CnpPayService校验）<br/>f2FPayRequestBo: 条码支付请求参数 |
| **返回值** | F2FPayResultVo — 含签名的支付结果 |
| **调用链** | → rpPayWayService.getByPayWayTypeCode()<br/>→ rpUserInfoService.getDataByMerchentNo()<br/>→ rpTradePaymentOrderDao.selectByMerchantNoAndMerchantOrderNo()<br/>→ sealF2FRpTradePaymentOrder() + insert()<br/>→ getF2FPayResultVo() |
| **数据读写** | READ[rp_trade_payment_order, rp_user_info, rp_pay_way]<br/>WRITE[rp_trade_payment_order (insert)] |
| **事务属性** | 无 |
| **异常抛出** | PayBizException(交易类型错误)<br/>UserBizException(配置异常/商户不存在)<br/>TradeBizException(订单异常) |
| **代码行号** | L165-L214 (49行) |

---

### RpTradePaymentManagerServiceImpl.getF2FPayResultVo()

| 属性 | 内容 |
|------|------|
| **分类** | core |
| **签名** | `private F2FPayResultVo getF2FPayResultVo(RpTradePaymentOrder, RpPayWay, String payKey, String merchantPaySecret, String authCode, List<RoncooPayGoodsDetails>)` |
| **参数** | rpTradePaymentOrder: 支付订单<br/>payWay: 商户支付通道配置<br/>payKey: 商户支付KEY<br/>merchantPaySecret: 商户签名密钥<br/>authCode: 用户支付授权码<br/>roncooPayGoodsDetailses: 商品详情列表（当前传null，预留功能） |
| **返回值** | F2FPayResultVo — 含签名的支付结果 |
| **调用链** | → sealRpTradePaymentRecord() + insert()<br/>→ WeiXinPayUtil.micropay() 或 AliPayUtil.tradePay()<br/>→ completeSuccessOrder() / completeFailOrder()<br/>→ MerchantApiUtil.getSign() |
| **数据读写** | WRITE[rp_trade_payment_record (insert)]<br/>READ[rp_user_pay_info] |
| **事务属性** | 无 |
| **异常抛出** | UserBizException(商户支付渠道配置不存在)<br/>TradeBizException(支付方式错误) |
| **代码行号** | L223-L310 (87行) |

---

### RpTradePaymentManagerServiceImpl.completeSuccessOrder()

| 属性 | 内容 |
|------|------|
| **分类** | core |
| **签名** | `void completeSuccessOrder(RpTradePaymentRecord rpTradePaymentRecord, String bankTrxNo, Date timeEnd, String bankReturnMsg)` |
| **注解** | `@Transactional(rollbackFor = Exception.class)` |
| **参数** | rpTradePaymentRecord: 支付记录<br/>bankTrxNo: 银行交易流水号<br/>timeEnd: 支付成功时间<br/>bankReturnMsg: 银行返回消息 |
| **返回值** | void |
| **调用链** | → rpTradePaymentRecordDao.update()<br/>→ rpTradePaymentOrderDao.update()<br/>→ rpAccountTransactionService.creditToAccount()<br/>→ getMerchantNotifyUrl()<br/>→ rpNotifyService.notifySend() |
| **数据读写** | WRITE[rp_trade_payment_record, rp_trade_payment_order] |
| **事务属性** | @Transactional(rollbackFor = Exception.class) |
| **代码行号** | L318-L344 (27行) |

---

### RpTradePaymentManagerServiceImpl.completeFailOrder()

| 属性 | 内容 |
|------|------|
| **分类** | core |
| **签名** | `private void completeFailOrder(RpTradePaymentRecord rpTradePaymentRecord, String bankReturnMsg)` |
| **参数** | rpTradePaymentRecord: 支付记录<br/>bankReturnMsg: 银行返回的失败原因 |
| **返回值** | void |
| **调用链** | → rpTradePaymentRecordDao.update()<br/>→ rpTradePaymentOrderDao.update()<br/>→ getMerchantNotifyUrl()<br/>→ rpNotifyService.notifySend() |
| **数据读写** | WRITE[rp_trade_payment_record, rp_trade_payment_order] |
| **事务属性** | 无（在调用方getF2FPayResultVo的事务上下文中执行） |
| **代码行号** | L390-L400 (~15行) |

---

### CnpPayService.checkParamAndGetUserPayConfig()

| 属性 | 内容 |
|------|------|
| **分类** | core |
| **签名** | `public RpUserPayConfig checkParamAndGetUserPayConfig(Object object, BindingResult bindingResult, HttpServletRequest httpServletRequest)` |
| **参数** | object: 请求参数对象<br/>bindingResult: 校验结果<br/>httpServletRequest: HTTP请求对象 |
| **返回值** | RpUserPayConfig — 商户支付配置 |
| **调用链** | → Validator.validate()<br/>→ BindingResult.hasErrors()<br/>→ getErrorResponse()<br/>→ JSONObject.toJSON()<br/>→ JSONObject.parseObject()<br/>→ RpUserPayConfigService.getByPayKey()<br/>→ checkIp()<br/>→ MerchantApiUtil.isRightSign() |
| **数据读写** | READ[rp_user_pay_config] |
| **事务属性** | 无 |
| **异常抛出** | PayBizException(请求参数异常/用户异常)<br/>TradeBizException(订单签名异常/非法IP) |
| **代码行号** | L72-L102 |

---

### RpTradePaymentManagerServiceImpl.sealF2FRpTradePaymentOrder()

| 属性 | 内容 |
|------|------|
| **分类** | core |
| **签名** | `private RpTradePaymentOrder sealF2FRpTradePaymentOrder(RpUserPayConfig, F2FPayRequestBo, RpUserInfo, RpPayWay)` |
| **职责** | 封装条码支付订单实体，设置商户信息、订单金额、状态等字段 |
| **调用链** | → BuildNoServiceImpl.buildBankOrderNo()<br/>→ BuildNoServiceImpl.buildTrxNo()<br/>→ DateUtils.parseDate() |
| **数据读写** | 创建 RpTradePaymentOrder 对象（不直接写库） |

---

### RpTradePaymentManagerServiceImpl.sealRpTradePaymentRecord()

| 属性 | 内容 |
|------|------|
| **分类** | core |
| **签名** | `private RpTradePaymentRecord sealRpTradePaymentRecord(...)` |
| **职责** | 封装支付记录实体，生成trxNo/bankOrderNo，计算平台收入/成本/利润 |
| **调用链** | → BuildNoServiceImpl.buildTrxNo()<br/>→ BuildNoServiceImpl.buildBankOrderNo()<br/>→ BigDecimal.valueOf() |
| **数据读写** | 创建 RpTradePaymentRecord 对象（不直接写库） |

---

## 支撑方法 (Supporting)

### RpTradePaymentManagerServiceImpl.getMerchantNotifyUrl()

| 属性 | 内容 |
|------|------|
| **分类** | supporting |
| **签名** | `private String getMerchantNotifyUrl(RpTradePaymentRecord, RpTradePaymentOrder, String sourceUrl, TradeStatusEnum)` |
| **职责** | 构建商户异步通知URL，拼接参数并计算MD5签名 |
| **调用链** | → RpUserPayConfigService.getByUserNo()<br/>→ DateUtils.formatDate()<br/>→ MerchantApiUtil.getParamStr()<br/>→ MerchantApiUtil.getSign() |
| **数据读取** | RpUserPayConfig (通过merchantNo查询) |

---

### CnpPayService.checkIp()

| 属性 | 内容 |
|------|------|
| **分类** | supporting |
| **签名** | `public void checkIp(RpUserPayConfig, HttpServletRequest)` |
| **职责** | IP白名单校验 |
| **调用链** | → SecurityRatingEnum.MD5_IP.name()<br/>→ NetworkUtil.getIpAddress()<br/>→ StringUtil.isEmpty() |

---

### CnpPayService.getErrorResponse()

| 属性 | 内容 |
|------|------|
| **分类** | supporting |
| **签名** | `public String getErrorResponse(BindingResult)` |
| **职责** | 获取校验错误信息 |
| **调用链** | → BindingResult.getAllErrors() |

---

## 共享方法 (Shared)

| 方法 | 所属模块 | 用途 | 被调用方 |
|------|---------|------|---------|
| WeiXinPayUtil.micropay() | trade/utils | 微信刷卡支付API | getF2FPayResultVo |
| AliPayUtil.tradePay() | trade/utils/alipay | 支付宝条码支付API | getF2FPayResultVo |
| MerchantApiUtil.getSign() | trade/utils | 商户API MD5签名 | getF2FPayResultVo, getMerchantNotifyUrl |
| MerchantApiUtil.isRightSign() | trade/utils | 商户签名验证 | checkParamAndGetUserPayConfig |
| MerchantApiUtil.getParamStr() | trade/utils | 参数字符串拼接 | getMerchantNotifyUrl |
| MD5Util.encode() | trade/utils | MD5加密 | getSign, isRightSign |
| RpNotifyService.orderSend() | notify模块 | 发起订单轮询通知 | getF2FPayResultVo |
| RpNotifyService.notifySend() | notify模块 | 发送商户异步通知 | completeSuccessOrder, completeFailOrder |
| RpAccountTransactionService.creditToAccount() | account模块 | 账户入账（加款） | completeSuccessOrder |
| RpUserPayConfigService.getByPayKey() | user模块 | 通过payKey获取支付配置 | checkParamAndGetUserPayConfig |
| RpUserPayConfigServiceImpl.getByPayKey() | user模块 | 实现类：查询配置 | getByPayKey接口 |
| RpUserPayConfigService.getByUserNo() | user模块 | 通过商户号获取支付配置 | getMerchantNotifyUrl |
| RpUserInfoService.getDataByMerchentNo() | user模块 | 获取商户信息 | f2fPay |
| RpPayWayService.getByPayWayTypeCode() | user模块 | 获取费率配置 | f2fPay |
| BuildNoServiceImpl.buildBankOrderNo() | user模块 | 生成银行订单号 | sealF2FRpTradePaymentOrder, sealRpTradePaymentRecord |
| BuildNoServiceImpl.buildTrxNo() | user模块 | 生成平台交易流水号 | sealF2FRpTradePaymentOrder, sealRpTradePaymentRecord |
| NetworkUtil.getIpAddress() | gateway | 获取客户端IP | checkIp |
| StringUtil.isEmpty() | common-core | 字符串判空 | checkIp |
| DateUtils.parseDate() | common-core | 日期解析 | sealF2FRpTradePaymentOrder |
| DateUtils.formatDate() | common-core | 日期格式化 | getMerchantNotifyUrl |
