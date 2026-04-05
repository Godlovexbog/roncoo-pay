# 模块结构扫描报告

## 入口方法

| URL | Controller方法 | 说明 |
|-----|---------------|------|
| POST /f2fPay/doPay | F2FPayController.initPay() | 条码支付入口 |
| GET /f2fPay/order/query | F2FPayController.orderQuery() | 订单查询入口 |

## BFS扫描统计

| 指标 | 值 |
|------|-----|
| 扫描清单初始大小 | 1（M:F2FPayController.initPay） |
| 最终扫描方法数 | 87 |
| 调用边数 | 125 |
| 最大调用深度 | 6 |
| 接口→实现类跳转数 | 3 |

## 模块依赖关系

| 依赖模块 | 依赖类/服务 | 依赖类型 |
|---------|------------|---------|
| roncoo-pay-web-gateway | CnpPayService | 同模块注入 |
| roncoo-pay-service | RpTradePaymentManagerService | 跨模块注入 |
| roncoo-pay-service | RpTradePaymentQueryService | 跨模块注入 |
| user模块 | RpUserPayConfigService | 跨模块调用 |
| user模块 | RpUserPayConfigServiceImpl | 接口实现类 |
| user模块 | RpUserPayConfigDao | DAO层 |
| user模块 | RpUserPayInfoService | 跨模块调用 |
| user模块 | RpUserInfoService | 跨模块调用 |
| user模块 | RpPayWayService | 跨模块调用 |
| user模块 | BuildNoServiceImpl | 编号生成 |
| notify模块 | RpNotifyService | 跨模块调用 |
| account模块 | RpAccountTransactionService | 跨模块调用 |
| common-core | StringUtil, SecurityRatingEnum, PayTypeEnum, PayWayEnum, TradeStatusEnum, FundInfoTypeEnum, DateUtils, PublicStatusEnum, PublicEnum, BaseDaoImpl | 工具/枚举/DAO基类 |
| trade/utils | WeiXinPayUtil, AliPayUtil, MerchantApiUtil, MD5Util | 工具类 |
| common-core/dao | BaseDaoImpl | DAO基类 |
| mybatis-spring | SqlSessionTemplate | 框架注入 |
| spring | Validator, BindingResult | 框架注入 |
| fastjson | JSONObject | 第三方库 |

## 接口→实现类映射

| 接口 | 实现类 | Bean名称 | 调用位置 |
|------|--------|---------|---------|
| RpUserPayConfigService | RpUserPayConfigServiceImpl | rpUserPayConfigService | CnpPayService.checkParamAndGetUserPayConfig |
| RpTradePaymentManagerService | RpTradePaymentManagerServiceImpl | rpTradePaymentManagerService | F2FPayController.initPay |
| BuildNoService | BuildNoServiceImpl | buildNoService | RpTradePaymentManagerServiceImpl.sealF2FRpTradePaymentOrder |

## 方法统计

- Controller方法: 2个 (initPay, orderQuery)
- Service核心方法: 7个 (checkParamAndGetUserPayConfig, f2fPay, getF2FPayResultVo, completeSuccessOrder, completeFailOrder, sealF2FRpTradePaymentOrder, sealRpTradePaymentRecord)
- 支撑方法: 6个 (checkIp, getErrorResponse, getMerchantNotifyUrl, DAO方法)
- 外部工具方法: 4个 (WeiXinPayUtil.micropay, AliPayUtil.tradePay, MerchantApiUtil.getSign/isRightSign)
- 跨模块服务: 8个 (RpUserPayConfigService, RpUserPayInfoService, RpUserInfoService, RpPayWayService, RpNotifyService×2, RpAccountTransactionService, RpTradePaymentQueryService)
- 框架/标准库方法: 15个 (SqlSessionTemplate, Validator, BindingResult, JSONObject, BigDecimal, String, Map, Date, LOG等)

## 代码行数

- F2FPayController: 81行
- CnpPayService: 120行
- RpTradePaymentManagerServiceImpl.f2fPay: 49行 (L165-L214)
- RpTradePaymentManagerServiceImpl.getF2FPayResultVo: 87行 (L223-L310)
- RpTradePaymentManagerServiceImpl.completeSuccessOrder: 27行 (L318-L344)
- RpTradePaymentManagerServiceImpl.completeFailOrder: ~15行 (L390-L401)
- RpTradePaymentManagerServiceImpl.sealF2FRpTradePaymentOrder: ~27行 (L944-L971)
- RpTradePaymentManagerServiceImpl.sealRpTradePaymentRecord: ~50行 (L997-L1047)
- RpTradePaymentManagerServiceImpl.getMerchantNotifyUrl: ~37行 (L346-L383)
- RpUserPayConfigServiceImpl.getByPayKey: 7行 (L403-L409)
- BaseDaoImpl.getBy/insert/update: ~30行
- NetworkUtil.getIpAddress: 46行 (L23-L68)
- StringUtil.isEmpty: ~5行 (L48-L50)
