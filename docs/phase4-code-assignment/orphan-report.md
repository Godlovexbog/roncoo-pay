# 孤儿代码报告

## 分析范围
F2FPayController.initPay() 调用链及其关联的DS-Code图节点

## 孤儿代码清单

| 代码 | 类型 | 位置 | 说明 | 建议 |
|------|------|------|------|------|
| RoncooPayGoodsDetails | DTO | trade/entity | 商品详情DTO，f2fPay方法参数中有引用但未实际使用(传null) | 标记为预留功能，当前未启用 |
| sealRpTradePaymentOrder() | 方法 | RpTradePaymentManagerServiceImpl | 通用订单封装方法，返回null，未被调用 | 可能是未完成的代码，建议删除或实现 |
| sealWeixinPerPay() | 方法 | RpTradePaymentManagerServiceImpl | 微信预支付实体封装，被getScanPayResultVo调用但未被getF2FPayResultVo调用 | 属于扫码支付能力(CAP-01)，非本能力孤儿 |

## 结论

在 F2FPayController.initPay 的调用链范围内，**未发现真正的孤儿代码**。所有方法都有明确的业务归属。

唯一值得关注的是：
1. `RoncooPayGoodsDetails` 在f2fPay中被声明为参数但实际调用时传入null，说明商品详情功能尚未启用，属于预留代码
2. `sealRpTradePaymentOrder()` 方法返回null且未被任何方法调用，可能是未完成的代码
