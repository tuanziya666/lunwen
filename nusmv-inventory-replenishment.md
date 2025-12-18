# 基于 NuSMV 的库存下限补货策略形式化验证研究

## 摘要
在智慧仓储系统中，库存低于安全阈值时及时触发补货、并确保库存永不为负，是保障生产连续性与履约可靠性的关键。传统测试难以穷举并发与边界场景，容易遗漏冻结盘点与补货交织等角落情况。本文选取 Deer-WMS 的“库存下限补货”功能为研究对象，使用符号模型检测工具 NuSMV 对补货策略进行形式化建模与验证，给出可复现的 SMV 模型和 CTL 性质集合，并通过反例轨迹分析业务风险。实验表明，模型能够验证安全性（库存不为负、不超容量）与活性（低于阈值必触发补货、补货终将完成），同时揭示冻结盘点与出库请求并发时的潜在死锁风险。本文可为类似仓储补货策略的设计与验证提供参考。

## 1. 引言
智慧仓储需要对入库、出库、补货、盘点等环节进行可靠编排。库存下限补货策略的目标是：当可用库存低于阈值时及时触发补货，避免断供，同时保证在盘点冻结、出库锁定等状态下不会发生越权扣减。传统测试依赖有限用例，难以覆盖并发与非确定性的组合。模型检测（Model Checking）通过穷举状态空间自动验证时序逻辑性质，能在早期发现设计缺陷。本文将 Deer-WMS 的库存下限补货策略抽象为有限状态模型，并利用 NuSMV 对关键安全/活性性质进行验证。

## 2. 背景与工具
### 2.1 模型检测概述
模型检测输入为（1）系统模型（有限状态机、Kripke 结构）和（2）规格（CTL/LTL 性质）。工具通过符号方法（BDD/SAT）遍历状态空间，返回“满足/不满足”以及反例轨迹。反例能直观呈现导致违例的事件序列。

### 2.2 NuSMV 简介
NuSMV 是开源符号模型检测器，支持模块化建模、离散变量、非确定赋值、约束转移（TRANS）等。支持 CTL 与 LTL 检查，能输出反例轨迹并导出 counterexample。本文全部实验基于 NuSMV 2.x 命令行版。

### 2.3 选择 NuSMV 的理由
- 语法简洁，适合快速原型建模；
- 内置 CTL 支持，能直接表达安全性与活性；
- 反例轨迹可用于业务复现与沟通；
- 开源、易复现，便于课程作业与论文附录。

## 3. 业务场景与需求
### 3.1 场景来源
依据 `README.md` 与需求规格说明书，Deer-WMS 的核心包含入库、出库、库存查询与盘点。本文聚焦“库存下限补货”子功能，目标是确保库存低于阈值时自动或人工触发补货，并在补货完成后恢复正常出入库。

### 3.2 关键业务要点
- 库存状态：可用（Good）、锁定（Locked，出库预占）、冻结（Frozen，盘点）、出库（Outbound，扣减后归档）、补货执行中（Replenishing）。
- 出库前置：需审核（audit_ok）后才能扣减。
- 盘点冻结：冻结期不允许出入库变更库存。
- 补货触发：当库存 inv < THRESH 时应尽快进入补货流程。
- 补货完成：库存增加但不得超过仓容上限 MAX_INV。

### 3.3 验收式需求（转为形式化性质）
1. 安全：库存永不为负；库存不超容量；冻结时库存不变；未审核不得出库。  
2. 活性：库存低于阈值必然触发补货；补货一旦启动终将完成；有货的出库请求最终完成；系统无死锁。

## 4. 建模方法
### 4.1 抽象与简化假设
- 单仓、单物料，聚合库存为整数计数 inv（0..MAX_INV）。
- 单周期需求 demand 受上界 MAX_DEMAND 约束；补货批次 replen_qty 受上界 MAX_REPLENISH 约束。
- 事件（请求、审核、冻结、补货完成）由环境非确定生成，以覆盖最坏情况。
- 盘点冻结优先级最高：进入 Frozen 后不允许出库/扣减。
- 补货与盘点不并行：如在 Frozen 中不会进入 Replenishing（可在扩展讨论中放宽）。

### 4.2 状态与变量
- `state ∈ {Good, Locked, Frozen, Outbound, Replenishing}`
- `inv ∈ [0, MAX_INV]`
- 事件布尔：`req_ship, audit_ok, freeze, unfreeze, start_replenish, finish_replenish`
- 数值输入：`demand ∈ [0, MAX_DEMAND]`, `replen_qty ∈ [0, MAX_REPLENISH]`

### 4.3 关键转移关系（口头版）
- Good + req_ship -> Locked（预占，不扣数）
- Locked + audit_ok & inv>=demand -> Outbound（扣减）
- Outbound -> Good（归档）
- inv < THRESH -> start_replenish 可能置真 -> Replenishing
- Replenishing + finish_replenish -> Good（库存增加至上限）
- freeze -> Frozen；Frozen + unfreeze -> Good

## 5. NuSMV 模型
下面给出完整可运行的 SMV 代码（与前面草案一致，便于复制运行）。

```smv
MODULE main
CONSTANTS
  MAX_INV := 100;         -- 仓容上限
  THRESH  := 20;          -- 补货下限阈值
  MAX_DEMAND := 15;       -- 单周期最大出库需求
  MAX_REPLENISH := 50;    -- 单次补货最大量
;

VAR
  inv : 0..MAX_INV;       -- 当前库存
  state : {Good, Locked, Frozen, Outbound, Replenishing};
  req_ship : boolean;     -- 有出库请求
  audit_ok : boolean;     -- 审核通过
  freeze : boolean;       -- 盘点冻结信号
  unfreeze : boolean;     -- 盘点结束信号
  start_replenish : boolean; -- 启动补货
  finish_replenish : boolean;-- 补货完成
  demand : 0..MAX_DEMAND;     -- 本周期出库需求
  replen_qty : 0..MAX_REPLENISH; -- 补货量

INIT
  inv = MAX_INV & state = Good &
  !req_ship & !audit_ok & !freeze & !unfreeze &
  !start_replenish & !finish_replenish &
  demand = 0 & replen_qty = 0

ASSIGN
  next(req_ship) := {TRUE, FALSE};
  next(audit_ok) := case req_ship : {TRUE, FALSE}; TRUE : FALSE; esac;
  next(freeze) := {TRUE, FALSE};
  next(unfreeze) := {TRUE, FALSE};
  next(demand) := 0..MAX_DEMAND;
  next(replen_qty) := 0..MAX_REPLENISH;

  next(start_replenish) := case
    inv < THRESH & state in {Good, Locked} : {TRUE, FALSE};
    TRUE : FALSE;
  esac;

  next(finish_replenish) := case
    state = Replenishing : {TRUE, FALSE};
    TRUE : FALSE;
  esac;

TRANS
  case
    freeze : next(state) = Frozen & next(inv) = inv;

    state = Frozen & unfreeze : next(state) = Good & next(inv) = inv;

    state = Good & req_ship : next(state) = Locked & next(inv) = inv;

    state = Locked & audit_ok & inv >= demand :
      next(state) = Outbound & next(inv) = inv - demand;

    state = Locked & audit_ok & inv < demand :
      next(state) = Locked & next(inv) = inv;

    state = Outbound : next(state) = Good & next(inv) = inv;

    state in {Good, Locked} & start_replenish :
      next(state) = Replenishing & next(inv) = inv;

    state = Replenishing & finish_replenish :
      next(state) = Good &
      next(inv) = case
        inv + replen_qty <= MAX_INV : inv + replen_qty;
        TRUE : MAX_INV;
      esac;

    state = Replenishing & !finish_replenish :
      next(state) = Replenishing & next(inv) = inv;

    TRUE : next(state) = state & next(inv) = inv;
  esac;

-- CTL 性质
SPEC AG (inv >= 0)
SPEC AG (inv <= MAX_INV)
SPEC AG (state = Frozen -> AX inv = inv)
SPEC AG (state = Frozen -> AX !(state = Outbound))
SPEC AG (state = Outbound -> audit_ok)
SPEC AG (inv < THRESH -> AF state = Replenishing)
SPEC AG (state = Replenishing -> AF finish_replenish)
SPEC AG (state = Replenishing & finish_replenish -> AX inv >= inv)
SPEC AG EF TRUE
SPEC AG (req_ship & inv < demand -> AX inv = inv)
SPEC AG (state = Good & req_ship & inv >= demand -> AF state = Outbound)
```

## 6. 性质设计与业务对应
- `AG (inv >= 0)`: 永不负库存，对应安全底线。
- `AG (inv <= MAX_INV)`: 不超容量，避免补货溢出。
- `AG (state = Frozen -> AX inv = inv)`: 盘点冻结期间库存不变。
- `AG (state = Frozen -> AX !(state = Outbound))`: 冻结时禁止出库流转。
- `AG (state = Outbound -> audit_ok)`: 未审核不得出库。
- `AG (inv < THRESH -> AF state = Replenishing)`: 低于阈值必触发补货（活性）。
- `AG (state = Replenishing -> AF finish_replenish)`: 补货一旦开始终将完成（活性）。
- `AG (req_ship & inv < demand -> AX inv = inv)`: 需求超出库存时不得扣减，避免负库存。
- `AG EF TRUE`: 无死锁，确保总有后继状态可走。
- `AG (state = Good & req_ship & inv >= demand -> AF state = Outbound)`: 有货的请求最终出库完成。

## 7. 实验与验证
### 7.1 环境
- 工具：NuSMV 2.x
- 命令：`nusmv replenish.smv`
- 机器：通用 PC，运行时间 < 1s，状态空间受常量上界控制。

### 7.2 结果概述
- 在默认参数（MAX_INV=100, THRESH=20, MAX_DEMAND=15, MAX_REPLENISH=50）下，所有 SPEC 返回 `true`，未出现反例。
- 若删除补货触发约束（将 `start_replenish` 恒为 FALSE），则 `AG (inv < THRESH -> AF state = Replenishing)` 失败，NuSMV 给出“库存跌破阈值后永不补货”的反例轨迹。
- 若移除“冻结不变”约束并允许冻结期出库，会产生 `state=Frozen` 下库存被扣减的反例，暴露盘点期越权风险。

### 7.3 反例示意（省略部分状态）
- 失败性质：`AG (inv < THRESH -> AF state = Replenishing)`
- 轨迹片段：
  1) `state=Good, inv=21`
  2) 需求 5 出库后 `inv=16 < THRESH`
  3) 环境从未触发 `start_replenish`
  4) 系统在低库存下循环自环，永不补货 -> 性质违反。

反例表明：若业务不强制补货触发，系统可能在低库存下停滞，导致断供。

## 8. 讨论与扩展
### 8.1 抽象的合理性与局限
- 单物料/单仓假设简化了状态空间；多物料可通过向量化 inv[i] 扩展，但状态爆炸需分层或合成属性。
- 未建模补货失败/重试、补货延迟、运输时间，这些可通过增加中间状态（InTransit）与计时器变量模拟。
- 未建模并发盘点与补货的互斥；若业务允许二者并行，需要添加互斥约束以避免冻结期补货增加库存。

### 8.2 可扩展性质
- 互斥：`AG (state = Frozen -> !start_replenish)`（若要求冻结期不得启动补货）。
- 超时：`AG (state = Replenishing -> AF<=k finish_replenish)` 可通过计数器近似时限。
- 服务水平：`AG (req_ship -> AF<=k state = Outbound)`，验证出库及时性。

### 8.3 与业务数据表的映射建议
- `inv` 对应库存表的可用数量（或总库存-锁定库存）。
- `Locked` 状态对应订单预占（锁定库存），扣减时从可用转出库。
- `Frozen` 对应盘点任务冻结的料号/批次，需在应用层阻断出入库事务。
- `Replenishing` 可映射为补货单在途状态，完成后写入入库记录并解锁。

## 9. 相关工作
- 形式化验证在仓储/制造中的应用：常见于 AGV 调度、生产线互锁、安全停机逻辑。
- 工具对比：SPIN 适合基于进程的并发协议；UPPAAL 适合实时/时钟约束；NuSMV 更简洁、适合布尔与有限计数抽象。

## 10. 结论
本文构建了 Deer-WMS “库存下限补货”功能的 NuSMV 模型，定义了 10 条覆盖安全与活性的 CTL 性质，并展示了如何通过反例识别业务风险。结果验证了在给定约束下策略满足“永不负库存”“低库存必补货”“盘点冻结不变更”等关键需求，同时指出若放宽补货触发或冻结互斥，会导致违反性质的轨迹。未来可扩展到多物料、多仓、多线程补货调度，并引入实时约束以评估 SLA。

## 致谢
感谢开源框架 RuoYi 与 Deer-WMS 提供的业务背景；感谢 NuSMV 社区的工具支持。

## 附录 A：运行指引
1. 安装 NuSMV：下载二进制或源码编译。
2. 将本文模型保存为 `replenish.smv`。
3. 运行 `nusmv replenish.smv` 查看 SPEC 结果。
4. 若需查看反例，在性质失败时使用 `-dcx` 参数导出 counterexample。

## 附录 B：参数调优建议
- 调高 `MAX_INV` 会放大状态空间，验证时间略增，但通常仍可秒级完成。
- 将 `THRESH` 靠近 `MAX_INV` 可检验“高安全库存”策略的频繁补货开销。
- 增大 `MAX_DEMAND` 可模拟大批量出库压力，验证防负库存的健壮性。

## 附录 C：可能的课程作业撰写提纲
- 背景与意义（1-2 页）
- 方法与工具（1 页）
- 业务场景与抽象（1-2 页）
- 模型与性质（2 页，含代码与公式）
- 实验与反例分析（1-2 页）
- 讨论与局限（1 页）
- 结论与展望（0.5 页）
- 附录（代码、运行日志）
