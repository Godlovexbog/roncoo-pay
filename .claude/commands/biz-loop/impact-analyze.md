---
name: "BizLoop: Impact Analyze"
description: 基于已有澄清文档跑多维影响分析
category: biz-loop
tags: [biz-loop, impact, analyze]
---

## /biz-impact analyze

跳过澄清，基于已存在的 `1-clarification.md` 运行 Phase 2 多维影响分析：代码链路（GitNexus impact/query/api_impact）+ 全景知识各维度（data-model/api/deps/config/rules/flows/architecture）。

**执行**: `Skill({skill: "biz-impact", args: "analyze <需求目录>"})`
