---
name: panorama-verify
description: 校对全景图产出之间的不一致——API vs 数据模型、依赖 vs 配置
tools: Read, Write
---

## 全景图校对

对 `docs/biz-loop/panorama/` 下已完成的基础文件做交叉校对。

### 检查项

1. **API vs 数据模型**: 提取 api-list.md 中所有接口入参/出参的对象名，在 data-model.md 中搜索这些对象的定义。不一致时报告：
   - "接口引用了未建模对象: XxxResponse (出现在 GET /api/xxx 返回中，但 data-model.md 无此对象)"
   - "数据模型中有未使用的对象: XxxVo (data-model.md 有定义，但无任何接口引用)"

2. **依赖 vs 配置**: 检查 external-deps.md 中的 HTTP 外部服务是否在 config.md 中有对应配置项。有 URL 但没有配置项 → 可能是硬编码，标记。

### 产出

校对报告追加到 `docs/biz-loop/panorama/api-list.md` 底部：

```markdown
## 校对报告
(生成时间)

### 接口引用了未建模对象
| 接口 | 引用的对象 | 状态 |
|------|-----------|------|

### 数据模型中有未使用的对象
| 对象 | 状态 |
|------|------|

### 外部服务缺少配置
| 服务 | 说明 |
|------|------|
```