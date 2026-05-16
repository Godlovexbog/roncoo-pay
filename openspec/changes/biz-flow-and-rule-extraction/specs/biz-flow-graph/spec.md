## ADDED Requirements

### Requirement: 入口方法定位
用户输入 `完整类路径#方法名` 格式的入口方法，系统 SHALL 通过 GitNexus context 查询定位到该方法的所在文件、行号范围、调用关系。

#### Scenario: 标准格式输入
- **WHEN** 用户输入 `com.roncoo.pay.controller.ScanPayController#initPay`
- **THEN** 系统定位到 `roncoo-pay-web-gateway/src/main/java/com/roncoo/pay/controller/ScanPayController.java:85-130`，获取 outgoing.calls 和 incoming.calls

#### Scenario: 方法名在多个文件中存在
- **WHEN** 同名方法在多个 Controller 中存在（如 initPay 在 ScanPayController 和 F2FPayController 都有）
- **THEN** 系统使用 file_path 参数精确匹配用户指定的类路径

### Requirement: 递进追踪调用链
系统 SHALL 从入口方法出发，沿 outgoing.calls 递进追踪最多 3 层调用链，每层使用 gitnexus_context 获取被调用方法的信息。方法体 ≤ 5 行的 getter/setter 方法记录但不深入追踪。

#### Scenario: 三层追踪覆盖
- **WHEN** 入口方法 outgoing.calls 包含 Service 方法，Service 方法又调用 Repository 方法，Repository 调用 DAO
- **THEN** 系统输出 3 层调用树，每层标注方法名、文件路径、行号范围

#### Scenario: 方法体过短跳过深入追踪
- **WHEN** 被调用方法是 getter（如 `getPayType()`，行号范围 3 行）
- **THEN** 系统记录该方法但不继续追踪其 outgoing.calls

### Requirement: 收集实体属性和字段读写
系统 SHALL 通过 Cypher 查询调用链上涉及的所有类的 Property 节点、ACCESSES write/read 关系，以及社区归属。

#### Scenario: 收集实体属性
- **WHEN** 调用链上包含类 `RpTradePaymentOrder`、`ScanPayRequestBo`、`ScanPayResultVo`
- **THEN** 系统查询 `HAS_PROPERTY` 关系，返回所有属性及其类型

#### Scenario: 收集字段读写
- **WHEN** 调用链上方法 `initDirectScanPay` 通过 `ACCESSES` 关系连接到属性 `orderAmount`
- **THEN** 系统区分 read 和 write 操作，分别记录

### Requirement: 提取控制流
系统 SHALL 对调用链上方法体 > 5 行的方法读取源码（使用 Read 工具，指定 startLine/endLine），提取所有 if/else/switch/throw/try-catch/return 语句。

#### Scenario: 提取条件分支
- **WHEN** 源码中包含 `if (StringUtil.isEmpty(payType))` 及其 else 分支
- **THEN** 系统记录决策点: {方法名, 行号, 条件表达式, 每个分支的动作/调用}

#### Scenario: 提取异常处理
- **WHEN** 源码中包含 `try { ... } catch (BizException e) { ... } catch (Exception e) { ... }`
- **THEN** 系统记录 2 条异常决策点: {异常类型, 捕获处理方式, 所在的 try 块范围}

### Requirement: 生成活动图
系统 SHALL 将调用树、决策点、状态变更组合成带泳道的 Mermaid flowchart 活动图，按 Controller/Service/Repository/External 划分泳道，菱形节点表示决策，箭头表示调用流，虚线表示异常路径。

#### Scenario: 活动图包含关键元素
- **WHEN** 所有数据收集完成
- **THEN** 输出的 Mermaid 图包含: 泳道分组、实线调用箭头、菱形决策节点、虚线异常路径、状态变更标注

### Requirement: 产出自检清单
系统 SHALL 产出结构化自检清单，对比数据源数量和图上标注数量，给出每个维度的覆盖率: 调用方法覆盖率、决策分支覆盖率、字段写操作覆盖率、异常出口覆盖率。

#### Scenario: 自检清单验证完整性
- **WHEN** 源码中有 5 个 if 分支、3 个 throw、2 个 catch
- **THEN** 自检清单显示: 决策分支 10/10 (100%)、异常出口 5/5 (100%)

### Requirement: 产出结构化表格数据
系统 SHALL 在 Markdown 输出中内嵌结构化表格（调用树表、决策点表、状态变更表），供 biz-rule-extraction Skill 解析使用。

#### Scenario: 结构化表格可解析
- **WHEN** biz-flow-graph 输出完成
- **THEN** 输出包含至少 3 个带表头的 Markdown 表格: 调用链路表、决策点表、状态变更表
