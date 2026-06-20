---
name: biz-panorama
description: 通用项目全景图——自动发现架构、依赖、接口、数据对象、配置、业务流程
category: biz-loop
tags: [biz-loop, panorama, discovery, knowledge, generic]
allowed-tools: Read, Write, Bash, Agent, Glob, mcp__gitnexus__query, mcp__gitnexus__context, mcp__gitnexus__cypher, mcp__gitnexus__route_map, mcp__gitnexus__list_repos, mcp__gitnexus__detect_changes
---

## 通用项目全景图

自动发现项目结构并提取全局知识到 `docs/biz-loop/panorama/`。
**不预设命名约定**，用 GitNexus + 代码扫描动态发现实际模式。
每个阶段通过 `Agent` 启动子 agent，`run_in_background: true` 并行执行。

**前置条件**: GitNexus 索引已就绪

### 命令

| 命令 | 说明 |
|------|------|
| `/biz-panorama` 或 `full` | Phase 0→1→2→3 完整提取 |
| `/biz-panorama quick` | 对比 snapshot，仅更新变更维度（无快照时降级 full） |
| `/biz-panorama api` | 仅接口清单 + 相关校对 |
| `/biz-panorama data` | 仅数据模型 + 相关校对 |
| `/biz-panorama deps` | 仅外部依赖 |
| `/biz-panorama config` | 仅配置项 |
| `/biz-panorama flows` | 仅业务流程 |
| `/biz-panorama claude` | 仅生成/更新 CLAUDE.md |

### 命令路由

```
full:      Phase 0 → Phase 1(6 agent) → Phase 2 → Phase 3
quick:     读 snapshot → detect_changes → 仅启动受影响 agent
api:       Phase 1(panorama-api) → Phase 2(API部分)
data:      Phase 1(panorama-data-model) → Phase 2(数据部分)
deps:      Phase 1(panorama-dependencies)
config:    Phase 1(panorama-config)
flows:     Phase 1(panorama-flows)
claude:    Phase 3
```

**quick 变更检测**: `detect_changes({scope: "compare", base_ref: "<snapshot.commit>"})`
- 配置文件变更 → dependencies + config
- 数据对象变更 → data-model + config
- Controller/Handler 变更 → api + flows
- 依赖文件变更 → architecture + dependencies

---

### Phase 0: 项目指纹 (仅 full)

```
Agent(description: "项目指纹识别", subagent_type: "panorama-fingerprint", run_in_background: true)
```

等待完成后继续。

### Phase 1: 并行收集 (6 agent)

同一轮 tool call 全部发出：

```
Agent(description: "架构",   subagent_type: "panorama-architecture", run_in_background: true)
Agent(description: "依赖",   subagent_type: "panorama-dependencies", run_in_background: true)
Agent(description: "接口",   subagent_type: "panorama-api",          run_in_background: true)
Agent(description: "数据",   subagent_type: "panorama-data-model",   run_in_background: true)
Agent(description: "配置",   subagent_type: "panorama-config",       run_in_background: true)
Agent(description: "流程",   subagent_type: "panorama-flows",        run_in_background: true)
```

等待全部完成。逐一校验产出文件存在且非空。失败的重试一次。

### Phase 2: 校对 (full/api/data 模式)

```
Agent(description: "校对全景图", subagent_type: "panorama-verify")
```

### Phase 3: CLAUDE.md (full/claude 模式)

```
Agent(description: "生成CLAUDE.md", subagent_type: "panorama-claude")
```

---

### 完成后

```
✅ 全景图完成
├── .fingerprint.md          项目指纹
├── architecture.md          分层架构
├── external-deps.md         外部依赖(库+HTTP服务+中间件)
├── api-list.md              API端点
├── data-model.md            数据对象(按实际分组)
├── config.md                配置项
├── flow-maps.md             核心流程
├── 校对报告
└── CLAUDE.md

下一步: /biz-rules 提取业务规则
```