---
name: ddd-analysis
description: "DDD domain analysis — extract entities, activities, business rules, events, and flows from backend code; generate test scenarios; detect rule gaps. Examples: 分析initPay方法涉及哪些DDD元素, 提取这段代码的领域事件和规则, 生成这个方法的业务场景测试案例, 检查订单支付流程缺失了哪些校验"
---

# DDD Domain Analysis & Rule Gap Detection

## When to Use

- "分析这个方法的 DDD 元素: 实体、活动、规则、事件、流程"
- "帮我提取 XX 的领域事件和业务规则"
- "基于 DDD 分析生成业务场景和测试案例"
- "检查这个方法流程中有哪些缺失的业务规则"
- "审计这些代码的规则覆盖完整度"

## Workflow

```
1. gitnexus_context({name: "<entry_method>"})                            → Get method code + callees
2. Read source files for each entity/BO/VO class                         → Extract entity attributes
3. gitnexus_context({name: "<each_callee>"})                             → Trace full call chain
4. DDD analysis output to .gitnexus/<method>-ddd-analysis.md             → Entities, Rules, Events, Flows
5. Test scenarios output to .gitnexus/<method>-test-scenarios.md          → GWT scenarios × rules × events
6. Rule gap check output to .gitnexus/<method>-rule-gap-analysis.md       → 12-category SBVR scan
```

## Analysis Outputs

### Phase 1: DDD Elements (生成到 .gitnexus/)

分析目标方法涉及的 DDD 元素，输出包含:

```
1. 流程总览 (Flow): ASCII 流程图，标注每个步骤调用的方法和分支
2. 实体 (Entities): 聚合根、实体、值对象的属性表，标注来源
3. 活动 (Activities): 应用层/领域服务方法清单和职责
4. 业务规则 (Business Rules): 按 SBVR 12 类体系分类
   - 校验规则、存在性规则、安全规则、不变规则
   - 计算规则、状态转换规则、路由规则、幂等规则
   - 时间规则、补偿规则、审计规则、通知规则
5. 领域事件 (Domain Events): 隐式/显式事件，携带数据，所属聚合，时序图
6. DDD 分层映射: 接口层 → 应用服务 → 领域模型 → 基础设施
```

### Phase 2: Test Scenarios (生成到 .gitnexus/)

基于 Phase 1 的规则/事件/流程生成:

```
1. 场景-规则-事件映射矩阵: 列出所有业务场景覆盖的规则和事件
2. 业务场景 (S1-S14): Given(聚合状态) × When(触发动作) × Then(事件序列+页面)
3. 测试案例 (TC1-TC15): GWT 格式，含事件验证+规则验证+状态验证+数据库断言
4. 覆盖矩阵: 规则覆盖率、事件覆盖率、聚合覆盖率统计
```

### Phase 3: Rule Gap Detection (生成到 .gitnexus/)

用 12 类 SBVR 规则体系逐类对比代码实现:

```
对每类规则:
  - 理论要求: 该类规则应覆盖的范围
  - 代码现状: 已实现的规则
  - 缺失场景: GAP-N 编号，含风险等级 (P0/P1/P2)、代码位置、修复建议
  - 风险等级: P0=资金安全/数据一致性, P1=业务逻辑缺陷, P2=健壮性/安全增强
```

## Rule Classification System (SBVR + BRG)

分析时强制扫描这 12 类规则，每类必查:

| # | 类别 | 检查要点 |
|---|------|----------|
| 1 | 校验规则 | 所有外部输入字段的类型/格式/范围/必填/长度/组合约束 |
| 2 | 存在性规则 | 跨聚合引用的存在性 + 各实体的 status/启用状态 |
| 3 | 安全规则 | 签名验证、IP白名单、防重放、时效性、防暴力破解 |
| 4 | 不变规则 | 聚合内部状态一致性、终态不可变、属性继承约束 |
| 5 | 计算规则 | 派生字段公式、签名算法、费率查询、金额调整 |
| 6 | 状态转换规则 | 聚合状态机的每个转换守卫、非法转换拦截 |
| 7 | 路由规则 | 所有 if/switch 分支的完备性、else 兜底 |
| 8 | 幂等规则 | 创建幂等、支付幂等、金额幂等 |
| 9 | 时间规则 | 时间字段一致性、过期判断、解析异常处理 |
| 10 | 补偿规则 | 事务边界、第三方失败回滚、异常分支的数据清理 |
| 11 | 审计规则 | 关键变更日志、操作人记录、错误信息脱敏 |
| 12 | 通知规则 | 关键事件的外部通知义务 |

## Completeness Metrics

分析完成后输出量化指标:

| 指标 | 含义 | 目标 |
|------|------|------|
| DPC | 决策点覆盖率 (规则映射的决策点/总决策点) | 100% |
| APC | 属性约束覆盖率 (有规则的属性/总属性) | >80% |
| ERC | 事件-规则完备性 (有触发规则的事件/总事件) | 100% |
| STC | 状态转换覆盖率 (已覆盖转换/状态机总转换) | 100% |

## Where to Write

- DDD 分析: `.gitnexus/<entryMethod>-ddd-analysis.md`
- 测试案例: `.gitnexus/<entryMethod>-test-scenarios.md`
- 规则审查: `.gitnexus/<entryMethod>-rule-gap-analysis.md`

## Example

User: "分析 ScanPayController.initPay 方法涉及哪些 DDD 元素"

```
1. gitnexus_context({name: "initPay", repo: "roncoo-pay"})
   → ScanPayController.initPay(), calls CnpPayService.checkParamAndGetUserPayConfig,
     RpTradePaymentManagerService.initDirectScanPay / initNonDirectScanPay

2. Read ScanPayRequestBo.java, RpUserPayConfig.java, RpTradePaymentOrder.java,
      RpPayWay.java, RpUserInfo.java, ScanPayResultVo.java, RpPayGateWayPageShowVo.java

3. gitnexus_context on checkParamAndGetUserPayConfig, initDirectScanPay, initNonDirectScanPay
   → Full call chain with implementations

4. Write .gitnexus/initPay-ddd-analysis.md
   → 4 entities, 5 value objects, 11 domain events, 43 rules across 6 categories,
     flow diagram, DDD layer map

5. Write .gitnexus/initPay-test-scenarios.md
   → 14 business scenarios, 15 test cases, coverage matrix (100% rules, 100% events)

6. Write .gitnexus/initPay-rule-gap-analysis.md
   → 25 gaps found (7 P0, 10 P1, 8 P2), completeness score 37% missing
```
