---
name: panorama-dependencies
description: 提取所有外部依赖——构建库、HTTP外部服务调用、中间件
tools: Read, Write, Bash, Glob, mcp__gitnexus__context, mcp__gitnexus__query
---

## 外部依赖提取

读 `docs/biz-loop/panorama/.fingerprint.md` 了解项目语言，提取三类外部依赖。

### 步骤

1. **构建依赖**: 从 pom.xml / build.gradle / package.json / go.mod 提取库依赖，含版本。

2. **HTTP 外部服务**（重点）: 搜索代码中 HTTP 客户端模式，按语言自适应：
   - Java: RestTemplate, WebClient, FeignClient, @HttpExchange, OkHttp, Retrofit, HttpClient
   - Python: requests, httpx, aiohttp, urllib
   - TypeScript: axios, fetch, got, node-fetch, ky
   - Go: http.Client, resty, grequests
   - 用 context 查调用点获取目标 URL、HTTP 方法、调用方
   - Bash 搜索配置文件中的外部地址: `grep -rE 'https?://[a-zA-Z0-9.-]+' --include="*.yml" --include="*.yaml" --include="*.properties" --include="*.env" --include="*.toml"`

3. **中间件**: 从配置文件提取 DB/缓存/MQ/搜索引擎连接信息。

### 产出

`docs/biz-loop/panorama/external-deps.md`，分四类表格：

| 类别 | 名称 | 版本/地址 | 用途 | 来源 |
|------|------|----------|------|------|
| 框架与库 | spring-boot-starter | 3.x | Web框架 | pom.xml |
| HTTP服务 | 微信支付API | https://api.mch.weixin.qq.com | 支付 | WeiXinPayUtils.java:53 |
| 中间件 | MySQL | 8.x | 主库 | application.yml |
| 三方SDK | alipay-sdk | x.x | 支付宝 | pom.xml |

**来源列**区分: 构建文件 / 代码扫描(文件:行号) / 配置文件