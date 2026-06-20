---
name: "BizLoop: Sync Snapshot"
description: 仅更新快照——记录当前 HEAD commit 和全景图文件校验和
category: biz-loop
tags: [biz-loop, sync, snapshot]
---

## /biz-sync snapshot

仅 Phase 4：人工确认全景图内容正确后，记录当前 HEAD commit + panorama/ 下所有 .md 的 checksum → `snapshots/snapshot.json`。

**执行**: `Skill({skill: "biz-sync", args: "snapshot"})`
