---
name: panorama-data-model
description: 提取所有数据对象——按实际命名模式自适应发现
tools: Read, Write, mcp__gitnexus__cypher, mcp__gitnexus__context
---

## 数据对象提取

读 `docs/biz-loop/panorama/.fingerprint.md` 了解项目的实际命名模式，然后用对应模式搜索数据对象。

### 步骤

1. **发现数据对象**: 用 GitNexus cypher 查所有 Class/Struct/Interface，按指纹中的命名模式聚类：
   - 指纹说什么就叫什么（是 Entity 就搜 Entity，是 Model 就搜 Model，是 Schema 就搜 Schema）
   - 多种模式共存就都列出来，如实分组

2. **注解辅助**: 搜索数据相关注解补充发现：
   - Java: `@Entity`, `@Table`, `@Document`, `@Embeddable`, `@Schema`
   - Python: `@dataclass`, SQLAlchemy Model, Pydantic BaseModel
   - TypeScript: `@Entity()`, `@Schema()`, Prisma model/type

3. **枚举/常量**: 查 Enum/Enumeration 节点，搜索 `public enum`, `enum class`, `Enum`, `const`

4. **获取详情**: 对每个数据对象用 context 获取字段名、类型、约束注解/tag、继承关系

### 产出

`docs/biz-loop/panorama/data-model.md`，按实际发现的分组：

```markdown
# 数据模型

## Entity (数据库实体)
| 名称 | 表名 | 文件 | 字段 |
|------|------|------|------|

## DTO (数据传输对象)
| 名称 | 用途 | 文件 | 字段 |
|------|------|------|------|

## Enum (枚举)
| 名称 | 用途 | 文件 | 值列表 |
|------|------|------|--------|

(分组名称来自实际代码,如代码中叫 Domain/Schema/Record/Model 就如实用这些名称)
```