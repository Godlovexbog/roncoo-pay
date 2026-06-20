---
name: panorama-api
description: 提取所有对外API端点清单
tools: Read, Write, Bash, mcp__gitnexus__route_map, mcp__gitnexus__context
---

## API 端点提取

读 `docs/biz-loop/panorama/.fingerprint.md` 了解框架类型，提取所有 API 端点。

### 步骤

1. **优先 GitNexus**: 先尝试 `route_map`（支持 Spring MVC/WebFlux 等框架自动提取）。

2. **回退代码扫描**: 如果 route_map 为空，按指纹的语言搜索对应模式：
   - Java: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@RequestMapping`, `@Path`
   - Python: `@app.route`, `@router.get`, `@router.post`, `@api.get`
   - TypeScript: `@Get(`, `@Post(`, `@Put(`, `@Delete(`, `app.get(`, `router.get(`
   - Go: `r.GET(`, `r.POST(`, `mux.HandleFunc(`, `gin.Context`

3. **获取详情**: 对每个端点用 context 获取完整方法签名、参数、返回类型。

4. **区分对内/对外**: 根据鉴权注解(@PreAuthorize/@Public/@Anonymous/openapi 路径前缀)和包路径判断。

### 产出

`docs/biz-loop/panorama/api-list.md`，按模块/Controller 分组：

| 方法 | 路径 | 处理函数 | 对内/对外 | 入参 | 返回 |
|------|------|---------|----------|------|------|
| POST | /api/users | UserController.create | 对外 | CreateUserRequest | UserResponse |
| GET | /internal/health | HealthController.check | 对内 | - | HealthStatus |