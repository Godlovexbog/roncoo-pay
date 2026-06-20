---
name: panorama-fingerprint
description: 识别项目指纹——语言、构建工具、模块结构、命名模式、框架类型
tools: Read, Bash, Glob, mcp__gitnexus__cypher, mcp__gitnexus__query
---

## 项目指纹识别

识别当前项目的技术特征和命名约定，写入 `docs/biz-loop/panorama/.fingerprint.md`。

### 步骤

1. **识别构建系统**: 读根目录 pom.xml / build.gradle / package.json / go.mod / Cargo.toml，确定语言、构建工具、模块列表

2. **分析命名模式**: 用 GitNexus cypher 抽样前 500 个类名，统计命名后缀分布：
   ```
   MATCH (c:Class) RETURN c.name, c.filePath LIMIT 500
   ```
   回答：数据对象叫什么？接口层叫什么？业务层叫什么？数据层叫什么？

3. **探测框架**: 用 GitNexus query 搜索框架特征关键词：
   ```
   query({query: "controller route handler endpoint framework"})
   ```

4. **输出指纹文件**：
   ```markdown
   # 项目指纹
   
   - 语言: <Java/Python/TypeScript/Go/...>
   - 构建工具: <Maven/Gradle/npm/pnpm/go mod/...>
   - 模块数: N
   
   ## 命名模式
   | 层 | 实际命名 | 示例 |
   |----|---------|------|
   | 数据对象 | XxxEntity, XxxDTO | ... |
   | 接口层 | XxxController | ... |
   | 业务层 | XxxService, XxxManager | ... |
   | 数据层 | XxxDao, XxxMapper | ... |
   
   ## 框架
   - Web/API: <Spring Boot/Express/FastAPI/Gin/...>
   ```

重要：如实反映代码中的实际命名，不预设 Entity/DTO/Model/Schema 等名称。