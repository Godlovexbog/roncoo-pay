## Context

roncoo-pay 是一个支付系统，已通过 GitNexus 索引生成了完整的代码知识图谱（23765 个符号、51079 条关系，包含 CALLS、ACCESSES(write/read)、HAS_PROPERTY、MEMBER_OF 等边类型）。本次设计利用该图谱，结合按需源码阅读，构建两个串行协作的 Skill：先画活动图，再提规则。

现有约束:
- 必须只使用 GitNexus 查询 + Read 读源码，不引入其他代码解析工具
- Java 项目，Controller → Service → Repository 分层
- 入口方法格式统一为 `完整类路径#方法名`

## Goals / Non-Goals

**Goals:**
- biz-flow-graph: 生成可验证完整性的活动图（调用树 + 决策点 + 状态变更 + 泳道）
- biz-rule-extraction: 从活动图数据提取业务规则，三理论交叉验证保证完整性
- 两个 Skill 产出结构化 Markdown，可被 Skill 间引用，也可独立阅读

**Non-Goals:**
- 不修改任何业务代码（只读分析）
- 不生成测试代码或自动化测试脚本
- 不支持非 Java 项目（虽然理论适用，但先聚焦）
- 不处理运行时动态行为（反射、动态代理、AOP 织入）
- 不覆盖所有 Controller，按需单入口分析

## Decisions

### D1: 两个 Skill 通过 Markdown 文件传递数据

biz-flow-graph 产出一个结构化的 Markdown 文件，biz-rule-extraction 读取该文件作为输入。

**为什么不是内存传递？** Skill 是独立的 Claude Code 调用，无法共享运行时状态。Markdown 文件既作为数据传递载体，也作为可独立阅读的产物。

**格式约定**: 活动图文件在 Markdown 中内嵌结构化元数据（调用树表格、决策点表格、状态变更表格），规则提取 Skill 通过解析这些表格获取结构化数据。

### D2: 调用链追踪深度固定为 3 层

入口 Controller → L1 Service → L2 Service/Repository → L3 DAO/Utils。

**为什么 3 层？** 实际验证发现在 roncoo-pay 中 3 层覆盖了 Controller → Service → Repository → DAO 的完整链路。超过 3 层通常是工具方法或框架层调用，对业务规则贡献极小。

**跳过策略**: getter/setter/toString 等单行方法不深入追踪，但记录在调用树上。

### D3: 活动图采用 Mermaid flowchart + 结构化表格双格式

Mermaid flowchart 用于可视化，结构化表格（决策点表、状态变更表）用于规则提取阶段解析。

**为什么不只用 Mermaid？** Mermaid 图是视觉载体，机器难以从中提取结构化数据。表格同时满足人类阅读和规则提取解析。

### D4: 源码读取只针对方法体 > 5 行的方法

**为什么 5 行？** 经验值：< 5 行的方法通常是 getter/setter/简单委托，控制流简单到可以忽略。> 5 行才可能有 if/throw/复杂决策。

### D5: 三个理论的分工

| 理论 | 输入来源 | 产出 | 保证的维度 |
|------|---------|------|-----------|
| 决策表 | 源码控制流 (if/switch/throw/return) | WHEN-THEN 分支规则 | 分支完备性 |
| 霍尔逻辑 | GitNexus ACCESSES read/write | {前置}→方法→{后置} 规则 | 状态完备性 |
| 状态机 | 前两者综合 | 实体状态跃迁图 | 跃迁完备性 |

**为什么是这三个？** 从 10 个候选理论中筛选，标准是: (1) 能在 GitNexus 数据上落地 (2) 每个保证不同的完整性维度 (3) 三者交叉验证有意义。

被排除的理论:
- 因果图: 与决策表的条件-动作分析重叠度过高
- OCL: 需要人工定义约束，无法自动提取
- 契约式设计: Java 代码中很少有显式契约标注
- MC/DC / 变异测试: 需要运行时代码覆盖率数据
- 规约模式: roncoo-pay 中没有使用 Specification 模式
- Rete/DRT: 运行时规则引擎，不适用

### D6: 交叉验证矩阵

```
             决策表(A)  霍尔逻辑(B)  状态机(C)
决策表(A)        -        A∩B        A∩C
霍尔逻辑(B)     A∩B        -          B∩C
状态机(C)       A∩C      B∩C          -
```

规则置信度判定:
- 被 3/3 理论确认 → 高置信度
- 被 2/3 理论确认 → 中置信度
- 仅 1 理论独有 → 待验证

一致性 = Σ(交集规则数) / Σ(各理论规则总数)

## Risks / Trade-offs

- **[R1] ACCESSES 数据稀疏]** → 当前数据中 ACCESSES 覆盖不完整（write 770 条、read 仅 40 条）。Mitigation: 补充从 getter/setter 方法推断字段操作
- **[R2] 间接调用和接口多态]** → GitNexus context 查不到通过接口/Bean注入的间接调用（如 Service接口 → ServiceImpl）。Mitigation: 同时查询接口方法和同名的 Impl 方法，标注"可能有实现差异"
- **[R3] 异常路径隐式]** → 有些 Runtime 异常（NPE、数组越界）没有显式 throw/catch。Mitigation: 缺失检测中提示潜在空值风险，但标记为 LOW 置信度
- **[R4] 两个 Skill 串行成本]** → 每次分析需要两轮对话。Mitigation: 最终可以合并为一个 Skill，当前分开是为了验证阶段间数据格式的正确性
- **[R5] Skill 文件较长]** → 两个 Skill 各包含 6-7 个步骤，SKILL.md 会比较长。Mitigation: 可在后续提炼公共流程为共享文档

## Open Questions

- 状态机推断中"状态值"的枚举是否需要结合数据库表结构或常量类？当前完全从代码中推断，可能漏掉数据库中定义的状态值
- 批量分析多个 Controller 入口时，活动图是否需要合并？还是每入口独立？
