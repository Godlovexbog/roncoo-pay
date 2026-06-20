# roncoo-pay 项目指南

> 精简路由型 CLAUDE.md，由全景图产出生成。详细内容查阅 `docs/biz-loop/panorama/` 下对应文档。
> 生成日期: 2026-06-20

---

## 项目定位

龙果开源支付系统 (roncoo-pay)，Spring Boot 2.1.2 + MyBatis + Shiro + ActiveMQ 多模块聚合支付平台，覆盖扫码支付、条码支付（F2F）、小程序支付及完整的对账/结算/通知生命周期。

详见 [.fingerprint.md](docs/biz-loop/panorama/.fingerprint.md)。

---

## 核心架构

10 个 Maven 模块，按 **四层** 组织，通过 **ActiveMQ** 实现应用层与业务层异步解耦。

```
表现层 (Web)                应用层 (App, 独立进程)         业务服务层             基础设施层
+-----------------------+  +--------------------------+  +-------------------+  +-------------------+
| gateway :8092/:bos    |  | app-notify       (8095)  |  | roncoo-pay-       |  | roncoo-pay-       |
| boss    :8091/:bos    |  | app-order-polling(8096)  |  | service           |  | common-core       |
| merchant:8093/:bos    |  | app-recon.      (8097)  |  | (7个子域+所有DAO)  |  | (基类/工具/枚举)   |
| sample  :8094/:bos    |  | app-settlement  (8098)  |  |                    |  |                    |
+-----------------------+  +--------------------------+  +-------------------+  +-------------------+
      |                           |                            |                      ^
      +-----> Service 接口 <------+------ ActiveMQ ------> DAO -> MySQL              |
                                                              |                      |
                                                         所有模块依赖 ---------------+
```

- **表现层**: Spring MVC + JSP，Controller 返回 ModelAndView（非 REST API），调用 Service 接口。
- **应用层**: 4 个独立 JAR，通过 ActiveMQ Listener 或定时任务触发，调 Service 执行业务。
- **业务服务层**: 核心模块，7 个子域（trade/user/account/notify/reconciliation/permission/banklink），Interface-Impl 模式。
- **基础设施层**: BaseEntity/BaseDao/BaseController 基类、分页、异常、工具类。

依赖方向单向无环: `common-core <- service <- web-* / app-*`。

详见 [architecture.md](docs/biz-loop/panorama/architecture.md)。

---

## 模块地图

详见 [architecture.md](docs/biz-loop/panorama/architecture.md) 模块清单与依赖矩阵。速览：

### 业务服务层 — roncoo-pay-service（单模块，7 子域）

| 子域 | 核心职责 | 入口 Service |
|------|---------|-------------|
| `trade` | 支付订单、扫码/F2F/小程序支付 | `RpTradePaymentManagerService` |
| `account` | 资金账户、交易记账、日终结算 | `RpAccountService`, `RpAccountTransactionService` |
| `user` | 商户管理、支付配置、流水号 | `RpUserInfoService`, `RpUserPayConfigService` |
| `notify` | 通知记录与日志 | `RpNotifyService` |
| `reconciliation` | 对账批次、差错记录 | `RpAccountCheckBatchService` |
| `permission` | RBAC 权限（操作员/角色/菜单） | `PmsOperatorService` |
| `banklink` | 微信支付工具类 | 工具类（非 Service） |

### 表现层 & 应用层速览

| 模块 | 端口 | 类型 | 一句话职责 |
|------|------|------|-----------|
| web-gateway | 8092 | Web | 支付网关入口（扫码/F2F/小程序/回调） |
| web-boss | 8091 | Web | 运营后台（交易/商户/对账/权限管理，DWZ UI） |
| web-merchant | 8093 | Web | 商户后台（交易/账户/结算查看，AdminLTE UI） |
| web-sample-shop | 8094 | Web | 模拟商户网站（演示对接流程） |
| app-notify | 8095 | App | 异步通知：MQ 消费 + HTTP POST 商户回调 |
| app-order-polling | 8096 | App | 订单轮询：MQ 消费 + 定时回查银行 |
| app-reconciliation | 8097 | App | 定时对账：下载账单、解析、比对、差错 |
| app-settlement | 8098 | App | 定时结算：日终汇总、生成结算记录（T+1） |

---

## 关键约定

### 核心命名模式（一行说明）

Entity 无后缀位于 `entity` 包（业务 `Rp` 前缀，权限 `Pms` 前缀）；VO → `XxxVo`；BO → `XxxBo`；DAO → `XxxDao`/`XxxDaoImpl`；Service → `XxxService`/`XxxServiceImpl`；编排 → `XxxBiz`；Controller → `XxxController`；任务 → `XxxTask`。

### 分层调用链

```
Controller → Service(接口) → ServiceImpl → Dao(接口) → DaoImpl(extends BaseDaoImpl) → MyBatis SqlSession
```

- DAO 和 Service 均采用 Interface-Impl 分离模式。
- Entity 为纯 POJO，无 ORM 注解，SQL 全量写在 Mapper XML 中。
- Controller 返回 ModelAndView + JSP（无 `@RestController`/`@ResponseBody`）。
- app 模块内部按 `app.{domain}.core/biz/entity/parser/scheduled` 组织，各自有 `@SpringBootApplication` 入口。

### 配置加载方式（混合四种模式）

1. Spring Boot 自动: `application.yml`（端口、视图、MyBatis）
2. `@PropertySource` + `@Value`: `jdbc.properties`, `mq_config.properties`
3. 静态块 `Properties.load()`: 支付通道参数（不走 Spring）
4. `@Configuration` 硬编码: 线程池、通知策略、轮询策略、Shiro

### 关键约束（5 条最重要的）

1. **无环境区分**: 无 dev/test/prod profile，所有配置单一文件，切换需手动修改。
2. **ActiveMQ 连接池硬编码**: 配置声明 `maxConnections=20`，但 `ActiveMqConfig.java` 强制 `@Value("#{10}")`，实际为 10。
3. **通知重试策略硬编码**: 5 级重试（0/1/2/5/15 分钟），写死在 `NotifyConfig.java`。
4. **订单轮询策略硬编码**: 6 级轮询（2/3/5/10/20/30 分钟），写死在 `PollingConfig.java`。
5. **Redis 依赖引入但未使用**: `spring-boot-starter-data-redis` 在 pom 中声明但无配置和调用代码。

详见 [config.md](docs/biz-loop/panorama/config.md)。

---

## 怎么跑

### 环境要求

- JDK 1.8+, Maven 3.3+, MySQL 5.7+, ActiveMQ 5.x

### 编译

```bash
cd roncoo-pay && mvn clean install -DskipTests
```

### 数据库初始化

```bash
# 数据库名: roncoo_mini_pay_demo, JDBC: jdbc:mysql://127.0.0.1:3306/roncoo_mini_pay_demo
mysql -u root -p roncoo_mini_pay_demo < sql/init.sql
```

### 启动 ActiveMQ

```bash
# macOS: brew services start activemq
# Linux: sudo systemctl start activemq
# 管理控制台: http://127.0.0.1:8161/admin
```

### 启动服务（推荐顺序，共 8 个进程）

| 顺序 | 模块 | 端口 | 命令 |
|------|------|------|------|
| 1 | app-notify | 8095 | `cd roncoo-pay-app-notify && mvn spring-boot:run` |
| 2 | app-order-polling | 8096 | `cd roncoo-pay-app-order-polling && mvn spring-boot:run` |
| 3 | app-reconciliation | 8097 | `cd roncoo-pay-app-reconciliation && mvn spring-boot:run` |
| 4 | app-settlement | 8098 | `cd roncoo-pay-app-settlement && mvn spring-boot:run` |
| 5 | web-boss | 8091 | `cd roncoo-pay-web-boss && mvn spring-boot:run` |
| 6 | web-gateway | 8092 | `cd roncoo-pay-web-gateway && mvn spring-boot:run` |
| 7 | web-merchant | 8093 | `cd roncoo-pay-web-merchant && mvn spring-boot:run` |
| 8 | web-sample-shop | 8094 | `cd roncoo-pay-web-sample-shop && mvn spring-boot:run` |

> 先启动 app 模块再启动 web 模块，确保 ActiveMQ 消费者就绪。

### 冒烟 URL

- 运营后台: http://localhost:8091/boss
- 支付网关: http://localhost:8092/roncoo-pay-web-gateway
- 商户后台: http://localhost:8093/mch
- 模拟商户: http://localhost:8094/

### 配置文件位置速查

| 配置 | 文件 |
|------|------|
| 数据库 | `roncoo-pay-service/src/main/resources/jdbc.properties` |
| ActiveMQ | `roncoo-pay-service/src/main/resources/mq_config.properties` |
| 微信支付 | `roncoo-pay-service/src/main/resources/weixinpay_config.properties` |
| 支付宝 | `roncoo-pay-service/src/main/resources/alipay_config.properties` |
| 对账/结算/鉴权/示例商户 | 各自模块 `src/main/resources/` 下 `.properties` 文件 |
| 日志 | 各模块 `src/main/resources/logback.xml` |

详见 [env/setup.md](docs/biz-loop/panorama/env/setup.md)。

---

## 外部依赖

### 支付通道

**微信支付**（统一下单/订单查询/被扫支付/下载对账单）与 **支付宝**（PC 网页支付 MD5 + 被扫/查询 RSA SDK），另有鉴权服务（auth_config 配置的示例接口）和模拟商户对接。

### 中间件

| 中间件 | 用途 | 连接 |
|--------|------|------|
| MySQL | 主数据库（Druid 连接池） | `127.0.0.1:3306/roncoo_mini_pay_demo` |
| ActiveMQ | 异步消息（通知队列 + 订单查询队列） | `failover:(tcp://127.0.0.1:61616)` |
| Ehcache | Shiro 权限缓存（仅 web-boss） | 本地 JVM |

> 消息队列: `opensource_demo_tradeNotify`（通知）、`opensource_demo_orderQuery`（订单轮询）。

详见 [external-deps.md](docs/biz-loop/panorama/external-deps.md)。

---

## 禁区

待补充。

---

## 历史包袱

待补充。

---

## 相关文档

| 文档 | 内容 |
|------|------|
| [.fingerprint.md](docs/biz-loop/panorama/.fingerprint.md) | 项目指纹：模块结构、框架、命名模式、分层约定 |
| [architecture.md](docs/biz-loop/panorama/architecture.md) | 架构全貌：分层、模块清单、依赖矩阵、核心类、数据流 |
| [config.md](docs/biz-loop/panorama/config.md) | 配置全景：端口、数据库、MQ、支付通道、日志、缓存、安全 |
| [external-deps.md](docs/biz-loop/panorama/external-deps.md) | 外部依赖：框架库版本、HTTP API URL、中间件连接信息 |
| [env/setup.md](docs/biz-loop/panorama/env/setup.md) | 环境搭建：编译、数据库初始化、启动步骤、注意事项 |
