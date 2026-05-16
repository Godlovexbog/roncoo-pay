---
name: biz-flow
description: "业务流程分析 — 输入 Controller 入口方法，递进追踪调用链，生成流程总览 + 实体属性 + 活动 + 领域事件 + DDD 分层映射 + 自检清单"
---

# 业务流程分析

## 输入

入口方法，格式: `完整类路径#方法名`

示例: `com.roncoo.pay.controller.ScanPayController#initPay`

## 输出

一份 Markdown 报告，包含:

1. 流程总览（缩进文本树，标注关键决策和状态变更）
2. 实体及属性（含校验规则注解）
3. 活动（方法 → 业务含义 → 所属分层）
4. 业务规则入口（按 12 类框架列出已发现的规则点）
5. 领域事件（从调用链推断的领域事件列表）
6. DDD 分层映射（ASCII 架构图）
7. Mermaid 活动图（带泳道）
8. 自检清单

---

## 执行步骤

### 第一步: 定位入口方法

```
gitnexus_context({name: "<方法名>", file_path: "<类路径对应的文件>", repo: "roncoo-pay"})
```

从结果中提取: `startLine`/`endLine`、`outgoing.calls`、`incoming.calls`、`processes`。

如果方法名在多个文件中存在，用 `file_path` 精确匹配。

---

### 第二步: 递进追踪调用链

对第一层 outgoing.calls 中的每个方法，用 `gitnexus_context` 查询，获取第二层调用。对第二层关键方法继续追踪到第三层。最多 3 层。

**跳过规则**: 方法体 ≤ 5 行的 getter/setter、第三方库方法、toString/hashCode/equals 等标准方法。

**识别接口实现**: 如果 context 返回 `method_implements`，标注接口定义和 Impl 实现。

---

### 第三步: 绘制流程总览（缩进文本树）

将调用链用缩进文本树表达，每个节点标注业务含义:

```
商户系统发起xxx支付请求
  │
  ▼
[1] initPay (控制器入口)
  │
  ├─[2] checkParamAndGetUserPayConfig (参数校验 + 商户配置加载)
  │     ├─ Bean Validation 注解校验 (XxxRequestBo)
  │     ├─ 根据 payKey 查询商户支付配置 (RpUserPayConfig)
  │     ├─ IP 白名单校验 (checkIp, 可选)
  │     └─ MD5 签名校验 (MerchantApiUtil.isRightSign)
  │
  └─[3] xxxPay (核心领域服务)
        ├─ 校验支付类型枚举
        ├─ 查询支付方式 (RpPayWay) + 费率
        ├─ 查询商户信息 (RpUserInfo)
        ├─ 查订单/新建订单 (sealXxx...)
        │     ├─ 订单不存在 → 创建
        │     ├─ 订单已存在 + 金额不一致 → 处理逻辑
        │     └─ 订单已存在 + 已支付成功 → 拒绝
        └─[4] getXxxPayResultVo
              ├─ 根据 payWayCode 设定 payType
              ├─ 更新订单支付方式 → 创建支付记录
              ├─ 调用第三方支付渠道:
              │    ├─ 微信: xxx API
              │    │     ├─ 成功 → completeSuccessOrder
              │    │     ├─ 明确失败 → completeFailOrder
              │    │     └─ 结果未知 → orderSend 轮询
              │    └─ 支付宝: xxx API → orderSend 轮询
              └─ 返回 XxxResultVo → 跳转 xxx 页面
```

**输出要求**: 每个分支标注清楚条件和动作，决策点用 ├─ 表达。

---

### 第四步: 收集实体属性（读实体类源码）

对调用链涉及的每个**实体类**（RequestBo/Entity/ResultVo/枚举），用 `Read` 工具读取完整类定义，提取:

- 每个字段的类型
- Bean Validation 注解: `@NotNull`/`@Size`/`@Digits`/`@Pattern`/`@Min`/`@Max`/`@DecimalMin` 等
- 字段 Javadoc 注释
- set 方法中的守卫条件 (如 `if (amount < 0) throw...}`)
- 父类继承的字段（特别关注 `BaseEntity.status/creater/editor`）

**输出格式**:

```markdown
## 实体及属性

### XxxRequestBo — 请求参数 (输入值对象)

| 属性 | 类型 | 校验规则 | 说明 |
|------|------|----------|------|
| payKey | String | @NotNull, @Size(min=16, max=32) | 商户Key |
| orderPrice | BigDecimal | @NotNull, @Digits(integer=12, fraction=2) | 订单金额 |
| payType | String | @NotNull, @Size(min=1, max=14) | 支付类型 |
| authCode | String | @NotNull, @Size(min=16, max=20) | **特有字段** — 付款码 |
| ... | ... | ... | ... |

### XxxEntity — 领域实体

| 属性 | 类型 | 校验/守卫 | 说明 |
|------|------|----------|------|
| status | String | (继承 BaseEntity) | 订单状态 |
| orderAmount | BigDecimal | set 方法无守卫 | 订单金额 |
| expireTime | Date | — | 订单过期时间 |
| creater | String | (继承 BaseEntity) | 创建人 |
| ... | ... | ... | ... |

### PayTypeEnum — 支付类型

| 枚举值 | 说明 |
|--------|------|
| SCANPAY | 扫码支付 |
| F2F_PAY | 条码支付 |
| MICRO_PAY | 微信刷卡支付 |
| ... | ... |
```

**注意**: 
- 实体类源码是理解"校验规则"、"存在性规则"、"不变规则"的关键来源
- 特别关注继承了 BaseEntity 的类，status/creater/editor 字段经常被忽略但不被检查
- 与同名类似流程（如 ScanPay vs F2FPay）的差异在实体层标注

---

### 第五步: 提取活动列表 + 控制流

**5.1 活动列表**

将调用链上的方法映射为业务活动:

| 活动名 | 方法 | 所属分层 | 说明 |
|--------|------|----------|------|
| 发起扫码支付 | ScanPayController.initPay() | 接口层 | 统一入口 |
| 参数校验与配置获取 | CnpPayService.checkParamAndGetUserPayConfig() | 应用服务 | 多层校验后获取 RpUserPayConfig |
| 扫码支付 | RpTradePaymentManagerServiceImpl.initDirectScanPay() | 领域服务 | 校验→查订单→创建→调用渠道 |
| ... | ... | ... | ... |

**5.2 决策点清单**

对**方法体 > 5 行**的方法读源码，提取控制流:

| 元素 | 提取内容 |
|------|---------|
| `if (条件) { ... }` | 条件表达式 + 分支内动作 |
| `else if (条件) { ... }` | 条件表达式 + 分支内动作 |
| `else { ... }` | 分支内动作 |
| `switch (值) { case: ... }` | 匹配值 + 分支动作 |
| `throw new XxxException` | 异常类型 + 抛出条件 |
| `try { ... } catch (Xxx e) { ... }` | 异常类型 + 处理方式 |
| `return xxx` | 返回值 + 所在分支 |

**输出: 决策点清单表**

| 编号 | 所在方法 | 行号 | 决策类型 | 条件 | 动作 |
|------|---------|------|---------|------|------|
| D1 | initPay | 93 | if | payType 为空 | initNonDirectScanPay → 返回 gateway |
| ... | ... | ... | ... | ... | ... |

---

### 第六步: 提取业务规则入口（按 12 类框架）

按 SBVR 12 类规则分类列出代码中"已发现"的规则入口。这里只列已实现的部分，缺失检测留给 biz-rule。

**12 类规则框架:**

| 类别 | 检查点 | 从哪找 |
|------|--------|--------|
| 1. 校验规则 | 字段的 Bean Validation 注解 | 实体类源码 (第四步) |
| 2. 存在性规则 | null 检查 + 实体 status 字段校验 | 方法源码中的 if (xx == null) |
| 3. 安全规则 | 签名/加密/IP白名单/时效性 | checkParamAndGetUserPayConfig, checkIp |
| 4. 不变规则 | set 守卫 + 聚合约束 + 唯一性 | 实体类 set 方法 + insert/update 配对 |
| 5. 计算规则 | 派生字段的赋值逻辑 | 方法源码中的计算逻辑 |
| 6. 状态转换规则 | status 字段的 if/赋值 | 决策点中涉及 status 的条目 |
| 7. 路由规则 | payType/payWay 等分发判断 | 决策点中的枚举分发 |
| 8. 幂等规则 | 重复提交时的处理 | selectBy then insert 模式 |
| 9. 时间规则 | 时间一致性 + 过期判断 | expireTime 相关的 if 判断 |
| 10. 补偿规则 | @Transactional + 失败回滚 | 方法注解 + catch 中的补偿动作 |
| 11. 审计规则 | LOG + creater/editor | 日志语句 + BaseEntity 字段赋值 |
| 12. 通知规则 | orderSend/消息队列 | 方法调用中的通知发送 |

**输出**: 每类规则下列出已发现的具体规则条目（表格，一行一条），标注实现位置。

---

### 第七步: 推断领域事件

从调用链和决策点中推断领域事件:

**事件来源映射:**

| 代码操作 | 对应事件类型 |
|---------|------------|
| `dao.insert(xxx)` | 实体创建事件 |
| `dao.update(xxx)` | 实体变更事件 |
| `setStatus()` / 状态字段 write | 状态变更事件 |
| `throw BizException` | 异常/失败事件 |
| `httpXmlRequest` / 第三方 API 调用 | 外部交互事件 |
| `orderSend` / 通知调用 | 通知事件 |
| `completeSuccessOrder` | 支付成功事件 |
| `completeFailOrder` | 支付失败事件 |
| `new XxxResultVo` + return | 结果返回事件 |

**输出格式**:

```markdown
## 领域事件

| # | 事件名 | 触发时机 | 所属聚合 | 携带数据 |
|---|--------|----------|----------|----------|
| E1 | RequestValidated | Bean Validation 通过 | 应用层 | XxxRequestBo 所有字段 |
| E2 | MerchantSecurityVerified | payKey+IP+签名验证通过 | RpUserPayConfig | payKey, merchantNo |
| E3 | PayOrderCreated | 订单不存在→新建 | RpTradePaymentOrder | orderId, merchantNo, orderNo, orderAmount |
| E4 | **PayOrderAmountConflict** | 重复下单,金额不一致 | RpTradePaymentOrder | oldAmount, newAmount |
| E5 | ThirdPartyPaymentRequested | 调用微信/支付宝 | RpTradePaymentRecord | payWayCode, bankOrderNo |
| E6 | PaymentSucceeded | 第三方返回成功 | RpTradePaymentOrder+Record | bankTrxNo, orderAmount |
| E7 | PaymentFailed | 第三方返回失败/验签失败 | RpTradePaymentOrder+Record | errCode, errMsg |
| ... | ... | ... | ... | ... |
```

**关键要点**: 事件的"触发时机"要指向具体的代码行号或方法调用点。

---

### 第八步: DDD 分层映射

按包路径将方法划分到四层:

```
接口层:    roncoo-pay-web-gateway/controller
应用层:    roncoo-pay-web-gateway/service (CnpPayService 等)
领域层:    roncoo-pay-service/.../service/impl (领域服务)
           roncoo-pay-service/.../entity + vo + bo + enums (领域模型)
基础设施:  roncoo-pay-service/.../dao + utils + httpclient
           外部 API (微信/支付宝)
```

**输出**: ASCII 架构图，标注每一层有哪些类/方法。

---

### 第九步: 生成 Mermaid 活动图

综合前八步数据，生成带泳道的 Mermaid flowchart:

**泳道**: 接口层 → 应用层 → 领域层 → 基础设施层 → 视图返回

**节点**: `[活动名]` 矩形 / `{条件}` 菱形 / `[[异常]]` 子程序形

**连线**: `-->` 实线 / `-.->` 虚线(异常)

---

### 第十步: 产出自检清单

| 维度 | 数据源 | 数量 | 图上标注 | 覆盖率 | 遗漏项 |
|------|--------|------|---------|--------|--------|
| 调用方法 | outgoing.calls 去重 | N | N' | % | ... |
| 决策分支 | 源码 if/throw/catch | N | N' | % | ... |
| 实体类读取 | 调用链涉及的实体类 | N | N' | % | ... |
| 领域事件 | 推断的事件数 | N | N' | % | ... |
| 规则分类 | 12 类中有数据源可查的类 | N | N' | % | ... |

---

## 输出总模板

```markdown
# 业务流程分析: `{入口方法全路径}`

> 分析日期: YYYY-MM-DD
> 入口: `ClassName.methodName()` (文件路径:行号)

## 一、流程总览
（缩进文本树，标注关键决策和状态变更）

## 二、实体及属性
（按实体类分组，每个属性列出类型/校验规则/说明）

## 三、活动
### 3.1 活动列表（活动名/方法/分层/说明）
### 3.2 决策点清单（编号/方法/行号/类型/条件/动作）

## 四、业务规则
（按 12 类分别列出已发现的规则，标注实现位置）

## 五、领域事件
（事件名/触发时机/所属聚合/携带数据）

## 六、DDD 分层映射
（ASCII 架构图）

## 七、活动图
（Mermaid flowchart 带泳道）

## 八、自检清单
（覆盖率对比表 + 遗漏项说明）
```

---

## 执行约束

- ⚠️ 只读分析，不修改任何代码
- ⚠️ 严格按一→十步顺序执行
- ⚠️ 如果 Cypher 查询超时或返回空，标注警告继续
- ⚠️ 方法体 ≤ 5 行的 getter/setter 不深入追踪但记录
- ⚠️ 实体类源码必须读取（第四步），这是校验规则/存在性规则/不变规则的关键来源
- ⚠️ 第三方库方法不追踪
- ⚠️ 事件推断基于代码操作的实际语义，不要臆造事件
