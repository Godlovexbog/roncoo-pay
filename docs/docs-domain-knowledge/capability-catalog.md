# 业务能力目录

## CAP-02：条码支付能力

### 基本信息

| 属性 | 值 |
|------|-----|
| **能力ID** | CAP-02 |
| **能力名称** | 条码支付能力 |
| **英文名称** | Barcode Payment Capability (F2F Payment) |
| **信号评分** | 0.88（归一化后） |
| **验证状态** | ✅ 全部通过（流程层/活动层/方法层） |

### 业务目的

提供商户通过扫码枪扫描用户付款码完成支付的业务能力。支持微信刷卡支付（MICRO_PAY）和支付宝条码支付（F2F_PAY）两种支付通道。

### 输入/输出

- **输入**: `F2FPayRequestBo`
  - payKey: 商户支付KEY（16-32位）
  - authCode: 用户支付授权码（16-20位）
  - productName: 商品名称
  - orderNo: 商户订单号（5-20位）
  - orderPrice: 订单金额
  - orderIp: 下单IP
  - orderDate: 订单日期
  - orderTime: 订单时间
  - payType: 交易类型（MICRO_PAY / F2F_PAY）
  - sign: 数据签名

- **输出**: `F2FPayResultVo`
  - status: 交易状态（SUCCESS / FAILED / WAITING_PAYMENT）
  - trxNo: 交易流水号
  - orderNo: 商户订单号
  - payKey: 支付KEY
  - productName: 产品名称
  - sign: 签名数据
  - remark: 支付备注

### 子需求

| 需求ID | 需求名称 | 目的 | 入口方法 |
|--------|---------|------|---------|
| REQ-010 | 条码支付-参数校验 | 验证商户身份、支付授权码、订单参数合法性 | F2FPayController.initPay() |
| REQ-011 | 条码支付-订单管理 | 查询或创建支付订单，防止重复支付 | RpTradePaymentManagerServiceImpl.f2fPay() |
| REQ-012 | 条码支付-执行支付 | 调用微信micropay或支付宝tradePay完成条码支付 | getF2FPayResultVo() |
| REQ-013 | 条码支付-结果处理 | 根据支付结果更新订单状态、入账、通知商户 | completeSuccessOrder() / completeFailOrder() |
| REQ-014 | 条码支付-订单查询 | 根据交易流水号查询支付记录 | F2FPayController.orderQuery() |

### 对外API

| 接口 | 方法 | 说明 |
|------|------|------|
| POST /f2fPay/doPay | F2FPayController.initPay() | 执行条码支付 |
| GET /f2fPay/order/query | F2FPayController.orderQuery() | 查询支付记录 |

### 依赖能力

| 依赖能力 | 依赖类型 | 说明 |
|---------|---------|------|
| CAP-09 交易查询 | 共享依赖 | 共用 RpTradePaymentQueryService 查询支付记录 |
| 用户鉴权能力 | 语义相似 | REQ-012 与鉴权支付执行功能相似 |

### 被依赖能力

| 被依赖能力 | 依赖类型 | 说明 |
|-----------|---------|------|
| CAP-01 扫码支付 | 语义相似 | REQ-012 ↔ REQ-002 同为支付执行，逻辑相似 |
| CAP-01 扫码支付 | 语义相似 | REQ-013 ↔ REQ-003 同为结果处理，逻辑相似 |

### 代码覆盖

| 指标 | 值 |
|------|-----|
| 核心方法数 | 5 |
| 支撑方法数 | 6 |
| 共享依赖数 | 9 |
| 数据实体数 | 5 |
| 代码行数 | ~350行 |
| 孤儿代码数 | 0 |
| 代码覆盖率 | 100% |

### 成熟度评估

| 维度 | 评分 | 说明 |
|------|------|------|
| 代码完整性 | ⭐⭐⭐⭐⭐ | 所有需求都有对应的方法实现 |
| 事务控制 | ⭐⭐⭐⭐ | completeSuccessOrder有事务，但f2fPay主流程无事务 |
| 异常处理 | ⭐⭐⭐⭐ | Controller层有try-catch，但部分方法缺少细粒度异常 |
| 测试覆盖 | ⭐⭐⭐ | 需补充测试用例（见测试文档） |
| 文档完整性 | ⭐⭐⭐⭐⭐ | 本能力已生成完整的领域知识、设计、测试文档 |
| **综合成熟度** | **⭐⭐⭐⭐** | **4.2/5.0** |
