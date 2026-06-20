---
name: "BizLoop: Panorama Rules"
description: 仅提取业务规则——发现入口方法→pipeline并行提取→构建五维INDEX
category: biz-loop
tags: [biz-loop, panorama, rules]
---

## /biz-panorama rules

Phase 3 规则提取：发现入口方法（读 api-list + flow-maps，筛掉 getter/DAO/工具方法）→ pipeline 并行跑 rules-extract agent → 构建五维 INDEX.md。

**执行**: `Skill({skill: "biz-panorama", args: "rules"})`
