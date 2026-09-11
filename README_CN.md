# lean-plan

[English version](README.md)

面向按次 / 用量计费编码计划的精简实现规划技能。少花请求，不砍工作。

## 是什么

按次计费或用量窗口下，稀缺资源是请求。这个技能减少请求数，但不靠跳过工作来省：省请求的办法是合并调用、选便宜的 worker，验证不能省，坏结果会花更多请求补。

## 特性

- 合并工具调用。一次 assistant 回合就是一次 API 请求，独立的工具调用都放进同一回合：读多个文件一个回合、写多个文件一个回合、验证一个回合。
- 按难度分配 worker。纯机械切片交给 fast worker（`sonic`，只读用 `scout`）；需要推理的切片留在主会话；大到主会话装不下的完整子系统才用 good worker（`task`）。
- 防滥用阈值。先估算单代理批量成本，低于 20 请求就自己做完。要派 worker，得有至少 6 个互不相干的文件域，且每个切片在主会话里做也要花 5 个以上请求。
- 计划一次成型。完整任务清单、文件归属、worker 分级、验证步骤一次给出。没有确认循环，验证一次做完，每个域一次提交。
- 保住缓存前缀。一个任务一个会话、工具面固定、少派 worker，缓存前缀不被打断，后面的请求主要为增量付费。实测命中率：干净单会话跑到 83%，有返工的落到 30%。命中率是回合数的副产品，不要为抬命中率加回合。

> [!NOTE]
> worker 是成本不是默认选项。派一个 worker 固定花约 3 个请求，外加失败重试的风险；一个分错等级的推理切片能烧掉 10+ 个请求。

## 安装

推荐用 skills CLI 一次装到所有 agent：

```bash
npx skills add adam-ikari/lean-plan -g -y
```

CLI 会写入 `~/.agents/skills`（通用目录）并 symlink 到各 agent 的 skills 目录。加 `-a '*'` 强制装到全部 agent，或 `-a <agent>` 只装一个。

按 agent 手动安装，clone 到对应目录：

| Agent | 目录 |
|---|---|
| Claude Code | `~/.claude/skills/lean-plan` |
| Codex | `~/.codex/skills/lean-plan` |
| Cursor | `~/.cursor/skills/lean-plan` |
| Gemini CLI | `~/.gemini/skills/lean-plan` |
| Kilo Code | `~/.kilocode/skills/lean-plan` |
| Roo Code | `~/.roo/skills/lean-plan` |
| Windsurf | `~/.windsurf/skills/lean-plan` |
| Amp、Antigravity、Cline、OpenCode、Warp 等 | `~/.agents/skills/lean-plan` |

```bash
git clone https://github.com/adam-ikari/lean-plan.git ~/.claude/skills/lean-plan
```

触发词：`coding plan`、`implementation plan`、`task breakdown`、`多步实现`、`按次计费`、`省调用`、`调用次数`、`合并调用`、`批量调用`、`减少调用`、`限额`、`配额`、`fast worker`、`good worker`、`抵扣系数`。

## 用法

1. 一个回合给出完整计划：任务清单、文件归属、worker 分级、验证步骤。不确认循环。
2. 所有独立工具调用放进同一回合。手头有多个可并行的调用时，别一个个单发。
3. 只有阈值判断 worker 划算才派。按难度分级，默认 fast worker。
4. 合并后的结果做一次批量验证：构建、相关测试、冒烟，全在一个回合。
5. 每个域提交一次。

> [!WARNING]
> 不要为省请求跳过验证。没验证的「完成」后面要花更多请求补。

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
- 缓存命中率遵循同一套规则。干净的单会话批量跑到 83% 缓存读取，多 worker 或有返工的落到 30% 到 57%。少开新会话、中途不动工具、少派 worker、不重复读文件，都在保住前缀。但命中率是回合数的副产品：某次 6 请求的运行命中率只有 44% 却照样完工，而请求数最少的运行不一定最便宜——长文档被截断后串行补读，每一页都按全新输入付费。

### 测量方法

```bash
jq -s 'map(select(.type=="message" and .message.usage != null)) |
  {req: length, tok: (map(.message.usage.totalTokens)|add)}' <session>.jsonl
```

一条带 usage 的记录就是一个 assistant 回合，也就是一次 API 请求。

4 次运行：Baseline（自然）、Lean2（单包）、Lean3（多模块，调阈值前）、Lean4（多模块，调阈值后）。工作区和规格留在 `/tmp/abtest/`。

### 局限

每组只有一次运行，不是统计结论。本环境 token 成本字段全为 0，给不出美元数字。
