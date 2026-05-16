## Why

从代码中提取业务规则一直靠人工阅读，效率低且无法保证完整性。现在有了 GitNexus 知识图谱（23765 个符号、51079 条关系、300 个执行流），可以在此基础上递进追踪调用链，按需读源码补充控制流，用多理论交叉验证保证提取的完整性和可追溯性。这能让开发、测试、架构师从 Controller 入口一键获取流程全貌和规则清单。

## What Changes

- 新增 Skill: **biz-flow-graph**（业务活动图生成），输入入口方法（如 `com.xx.controller.ScanPayController#initPay`），递进追踪调用链，结合 GitNexus 图数据和源码控制流分析，输出 Mermaid 活动图及自检清单
- 新增 Skill: **biz-rule-extraction**（业务规则提取），从活动图数据出发，使用决策表、霍尔逻辑、状态机三个理论独立提取规则，交叉验证后输出规则列表 + 缺失检测 + 完整性报告
- 两个 Skill 串行协作：biz-flow-graph 产出活动图数据，biz-rule-extraction 消费活动图数据产生规则

## Capabilities

### New Capabilities

- `biz-flow-graph`: 业务活动图生成 — 从 Controller 入口方法递进追踪调用链（3层），通过 GitNexus 查询实体属性和字段读写，按需读源码提取控制流（if/throw/return），生成带泳道的 Mermaid 活动图，附带自检清单验证骨架完整
- `biz-rule-extraction`: 业务规则提取 — 从活动图数据中用决策表提取分支规则、用霍尔逻辑提取状态规则、用状态机提取跃迁规则，三理论交叉验证，输出规则列表 + 缺失检测 + 完整性量化报告

### Modified Capabilities

（无现有能力变更）

## Impact

- 新增文件: `.claude/skills/biz-flow-graph/SKILL.md`、`.claude/skills/biz-rule-extraction/SKILL.md`
- 依赖: GitNexus MCP 工具（context、cypher）、源码文件读取（Read 工具）
- 无代码变更、无 API 变更、纯只读分析操作
