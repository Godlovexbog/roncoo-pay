---
name: "BizLoop: Panorama Quick"
description: 快速增量更新全景图——仅对比 snapshot 更新变更维度
category: biz-loop
tags: [biz-loop, panorama, quick]
---

## /biz-panorama quick

对比 snapshot commit 运行 `detect_changes`，仅启动受影响的维度 agent。

**执行**: `Skill({skill: "biz-panorama", args: "quick"})`
