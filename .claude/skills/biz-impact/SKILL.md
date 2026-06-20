---
name: biz-impact
description: 需求澄清 + 多维影响分析——全景图/规则/测试/代码链路综合分析
category: biz-loop
tags: [biz-loop, impact, analysis, requirement, generic]
allowed-tools: Read, Write, mcp__gitnexus__impact, mcp__gitnexus__detect_changes, mcp__gitnexus__api_impact, mcp__gitnexus__context, mcp__gitnexus__query, mcp__gitnexus__cypher, mcp__gitnexus__shape_check
---

## 需求影响分析

基于已沉淀的全局知识 + GitNexus 实时事实，对需求做澄清和多维影响分析。

**前置条件**: `docs/biz-loop/panorama/` 已就绪
**产出**: `docs/biz-loop/requirements/<YYYY-MM-DD>-<需求简称>/`

### 命令

| 命令 | 说明 |
|------|------|
| `/biz-impact <需求描述>` | 完整流程：澄清 → 影响分析 → 测试案例 |
| `/biz-impact clarify <需求描述>` | 仅需求澄清 |
| `/biz-impact analyze <需求目录>` | 基于已有澄清文档跑影响分析 + 测试案例 |
| `/biz-impact quick <需求描述>` | 跳过澄清，直接出简化影响报告(不含测试案例) |

---

### Phase 1: 需求澄清 (full/clarify 模式)

读全景知识 + 需求描述，按 6 个维度写澄清文档 `docs/biz-loop/requirements/<date>-<name>/1-clarification.md`：

1. 先读 `docs/biz-loop/panorama/` 下所有 .md 文件 + CLAUDE.md
2. 按以下维度写文档：

**业务目标** — 一句话说清要达成什么业务目的

**用户场景** — 典型使用场景 + 当前痛点。如果 flow-maps.md 包含相关流程，链接过去

**接口契约** — 对齐 api-list.md 风格，表格：方法/路径/入参(字段+类型+必填+说明)/返回(字段+类型+说明)/错误码

**边界场景** — 至少 8 条，每条标 `[预期行为]` 或 `[待产品决策]`。从现有代码反推优先

**老项目约束** — 从 CLAUDE.md 禁区+历史包袱提取相关条目，从 config.md 提取限制，每条标注来源

**不在范围里的事** — 候选清单，标"本期不做"或"留到下期"

完成后**暂停，等待用户确认**。确认后将[待产品决策]条目替换为确认结果。

---

### Phase 2: 多维影响分析 (full/analyze/quick 模式)

**2.1 确定可用维度**

读 `docs/biz-loop/panorama/` 下文件列表，确认实际存在哪些维度的知识文件。

**2.2 代码链路影响**（始终运行）

从需求提取关键词 → `query({query: "<关键词>"})` 定位核心符号 → `impact({direction: "upstream"})` 获取调用者和受影响 Process → 如有 API 变更则 `api_impact()` + `shape_check()`

**2.3 全景知识影响**（按实际存在的维度顺序分析）

对每个存在的维度文件，读内容 → 判断需求影响：
- 有 data-model.md → 哪些数据对象增删改字段
- 有 api-list.md → 哪些接口受影响
- 有 external-deps.md → 是否新增 HTTP 调用/依赖
- 有 config.md → 是否新增/修改配置项
- 有 rules/INDEX.md → 受影响规则(不变/修改/新增/删除)
- 有 flow-maps.md → 哪些流程被改动
- 有 architecture.md → 哪些模块受波及

**2.4 合并输出** `2-impact-report.md`：影响概览表 + 各维度详情 + 风险汇总

---

### Phase 3: 测试影响与验证案例 (full/analyze 模式)

> quick 模式跳过此阶段

**3.1 场景建模 —— C-E-A-R 四元组**

对每条受影响入口链路，用 **C-E-A-R 模型** 结构化描述场景，确保可追溯：

```
C  (Command)   谁/什么触发 — API请求 / 外部回调 / 定时任务 / 用户操作
E1 (Entity)    涉及哪些数据 — 输入 Entity / 输出 VO / 持久化 Entity
A  (Activity)  执行哪些活动 — 调用链: Controller → Service → DAO → 外部调用
R  (Rule)      受哪些规则约束 — 校验规则 / 路由规则 / 状态机规则 / 资金规则
E2 (Event)     产生什么结果 — 状态变更 / 数据写入 / 通知 / 拒绝返回
```

每个场景标注完整度: C✓ | E1✓ | A✓ | R✓ | E2✓ = N/5

**场景分类与推导规则**：
- ✅ 正常场景 — 所有 R 通过, routing 走正常分支
- ⚠️ 异常场景 — 至少一条 R 不满足, 或 external 调用失败
- 🔒 安全场景 — 签名/权限/幂等 rule 被绕过
- 🔄 兼容场景 — 已有调用方不受影响

**3.2 识别受影响入口链路**

对 Phase 2.2 中每个受影响的入口 API，用 GitNexus context 追踪完整调用栈：

```
入口 API: POST /scanPay/initPay
  └── ScanPayController.initPay             [入口]
      ├── CnpPayService.checkParamAndGetUserPayConfig  [校验]
      ├── RpTradePaymentManagerServiceImpl.initDirectScanPay  [核心, 🔴 变更]
      │   ├── RpTradePaymentOrderDao.insert              [DB]
      │   ├── RpTradePaymentRecordDao.insert             [DB]
      │   └── RpPayWayService.getByPayWayTypeCode         [费率]
      └── getScanPayResultVo                              [渠道, 🔴 变更]
          ├── WeiXinPayUtils.httpXmlRequest               [外部调用]
          └── RpNotifyService.orderSend                   [MQ]
```

每条链路标注：
- 哪些节点**不变**(🟢)、**修改**(🟡)、**新增**(🔴)
- 对应的 flow-maps.md 流程名称和步骤号

**3.2 回归测试案例**

对每条受影响链路，从 flow-maps.md 反推已有的核心测试场景。每条场景标注：

| 场景编号 | 入口链路 | 测试目的 | 为什么需要回归 | 优先级 |
|---------|---------|---------|---------------|--------|
| REG-001 | POST /scanPay/initPay (SCANPAY) | 现有微信扫码支付不受影响 | payType路由新增COMPOSITE, 需确保SCANPAY路径不改道 | P0 |
| REG-002 | POST /scanPay/initPay (DIRECT_PAY) | 现有支付宝即时到账不受影响 | 同上 | P0 |

**P0**: 必须回归, P1: 建议回归, P2: 可选

**3.3 新增测试案例（含 curl 报文）**

对每个新增/修改的接口，按"正常→边界→异常"顺序写测试案例。每条案例包含**可执行的 curl 命令**：

```markdown
### NEW-001: 组合支付-银行卡+积分-正常支付

**curl**:
```bash
curl -X POST http://localhost:8092/roncoo-pay-web-gateway/scanPay/initPay \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "payKey=abc123def456..." \
  -d "orderNo=TEST-COMPOSITE-001" \
  -d "productName=测试组合支付商品" \
  -d "orderPrice=100.00" \
  -d "payType=COMPOSITE" \
  -d "compositeSources=[{\"sourceType\":\"POINTS\",\"amount\":30.00,\"priority\":1},{\"sourceType\":\"BANK_CARD\",\"amount\":70.00,\"priority\":2}]" \
  -d "returnUrl=http://example.com/return" \
  -d "notifyUrl=http://example.com/notify"
```

**预期**:
- HTTP 200
- 返回 CompositePayResultVo, status=WAITING
- compositeSources[0].payUrl 非空

---

### NEW-002: 组合支付-金额不匹配-应抛异常

**curl**:
```bash
curl -X POST http://localhost:8092/roncoo-pay-web-gateway/scanPay/initPay \
  -d "payKey=xxx" \
  -d "orderNo=TEST-002" \
  -d "productName=测试" \
  -d "orderPrice=100.00" \
  -d "payType=COMPOSITE" \
  -d "compositeSources=[{\"sourceType\":\"POINTS\",\"amount\":50.00}]" \
  ...
```

**预期**:
- 返回异常: "组合支付金额不等于订单金额"
```

案例覆盖矩阵：

| 类型 | 最少条数 | 要求 |
|------|---------|------|
| 正常路径 | ≥2 | 覆盖核心场景组合 |
| 边界场景 | ≥3 | 对应澄清文档的边界场景 |
| 异常/错误 | ≥3 | 参数校验、签名错误、幂等、超时 |
| 回归验证 | ≥3 | 确保已有功能不受影响 |

**3.4 输出** `3-test-cases.md`

结构：
```
# 测试案例

## 一、受影响入口链路 (C-E-A-R 场景)
  ### S1: <场景名>
  C  ... | E1 ... | A  ... | R  ... | E2 ...
  完整度: N/5

## 二、规则覆盖清单
  每条规则绑定测试案例, 量化覆盖:
  | 规则ID | 通过案例 | 拒绝/异常案例 | 边界案例 | 覆盖状态 |
  |--------|---------|-------------|----------|----------|
  | N-001  | S1      | S4          | S6       | ✓ 3/3   |

## 三、覆盖矩阵
  ### C×R 矩阵
  (Command类型 × 规则ID, 标注场景编号, 算百分比)
  当前覆盖率: N/M = X%, 目标 ≥ 85%

  ### R×E 矩阵
  (规则ID × Event类型, 标注场景编号)
  当前覆盖率: N/M = X%, 目标 ≥ 75%

## 四、回归测试案例
  (表格: 场景/入口/目的/为什么回归/优先级)

## 五、新增测试案例
  ### 5.1 正常路径
  ### 5.2 边界场景
  ### 5.3 异常/错误
  (每条含 curl 报文 + 预期结果)

## 六、curl 速查表
```

**覆盖标准**:
- 每条规则 ≥ 2 条案例（通过 + 拒绝）
- 每个 C-E-A-R 场景 ≥ 1 条案例
- C×R 组合覆盖率 ≥ 85%
- R×E 组合覆盖率 ≥ 75%

---

### Phase 4: 决策记录

提取澄清文档的[待产品决策] + 影响报告中 🔴 高风险项 → `4-decision-log.md`

---

### 完成后

```
✅ 影响分析完成
├── 1-clarification.md       需求澄清(6维)
├── 2-impact-report.md       多维影响分析(N个维度+风险)
├── 3-test-cases.md          测试案例(回归+新增+curl报文)
└── 4-decision-log.md        决策记录

下一步: /biz-design 生成方案设计
```
