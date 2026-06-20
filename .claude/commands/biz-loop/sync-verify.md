---
name: "BizLoop: Sync Verify"
description: 仅交叉校验——检查知识文件与代码实际状态的一致性，报告不修改
category: biz-loop
tags: [biz-loop, sync, verify]
---

## /biz-sync verify

仅 Phase 3 交叉校验：data-model ↔ context 检查废弃/缺失对象 → api-list ↔ route_map 检查废弃/缺失接口 → rules/INDEX.md 检查断链 → tests/ 检查失效引用。只报告，不修改。

**执行**: `Skill({skill: "biz-sync", args: "verify"})`
