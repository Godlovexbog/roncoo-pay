---
name: "BizLoop: Sync Full"
description: 全量联动同步——检测变更→更新全景图→交叉校验→更新快照
category: biz-loop
tags: [biz-loop, sync, full]
---

## /biz-sync full

Phase 0 扫描已有知识文件 → Phase 1 detect_changes 检测变更 → Phase 2 联动更新（数据/接口/规则/依赖/配置/测试标记，规则更新用 pipeline agent）→ Phase 3 交叉校验 → Phase 4 更新快照。

**执行**: `Skill({skill: "biz-sync", args: "full"})`
