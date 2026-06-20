---
name: sync-update-method
description: 对比方法新旧源码，更新方法行为文件和内联规则
tools: Read, Write, mcp__gitnexus__context
---

## 更新方法文件

对比方法的当前源码与已有方法文件，增量更新行为和规则。

### 输入
- 方法名 + 文件路径（从 detect_changes 结果获取）

### 步骤

1. **获取最新源码**: `context({name: "<methodName>", include_content: true})`

2. **读旧方法文件**: 检查 `docs/biz-loop/panorama/rules/methods/<module>/<ClassName>/<methodName>.md`
   - 存在 → 对比更新
   - 不存在 → 判断是否应新增（入口方法?有 if/throw/状态判断?）

3. **对比行为表**:
   - 步骤增减? → 更新步骤表
   - 输入输出变化? → 更新参数描述
   - 关联变化? → 更新链接

4. **对比规则**（规则ID 保持稳定，基于代码位置匹配）:
   - 源码新增 if-throw → 追加新规则，分配新编号
   - 旧规则对应的 if 代码消失 → 标记 `~~删除线~~`，不真删
   - if 条件或 throw 异常变了 → 更新规则的条件和动作

5. **记录变更摘要**: {方法, 新增R N条, 删除R M条, 修改R K条}

### 产出
- 更新后的 `panorama/rules/methods/<module>/<ClassName>/<methodName>.md`
- 变更摘要（供 sync 主流程汇总）
