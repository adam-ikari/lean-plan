# lean-plan

[English version](README.md)

面向按次 / 用量计费编码计划的精简实现规划技能。让每个请求都买到进展：批量合并工具调用，把切片交给最便宜的 worker，计划一次成型。

## 简介

按次计费或用量窗口下，稀缺资源是请求。这个技能减少请求数，但不靠跳过工作来省：省请求的办法是合并调用、选便宜的 worker，验证不能省，坏结果会花更多请求补。

## 核心原则

1. 批量。一次 assistant 回合就是一次 API 请求，所有独立的工具调用都放进同一回合。读多个文件、写多个文件、验证，各用一次回合完成。
2. 路由便宜。切片交给最便宜的 worker。默认 fast worker（`sonic`，只读用 `scout`），需要推理的活留在主会话。
3. 决定一次。计划一次成型，不反复确认，验证一次做完。

### Worker 分级

| 切片难度 | 在哪跑 | 原因 |
|---|---|---|
| 纯机械：样板、配置、文档、复制粘贴 | fast worker（`sonic`；只读 `scout`） | 约 3 请求一片，抵扣系数低 |
| 需判断：逻辑、测试交互、argparse、边界 | 主会话批量做，不派 | 派出去要 10+ 请求加编排加等待，主会话批量同价零开销 |
| 大到主会话装不下的推理片（完整子系统） | good worker（`task`） | 才值得付全能力 |

### 防滥用阈值

先估算单代理批量成本。低于 20 请求就自己做完，不派 worker。要派 worker，得有至少 6 个互不相干的文件域，且每个切片在主会话里做也要花 5 个以上请求。

## 安装

```bash
git clone https://github.com/adam-ikari/lean-plan.git ~/.agents/skills/lean-plan
```

触发词：`coding plan`、`implementation plan`、`task breakdown`、`多步实现`、`按次计费`、`省调用`、`调用次数`、`合并调用`、`批量调用`、`减少调用`、`限额`、`配额`、`fast worker`、`good worker`、`抵扣系数`。

## 实验数据

同一任务、同模型、隔离工作区并行跑，唯一差别是是否执行 lean-plan。请求数和 tokens 从转录的 `.message.usage` 字段实采，不是估算。

### 任务 A（单包 textstats，7 文件）

| 指标 | 无 skill | 使用 lean-plan |
|---|---|---|
| 请求数 | 20 | 9 |
| tokens | 583,279 | 276,813 |

### 任务 B（多模块 textmon，12 文件）

| 指标 | 无 skill | 使用 lean-plan |
|---|---|---|
| 请求数 | 15 | 10 |
| tokens | 325,363 | 311,204 |

### 结论

- 批量合并是主要的收益来源。任务 A 请求从 20 降到 9（-55%），tokens 差不多减半；任务 B 从 15 降到 10（-33%）。一个回合写完 7 个文件只算一次请求。
- fast worker 确实便宜。sonic 处理纯机械切片大约 3 个请求，task 处理推理切片要 13 个。所以推理切片留在主会话。
- 阈值有用。任务 B 第一次按「3 个文件域」就派 worker，4 个 worker 花了 39 个请求，比自然跑法的 15 个还多。改成「估算低于 20 不派」之后，正确选择不派 worker，10 个请求完成。
- 代理会低估自己的请求数。自己估 8 实际 9，估 7 实际 10，但实测数据确认下降存在。

### 测量方法

```bash
jq -s 'map(select(.type=="message" and .message.usage != null)) |
  {req: length, tok: (map(.message.usage.totalTokens)|add)}' <session>.jsonl
```

一条带 usage 的记录就是一个 assistant 回合，也就是一次 API 请求。

4 次运行：Baseline（自然）、Lean2（单包）、Lean3（多模块，调阈值前）、Lean4（多模块，调阈值后）。工作区和规格留在 `/tmp/abtest/`。

### 局限

每组只有一次运行，不是统计结论。本环境 token 成本字段全为 0，给不出美元数字。
