---
name: "BizLoop: Sync Check"
description: 只读检测——报告哪些知识文件过期，不修改任何内容
category: biz-loop
tags: [biz-loop, sync, check]
---

## /biz-sync check

仅 Phase 0 + Phase 1：扫描已有知识文件 → detect_changes 对比 snapshot → 报告过期文件清单，不执行任何更新。

**执行**: `Skill({skill: "biz-sync", args: "check"})`
