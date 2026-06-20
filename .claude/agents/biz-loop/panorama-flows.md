---
name: panorama-flows
description: 从 GitNexus 提取核心业务流程的端到端调用链
tools: Read, Write, ReadMcpResourceTool
---

## 业务流程提取

从 GitNexus processes 资源提取核心业务流程。

### 步骤

1. **读取流程**: `ReadMcpResourceTool` 读 GitNexus processes 资源（`gitnexus://repo/{name}/processes`）

2. **按业务域分组**: 利用 GitNexus cluster 分类，或按 process 名称前缀归类（支付/通知/对账/轮询/鉴权等）

3. **标注步骤类型**:
   - 入口: Controller/Handler 方法
   - 计算: 数据转换、金额计算
   - 决策: if/throw/状态判断
   - DB操作: DAO/Repository 调用
   - 外部调用: HTTP/RPC/MQ

4. **不预设节点名称**，如实反映 GitNexus 返回的 process 步骤

### 产出

`docs/biz-loop/panorama/flow-maps.md`，按业务域分组：

```markdown
# 核心业务流程

## 支付流程
### proc_125_topay: ToPay → GetBy
| 步骤 | 类型 | 类.方法 | 说明 |
|------|------|--------|------|
| 1 | 入口 | ScanPayController.toPay | 接收用户请求 |
| 2 | 计算 | RpTradePaymentManagerServiceImpl.initDirectScanPay | 校验+创建订单 |
| 3 | 外部调用 | WeiXinPayUtils.httpXmlRequest | 调用微信统一下单 |
| ...
```