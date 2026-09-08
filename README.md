# lean-plan

面向**按次 / 用量计费**编码计划的精简实现规划技能（5 小时窗口、Claude Pro/Max、quota 计划）。目标不是最小化请求数本身，而是**让每个请求都买到进展**：批量合并工具调用、按难度把切片路由给最便宜的 worker、计划一次成型。

## 核心原则（按优先级）

1. **批量** — 1 个 assistant 回合 = 1 次 API 请求。所有独立工具调用塞进同一回合：一次读多文件、一次写多文件、一次验证。这是最大的杠杆。
2. **路由便宜** — 每个切片交给能干它活的最便宜 worker。默认 fast worker（`sonic` / `scout`），推理片留主会话批量做。
3. **决定一次** — 计划全量成型，无确认循环，一次性批量验证。

### Worker 分级

| 切片难度 | 在哪跑 | 原因 |
|---|---|---|
| 纯机械：样板/配置/文档/复制粘贴 | **fast worker**（`sonic`；只读用 `scout`） | 便宜（约 3 请求/片），抵扣系数低 |
| 需判断：逻辑、测试交互、argparse、边界 | **主会话批量做，不派** | 派 good worker 要 10+ 请求 + 编排 + 等待，主会话批量同价零开销 |
| 大到主会话装不下的推理片（完整子系统） | **good worker**（`task`） | 才值得付全能力 |

**防滥用阈值**：先算账——单代理批量估算 <20 请求就不派 worker；派 worker 需同时满足 ≥6 个独立文件域、每个片在主会话做要 ≥5 请求、片之间真独立。

## 安装

```bash
mkdir -p ~/.agents/skills && git clone https://github.com/adam-ikari/lean-plan.git ~/.agents/skills/lean-plan
```

触发词：`coding plan`、`implementation plan`、`task breakdown`、`多步实现`、`按次计费`、`省调用`、`调用次数`、`合并调用`、`批量调用`、`减少调用`、`限额`、`配额`、`fast worker`、`good worker`、`抵扣系数`。

## 实验统计数据（真实 A/B 测试）

同一任务、同模型（`home-llm-gateway/stack/medium`）、隔离工作区并行运行，唯一差异 = 是否执行 lean-plan 协议。请求数与 tokens 从运行转录的 `.message.usage` 字段**实采**，非估算。

### 任务 A — 单包（textstats，7 文件）

| 指标 | 无 skill | 旧版 lean | 新版 lean |
|---|---|---|---|
| 请求数 | 20 | 28 | **9** |
| tokens | 583,279 | 666,145 | **276,813** |
| 墙钟 | 327.1s | 270.9s | **81.3s** |
| 测试通过 | 9 | 11 | 7 |
| worker | 0 | 3 (task) | 0 |
| git 提交 | 1 | 1 | 1 |

### 任务 B — 多模块（textmon，12 文件）

| 指标 | 无 skill | 新版调阈值前 | 新版调阈值后 |
|---|---|---|---|
| 请求数 | 15 | 39 | **10** |
| tokens | 325,363 | 885,199 | **311,204** |
| 墙钟 | 341s | 979s | 442s |
| 测试通过 | 19 | 19 | 13 |
| worker | 0 | 4 (1 task + 3 sonic) | **0（算术判定单代理）** |
| git 提交 | 3 | 1 | 1 |

### 结论

- **批量合并是主杠杆**：任务 A 请求 20→9（**-55%**）、tokens -53%；任务 B 15→10（**-33%**）。一次回合写 7 文件 = 1 请求。
- **fast worker 真便宜**：sonic 干纯机械片只需 **3 请求/片**；good worker（task）干推理片 churn 到 **13 请求** → 所以「推理片留主会话」写进技能。
- **防滥用阈值有效**：初版阈值（3 文件域）导致任务 B 派 4 worker = 39 请求，反输给自然基线 15；调后（估算 <20 不派）正确选择 0 worker = 10 请求。
- **已知弱点**：代理自估请求数系统性偏低（自估 8 实 9、自估 7 实 10），但 jsonl 实采仍确认大幅下降。

### 测量方法

```bash
# 每个 assistant 回合 = 1 条带 usage 的记录 = 1 次 API 请求
jq -s 'map(select(.type=="message" and .message.usage != null)) |
  {req: length, tok: (map(.message.usage.totalTokens)|add)}' <session>.jsonl
```

5 次独立运行：`Baseline`（自然）/ `Lean`（旧版）/ `Lean2`（新版，任务 A）/ `Lean3`（新版调阈值前，任务 B）/ `Lean4`（新版调阈值后，任务 B）。工作区与规格留存于 `/tmp/abtest/`。

### 局限

n=1/边，单任务单样本，非统计结论；墙钟受网关延迟影响有噪声；本环境 token 成本字段全 0，无法换算美元。
