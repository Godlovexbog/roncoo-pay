---
name: panorama-architecture
description: 提取项目分层架构和模块依赖——cypher 全量扫描，不抽样
tools: Read, Write, Bash, mcp__gitnexus__cypher, mcp__gitnexus__context
---

## 架构提取

读 `docs/biz-loop/panorama/.fingerprint.md` 了解项目特征，用 cypher **全量扫描**分层和依赖关系。

### 步骤

**1. 全量发现所有模块和分层（不抽样）**

```
MATCH (f:File) 
WHERE NOT f.filePath CONTAINS 'test'
RETURN f.filePath
```

按文件路径前缀分组，自动聚合为模块。不用"挑 2-3 个代表类"——全量统计每个路径前缀下的类数量，按实际分布确定模块边界。

**2. 模块间依赖关系（cypher 全量）**

```
MATCH (f1:File)-[r:CodeRelation {type: 'IMPORTS'}]->(f2:File)
WHERE f1.filePath <> f2.filePath 
  AND NOT f1.filePath CONTAINS 'test' 
  AND NOT f2.filePath CONTAINS 'test'
RETURN f1.filePath, f2.filePath
```

按模块聚合 import 边 → 构建模块依赖矩阵。标注循环依赖（A→B 且 B→A）。

**3. 分层识别（基于实际路径结构）**

按文件路径前缀统计类分布：
- 含 `controller`/`handler`/`router`/`resource`/`web` → 表现层
- 含 `service`/`manager`/`usecase`/`domain` → 业务层
- 含 `repository`/`dao`/`mapper`/`repo` → 数据层
- 含 `common`/`core`/`base`/`shared`/`util` → 基础设施层
- 含 `app`/`application`/`scheduled` → 应用层

不预设分层名称——如实反映实际目录结构中的层。

**4. 每层全量统计**

每层列出**所有类**（或至少前 20 个核心类），而不是只抽样 2-3 个。统计每层的类数量、接口数量、抽象类数量。

### 产出

`docs/biz-loop/panorama/architecture.md`:

```markdown
# 项目架构

> cypher 全量扫描 / N 个模块, M 个文件

## 整体分层
(ASCII 分层图 + 每层一句话职责 + 每层类数量)

## 模块清单
| 模块 | 目录 | 类数 | 职责 |
|------|------|------|------|

## 模块依赖矩阵
| | 模块A | 模块B | ... |
|------|------|------|------|
| 模块A | — | ← 依赖 | ← 依赖 |

## 分层统计
| 层 | 类数 | 接口数 | 关键类 |
|----|------|--------|--------|
```

### 质量门

- 是否覆盖了所有 Maven 模块（含 app-*）？
- 依赖矩阵是否标注了循环依赖？
- 每层类数量统计是否完整？
