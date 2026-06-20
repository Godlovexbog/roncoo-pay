---
name: panorama-config
description: 提取项目所有配置项——按主题分组，敏感信息脱敏
tools: Read, Write, Bash, Glob
---

## 配置提取

扫描项目所有配置文件，按主题分组整理配置项全景。

### 步骤

1. **Glob 发现**: 搜索所有配置文件格式
   `**/*.yml`, `**/*.yaml`, `**/*.properties`, `**/.env*`, `**/*.toml`, `**/*.json` (config类), `**/settings.py`, `**/config.go`

2. **按主题分组提取**:
   - 数据库: datasource, jdbc, mongodb, redis, postgres, mysql
   - 消息队列: kafka, rabbitmq, pulsar, activemq, rocketmq
   - 外部服务: 任何含 `url`, `endpoint`, `host`, `base-path`, `api.` 的配置
   - 认证: jwt, oauth2, api-key, secret, token, credential
   - 业务参数: 业务相关 timeout/rate/limit/fee 等
   - 环境: active profile, server port, log level, debug

3. **脱敏**: 密码/密钥/Token/Secret 类配置值替换为 `***`，标注 🔒

### 产出

`docs/biz-loop/panorama/config.md`:

| 配置项 | 值 | 分组 | 用途 | 来源文件 |
|--------|----|------|------|---------|
| spring.datasource.url | jdbc:mysql://... | 数据库 | 主库 | application.yml |
| wechat.pay.api-key | `***` 🔒 | 外部服务 | 微信支付密钥 | application.yml |