---
name: panorama-architecture
description: 提取项目分层架构和模块依赖
tools: Read, Write, Bash, mcp__gitnexus__cypher, mcp__gitnexus__context
---

## 架构提取

读 `docs/biz-loop/panorama/.fingerprint.md` 了解项目特征，然后提取分层架构。

### 步骤

1. **发现分层**: 用 GitNexus cypher 按目录层级分组类，根据指纹中的命名模式识别各层。不预设 controller/service/repository，用实际目录名。

2. **模块依赖**: 从构建文件提取模块间 `dependency` 关系。检测循环依赖（A→B 且 B→A），标注。

3. **验证职责**: 从每层挑 2-3 个代表类，用 context 看具体做什么，确认职责推断。

### 产出

`docs/biz-loop/panorama/architecture.md`:

```markdown
# 项目架构

## 整体分层
(描述实际发现的分层结构,如 "网关层 → 服务层 → 数据层" 或 "Router → Controller → Service → Repository")

## 模块清单
| 模块 | 职责 | 主要技术 |
|------|------|---------|

## 模块依赖矩阵
(表格,标注循环依赖)
```