---
name: panorama-flows
description: 从 GitNexus 代码图提取核心业务流程——cypher 全量扫描 STEP_IN_PROCESS 边
tools: Read, Write, mcp__gitnexus__cypher, mcp__gitnexus__context
---

## 业务流程提取

用 GitNexus cypher 全量扫描 Process 和步骤，**不使用** ReadMcpResourceTool（不可用环境下降级到 cypher）。

### 步骤

**1. 全量发现所有 Process**

```
MATCH (s)-[r:CodeRelation {type: 'STEP_IN_PROCESS'}]->(p:Process)
RETURN p.heuristicLabel, p.processType, s.name, s.filePath, r.step
ORDER BY p.heuristicLabel, r.step
```

这会返回每个 Process 的所有步骤，精确到方法名+文件路径。

**2. 按业务域分组**

按 process 名称前缀聚类（如 Pay/Notify/Reconciliation/Sett/Auth/Account 等），或利用 GitNexus cluster 信息。

**3. 标注步骤类型**（基于方法名和 context 推断）

对每个步骤方法，根据名称模式推断类型：
- 入口: 方法属于 Controller/Handler/Application
- 决策: 方法名含 validate/check/decide/route/audit
- 计算: 方法名含 calculate/compute/build/seal/assemble/parse
- DB操作: 方法属于 DAO/Repository/Mapper
- 外部调用: 方法名含 send/request/call/invoke/http
- MQ发送: 方法名含 send/publish/produce/notify

对推断不确定的（名称不典型），用 context 快速确认。

**4. 标注流程覆盖**

每个流程标注入口端点路径和 HTTP 方法（从 api-list.md 交叉引用）。

### 产出

`docs/biz-loop/panorama/flow-maps.md`:

```markdown
# 核心业务流程

> cypher 全量扫描 / 发现 N 个 Process, M 个步骤 / 按 X 个业务域分组

## 一、支付流程
### 1.1 直连扫码支付
| 步骤 | 类型 | 类.方法 | 说明 |
|------|------|--------|------|

## 流程覆盖统计
- 总 Process: N
- 总步骤: M
- 入口端点: K 个(有对应 API)
- 决策节点: D 个(有对应规则方法)
```

### 质量门

- cypher 返回的 Process 是否全部包含？无遗漏？
- Process 类型分布（cross_community / within_community）？
- 步骤数是否合理（不少于 Process 数量的 3 倍）？
