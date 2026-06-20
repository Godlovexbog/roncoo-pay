---
name: biz-sync
description: 代码变更后全量联动同步——检测变更→更新全景图→交叉校验
category: biz-loop
tags: [biz-loop, sync, incremental, generic]
allowed-tools: Read, Write, Bash, Agent, Glob, mcp__gitnexus__detect_changes, mcp__gitnexus__context
---

## 全量联动同步

代码变更后增量更新 `docs/biz-loop/panorama/` 下所有已存在的知识文件。
**不预设哪些文件存在**，先扫描再按映射决定更新范围。

**前置条件**: panorama/ 已就绪，snapshot.json 存在

### 命令

| 命令 | 说明 |
|------|------|
| `/biz-sync` 或 `full` | 检测→更新→校验→快照 |
| `/biz-sync check` | 只读检测，报告过期文件，不修改 |
| `/biz-sync verify` | 仅交叉校验，不更新内容 |
| `/biz-sync snapshot` | 仅更新快照（人工确认后） |

---

### Phase 0: 扫描已有知识文件

```
Glob docs/biz-loop/panorama/**/*.md → 实际存在的文件列表
建立映射: 知识文件类型 → 代码变更类型
```

**check 模式**在此停下，输出过期文件报告。

---

### Phase 1: 检测变更 (full 模式)

```
读 snapshots/snapshot.json 获取上次快照 commit
detect_changes({scope: "compare", base_ref: "<snapshot.commit>"})
→ 按符号类型分组: 实体/Controller/Service/配置/依赖
→ 映射到 Phase 0 的知识文件
→ 无变更则退出
```

---

### Phase 2: 联动更新 (full 模式)

按依赖顺序 pipeline。每个阶段只启动实际需要更新的 agent：

```
# 数据对象变更 (优先级最高)
如果 data-model.md 存在 AND 有实体变更:
  对每个变更实体: context 获取最新定义 → 对比 data-model.md → 更新对应表格

# 接口变更
如果 api-list.md 存在 AND 有 Controller 变更:
  对每个变更端点: route_map 或 context 获取最新签名 → 更新 api-list.md
  如果 flow-maps.md 引用了该端点 → 同步更新

# 方法+规则变更 (唯一需要 agent 的阶段 —— pipeline 并行处理)
如果 rules/INDEX.md 存在 AND 有方法变更:
  对每个变更方法:
    Agent(description: "更新方法: <name>", subagent_type: "sync-update-method", run_in_background: true)
  全部完成后:
    扫描 rules/methods/ 所有文件 → 重建 INDEX.md 五维表
    读 tests/scenarios/ → 更新测试→规则映射

# 依赖变更
如果 external-deps.md 存在 AND (构建文件 OR HTTP调用变更):
  读新构建文件 → 对比 external-deps.md → 更新依赖和版本

# 配置变更
如果 config.md 存在 AND 配置文件变更:
  读变更的配置文件 → 对比 config.md → 更新配置项

# 测试标记
如果 tests/INDEX.md 存在 AND 规则有变更:
  读 rules 变更摘要 → 查 tests/INDEX.md 测试→规则表 → 标记受影响场景
  新增规则 → 追加到 gaps.md
```

---

### Phase 3: 交叉校验 (full/verify 模式)

顺序校验实际存在的知识文件，只报告不自动修改：

```
有 data-model.md:
  → 列出其中所有数据对象名
  → 逐个 context 检查是否在代码中存在
  → 报告: 废弃对象(类已删除) / 缺失对象(代码有文档无)

有 api-list.md:
  → route_map 获取实际路由列表
  → 对比 api-list.md
  → 报告: 废弃接口 / 缺失接口

有 rules/INDEX.md:
  → 检查 INDEX 中所有方法文件链接目标是否存在
  → 报告: 断链清单

有 tests/:
  → 检查场景文件中引用的规则 ID 是否还有效
  → 报告: 失效引用
```

---

### Phase 4: 更新快照 (full/snapshot 模式)

记录当前 HEAD commit + panorama/ 下所有 .md 的 checksum → `snapshots/snapshot.json`

---

### 完成后

```
✅ Sync 完成
变更: 实体 N | 接口 M | 方法 K | 规则 +P -S ~T | 测试 U 个标记
校验: (仅列出有问题项)
快照: commit=<hash>
```
