---
name: panorama-data-model
description: 提取所有数据对象——cypher 全量扫描，不预设包名
tools: Read, Write, mcp__gitnexus__cypher, mcp__gitnexus__context
---

## 数据对象提取

读 `docs/biz-loop/panorama/.fingerprint.md` 了解项目语言，然后用 GitNexus cypher **全量扫描**所有数据类。

### 步骤

**1. 全量发现所有 Class（不预设包名）**

```
MATCH (c:Class) 
WHERE NOT c.filePath CONTAINS 'test'
  AND NOT c.filePath CONTAINS 'controller'
  AND NOT c.filePath CONTAINS 'service'
  AND NOT c.filePath CONTAINS 'config'
  AND NOT c.filePath CONTAINS 'utils'
  AND NOT c.filePath CONTAINS 'filter'
  AND NOT c.filePath CONTAINS 'interceptor'
  AND NOT c.filePath CONTAINS 'listener'
  AND NOT c.filePath CONTAINS 'aspect'
RETURN c.name, c.filePath
ORDER BY c.filePath
```

**2. 自动分组（不预设 Entity/VO/DTO 等名称）**

按实际包路径后缀自动聚类：
- 包路径含 `entity`/`domain`/`model`/`po`/`do`/`persistence` → 持久化实体
- 包路径含 `dto`/`vo`/`bo`/`request`/`response`/`bean` → 数据传输对象
- 包路径含 `enums`/`enumeration`/`constant` → 枚举

对无法归类的，按继承关系判断：
- 继承 `BaseEntity`/`BaseDO`/`BasePO` → 持久化实体
- 实现 `Serializable` 且无持久化基类 → DTO/VO
- 是 `Enum`/`enum` → 枚举

**3. 全量获取详情**

对每个数据对象用 context 获取字段、类型、约束注解/tag。

**4. 发现枚举**

```
MATCH (e:Enum) RETURN e.name, e.filePath
```

对 App 层的枚举（如有）也纳入。

### 产出

`docs/biz-loop/panorama/data-model.md`，按实际发现的分组：

```markdown
# 数据模型

> cypher 全量扫描 / 发现 N 个数据对象: Entity X / VO Y / Enum Z

## 公共基类
(如有)

## Entity
| 名称 | 说明 | 文件 | 核心字段 |

## VO/DTO/BO
| 名称 | 用途 | 文件 | 字段 |

## Enum
| 名称 | 用途 | 文件 | 值 |

## 发现统计
- 总扫描 Class: N
- 持久化实体: X
- VO/DTO/BO: Y
- Enum: Z
```

### 质量门

- 是否扫描了所有模块（含 app-* 独立进程）？
- 枚举是否涵盖 common-core + service + app？
- 是否有遗漏的包路径模式（如 `record`/`schema`/`document`）？
