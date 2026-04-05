# 能力提取报告：F2FPayController.initPay 条码支付流程

## 概览

- 识别出 **1** 个能力（CAP-02 条码支付能力）
- 映射了 **1** 个流程（PROC-01 支付交易流程）
- 分析了 **5** 个需求节点（REQ-010 ~ REQ-014）
- DS-Code代码图：**87** 个节点，**125** 条边（BFS清单驱动，最大深度6）
- 需求图谱：**7** 个节点，**9** 条边
- 双图映射：**5** 条映射关系
- 代码覆盖率 **100%**（孤儿率 **0%**）
- BFS扫描统计：初始清单1个入口 → 最终扫描87个方法 → 125条调用边

## 能力清单

### CAP-02：条码支付能力

- **业务目的**：提供商户通过扫码枪扫描用户付款码完成支付的业务能力。支持微信刷卡支付（MICRO_PAY）和支付宝条码支付（F2F_PAY）两种支付通道。
- **对外API**：
  - `POST /f2fPay/doPay` — 执行条码支付
  - `GET /f2fPay/order/query` — 查询支付记录
- **核心代码**：F2FPayController.initPay(), CnpPayService.checkParamAndGetUserPayConfig(), RpTradePaymentManagerServiceImpl.f2fPay(), getF2FPayResultVo(), completeSuccessOrder(), completeFailOrder()
- **数据实体**：RpTradePaymentOrder, RpTradePaymentRecord, RpUserPayConfig, RpUserPayInfo, RpUserInfo, RpPayWay
- **外部依赖**：
  - 微信刷卡支付API (WeiXinPayUtil.micropay)
  - 支付宝条码支付API (AliPayUtil.tradePay)
  - notify模块 (订单轮询通知、商户异步通知)
  - account模块 (账户入账)
  - user模块 (商户配置、费率配置、编号生成)
  - MyBatis框架 (SqlSessionTemplate)
  - Spring框架 (Validator, BindingResult)
  - fastjson (JSONObject)
  - mybatis-plus (IdWorker雪花算法)
- **业务规则**：
  1. 同一(商户号+商户订单号)只能有一个活跃订单
  2. 已支付成功的订单不能重复支付
  3. 只支持 MICRO_PAY(微信刷卡) 和 F2F_PAY(支付宝条码)
  4. 微信结果未知时发起订单轮询
  5. 支付宝条码支付统一通过订单轮询确认结果
  6. 商户IP白名单校验（安全等级=MD5_IP时）
  7. 请求参数MD5签名验证

## 能力-代码映射矩阵

| 能力 | 核心类 | 核心方法 | 支撑类 | 共享依赖 | 实体 | 代码行数 | 孤儿 |
|------|--------|---------|--------|---------|------|---------|------|
| CAP-02 条码支付 | F2FPayController, RpTradePaymentManagerServiceImpl, CnpPayService, RpUserPayConfigServiceImpl | initPay, checkParamAndGetUserPayConfig, f2fPay, getF2FPayResultVo, completeSuccessOrder, completeFailOrder, sealF2FRpTradePaymentOrder, sealRpTradePaymentRecord, getByPayKey | RpTradePaymentOrderDaoImpl, RpTradePaymentRecordDaoImpl, BaseDaoImpl, CnpPayService.checkIp, getMerchantNotifyUrl | WeiXinPayUtil, AliPayUtil, MerchantApiUtil, RpNotifyService, RpAccountTransactionService, BuildNoServiceImpl, NetworkUtil, StringUtil | RpTradePaymentOrder, RpTradePaymentRecord, RpUserPayConfig, RpUserPayInfo, RpUserInfo, RpPayWay | ~1200 | 0 |

## BFS扫描发现

### 完整调用链（BFS清单驱动）

本次使用BFS清单驱动方法，确保每个方法都被扫描到，不会遗漏：

1. **RpUserPayConfigService.getByPayKey() 完整调用链**（深度5）：
   ```
   checkParamAndGetUserPayConfig()
     └→ RpUserPayConfigService.getByPayKey()         ← 接口
          └→ RpUserPayConfigServiceImpl.getByPayKey() ← 实现类（接口跳转）
               └→ RpUserPayConfigDao.getBy()          ← DAO调用
                    └→ BaseDaoImpl.getBy()            ← 父类方法
                         └→ BaseDaoImpl.getStatement() ← 构建statement
                         └→ SqlSessionTemplate.selectOne() ← MyBatis执行
   ```

2. **CnpPayService.checkIp() 完整调用链**（深度3）：
   ```
   checkParamAndGetUserPayConfig()
     └→ checkIp()
          └→ SecurityRatingEnum.MD5_IP.name()
          └→ NetworkUtil.getIpAddress()
               └→ HttpServletRequest.getHeader()
               └→ HttpServletRequest.getRemoteAddr()
          └→ StringUtil.isEmpty()
   ```

3. **DAO层完整调用链**（深度4）：
   ```
   DAO方法 (selectByMerchantNoAndMerchantOrderNo / insert / update)
     └→ BaseDaoImpl.getBy/insert/update()
          └→ BaseDaoImpl.getStatement()
          └→ SqlSessionTemplate.selectOne/insert/update()
   ```

4. **BuildNoServiceImpl 编号生成链**（深度2）：
   ```
   sealF2FRpTradePaymentOrder()
     └→ BuildNoServiceImpl.buildBankOrderNo()
          └→ IdWorker.getId()              ← mybatis-plus雪花算法
     └→ BuildNoServiceImpl.buildTrxNo()
          └→ IdWorker.getId()
   ```

## 孤儿代码
无真正孤儿代码。仅发现2个预留/未完成代码：
- RoncooPayGoodsDetails（DTO，当前未启用）
- sealRpTradePaymentOrder()（方法返回null，未被调用）

## 架构改进建议

1. **跨模块耦合偏高** — 建议引入Facade封装外部依赖
2. **支付通道分支逻辑重复** — 建议提取策略模式
3. **DAO层继承链过长** — MyBatis标准模式，无需修改，但需明确标注为框架调用
4. **接口→实现类跳转** — 已在全量扫描阶段建立映射表，BFS确保调用链完整
5. **条码支付结果处理不统一** — 建议统一结果确认机制
6. **预留代码未启用** — RoncooPayGoodsDetails参数始终传null
7. **异常处理粒度不够** — 建议细化异常分类
8. **缺少授权码幂等保护** — 建议增加authCode级别防重
