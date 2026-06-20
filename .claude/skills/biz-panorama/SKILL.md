---
name: biz-panorama
description: 通用项目全景图——自动发现架构、依赖、接口、数据对象、配置、业务流程、业务规则、环境信息
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
| `/biz-panorama` 或 `full` | Phase 0→1→2→3→4→5 完整提取 |
| `/biz-panorama quick` | 对比 snapshot，仅更新变更维度（无快照时降级 full） |
| `/biz-panorama api` | 仅接口清单 + 相关校对 |
| `/biz-panorama data` | 仅数据模型 + 相关校对 |
| `/biz-panorama deps` | 仅外部依赖 |
| `/biz-panorama config` | 仅配置项 |
| `/biz-panorama flows` | 仅业务流程 |
| `/biz-panorama rules` | 仅业务规则提取（发现入口方法→并行提取→INDEX） |
| `/biz-panorama env` | 仅环境信息 |
| `/biz-panorama claude` | 仅生成/更新 CLAUDE.md |

### 命令路由

```
full:      Phase 0 → Phase 1(6 agent) → Phase 2 → Phase 3 → Phase 4 → Phase 5
quick:     读 snapshot → detect_changes → 仅启动受影响 agent
api:       Phase 1(panorama-api) → Phase 2(API部分)
data:      Phase 1(panorama-data-model) → Phase 2(数据部分)
deps:      Phase 1(panorama-dependencies)
config:    Phase 1(panorama-config)
flows:     Phase 1(panorama-flows)
rules:     Phase 3
env:       Phase 4
claude:    Phase 5
```

**quick 变更检测**: `detect_changes({scope: "compare", base_ref: "<snapshot.commit>"})`
- 配置文件变更 → dependencies + config + env
- 数据对象变更 → data-model + config
- Controller/Handler 变更 → api + flows + **rules**(重新提取受影响方法)
- 依赖文件变更 → architecture + dependencies + env
- Service/业务逻辑变更 → rules(重新提取受影响方法)

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

---

### Phase 3: 业务规则提取 (full/rules 模式)

**前置**: Phase 1 的 api-list.md + flow-maps.md 已就绪

**3.1 发现入口方法**

读 `docs/biz-loop/panorama/api-list.md` + `docs/biz-loop/panorama/flow-maps.md` 提取入口方法清单。

筛出有业务逻辑的方法（排除以下）：
- getter/setter（单行 return this.xxx 或 this.xxx = xxx）
- 纯 DAO 委托（方法体仅 dao.xxx() 调用，无条件判断）
- 纯技术工具（HTTP 请求构造、序列化/反序列化、加解密）

输出发现清单：`docs/biz-loop/panorama/rules/.discovery.md`（方法名 + 文件路径 + 所属模块）

**3.2 Pipeline 并行提取**

对发现清单中的每个方法，同一轮发出所有 agent：

```
对每个方法:
  Agent(description: "提取规则: <methodName>",
        subagent_type: "rules-extract",
        run_in_background: true)
```

等待全部完成。失败的（文件不存在或产出为空）重试一次。

**3.3 构建 INDEX.md**

扫描 `docs/biz-loop/panorama/rules/methods/` 下所有方法文件，提取每个文件的规则 ID、类型、关联实体、关联接口、关联流程 → 构建五维索引入口：

`docs/biz-loop/panorama/rules/INDEX.md`:
- **按模块→类→方法**: 方法文件链接 + 规则数量
- **按规则类型**: 校验/路由/状态机/资金/其他
- **按关联实体**: 规则→数据对象（链接 data-model.md）
- **按关联接口**: 规则→API 端点（链接 api-list.md）
- **按关联流程**: 规则→业务流程图（链接 flow-maps.md）

---

### Phase 4: 环境信息 (full/env 模式)

扫描项目文件，生成 3 个文件：

**4.1 依赖清单** → `docs/biz-loop/panorama/env/deps.md`
- 读构建文件（pom.xml / build.gradle / package.json / go.mod / Cargo.toml 等）
- 提取: 依赖名 / 版本 / 类型(runtime/test/build) / 用途(一句话)
- 读 Phase 1 的 external-deps.md 补充外部服务依赖

**4.2 安装启停** → `docs/biz-loop/panorama/env/setup.md`
- 从构建文件推断: 安装命令 / 编译命令 / 启动命令 / 测试命令
- 从配置文件推断: 数据库连接 / 中间件地址 / 端口号
- 从 README 或 docker-compose 提取额外启停信息

**4.3 接口冒烟** → `docs/biz-loop/panorama/env/smoke.md`
- 读 api-list.md 选取核心接口（对内标记的优先）
- 尝试 curl 验证每个接口可达性
- 记录: 接口路径 / 预期状态码 / 实际状态码 / 响应时间

---

### Phase 5: CLAUDE.md (full/claude 模式)

生成精简路由型 CLAUDE.md 到**项目根目录**（≤300 行），只做导航不复制内容：

- **定位**: 1-2 句说清项目是什么
- **架构**: 分层图 + 一句话职责，详细内容 `→ [architecture.md](...)`
- **模块**: 速览表（模块名/端口/一句话职责），详细内容 `→ [architecture.md](...)`
- **约定**: 关键命名模式 + 配置约束（3-5 条），详细内容 `→ [config.md](...)`
- **怎么跑**: 编译/启动/冒烟 一键命令，详细内容 `→ [env/setup.md](...)`
- **外部依赖**: 仅列支付通道+中间件名称，详细内容 `→ [external-deps.md](...)`
- **禁区/历史包袱**: 留空待人工补充
- **相关文档**: 底部列出 panorama/ 下所有文档链接

核心原则：**每个章节 3-5 行概括 + 1 个链接**，不复制表格、不列全量清单。

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
├── rules/                   业务规则
│   ├── .discovery.md        入口方法发现清单
│   ├── INDEX.md             五维索引入口
│   └── methods/             方法文件(每个入口方法一个)
├── env/                     环境信息
│   ├── deps.md              依赖清单
│   ├── setup.md             安装/启停/编译
│   └── smoke.md             接口冒烟结果
├── 校对报告
└── CLAUDE.md (项目根目录)

下一步: /biz-impact <需求描述> 开始需求影响分析
```
