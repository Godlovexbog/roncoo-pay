## 1. biz-flow-graph Skill 编写

- [x] 1.1 创建 `.claude/skills/biz-flow-graph/SKILL.md`，包含入口方法定位、递进追踪调用链、收集实体属性和字段读写、提取控制流、生成活动图、产出自检清单的全部步骤
- [x] 1.2 定义活动图输出的 Markdown 模板（章节结构、表格格式），确保结构化数据可被 biz-rule-extraction 解析
- [x] 1.3 用 `com.roncoo.pay.controller.ScanPayController#initPay` 实际验证 biz-flow-graph 的执行

## 2. biz-rule-extraction Skill 编写

- [x] 2.1 创建 `.claude/skills/biz-rule-extraction/SKILL.md`，包含读取活动图数据、决策表规则提取、霍尔逻辑规则提取、状态机规则提取、交叉验证、缺失检测、完整性报告的完整步骤
- [x] 2.2 用阶段1产出的活动图数据实际验证 biz-rule-extraction 的执行

## 3. 端到端验证

- [x] 3.1 完整链路验证: 从入口方法到活动图到规则列表到缺失检测，检查各步骤数据一致性
- [x] 3.2 换一个入口方法（如 `com.roncoo.pay.controller.F2FPayController#initPay`）再做一次端到端，验证 Skill 的通用性
