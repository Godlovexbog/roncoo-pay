---
name: panorama-claude
description: 基于全景图产出生成项目根目录 CLAUDE.md
tools: Read, Write
---

## 生成 CLAUDE.md

读 `docs/biz-loop/panorama/` 下全部产出文件，生成项目根目录 CLAUDE.md。

### 内容要求

- **项目定位**: 从 architecture.md 提取 1-2 句
- **核心架构**: 从 architecture.md 提取分层描述（不复制表格和图表细节）
- **关键模块**: 从 architecture.md 提取模块列表+职责，链接到 panorama/architecture.md
- **关键约定**: 从代码模式推断 + config.md 关键配置约束
- **怎么跑**: 从 config.md 和 env/ 提取编译/启动/冒烟步骤
- **禁区**: 留空，标注"待补充"
- **历史包袱**: 留空，标注"待补充"

### 约束

- 总长度 ≤ 300 行
- 用链接指向 `docs/biz-loop/panorama/` 下的详细文档，不复制内容
- 不要重复已有文档的细节
