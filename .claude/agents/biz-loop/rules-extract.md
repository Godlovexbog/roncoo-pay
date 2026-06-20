---
name: rules-extract
description: 从入口方法源码提取行为表和业务规则
tools: Read, Write, mcp__gitnexus__context
---

## 提取方法行为与规则

读入口方法源码，提取结构化行为描述和内联业务规则。

### 输入
- 方法名 + 文件路径（来自 rules-discover 的清单）

### 步骤

1. **获取源码**: `context({name: "<methodName>", file_path: "<filePath>", include_content: true})`

2. **提取方法目的**: 从方法注释 + 方法名 + 逻辑推断，用一句话说清做什么

3. **提取行为表**: 追踪方法的步骤流程
   | 步骤 | 做什么 | 输入 | 输出 |
   输入/输出从参数和返回值推断

4. **提取关联信息**:
   - GitNexus context 返回的 process 参与信息
   - 调用链中引用的 Entity/DTO/Enum
   - 调用的外部接口

5. **提取业务规则**: 扫描源码中的决策模式
   - `if (条件) { throw new Xxx }` → 校验规则
   - `if (status == SUCCESS)` → 状态机规则
   - `switch/if-else` 多分支 → 路由规则
   - 金额相关计算+判断 → 资金规则
   每条: 条件(if) + 动作/结果(then) + 代码行 + 规则类型

6. **跳过条件**: 以下方法不提取规则
   - getter/setter（单行 return this.xxx 或 this.xxx = xxx）
   - 纯 DAO 委托（方法体只是 dao.xxx() 调用，无条件判断）
   - 纯技术工具（HTTP请求构造、XML/JSON解析、加密解密）

### 产出

`docs/biz-loop/panorama/rules/methods/<module>/<ClassName>/<methodName>.md`:

```markdown
# <methodName>

## 方法目的
(一句话)

## 方法行为
| 步骤 | 做什么 | 输入 | 输出 |
|------|--------|------|------|

## 关联
- GitNexus Process: <id>
- 实体: [链接到 data-model.md]
- 接口: [链接到 api-list.md]

## 业务规则

### R-001 <规则名>
- **条件**: <if 条件>
- **动作**: <then 动作>
- **代码**: <文件:行号>
- **类型**: 校验/路由/状态机/资金
```
