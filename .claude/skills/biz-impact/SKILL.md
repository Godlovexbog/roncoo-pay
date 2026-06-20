---
name: biz-impact
description: 需求澄清 + 多维影响分析——全景图/规则/测试/代码链路综合分析
category: biz-loop
tags: [biz-loop, impact, analysis, requirement, generic]
allowed-tools: Read, Write, mcp__gitnexus__impact, mcp__gitnexus__detect_changes, mcp__gitnexus__api_impact, mcp__gitnexus__context, mcp__gitnexus__query, mcp__gitnexus__cypher, mcp__gitnexus__shape_check
---

## 需求影响分析

基于已沉淀的全局知识 + GitNexus 实时事实，对需求做澄清和多维影响分析。

**前置条件**: `docs/biz-loop/panorama/` 已就绪
**产出**: `docs/biz-loop/impacts/<YYYY-MM-DD>-<需求简称>/`

### 命令

| 命令 | 说明 |
|------|------|
| `/biz-impact <需求描述>` | 完整流程：澄清 → 影响分析（默认） |
| `/biz-impact clarify <需求描述>` | 仅需求澄清 |
| `/biz-impact analyze <需求目录>` | 基于已有澄清文档跑影响分析 |
| `/biz-impact quick <需求描述>` | 跳过澄清，直接出简化影响报告 |

---

### Phase 1: 需求澄清 (full/clarify 模式)

读全景知识 + 需求描述，按 6 个维度写澄清文档 `docs/biz-loop/impacts/<date>-<name>/1-clarification.md`：

1. 先读 `docs/biz-loop/panorama/` 下所有 .md 文件 + CLAUDE.md
2. 按以下维度写文档：

**业务目标** — 一句话说清要达成什么业务目的

**用户场景** — 典型使用场景 + 当前痛点。如果 flow-maps.md 包含相关流程，链接过去

**接口契约** — 对齐 api-list.md 风格，表格：方法/路径/入参(字段+类型+必填+说明)/返回(字段+类型+说明)/错误码

**边界场景** — 至少 8 条，每条标 `[预期行为]` 或 `[待产品决策]`。从现有代码反推优先

**老项目约束** — 从 CLAUDE.md 禁区+历史包袱提取相关条目，从 config.md 提取限制，每条标注来源

**不在范围里的事** — 候选清单，标"本期不做"或"留到下期"

完成后**暂停，等待用户确认**。确认后将[待产品决策]条目替换为确认结果。

---

### Phase 2: 多维影响分析 (full/analyze/quick 模式)

**2.1 确定可用维度**

读 `docs/biz-loop/panorama/` 下文件列表，确认实际存在哪些维度的知识文件。

**2.2 代码链路影响**（始终运行）

从需求提取关键词 → `query({query: "<关键词>"})` 定位核心符号 → `impact({direction: "upstream", summaryOnly: true})` 获取调用者 → 如有 API 变更则 `api_impact()` + `shape_check()`

**2.3 全景知识影响**（按实际存在的维度顺序分析）

对每个存在的维度文件，读内容 → 判断需求影响：
- 有 data-model.md → 哪些数据对象增删改字段
- 有 api-list.md → 哪些接口受影响
- 有 external-deps.md → 是否新增 HTTP 调用/依赖
- 有 config.md → 是否新增/修改配置项
- 有 rules/INDEX.md → 受影响规则(不变/修改/新增/删除)
- 有 tests/INDEX.md → 受影响场景 + 需新增场景
- 有 flow-maps.md → 哪些流程被改动
- 有 architecture.md → 哪些模块受波及

**2.4 合并输出** `2-impact-report.md`：影响概览表 + 各维度详情 + 风险汇总

---

### Phase 3: 决策记录

提取澄清文档的[待产品决策] + 影响报告中 🔴 高风险项 → `4-decision-log.md`

---

### 完成后

```
✅ 影响分析完成
├── 1-clarification.md
├── 2-impact-report.md   (分析了 N 个维度)
└── 4-decision-log.md

下一步: /biz-design 生成方案设计
```
