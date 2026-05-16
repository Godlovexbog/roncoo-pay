## ADDED Requirements

### Requirement: 读取活动图数据
系统 SHALL 读取 biz-flow-graph 产出的 Markdown 文件，解析其中的调用链路表、决策点表、状态变更表，获取结构化数据用于规则提取。

#### Scenario: 解析结构化表格
- **WHEN** biz-flow-graph 产出包含 "调用链路" 表格（方法名、层级、文件路径）和 "决策点" 表格（方法、行号、条件、分支动作）
- **THEN** biz-rule-extraction 成功解析表格数据，构建内部规则提取数据集

### Requirement: 决策表规则提取（理论A）
系统 SHALL 从活动图的决策点表格中提取 WHEN-THEN 分支规则，每条决策分支（if/else/switch/throw）生成一条对应的业务规则。保证提取的规则数量等于源码中的决策分支数量（分支完备性）。

#### Scenario: if-else 分支规则提取
- **WHEN** 决策点表格包含 `initPay:93 | if (payType 为空) → initNonDirectScanPay` 和 `initPay:96 | else → initDirectScanPay`
- **THEN** 系统生成 2 条规则: R1: "WHEN payType 为空 THEN 走非直连扫码"、R2: "WHEN payType 非空 THEN 走直连扫码"

#### Scenario: 异常路径规则提取
- **WHEN** 决策点表格包含 `try-catch BizException → 返回异常页面` 和 `try-catch Exception → 返回系统异常页面`
- **THEN** 系统生成 2 条异常规则: "WHEN 发生 BizException THEN 返回业务异常页面"、"WHEN 发生其他 Exception THEN 返回系统异常页面"

#### Scenario: 规则数量与分支数量一致
- **WHEN** 源码中总计有 N 个决策分支
- **THEN** 决策表规则条目数 = N，覆盖率达到 100%

### Requirement: 霍尔逻辑规则提取（理论B）
系统 SHALL 从活动图的状态变更表中提取 {前置条件}→方法→{后置条件} 规则，每个字段写操作的读依赖构成前置条件，写结果构成后置条件（状态完备性）。

#### Scenario: 状态前后条件提取
- **WHEN** 活动图显示 `initDirectScanPay` 方法读 `rpUserPayConfig`、`scanPayRequestBo`，写 `ScanPayResultVo.codeUrl`
- **THEN** 系统生成规则: "{rpUserPayConfig 已加载 AND request 已验证} initDirectScanPay() {ScanPayResultVo.codeUrl 已生成}"

#### Scenario: 状态规则覆盖所有写操作
- **WHEN** ACCESSES write 总数 = M
- **THEN** 霍尔规则数量 = M，覆盖率达到 100%

### Requirement: 状态机规则提取（理论C）
系统 SHALL 从决策规则和状态变更中推断实体状态字段的合法值集合和跃迁路径，构建 Mermaid stateDiagram，检查所有状态值的可达性（跃迁完备性）。

#### Scenario: 状态值枚举
- **WHEN** 调用链中包含对 `order.status` 的 write 操作，且决策规则中引用了状态值 `CREATED, PAYING, SUCCESS`
- **THEN** 系统枚举所有状态值，构建状态值集合

#### Scenario: 跃迁路径构建
- **WHEN** `initPay` 写 status=PAYING，`completeScanPay` 写 status=SUCCESS
- **THEN** 系统生成跃迁: `CREATED → PAYING: initPay()`, `PAYING → SUCCESS: completeScanPay()`

#### Scenario: 死状态检测
- **WHEN** 存在状态值 `CLOSED` 但没有任何方法写入该状态
- **THEN** 系统在缺失检测中报告 "无法达成的状态 CLOSED"

### Requirement: 交叉验证
系统 SHALL 计算三理论规则的交集和差异，生成交叉验证矩阵，标注每条规则的置信度（高/中/待验证），列出冲突规则和不一致项。

#### Scenario: 高置信度规则
- **WHEN** 规则 "WHEN payWay=WEIXIN THEN 返回微信扫码页" 被决策表(A)、霍尔逻辑(B)、状态机(C) 三个理论共同确认
- **THEN** 该规则置信度标记为 "高"

#### Scenario: 独有规则标记
- **WHEN** 某条状态规则仅被霍尔逻辑(B) 发现，未被决策表(A) 和状态机(C) 覆盖
- **THEN** 该规则标记为 "仅 B 发现 — 待验证"

#### Scenario: 一致性计算
- **WHEN** 三理论总规则数为 T，交集规则数为 I
- **THEN** 一致性 = I / T，输出百分比

### Requirement: 缺失检测
系统 SHALL 基于规则集和活动图结构，检测以下缺失: 未覆盖的条件组合、缺失异常处理、状态跃迁缺失/死状态、隐式 fallthrough、逆向操作缺失、空值风险。

#### Scenario: 未覆盖分支检测
- **WHEN** payWay 枚举值包含 WEIXIN, ALIPAY, UNIONPAY，但决策表中仅处理了 WEIXIN, ALIPAY
- **THEN** 系统报告: "未覆盖分支: payWay=UNIONPAY 时无明确处理逻辑 (严重度 HIGH)"

#### Scenario: 空值风险检测
- **WHEN** 方法中先 ACCESSES read 某属性，紧接 ACCESSES write 同一属性，中间无 null 检查
- **THEN** 系统报告: "空值风险: {方法名} 中读取 {属性} 后直接写入，未检查 null (严重度 HIGH)"

### Requirement: 完整性量化报告
系统 SHALL 输出完整性量化报告，包含: 决策分支覆盖率、状态写操作覆盖率、状态跃迁覆盖率、交叉验证一致性、高置信度规则占比、检测到的缺失项统计。

#### Scenario: 输出量化指标
- **WHEN** 规则提取和验证全部完成
- **THEN** 报告包含至少 6 个量化指标，每个指标有明确的计算公式和数据来源
