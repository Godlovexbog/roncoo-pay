# 架构改进建议

## 基于 F2FPayController.initPay 调用链分析

### 1. 跨模块耦合偏高

**问题**: `RpTradePaymentManagerServiceImpl` 直接依赖了5个外部模块的服务：
- user模块: RpUserPayConfigService, RpUserPayInfoService, RpUserInfoService, RpPayWayService
- notify模块: RpNotifyService
- account模块: RpAccountTransactionService

**建议**: 引入领域服务层，将外部依赖封装为内部Facade接口，降低trade模块对外的耦合度。

### 2. 支付通道分支逻辑重复

**问题**: `getF2FPayResultVo()` 和 `getScanPayResultVo()` 中有高度相似的微信/支付宝分支判断逻辑：
```java
if (PayWayEnum.WEIXIN.name().equals(payWayCode)) { ... }
else if (PayWayEnum.ALIPAY.name().equals(payWayCode)) { ... }
```

**建议**: 提取策略模式，将微信和支付宝的支付逻辑封装为独立的Strategy实现类，通过策略工厂根据payWayCode选择对应实现。

### 3. DAO层继承链过长

**问题**: DAO调用链深度达4层：
```
RpTradePaymentOrderDao.selectByMerchantNoAndMerchantOrderNo()
  → BaseDaoImpl.getBy()
    → BaseDaoImpl.getStatement()
      → SqlSessionTemplate.selectOne()
```

**建议**: 这是MyBatis通用DAO的标准模式，无需修改。但在可视化图中应明确标注为框架调用链，避免误判为业务逻辑。

### 4. 接口→实现类跳转缺失风险

**问题**: 之前分析遗漏了 `RpUserPayConfigService.getByPayKey()` 到 `RpUserPayConfigServiceImpl.getByPayKey()` 的跳转，导致DAO层调用链缺失。

**建议**: 在全量扫描阶段建立完整的接口→实现类映射表，确保调用链追踪时能正确跳转。

### 5. 条码支付结果处理不统一

**问题**: 微信条码支付在 `getF2FPayResultVo()` 中直接处理结果并调用 `completeSuccessOrder/completeFailOrder`，而支付宝条码支付统一走订单轮询确认结果。两种通道的结果确认机制不一致。

**建议**: 统一改为通过订单轮询确认结果，或在方法注释中明确说明差异原因。

### 6. RoncooPayGoodsDetails 预留代码未启用

**问题**: `getF2FPayResultVo()` 方法接收 `roncooPayGoodsDetailses` 参数但调用方始终传null，支付宝 `tradePay` 方法接收了该参数但实际也未使用。

**建议**: 如果短期内不启用商品详情功能，建议移除该参数以减少方法签名复杂度。

### 7. 异常处理粒度不够

**问题**: `F2FPayController.initPay()` 只捕获了 `BizException` 和 `Exception` 两类异常，没有区分更细粒度的异常类型（如参数异常、配置异常、银行调用异常）。

**建议**: 在Controller层增加更细粒度的异常处理，返回更明确的错误信息给前端。

### 8. 缺少授权码幂等保护

**问题**: 条码支付是实时扣款，如果商户重复提交相同的authCode，可能导致重复扣款。当前只在订单层面做了幂等（同一商户订单号不能重复支付），但没有在授权码层面做幂等。

**建议**: 增加授权码级别的幂等控制，或在调用微信micropay前检查该authCode是否已在处理中。
