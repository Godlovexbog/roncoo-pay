# 流程图：条码支付流程（F2FPayController.initPay）

## 需求→代码映射矩阵

| 需求ID | 业务目的 (Purpose) | 输入 (Input) | 输出 (Output) | 入口方法 | 调用链方法数 |
|--------|-------------------|-------------|--------------|---------|-------------|
| REQ-010 | 验证商户身份、支付授权码、订单参数合法性 | F2FPayRequestBo(payKey, authCode, productName, orderNo, orderPrice, orderIp) | RpUserPayConfig(商户支付配置) | F2FPayController.initPay | 13 |
| REQ-011 | 查询或创建支付订单，防止重复支付 | merchantNo, orderNo, orderPrice | RpTradePaymentOrder(已存在或新建) | RpTradePaymentManagerServiceImpl.f2fPay | 12 |
| REQ-012 | 调用微信micropay或支付宝tradePay完成条码支付 | bankOrderNo, authCode, productName, orderAmount, orderIp | 支付结果(成功/失败/未知) | RpTradePaymentManagerServiceImpl.getF2FPayResultVo | 13 |
| REQ-013 | 根据支付结果更新订单状态、入账、通知商户 | 支付结果(成功/失败/未知) | F2FPayResultVo(含签名) | RpTradePaymentManagerServiceImpl.completeSuccessOrder | 14 |
| REQ-014 | 根据交易流水号查询支付记录 | trxNo | RpTradePaymentRecord (JSON) | F2FPayController.orderQuery | 2 |

---

## 主流程图

```mermaid
flowchart TD
    Start["REQ-010: 参数校验\nPOST /f2fPay/doPay\nF2FPayController.initPay"] --> A["checkParamAndGetUserPayConfig\n验证商户身份和参数合法性"]
    
    A --> B{"Validator.validate\n参数校验通过?"}
    B -->|否| B_ERR1["PayBizException\n请求参数异常"]
    B -->|是| C{"payKey对应的商户存在?\nRpUserPayConfigService.getByPayKey"}
    C -->|否| B_ERR2["PayBizException\n用户异常"]
    C -->|是| D{"IP白名单校验通过?\ncheckIp"}
    D -->|否| B_ERR3["TradeBizException\n非法IP请求"]
    D -->|是| E{"MD5签名验证通过?\nMerchantApiUtil.isRightSign"}
    E -->|否| B_ERR4["TradeBizException\n订单签名异常"]
    E -->|是| F["REQ-011: 订单管理\nf2fPay 查询或创建订单"]
    
    F --> G{"payType是否为\nMICRO_PAY或F2F_PAY?"}
    G -->|否| F_ERR1["PayBizException\n交易类型有误"]
    G -->|是| H["获取费率配置\nRpPayWayService.getByPayWayTypeCode"]
    H --> I["获取商户信息\nRpUserInfoService.getDataByMerchentNo"]
    I --> J{"订单是否存在?\nselectByMerchantNoAndMerchantOrderNo\nREAD rp_trade_payment_order"}
    J -->|不存在| K["sealF2FRpTradePaymentOrder\n创建订单实体\n生成bankOrderNo/trxNo"]
    K --> L["RpTradePaymentOrderDao.insert\nWRITE rp_trade_payment_order"]
    L --> M
    J -->|已存在| M{"订单金额一致?\norderAmount.compareTo"}
    M -->|否| F_ERR2["TradeBizException\n错误的订单"]
    M -->|是| N{"订单状态=SUCCESS?\nTradeStatusEnum.name"}
    N -->|是| F_ERR3["TradeBizException\n订单已支付成功"]
    N -->|否| O["REQ-012: 执行支付\ngetF2FPayResultVo 执行支付"]
    
    O --> P["设置订单支付类型和通道\nRpTradePaymentOrderDao.update\nWRITE rp_trade_payment_order"]
    P --> Q["sealRpTradePaymentRecord\n封装支付记录\n生成trxNo/bankOrderNo\n计算platIncome/platCost/platProfit"]
    Q --> R["RpTradePaymentRecordDao.insert\nWRITE rp_trade_payment_record"]
    R --> S{"支付方式?\nPayWayEnum.name"}
    
    S -->|WEIXIN| T["WeiXinPayUtil.micropay\n调用微信刷卡支付API\nhttps://api.mch.weixin.qq.com/pay/micropay"]
    T --> U{"微信返回结果?"}
    U -->|为空| V["RpNotifyService.orderSend\n发起订单轮询\nnotify模块 → JmsTemplate.send\nORDER_NOTIFY_QUEUE"]
    U -->|有结果| W{"验签通过?\nwxResultMap.verify=YES"}
    W -->|否| X["completeFailOrder\n签名校验失败"]
    W -->|是| Y{"通讯成功且业务成功?\nreturn_code=SUCCESS\nresult_code=SUCCESS"}
    Y -->|是| Z["REQ-013: 结果处理\ncompleteSuccessOrder 处理成功"]
    Y -->|否| AA{"错误码类型?"}
    AA -->|BANKERROR/USERPAYING/SYSTEMERROR| AB["RpNotifyService.orderSend\n结果未知 发起轮询\nJmsTemplate.send → ORDER_NOTIFY_QUEUE"]
    AA -->|其他| AC["completeFailOrder\n业务失败"]
    
    S -->|ALIPAY| AD["AliPayUtil.tradePay\n调用支付宝条码支付API\nAlipayTradePayRequest"]
    AD --> AE["RpNotifyService.orderSend\n统一发起订单轮询\nJmsTemplate.send → ORDER_NOTIFY_QUEUE"]
    
    Z --> AF["返回 /f2fAffirmPay 视图\nF2FPayResultVo含签名"]
    X --> AF
    AC --> AF
    V --> AF
    AB --> AF
    AE --> AF
    
    B_ERR1 --> End["返回异常页面\nexception/exception"]
    B_ERR2 --> End
    B_ERR3 --> End
    B_ERR4 --> End
    F_ERR1 --> End
    F_ERR2 --> End
    F_ERR3 --> End
    AF --> End
```

## 状态流转图

```mermaid
stateDiagram-v2
    [*] --> WAITING_PAYMENT: sealF2FRpTradePaymentOrder\n创建订单\nstatus=WAITING_PAYMENT
    
    WAITING_PAYMENT --> SUCCESS: completeSuccessOrder\n支付成功\nstatus=SUCCESS\n@事务: rollbackFor=Exception.class
    WAITING_PAYMENT --> FAILED: completeFailOrder\n支付失败\nstatus=FAILED
    WAITING_PAYMENT --> WAITING_PAYMENT: 结果未知\nBANKERROR/USERPAYING/SYSTEMERROR\n发起RpNotifyService.orderSend轮询\nJmsTemplate.send → ORDER_NOTIFY_QUEUE
    
    SUCCESS --> [*]: 终结状态\n触发: 更新记录+更新订单+入账(平台收款时)+通知商户
    FAILED --> [*]: 终结状态\n触发: 更新记录+更新订单+通知商户
    
    note right of WAITING_PAYMENT
        初始状态
        等待上游支付结果
        数据写入: rp_trade_payment_order, rp_trade_payment_record
    end note
    
    note right of SUCCESS
        终结状态
        事务: @Transactional(rollbackFor=Exception.class)
        数据写入: rp_trade_payment_record.status=SUCCESS
        数据写入: rp_trade_payment_order.status=SUCCESS, trxNo
        账户操作: RpAccountTransactionService.creditToAccount (仅平台收款)
        通知操作: RpNotifyService.notifySend → JmsTemplate.send\n发送到 MERCHANT_NOTIFY_QUEUE
    end note
    
    note right of FAILED
        终结状态
        数据写入: rp_trade_payment_record.status=FAILED
        数据写入: rp_trade_payment_order.status=FAILED
        通知操作: RpNotifyService.notifySend → JmsTemplate.send\n发送到 MERCHANT_NOTIFY_QUEUE
    end note
```

## 子流程：参数校验流程（REQ-010）

```mermaid
flowchart TD
    A["checkParamAndGetUserPayConfig\n验证商户身份、支付授权码、订单参数合法性"] --> B["Validator.validate\n参数校验(object, bindingResult)"]
    B --> C{"bindingResult.hasErrors?\n参数校验失败?"}
    C -->|是| D["getErrorResponse\n拼凑错误信息"]
    D --> E["抛出 PayBizException\n请求参数异常"]
    C -->|否| F["JSONObject.toJSON\n序列化请求对象"]
    F --> G["JSONObject.parseObject\n反序列化为Map"]
    G --> H["获取payKey参数\nMap.get(payKey)"]
    H --> I["RpUserPayConfigService.getByPayKey\nREAD rp_user_pay_config\n查询商户支付配置"]
    I --> J{"rpUserPayConfig==null?\n商户配置不存在?"}
    J -->|是| K["抛出 PayBizException\n用户异常"]
    J -->|否| L["checkIp\nIP白名单校验"]
    L --> M{"SecurityRatingEnum==MD5_IP?\n安全等级为MD5_IP?"}
    M -->|是| N{"merchantServerIp.indexOf(ip)<0?\nIP不在白名单中?"}
    N -->|是| O["抛出 TradeBizException\n非法IP请求"]
    N -->|否| P
    M -->|否| P["MerchantApiUtil.isRightSign\nMD5签名验证"]
    P --> Q{"签名正确?\nMD5(排序参数+paySecret)==sign"}
    Q -->|否| R["抛出 TradeBizException\n订单签名异常"]
    Q -->|是| S["返回 RpUserPayConfig\n输出: 商户支付配置"]
```

## 子流程：支付结果判断逻辑（REQ-012）

```mermaid
flowchart LR
    A["上游API返回\n微信micropay/支付宝tradePay"] --> B{"返回为空?\nwxResultMap==null"}
    B -->|是| C["结果未知\n发起RpNotifyService.orderSend轮询\nJmsTemplate.send → ORDER_NOTIFY_QUEUE"]
    B -->|否| D{"验签通过?\nverify=YES"}
    D -->|否| E["支付失败\n签名校验失败\ncompleteFailOrder"]
    D -->|是| F{"通讯成功?\nreturn_code=SUCCESS"}
    F -->|否| G["支付失败\n通讯失败\ncompleteFailOrder"]
    F -->|是| H{"业务成功?\nresult_code=SUCCESS"}
    H -->|是| I["支付成功\ncompleteSuccessOrder\n更新状态+入账+通知"]
    H -->|否| J{"错误码类型?"}
    J -->|BANKERROR_USERPAYING_SYSTEMERROR| K["结果未知\n发起RpNotifyService.orderSend轮询\nJmsTemplate.send → ORDER_NOTIFY_QUEUE"]
    J -->|其他| L["支付失败\n业务错误\ncompleteFailOrder"]
```

## 子流程：结果处理流程（REQ-013）

```mermaid
flowchart TD
    A["REQ-013: 结果处理\n根据支付结果更新订单状态、入账、通知商户\n输入: 支付结果(成功/失败/未知)"] --> B{"支付结果?\nTradeStatusEnum"}
    
    B -->|SUCCESS| C["completeSuccessOrder\n处理成功订单\n@Transactional(rollbackFor=Exception.class)"]
    C --> D["设置支付记录\npaySuccessTime=当前时间\nbankTrxNo=transaction_id\nstatus=SUCCESS"]
    D --> E["RpTradePaymentRecordDao.update\nWRITE rp_trade_payment_record"]
    E --> F["查询关联订单\nselectByMerchantNoAndMerchantOrderNo\nREAD rp_trade_payment_order"]
    F --> G["设置订单状态\nstatus=SUCCESS\ntrxNo=平台流水号"]
    G --> H["RpTradePaymentOrderDao.update\nWRITE rp_trade_payment_order"]
    H --> I{"资金流入类型?\nFundInfoTypeEnum.name"}
    I -->|PLAT_RECEIVES 平台收款| J["RpAccountTransactionService.creditToAccount\n账户入账\n金额=orderAmount-platIncome\naccount模块"]
    I -->|MERCHANT_RECEIVES 商户收款| K["跳过入账\n资金直接到商户账户"]
    J --> L
    K --> L["getMerchantNotifyUrl\n构建商户通知URL\nMD5签名"]
    L --> M["RpNotifyService.notifySend\n发送商户异步通知\nnotify模块"]
    M --> N["RpNotifyServiceImpl.notifySend\n创建通知记录 RpNotifyRecord"]
    N --> O["JSONObject.toJSON(record)\n序列化为JSON"]
    O --> P["JmsTemplate.send\n发送消息到ActiveMQ\nMERCHANT_NOTIFY_QUEUE"]
    P --> Q["返回 F2FPayResultVo\n含签名数据"]
    
    B -->|FAILED| R["completeFailOrder\n处理失败订单"]
    R --> S["设置支付记录\nbankReturnMsg=失败原因\nstatus=FAILED"]
    S --> T["RpTradePaymentRecordDao.update\nWRITE rp_trade_payment_record"]
    T --> U["查询关联订单\nselectByMerchantNoAndMerchantOrderNo\nREAD rp_trade_payment_order"]
    U --> V["设置订单状态\nstatus=FAILED"]
    V --> W["RpTradePaymentOrderDao.update\nWRITE rp_trade_payment_order"]
    W --> X["getMerchantNotifyUrl\n构建商户通知URL\nMD5签名"]
    X --> Y["RpNotifyService.notifySend\n发送商户异步通知\nnotify模块"]
    Y --> Z["RpNotifyServiceImpl.notifySend\n创建通知记录 RpNotifyRecord"]
    Z --> AA["JSONObject.toJSON(record)\n序列化为JSON"]
    AA --> AB["JmsTemplate.send\n发送消息到ActiveMQ\nMERCHANT_NOTIFY_QUEUE"]
    AB --> AC["返回 F2FPayResultVo\n含签名数据"]
```

## 子流程：订单查询流程（REQ-014）

```mermaid
flowchart TD
    A["REQ-014: 订单查询\n根据交易流水号查询支付记录\n输入: trxNo"] --> B["F2FPayController.orderQuery\nGET /f2fPay/order/query?trxNo=xxx"]
    B --> C["RpTradePaymentQueryService.getRecordByTrxNo\nREAD rp_trade_payment_record"]
    C --> D["JSONObject.toJSONString\n序列化为JSON"]
    D --> E["返回 JSON\nRpTradePaymentRecord"]
```

## 子流程：通知发送流程（notify模块下钻）

```mermaid
flowchart TD
    A["RpNotifyService.notifySend\n输入: notifyUrl, merchantOrderNo, merchantNo"] --> B["RpNotifyServiceImpl.notifySend\n接口→实现类跳转"]
    B --> C["创建 RpNotifyRecord 对象\nnotifyTimes=0\nlimitNotifyTimes=5\nstatus=CREATED\nurl=notifyUrl\nnotifyType=MERCHANT"]
    C --> D["JSONObject.toJSON(record)\n序列化为JSON字符串"]
    D --> E["JmsTemplate.setDefaultDestinationName\nMqConfig.MERCHANT_NOTIFY_QUEUE"]
    E --> F["JmsTemplate.send\n发送TextMessage到ActiveMQ\nnotify模块异步消费"]
    
    G["RpNotifyService.orderSend\n输入: bankOrderNo"] --> H["RpNotifyServiceImpl.orderSend\n接口→实现类跳转"]
    H --> I["JmsTemplate.setDefaultDestinationName\nMqConfig.ORDER_NOTIFY_QUEUE"]
    I --> J["JmsTemplate.send\n发送TextMessage到ActiveMQ\norder-polling模块异步消费"]
```
