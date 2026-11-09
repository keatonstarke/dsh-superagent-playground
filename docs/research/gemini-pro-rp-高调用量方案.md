# Gemini Pro 级模型跑 RP：高调用量方案（2026-11）

> 状态：调研稿（v1）
> 采样时间：2026-11
> 反代全流程（账号来源/凭证/项目/部署）见 `docs/research/gemini-反代全流程-账号与项目.md`。
> 后续：3.8 Flash 的落地方案见 `docs/research/gemini-3.8-flash-rp-落地方案.md`（无 Pro 免费层时的主力方案）。
> 承接：`docs/research/gemini-白嫖调研.md`（下称"前篇"）。前篇回答"有哪些白嫖路径"，本篇只回答一个问题：**用 Gemini 至少 Pro 档的模型跑 RP，且调用量很大，怎么搭、花多少钱、哪里会崩。**
> 口径：Google 官方文档 + 社区公开实测 + 工程推演。价格与配额以官方页面为准，且 2026-12-01 / 2027-01-01 两个涨价节点会改变结论。
> 标注：`[官方]` 可查文档；`[实测]` 社区复现；`[估算]` 本文推算，只取量级。

---

## 0. 结论摘要

| 问题 | 结论 |
| --- | --- |
| Pro 级模型还能白嫖吗 | **API 免费层已经没有 Pro**：`gemini-3.1-pro-preview` 免费层标 "Not available"，`gemini-4-argon` 同样无免费层 `[官方]`。只剩订阅配额（Antigravity / Gemini App）和灰产号池两条路 |
| 最便宜的"合法 Pro" | Google AI Pro **$19.99/月**，在 Antigravity 里可用 **Gemini 4 Argon**（付费订阅，不含试用）；Ultra 5x/20x 才有 3.1 Pro 与更高配额 `[官方]` |
| 最稳的工程路径 | 付费 API：**3.1 Pro + 显式缓存 + `thinking_level=low`**，40k 输入 / 900 输出 / 90% 缓存命中时约 **$0.026/轮** `[估算]` |
| 高调用量的真正瓶颈 | 不是并发，是 **token 成本** 和 **Tier 花费上限**（Tier 1 仅 $10/10 分钟，约 $1,440/日）`[官方]` |
| 降本杠杆排序 | 缓存（输入 5~10×）> 模型分层（总成本 −60%~−80%）> thinking/输出长度（输出 −40%~−70%）> 摘要压缩 > Batch/Flex（−50%，非实时） |
| 订阅 vs API 的临界点 | 单账号月产 **> 约 770 轮**（按 3.1 Pro API 价折算）订阅就更便宜；实际配额通常远高于此，所以"订阅套利"在经济上成立，但多账号有 **封号实证** |
| RP 质量 | 3.1 Pro 文笔上限高，但有一组稳定退化：时间漂移、体格描写通胀、感官复读、特质复述、空间错误、角色全知 `[实测]`。必须用状态账本 + 反复读规则 + 自动评测压住 |
| 内容边界 | 四类可调过滤器默认 **Off**，但核心伤害保护不可调、模型仍会自拒 `[官方]`。Gemini 不是硬核 explicit 的合适载体，硬需求要换模型族或本地 |
| 一句话 | **Pro 级 RP 的成本问题已经变成"缓存 + 分层 + 配额套利"的工程问题，不再是"找免费 Key"的问题。** |

---

## 1. 范围与口径

### 1.1 什么叫 RP（本文的定义）

不是"让模型写一段小说"，而是**多轮、有状态、有角色约束的互动叙事**：

- 有角色卡（外貌、口吻、欲望、恐惧、知识边界）
- 有世界观/设定集（lore），且不能每轮全量塞进去
- 有会话状态（时间、地点、伤势、物品、关系、承诺）
- 有"玩家角色"与"NPC/旁白"的视角隔离
- 单会话几十到几百轮，跨会话还要记忆

这决定了 token 结构和成本模型：**输入远大于输出，且输入随轮数增长**。

### 1.2 一轮 RP 的 token 构成（本文的统一假设）

| 组成 | 典型 token | 是否稳定 | 备注 |
| --- | --- | --- | --- |
| 系统规则 / 风格圣经 | 800 ~ 1,500 | 稳定 | 放最前，做缓存前缀 |
| 角色卡（1~3 人） | 600 ~ 2,000 | 稳定 | 结构化字段，别写成散文 |
| 世界观检索片段 | 1,000 ~ 4,000 | 半稳定 | 按场景检索，别全量注入 |
| 状态账本（ledger） | 300 ~ 800 | 每轮变 | JSON/表格式，机器可校验 |
| 滚动摘要 | 500 ~ 1,500 | 每 N 轮变 | 用 Flash 生成 |
| 近期原文窗口 | 6,000 ~ 25,000 | 每轮变 | 10~20 轮，按模型窗口调 |
| 用户输入 | 50 ~ 300 | 每轮变 | |
| **输入合计（中期）** | **~40,000** | | 早期 10k，后期 80k+ |
| 输出（正文） | 400 ~ 1,200 | | |
| 输出（thinking） | 0 ~ 4,000 | | 不控就是纯浪费 |

> 本文所有单价按"40k 输入 + 900 输出（含 thinking）"计算，除非另注。

### 1.3 与前篇的关系

前篇的结论是"白嫖的真实成本在协议适配、号池保鲜和失效应急"。本篇把场景收窄到 **Pro 级 + 高调用量**，结论会变：

- 前篇的主力是 CLI / Antigravity 的免费额度；2026-06-18 之后 **Gemini CLI 已被 Antigravity CLI 取代**，免费层只剩 3.8 Flash `[官方]`
- 前篇按"号池"算 TCO；本篇按"**每轮成本 + 配额上限 + 质量损失**"算 TCO
- 前篇的 Web Cookie 反代在 RP 场景基本不可用：长上下文 + 流式 + 高频，Cookie 生命周期撑不住

---

## 2. 2026-11 的模型与供给现状

### 2.1 API 模型与价格 `[官方]`

| 模型 | 输入 $/1M | 缓存输入 $/1M | 输出 $/1M（含 thinking） | 上下文 | 输出上限 | 免费层 |
| --- | --- | --- | --- | --- | --- | --- |
| `gemini-3.1-pro-preview` | 2.00（≤200k）<br>4.00（>200k） | 0.20 / 0.40 | 12.00 / 18.00 | 1,048,576 | 65,536 | 无 |
| `gemini-4-argon` | 2.00（至 11-30）<br>4.00（12-01 起） | 0.10 / 0.20（仅隐式） | 10.00（至 11-30）<br>20.00（12-01 起） | 1,048,576 | 65,536（long decode 可到 1M） | 无 |
| `gemini-3.8-flash` | 0.75（至 12-31）<br>1.50（2027-01-01 起） | 0.075 / 0.15 | 3.75 / 7.50 | 1,048,576 | 65,536 | 有 |
| `gemini-3.5-flash-lite` | 0.30 | 0.03 | 2.50 | 1,048,576 | 65,536 | 有 |

几个会直接影响 RP 的事实：

1. **3.1 Pro 仍是 preview**，2026-02-19 发布，至今没有下线日期，但 preview 模型随时可能被替换（前代 3 Pro preview 从发布到关停不到四个月）`[官方]`。
2. **3.1 Pro 默认 thinking = high**，且不支持 `minimal`。RP 不控 thinking，等于每轮多付 1k~4k 输出 token 的钱，还更慢、更容易写成"分析报告"`[官方]`。
3. **Argon 的缓存只有隐式**，不能手动 pin 前缀；3.1 Pro / Flash 支持显式缓存（Beta，默认 TTL 1 小时）`[官方]`。

4. **2.5 Pro 已废弃**：2026-12-01 起所有 `gemini-2.5-pro` 请求自动路由到 3.8 Flash 并按 Flash 计费。老 preset 里写死 2.5 Pro 的，12 月后会"悄悄换模型" `[官方]`。
5. **Batch 五折、Flex 五折、Priority 贵 75%~100%**：Batch 目标 24 小时内完成，适合离线批量生成；Flex 是同步但延迟不保证；Priority 用于首 token SLO `[官方]`。

> Argon 的官方定位是 coding / reasoning / agent，不是写作模型；它在 RP 上的文风没有像 3.1 Pro 那样的社区证据。选它是因为「订阅里能拿到的最强模型」，不是因为「最适合写小说」。正式投产前必须用 §6.5 的评测集自测，不要默认它比 3.1 Pro 更会写。

### 2.2 三张网：App / Antigravity / API

| 维度 | Gemini App（网页/手机） | Antigravity（App/CLI/IDE） | Gemini API |
| --- | --- | --- | --- |
| 计费 | 订阅（Free/Plus/Pro/Ultra） | 订阅配额（5 小时窗口 + 周上限） | 按 token |
| Pro 级模型 | Pro 档起可用 3.1 Pro；免费档 "varying access" | Pro 档：Argon；Ultra：Argon + 3.1 Pro；Free：仅 3.8 Flash | 3.1 Pro / Argon 直接可调 |
| 上下文 | Free 32k / Plus 128k / Pro·Ultra 1M | 与模型一致（1M） | 1M |
| 适合 | 人工玩、调试 prompt | 高产作者、脚本化批跑 | 产品、服务、可观测的高并发 |
| 关键限制 | 配额不透明，按"复杂度"扣 | 配额不透明，按"工作量"扣；第三方工具复用登录会封号 | Tier 花费上限、RPM/TPM |
| 数据用途 | 免费档用于改进产品 | 同左 | 免费层用于改进产品；付费层不用 `[官方]` |

**结论**：想做"高调用量"，只有 API 和 Antigravity 订阅两条路值得工程化；App 是人工入口，不是后端。

### 2.3 免费层的真相（2026-11）

- **API 免费层**：只有 Flash / Flash-Lite 系列；Pro / Argon 一律 "Not available" `[官方]`。Gemini CLI 文档口径的免费 API Key 是 **250 请求/日且仅 Flash** `[官方]`。
- **Antigravity 免费层**：只有 3.8 Flash，配额**按周刷新**，且"有意义的额度"没有公开数字 `[官方]`。
- **Gemini App 免费层**：3.6 Flash + "varying access to 3.1 Pro"，上下文 32k `[官方]`。32k ≈ 48 页，长团本很快见底。

所以"至少 Pro + 大调用量 + 免费"这三个条件**在 2026-11 不可能同时成立**。任何声称能做到的，要么是号池（灰），要么是共享池（内容外泄 + 随时挂），要么是拿 Flash 冒充 Pro。

### 2.4 2026 年影响 RP 的变更时间线

| 日期 | 变更 | 对 RP 的影响 |
| --- | --- | --- |
| 2026-02-19 | 3.1 Pro Preview 发布 | 当前唯一的 Pro 文本模型 |
| 2026-06-18 | Gemini CLI 被 Antigravity CLI 取代（免费/One 用户） | 老教程里的 `gemini` 命令与配额全部失效 |
| 2026-07-21 | `temperature` / `top_p` / `top_k` 废弃；`thinking_budget` → `thinking_level`；prefill 移除（3.8 Flash 起） | 老 SillyTavern preset 必须删采样参数和预填 |
| 2026-09-02 | 3.8 Flash 发布（$0.75/$3.75 intro） | 日常轮次的最佳性价比 |
| 2026-09-18 | 2.5 系列对新用户关闭 | 新项目别再用 2.5 Pro |
| 2026-10-13 | Gemini 4 Argon GA，进入 Antigravity（Pro 付费档） | 订阅路线的质量上限提高 |
| 2026-11-02 | 2.5 Pro 正式废弃（12-01 起路由到 3.8 Flash） | 老模型字符串必须改 |
| 2026-12-01 | Argon API 价格翻倍（$2/$10 → $4/$20） | 12 月前把预算模型换成 3.1 Pro 或订阅 |
| 2027-01-01 | 3.8/3.7/3.6 Flash 价格翻倍（$0.75/$3.75 → $1.50/$7.50） | 年度预算要按新价重算 |

**这条时间线的工程含义**：把"模型字符串、价格、配额"全部做成配置，不要硬编码；每次大版本发布前后留 1~2 周适配窗口（前篇的协议抖动结论在这里同样成立）。

---

## 3. 单位经济学：一轮 RP 到底多少钱

### 3.1 单轮成本（40k 输入 / 900 输出）

缓存命中按 90% 输入计（36k 命中，4k 未命中）：

| 模型 | 无缓存/轮 | 90% 缓存/轮 | 说明 |
| --- | --- | --- | --- |
| 3.1 Pro（≤200k） | $0.0908 | **$0.0260** | 显式缓存可控 |
| Argon（至 11-30） | $0.0890 | **$0.0206** | 仅隐式缓存，命中率不可控 |
| Argon（12-01 起） | $0.1780 | **$0.0412** | 涨价后明显贵于 3.1 Pro |
| 3.8 Flash（2026） | $0.0334 | **$0.0091** | 免费层也有 |
| 3.8 Flash（2027） | $0.0668 | **$0.0182** | 涨价后接近 3.1 Pro 现价 |
| 3.5 Flash-Lite | $0.0143 | **$0.0045** | 只适合工具调用/短场景 |

> 计算示例（3.1 Pro）：`4k×$2/M + 36k×$0.2/M + 900×$12/M = $0.008 + $0.0072 + $0.0108 = $0.026`。

### 3.2 会话成本是二次的

每轮都要重发历史，未压缩时输入 token 随轮数近似二次增长：

```
Input_total(N) ≈ N × P + a × N(N+1)/2
P = 固定前缀（规则+角色卡+账本+摘要）
a = 每轮新增 token（用户输入 + 模型正文，约 600~900）
```

以 P=6k、a=700、3.1 Pro 为例（这里刻意用较小的固定前缀和 20k 窗口上限，便于看清二次增长；§3.1 的 40k 是中期实际口径）：

| 轮数 | 未压缩输入总量 | 无缓存（含输出） | 每 20 轮摘要 | 摘要 + 90% 缓存 |
| --- | --- | --- | --- | --- |
| 20 | 0.27M | $0.75 | $0.75 | $0.32 |
| 50 | 1.19M | $2.92 | $1.92 | $0.80 |
| 100 | 4.13M | $9.35 | $4.07 | $1.65 |
| 200 | 15.27M | $32.70 | $8.22 | $3.31 |

结论：

1. **不压缩 + 无缓存**，200 轮会话在 3.1 Pro 上要 $32.7；同样的会话，摘要 + 缓存可以压到 $3.3 左右（约 10×）。
2. 200 轮以后，**摘要策略比模型选择更影响成本**。
3. 超过 200k 输入会触发 3.1 Pro 的**双倍计价**（$4/$18），一次跨线，整段 prompt 都按高价算。长团本必须做窗口上限，别让单请求越过 200k。

### 3.3 规模成本（90% 缓存 + 摘要）

| 日轮次 | 月轮次 | 3.1 Pro | Argon（11 月价） | Argon（12 月起） | 3.8 Flash（2026） | 3.5 Flash-Lite |
| --- | --- | --- | --- | --- | --- | --- |
| 1,000 | 30k | $780 | $618 | $1,236 | $272 | $136 |
| 20,000 | 600k | $15,600 | $12,360 | $24,720 | $5,445 | $2,718 |
| 200,000 | 6M | $156,000 | $123,600 | $247,200 | $54,450 | $27,180 |

> 这是"纯 API、无订阅、无 batch"的口径。真实账单还会被重试、失败重发、thinking 超标、跨 200k 加价拉高，建议预算乘 1.2~1.4 的安全系数。

### 3.4 降本杠杆排序（按投入产出）

| 杠杆 | 典型节省 | 实现成本 | 风险/副作用 |
| --- | --- | --- | --- |
| 1. 显式缓存稳定前缀（3.1 Pro / Flash） | 输入成本 −80%~−90% | 低（改 prompt 布局） | 缓存 TTL 1h，需保活；前缀必须逐字节一致，个性化记忆不能进前缀 |
| 2. 模型分层（Pro 写关键场景，Flash 写日常） | 总成本 −60%~−80% | 中（路由 + 质量评测） | 风格不连续，需要"文风锚定" |
| 3. `thinking_level=low` + 输出上限 | 输出成本 −40%~−70% | 低 | 复杂场景质量下降 |
| 4. 摘要 + 账本 + lore 检索 | 输入封顶，避免二次增长 | 中（记忆管线） | 摘要丢细节，需可回查 |
| 5. Batch（离线批量） | −50% | 低 | 24h 级延迟，不适合实时 |
| 6. Flex（同步、延迟不保证） | −50% | 低 | 首 token 可能很慢 |
| 7. 订阅配额（Antigravity） | 按轮次可到 API 的 1/3~1/10 | 中（多账号/运维） | 配额不透明、封号、ToS |
| 8. 多项目分摊 RPM | 提升吞吐上限 | 低 | 花费上限仍按账单账户 |
| 9. Priority | 反向 +75%~100% | 低 | 只在对首 token 有 SLA 时用 |

---

## 4. 五条供给路径

### 4.1 路径 A：官方 API 付费（推荐作为主链路）

**形态**：Gemini API Key（AI Studio）或 Vertex AI，直接调 3.1 Pro / Argon / Flash。

**成本**：见 §3。**上限**：受 Tier、RPM、TPM、花费上限约束。

| Tier | 资格 | 10 分钟花费上限 | 折算日上限 |
| --- | --- | --- | --- |
| Free | 活跃项目 | N/A | 仅 Flash |
| Tier 1 | 绑定账单 | $10 | ≈ $1,440/日 |
| Tier 2 | 累计付费 $100 + 3 天 | $200 | ≈ $28,800/日 |
| Tier 3 | 累计付费 $1,000 + 30 天 | $200 | 账单上限 $20k~$100k+，但速率仍受 $200/10min 约束 |

`[官方]` 关键：**Tier 1 的 $10/10min 是很多"高调用量"项目第一次撞墙的地方**。20k 轮/日、$0.026/轮，峰值很容易在 10 分钟内超过 $10。上线前先规划升 Tier。

优点：可观测、可预测、可退款争议小、数据不被用于训练（付费层）、支持显式缓存/Batch/Flex/Priority。
缺点：Pro 档单价高；preview 模型可能下线；跨 200k 双倍计价。

### 4.2 路径 B：Antigravity 订阅 + 官方 CLI headless（最便宜的正规 Pro）

**形态**：Google AI Pro $19.99/月（或 Ultra 5x $99.99 / 20x $199.99，美国区价格，按官方页面为准），在 Antigravity 里用 Argon / 3.1 Pro，通过官方 `agy` CLI 的 headless 模式脚本化。注意 Argon 不在 Antigravity IDE 里，只在 App、CLI 和 IDE 扩展中可选。

```bash
# 先交互登录一次，凭证会缓存
agy

# 单轮调用，JSON 输出带 usage（含 cache_read_tokens）
agy -p "<PROMPT>" \
  --model gemini-4-argon-medium \
  --output-format json \
  --print-timeout 10m
```

`agy models` 可列出 slug：`gemini-4-argon-high/medium`、`gemini-3.8-flash-high/medium`、`gemini-3.1-pro-high` 等；effort 也可以直接用 `--effort low|medium|high` 覆盖 `[官方]`。

**配额规则** `[官方]`：

| 计划 | 刷新 | 模型 |
| --- | --- | --- |
| Free / Plus | 每周一次 | 仅 3.8 Flash（第三方模型处于过渡期，随时下线） |
| AI Pro $19.99 | 每 5 小时，直到周上限 | Argon（仅付费订阅，不含试用）、3.8 Flash、Claude 5.5 |
| Ultra 5x $99.99 | 每 5 小时 + 更高周上限 | 上述 + 3.7/3.6 Flash + 3.1 Pro |
| Ultra 20x $199.99 | 最高 | 同上，配额最高 |
| Enterprise | 按合同 | 3.1 Pro 等 |

**关键坑**：

1. **配额不按 prompt 数算，按"工作量"算**。长上下文、长输出、高 thinking 都扣得更快。RP 每轮 40k 输入，属于"重"任务，别用 agent 任务的消耗去外推。
2. **headless 有固定开销**：官方示例里问一句 "what is a git rebase" 就用了 **10,415 输入 token**（其中 8,113 命中缓存）、657 输出、616 thinking `[官方]`。也就是说 CLI 自带一套 coding agent 的 system prompt / 工具描述。用它跑 RP，每轮白付 ~2k 未缓存输入，而且 coding 语境会污染文风。
3. **第三方工具复用 Antigravity 登录会封号**。2026-11-05~06 有 4 个账号因"将 OAuth 路由进第三方网关"被 403 ToS 封禁的实证 `[实测]`。用官方 `agy` 脚本化是一回事，把 OAuth 塞进 OpenAI 兼容反代是另一回事，后者不要做。
4. **多账号**：每个账号需要独立的 `HOME`/配置目录、独立登录态、独立出口 IP；支付方式、家庭组共享、区域定价都会触发风控。工程上可行，但它是"运维号池"，不是"配置一下"。
5. **Ultra 20x 的单位配额优于 10 个 Pro**（10 倍价格换 20 倍配额），优先纵向扩，别急着横向堆号。
6. **官方 CLI 也不是通用文本 API**：它本质是 coding agent，带工具、带工作区上下文，Google 没有承诺「用它跑 RP」的配额口径；大规模自动化前先小流量验证，别把它当稳定后端。

**如果确实要跑订阅号池，最小工程要点**：

- 一号一 `HOME`：`HOME=/srv/agy/acc-01 agy -p ...`，让配置/凭证彻底隔离；不要共用 `~/.antigravity`。
- 一号一出口：固定住宅/机房 IP，别让多个号落在同一 /24；登录与调用尽量同地区。
- 配额采集：每次 headless 的 JSON 里有 `usage`，另用 `/usage` 定期采样，写入 Redis 做"剩余窗口 / 冷却"。
- 计费隔离：每号独立支付方式；家庭组共享、同一张卡多号会强关联。
- 灰度入池：新号先跑 20 轮标准 RP，成功率 > 90%、无 403 才进主池；连续 401/403 立即摘除。
- 只跑官方客户端：不要把 OAuth 导出给第三方反代；封号实证见 §4.2 第 3 条。

**经济性**：$19.99 / $0.026 ≈ **769 轮**（按 3.1 Pro API 价）或 **970 轮**（按 Argon 11 月价）。只要一个 Pro 账号月产超过 ~800 轮，订阅就比 API 便宜；如果实际能跑到 3,000~5,000 轮/月，等效单价降到 $0.004~$0.007/轮，是 API 的 1/4~1/6 `[估算]`。

**但**：配额不透明 + 封号 + 多账号运维，使它的"有效成本"高于账面。适合"自用/小团队/高产作者"，不适合作为对外产品的主链路。

### 4.3 路径 C：Gemini App + Gems（人工，小量）

适合作者本人写稿：Pro 档 1M 上下文，Gem 保存规则和世界观。缺点是配额按"复杂度+长度"扣、不能脚本化、不能并发。**不作为高调用量方案**。

### 4.4 路径 D：Vertex AI / Gemini Enterprise（有 SLA 的正规军）

- 同样的 3.1 Pro / Argon 价格，但走 GCP 账单、支持 VPC-SC、数据驻留、Provisioned Throughput（预留吞吐）。
- 适合：对外产品、需要合同/SLA/发票、需要保证峰值不 429。
- 代价：开通与合规流程长；Provisioned Throughput 有起订量，适合稳定在数千 TPM 以上的负载。

### 4.5 路径 E：号池 / 共享池 / 第三方反代（灰，不建议做主链路）

前篇已经把工程细节讲透，这里只补 2026-11 的新事实：

- **免费 API 已无 Pro**，号池只能来自"订阅账号的 OAuth/Cookie"，而订阅账号本身要付 $19.99/月，所以"零成本 Pro"不存在。
- **封号是实证，不是传闻**：4 个账号因第三方 OAuth 路由被 ToS 封禁 `[实测]`；一旦封禁，`403 This service has been disabled in this account`，申诉周期不可控。
- **内容外泄**：共享池/第三方反代意味着你的 RP 文本（可能含个人信息、私密创作）经过不受控中间人。
- **质量不可控**：池子里混着 Flash/Pro、不同 thinking 档、不同区域；同一会话被打散到不同号，角色口吻会漂移。

**结论**：号池只适合"成本极度敏感 + 内容不敏感 + 可接受随时归零"的实验；对外产品、付费用户、含隐私的创作，一律走 A/B/D。

### 4.6 路径对比

| 路径 | 单轮成本（Pro 级） | 上限 | 稳定性 | 封号/合规风险 | 适用规模 |
| --- | --- | --- | --- | --- | --- |
| A API 付费 | $0.021~$0.041 | Tier/TPM | ★★★★★ | 低 | 任意，需升 Tier |
| B Antigravity 订阅 | $0.004~$0.020（按配额摊） | 5h + 周上限 | ★★★ | 中（多账号/ToS） | 个人~小团队 |
| C App + Gems | 订阅内 | 人工 | ★★★★ | 低 | 单人 |
| D Vertex/Enterprise | ≈ API | 可预留 | ★★★★★ | 低 | 产品/企业 |
| E 号池/共享池 | $0~$0.01 | 不可控 | ★ | 高 | 实验 |

### 4.7 选择树

```
是否需要对外提供 / 有付费用户 / 有 SLA?
├─ 是 → D（Vertex/Enterprise）或 A（API + 升 Tier + 多项目）
└─ 否 → 月轮次?
    ├─ < 5,000   → B（1 个 AI Pro，Argon，官方 CLI/App）
    ├─ 5,000~50,000
    │   ├─ 质量优先 → B（Ultra 20x）+ A 溢出
    │   └─ 成本优先 → A（3.1 Pro 关键场景 + 3.8 Flash 日常）+ 显式缓存
    └─ > 50,000  → A/D 为主，B 只做"人工高优通道"；不建议靠堆号
```

---

## 5. 高调用量架构

### 5.1 参考架构

```
client / bot / 写作台
        │  (SSE 流式)
        ▼
┌─────────────────────────────────────────────┐
│ gateway                                     │
│  · 会话粘性（conversation_id → 上游）        │
│  · 上下文装配（canon/summary/ledger/window） │
│  · token 预算与 200k 熔断                    │
│  · 模型路由（Pro / Flash / Lite）            │
│  · 重试/降级/限流/计费                       │
└───────┬───────────────┬───────────────┬─────┘
        ▼               ▼               ▼
   3.1 Pro 池      3.8 Flash 池     订阅账号池
   (API, 缓存)     (API, 便宜)      (agy headless)
        │               │               │
        └───────┬───────┴───────┬───────┘
                ▼               ▼
          Redis: 会话/额度/健康分   Postgres: 正文/账本/摘要
                │
          Flash 工具链：摘要、记忆抽取、lore 检索 embedding、质量检测
```

### 5.2 模型路由策略

| 请求类型 | 模型 | thinking | 理由 |
| --- | --- | --- | --- |
| 关键剧情/章节收束/用户手动"高质量" | 3.1 Pro 或 Argon | low | 文笔上限 |
| 日常对话轮次 | 3.8 Flash | low | 成本 1/3，质量够用 |
| 摘要 / 记忆抽取 / 账本更新 | 3.8 Flash 或 3.5 Flash-Lite | minimal/low | 结构化任务，不需要文采 |
| lore 检索 / 去重 / 分类 | Embedding + Flash-Lite | — | 不生成正文 |
| 质量检测（复读、OOC、账本矛盾） | Flash | minimal | 判定类任务 |
| 离线批量分支/多条候选 | Batch 3.1 Pro / Flash | low | 五折 |

**分层比例的经验值**：Pro 占 15%~25% 的轮次，能拿到 80% 的质量收益；再往上加 Pro 比例，边际质量提升很小、成本线性上涨 `[估算]`。用 §6.5 的评测集校准这个比例，不要拍脑袋。

### 5.3 上下文与记忆三层

1. **Canon（不变层）**：规则、风格圣经、角色卡、固定世界观。放最前，做缓存前缀。**不要**在这里插时间戳、随机 ID、用户昵称——会让缓存全灭。
2. **State（半稳定层）**：滚动摘要 + 状态账本（时间/地点/伤势/物品/关系/承诺/已知信息）。每 N 轮用 Flash 重写一次，控制在 800~1,500 token。
3. **Window（易变层）**：最近 10~20 轮原文 + 本轮用户输入。

**Lore 不要全量注入**：把设定集切块 + Gemini Embedding 2 建索引，按当前场景检索 top-k（1k~4k token）。全量注入是长团本成本失控的第一原因。

**摘要的写法**：不要"复述剧情"，要"抽取状态"：

```json
{
  "time": "第 3 天 傍晚",
  "location": "北港·旧灯塔",
  "injuries": ["左臂刀伤（已包扎）"],
  "items": ["铜钥匙", "半张海图"],
  "relationships": {"Mira": "信任+1，仍隐瞒身份"},
  "open_threads": ["灯塔地下室的脚步声", "船长的欠条"],
  "facts_learned": ["Mira 怕火", "港口宵禁从今晚开始"]
}
```

账本每轮注入，模型的时间漂移和"角色全知"会明显减少（见 §6.1）。

### 5.4 缓存策略（最大的成本杠杆）

#### 显式缓存（3.1 Pro / 3.8 Flash，Beta）

```python
from google import genai
from google.genai import types

client = genai.Client()
cache = client.caches.create(
    model="gemini-3.1-pro-preview",
    config=types.CreateCachedContentConfig(
        display_name="rp-canon-<SESSION_ID>",
        system_instruction=CANON_TEXT,          # 规则+风格+角色卡，≥4096 token
        contents=[types.Content(role="user", parts=[types.Part(text=WORLD_LORE)])],
        ttl="3600s",                             # 默认 1 小时，可续
    ),
)
# 后续请求引用 cache.name，命中部分按 $0.20/M 计
```

**布局原则**：稳定内容全部放前面，易变内容放最后；同一前缀在 1 小时内反复用；TTL 到期前续期，避免"刚热就冷"。

**共享摊薄**：如果产品是"固定角色/固定世界观 + 多用户"，所有用户共用同一段 canon，就可以共享同一个 cache 对象（前缀逐字节一致），存储成本被用户数摊薄；个性化记忆（账本、摘要）必须放在缓存之后，不能进前缀。

**盈亏平衡**（3.1 Pro）：

```
存储成本 = $4.50 / 1M token / 小时
缓存读节省 = ($2.00 − $0.20) / 1M = $1.80 / 1M / 次
平衡点 = 4.50 / 1.80 = 2.5 次/小时
```

也就是说，**同一前缀每小时被用 3 次以上，显式缓存就赚钱**。RP 会话通常每几秒到几分钟一轮，远超平衡点。40k 前缀的存储成本是 $0.18/小时，每次命中节省 $0.072，约 2.5 次回本——按 RP 的轮次密度，基本是稳赚。

> 3.8 Flash 的存储只要 $0.50/M/小时，缓存读节省 $0.675/M，平衡点 **0.74 次/小时**，几乎无脑开。

#### 隐式缓存（Argon）

- Argon **只支持隐式缓存**，不能 pin 前缀，命中率不可控 `[官方]`。
- Gemini 4 的规则：**新请求比上一轮多出 ≥2,050 token，才缓存整段上一轮请求**；不足 2,050 的部分会按标准输入价重算 `[官方]`。
- RP 一轮通常新增 700~1,500 token，**低于 2,050**，所以 Argon 跑 RP 会频繁"只缓存一部分"，实际输入成本高于账面。对策：
  1. 把两轮合并成一次请求（让新增 ≥2,050）；
  2. 每轮附一段 600~1,000 token 的"场景块"（环境、NPC 状态），把增量顶过阈值；
  3. 关键长会话直接用 3.1 Pro 的显式缓存，别在 Argon 上赌命中率。

### 5.5 限流、重试、降级

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `429 RESOURCE_EXHAUSTED` | RPM/TPM/花费上限 | 指数退避 + 抖动；切项目/切账号；降级到 Flash；不要原地重试 |
| `503 UNAVAILABLE` | 上游容量 | 重试 2~3 次，仍失败则降级 |
| `finishReason: SAFETY` | 内容过滤/自拒 | **不要原样重试**；改写场景或换模型；记录样本 |
| `finishReason: MAX_TOKENS` | 输出上限/thinking 吃满 | 提高 `max_output_tokens` 或降 thinking；分段续写 |
| `MALFORMED_FUNCTION_CALL` | 工具调用（RP 一般不用） | 重试或去掉工具 |
| 流式中断 | 代理/超时 | 15s 心跳；客户端断点续传；服务端保存已生成部分 |
| 长上下文变慢 | 输入 80k+ | 摘要/检索；把窗口压到 20k~30k |

**降级链**：

```
3.1 Pro ──fail──▶ Argon(订阅) ──fail──▶ 3.8 Flash ──fail──▶ 明确报错（不静默重试）
```

降级要带**质量标记**：降到 Flash 的轮次在正文里标记 `model=flash`，方便事后做质量归因。

### 5.6 容量估算

```
峰值 RPS = 日轮次 × 峰值占比 / 峰值秒数
并发 C ≈ RPS × 平均时延 L
TPM = 日输入 token / 86400 × 峰值系数
```

例：20,000 轮/日，60% 集中在 12 小时，L=8s：

```
RPS = 12,000 / 43,200 ≈ 0.28
C ≈ 0.28 × 8 ≈ 2.2 并发
TPM ≈ 20,000×40,000/86,400 × 3 ≈ 27.8M TPM（峰值）
```

**注意**：20k 轮/日的并发只有个位数，瓶颈是 **TPM 和钱**，不是连接数。200k 轮/日才到 ~22 并发、~278M TPM 峰值，这时必须 Tier 3 + 多项目 + Vertex 预留。

### 5.7 观测指标

| 指标 | 维度 | 告警 |
| --- | --- | --- |
| `rp_turns_total` | model, tier, status | — |
| `rp_cost_usd` | model, session | 日预算 80% |
| `rp_tokens` | input, cached, output, thinking | thinking/输出 > 0.5 告警（说明没压住） |
| `rp_first_token_ms` | model | P95 > 3s |
| `rp_finish_reason` | SAFETY/MAX_TOKENS/STOP | SAFETY > 3% |
| `rp_repetition_rate` | model | 4-gram 重复率 > 0.08 |
| `rp_ledger_conflicts` | session | 单会话 > 2 次 |
| `rp_cache_hit_ratio` | prefix | < 60% 检查前缀是否被污染 |
| `rp_upstream_429` | project/account | 5 分钟 > 5% |

---

## 6. RP 质量工程：3.1 Pro / Argon 的已知毛病与对策

### 6.1 社区实测的退化清单 `[实测]`

2026-02~03 的官方论坛长帖 + 后续讨论，集中反馈了 3.1 在长文协作写作上的问题（对 2.5 而言是退化）：

| 现象 | 表现 | 对策 |
| --- | --- | --- |
| 时间漂移 | 写了三次睡眠/醒来，模型仍当成"第一天" | 账本显式记录 `day/time`，每轮注入；摘要里保留时间锚点 |
| 体格/行为通胀 | "高大"角色越写越夸张：巨手、轰鸣般的声音、非场景需要的压迫感 | 角色卡写**固定数值**（身高/体型/力量上限），规则里禁止升级形容词；后处理检测夸张词 |
| 感官复读 | 反复用同一批细节（臭氧味、沙哑嗓音、晨起嗓） | 禁用词表 + 每场景轮换感官库；n-gram 重复检测，超阈值重写 |
| 特质复述 | 50k 后不断"提醒"角色很克制/很警惕，而不是演出来 | 规则："已被演示过的特质不再复述；用行动与潜台词" |
| 空间错误 | 5'10" 的角色抱住 6' 的角色，头却在胃部 | 账本记录相对位置/高度；要求"先写站位再写动作" |
| 角色全知 | NPC 知道不该知道的事，或零信息直接得出正确结论 | 反全知条款（见 §6.2），明确"只能基于可观察线索推断，且可以推断错" |
| 字数不听话 | 要求长回复，因 thinking 挤占而变短 | `thinking_level=low`；把"字数"改成"节拍数/场景数"；`max_output_tokens` 给足 |
| 文风"合规化" | 比 2.5 更"听话"、更少惊喜，像工具而非合作者 | 风格圣经给具体样本句；提高多样性（多候选 + 选优）；关键场景用 Argon/更强模型 |
| 双形容词堆叠 | 50k+ 后每段都是"Large, bare feet."式结构 | 句法规则 + 后处理检测连续"形容词, 形容词"开头 |

**结论**：这些不是"换个 prompt 就没了"的小问题，而是模型行为。工程上要**用状态机 + 检测器兜底**，不能只靠一句"请勿重复"。

### 6.2 Prompt 骨架

```text
<system>
你是互动小说的叙述者，兼任所有 NPC。玩家扮演主角；你绝不替主角做决定、绝不写主角的台词。

[STYLE BIBLE]
- 视角：第三人称限知，过去时；只写主角能感知到的信息。
- 句长：以 12~25 字为主，紧张场景用短句；每段不超过 4 句。
- 禁止：总结上文、解释情绪、替读者下结论、"仿佛/似乎"连用。
- 禁用词/意象：{{BANNED_LIST}}（每章轮换）
- 每轮结尾：一个具体的、可回应的动作或问题（hook），不要总结。

[CHARACTER CARD]
姓名 / 年龄 / 身份：
外貌（固定数值，不得夸张）：
口吻样本（3~5 句原话）：
欲望 / 恐惧 / 底线：
知识边界（不知道什么）：
关系状态：

[STATE LEDGER]
{{LEDGER_JSON}}

[CANON]
{{WORLD_LORE_TOP_K}}

[RULES]
1. 角色不是全知的：只能基于对话、动作、表情、环境线索推断；推断可以错。
2. 已演示过的特质不要复述；用行动与潜台词。
3. 外貌与体格是固定事实；禁止升级形容词，禁止无依据的力量展示。
4. 时间/地点/伤势/物品以 LEDGER 为准；与正文冲突时以 LEDGER 为准。
5. 玩家角色的内心与台词不可代写；留白。
6. 不复述玩家刚说过的话；不复述上一段。
7. 不出现"作为 AI/作为一个模型"等元话语。
</system>
```

**每轮消息**：

```text
[SUMMARY]
{{ROLLING_SUMMARY}}

[RECENT]
{{LAST_10_TURNS}}

[SCENE]
时间/地点/在场者/环境线索：{{SCENE_BLOCK}}

[PLAYER ACTION]
{{USER_INPUT}}
```

### 6.3 参数（2026-07 之后）

| 参数 | 建议 | 原因 |
| --- | --- | --- |
| `temperature` / `top_p` / `top_k` | **不要传** | 2026-07-21 起废弃；官方建议删掉，旧值可能导致循环/退化 `[官方]` |
| `thinking_level` | 3.1 Pro / Argon：`low`；Flash：`low` 或 `minimal`（3.6 Flash 支持 minimal，3.8 Flash 不支持） | thinking 按输出计费；RP 不需要长推理 |
| `max_output_tokens` | 1,200~2,000（正文 600~1,200 + 余量） | 太小会被 thinking 挤断 |
| prefill（预填模型开头） | **不要用** | 3.8 Flash 起已移除 prefilled model turns，老 preset 的预填会失效 `[官方]` |
| `safety_settings` | 默认（Off）即可；不要主动调更严 | 四类过滤器默认 Off；调严只会增加误杀 |
| 流式 | 必开 | 首 token 体验；也便于中断 |

### 6.4 长会话的四个动作

1. **每 20 轮摘要一次**：Flash 抽取"状态 + 未闭合线索"，不写文学化摘要。
2. **每轮更新账本**：Flash 读"上一轮账本 + 本轮正文"，输出新 JSON；失败则沿用旧账本并告警。
3. **lore 检索代替全量**：Embedding top-k，按场景标签过滤。
4. **窗口硬上限**：单请求输入不超过 120k（给 200k 双倍计价留安全边界）；超过则强制摘要 + 截断最旧原文。

### 6.5 自动评测（不然调优就是玄学）

**离线评测集**：50 个场景 × 每个 20 轮，覆盖：日常对话、冲突、情感、动作、悬疑、多人场景、时间跳跃。

**自动指标**：

| 指标 | 算法 | 阈值 |
| --- | --- | --- |
| 复读率 | 4-gram 重复占比 | < 0.08 |
| 字数达标 | 实际/要求 | 0.8~1.3 |
| POV 违规 | 正文出现"你"+ 主角台词 | 0 |
| 账本矛盾 | LLM 判定正文 vs LEDGER | < 2/会话 |
| 角色全知 | LLM 判定"无来源信息" | < 1/10 轮 |
| 文风漂移 | 与风格样本的 embedding 距离 | 前 10 轮基线 ±20% |

**LLM-as-judge**（用 Flash，成本可忽略）：

```text
你是 RP 质量评审。给下面这段续写打 1~5 分，只输出 JSON：
{"prose":n,"character_voice":n,"continuity":n,"instruction_following":n,"repetition":n,"hook":n,"notes":"..."}
评分标准：5=出版级；4=稳定可读；3=有明显瑕疵但可用；2=出戏；1=不可用。
```

**A/B 方法**：同一场景、同一历史，分别用 Pro / Flash / 不同 prompt，盲评 100 轮，统计"胜率 + 每轮成本"，得到**每美元质量**曲线。模型分层比例按这条曲线定，不按感觉。

---

## 7. 内容边界与拒绝处理

### 7.1 事实 `[官方]`

- 四类可调过滤器：harassment、hate speech、sexually explicit、dangerous content；**Gemini 2.5 及以后默认阈值是 Off**。
- 另有一类 `jailbreak` 过滤器（提示注入/越狱）。
- **核心伤害保护（如儿童安全）始终开启，不可调**。
- 即使全部 Off，**模型仍可能自己拒绝**；换 Pro 不会改变这一点。
- 使用更宽松的安全设置的应用"可能被审查"（ToS）。

### 7.2 降低误杀（不保证成功）

1. **年龄/身份明确**：角色卡写清成年、关系、场景边界；模糊年龄是最高频的误杀源。
2. **文学框架**：把请求写成"小说续写/剧本"，而不是"你现在和我……"。
3. **场景切分**：把高风险场景拆成"前—中—后"，用省略/暗场过渡，而不是一次性要求露骨细节。
4. **避免越狱话术**：`jailbreak` 过滤器会识别"忽略你之前的指令"这类模式；越狱话术反而提高拦截率。
5. **失败处理**：`SAFETY` 不原样重试；改写触发句、降低直白度、换模型；记录被拦样本做回归。

### 7.3 如果 explicit 是硬需求

Gemini 不是合适载体。按"permissiveness × 质量 × 成本"选替代：

| 选项 | 优势 | 代价 |
| --- | --- | --- |
| Grok 4.1 | 明显更少拒绝，情感/角色感强 | 逻辑与长程一致性弱；重度使用要订阅 |
| MiniMax M2 (Her) | RP 专用微调，100+ 轮角色一致性，200k 上下文 | 渠道少（Shiori/OpenRouter） |
| DeepSeek V4.1 Flash / V4 Pro | 便宜，世界状态跟踪不错 | 文笔上限低于 Claude/Gemini |
| 本地 Llama 3.3 70B / Qwen3 32B + SillyTavern | 无过滤、隐私 | 需 ≥12GB VRAM；质量看调校 |

**混合策略**（推荐）：Gemini 写剧情/日常/情感铺垫，explicit 场景路由到 Grok 或本地；用统一的消息格式和账本，保证切换时不跳戏。代价是文风可能不一致，需要"文风锚定"（把上一段 Gemini 正文作为 style sample 传给替代模型）。

---

## 8. 推荐方案

### 8.1 按规模

| 规模（轮/日） | 推荐 | 月成本量级 `[估算]` | 关键动作 |
| --- | --- | --- | --- |
| < 500 | 1× Google AI Pro（$19.99），App/`agy` | $20 | 关 thinking、写账本、20 轮摘要 |
| 500 ~ 5,000 | AI Pro 或 Ultra 5x + API 溢出 | $20~$150 | 模型分层（Pro 20% + Flash 80%）、显式缓存 |
| 5,000 ~ 50,000 | API 3.1 Pro + 3.8 Flash + 2~5 个订阅号做高优通道 | $1.9k~$19k | 升 Tier 2、缓存、摘要、评测集、预算闸门 |
| > 50,000 | Vertex/Enterprise + 多项目 + 预留吞吐 | $18k+（预留另计） | SLA、容量规划、合规、专职工信 |

### 8.2 三套参考配置

**A. 省钱型（个人/小团队，质量优先）**

- 1× Google AI Pro $19.99 → Antigravity Argon（`--model gemini-4-argon-medium` 或 `--effort low`）
- 官方 `agy` headless + 自建队列；每账号独立 `HOME`
- 关键场景用 Argon，日常用 3.8 Flash（免费层或 API）
- 预算：$20~$60/月；产能：数百~数千轮/月（需实测配额）
- 风险：配额不透明；不要接第三方反代

**B. 均衡型（小产品，5k~20k 轮/日）**

- 主：API 3.1 Pro（关键 20%）+ 3.8 Flash（日常 80%）
- 3.1 Pro 显式缓存 canon；Flash 隐式缓存
- Flash 做摘要/账本/检测；Embedding 做 lore 检索
- Batch 跑离线分支；Flex 跑非实时补全
- 预算：$2k~$10k/月（20k 轮/日时约 $10.5k，见 §8.3）
- 风险：Tier 1 花费上限；12 月/1 月涨价

**C. 质量型（对外产品/工作室，> 20k 轮/日）**

- Vertex AI：3.1 Pro + Argon（关键场景）+ 3.8 Flash
- Provisioned Throughput 保峰值；多项目分摊 RPM
- 自建评测集 + 人工抽检；账本/摘要/检测全链路
- 预算：$10k+/月；需要 SRE/合规
- 风险：preview 模型下线；成本随规模线性

### 8.3 预算示例（20k 轮/日，均衡型）

| 项 | 计算 | 月成本 |
| --- | --- | --- |
| 3.1 Pro（20% = 4k 轮/日） | 4k×30×$0.026 | $3,120 |
| 3.8 Flash（80% = 16k 轮/日） | 16k×30×$0.0091 | $4,368 |
| 摘要/账本/检测（Flash） | 约 20% 额外调用 | $900 |
| 重试/失败/超长（×1.25） | | $2,097 |
| **合计** | | **≈ $10,485/月** |

> 若把 Pro 比例降到 10%，合计约 $7.9k；若全用 Flash，约 $5.5k；若 10 个 Ultra 20x 的配额真能承载 20k 轮/日，订阅费约 $2k（配额未证实，且多账号运维/封号风险另计）。

### 8.4 最小落地清单（30 分钟版）

1. 开一个 AI Studio 项目，拿 API Key；账单绑定后确认 Tier。
2. 用 `gemini-3.1-pro-preview` + `thinking_level=low` + `max_output_tokens=1600` 跑通一轮。
3. 把规则 + 角色卡 + 世界观拼成 ≥4,096 token 的稳定前缀，建显式缓存，验证第二轮 `usage.total_cached_tokens > 0`。
4. 每轮记录 `input / cached / output / thinking` 四个数；用附录 A 脚本算单轮成本。
5. 跑 50 轮，观察：time drift、复读、字数、SAFETY；把失败样本收进评测集。
6. 定预算闸门：日预算 = 目标轮次 × 单轮成本 × 1.3；超过 80% 告警、100% 降级到 Flash。

### 8.5 不建议的做法

1. **用 Gemini App 免费档跑长团本**：32k 上下文，50 轮后开始丢设定。
2. **全量注入设定集**：输入成本翻倍，且模型更容易抓错重点。
3. **不控 thinking**：3.1 Pro 默认 high，输出账单可能翻倍。
4. **把订阅 OAuth 塞进第三方反代**：已有封号实证。
5. **单一模型打天下**：Pro 写日常=烧钱，Flash 写高潮=泄气。
6. **不做评测就调 prompt**：无法判断"变好"还是"变贵"。

---

## 9. 落地路线图

### Week 1：打通与测量

- [ ] 选一条路径（建议先用 AI Pro 订阅 + `agy` 或 API 3.1 Pro）
- [ ] 写 10 个场景 × 20 轮的评测集
- [ ] 记录：输入/缓存/输出/thinking token、首 token、finishReason、成本
- [ ] 校准"单账号配额"：连续跑 50 轮标准 RP，记录 `/usage` 消耗百分比

### Week 2：成本与质量

- [ ] 上下文三层（canon / ledger+summary / window）落地
- [ ] 3.1 Pro 显式缓存；验证命中率 > 60%
- [ ] `thinking_level=low`；输出上限；删 temperature/prefill
- [ ] 反复读/反全知/反通胀规则；自动检测器
- [ ] Pro/Flash 分层比例 A/B

### Week 3~4：规模化

- [ ] 网关：会话粘性、重试、降级、预算闸门
- [ ] 升 Tier / 多项目；压测峰值 TPM
- [ ] 观测看板与告警
- [ ] 成本回归：每轮成本、每美元质量
- [ ] 写 runbook：封号/429/模型下线/涨价

### 验收指标

| 指标 | 目标 |
| --- | --- |
| 每轮成本（混合） | < $0.015 |
| 缓存命中率 | > 60% |
| 首 token P95 | < 3s |
| 请求成功率（含重试） | > 99% |
| 复读率 | < 0.08 |
| 账本矛盾 | < 2/会话 |
| SAFETY 拦截率 | < 3%（超过说明场景或写法有问题） |

---

## 10. 风险登记

| 风险 | 触发条件 | 影响 | 缓解 |
| --- | --- | --- | --- |
| 订阅封号 | 第三方反代、多账号同 IP、异常高频 | 账号归零 | 只用官方客户端；一号一环境；不共享登录 |
| API 花费失控 | 忘记设预算、重试风暴、跨 200k | 账单翻倍 | 预算闸门、单请求 token 上限、告警 |
| Tier 上限 | 峰值 > $10/10min | 429，用户可见失败 | 提前升 Tier 2/3；多项目 |
| 模型下线 | preview 被替换（3.1 Pro 无日期，但先例 <4 个月） | 质量/价格突变 | 抽象模型别名；保留 3.8 Flash 回退 |
| 涨价 | 12-01 Argon、01-01 Flash | 预算失效 | 价格配置化；按新价重算 |
| 数据留存 | 付费层 GenerateContent 默认 `store=true`（2026-10-26 起新项目） | RP 内容被存储 | 显式 `store=false`；读日志策略 |
| 免费层训练 | 免费 API/免费 App | 内容被用于改进产品 | 敏感创作走付费层 |
| 质量退化 | 长会话、模型更新 | 出戏、复读 | 评测集 + 检测器 + 版本 pin |
| 供应商锁定 | prompt 绑定 Gemini 特性 | 迁移成本 | 统一消息格式；保留 OpenAI 兼容层 |

---

## 11. 待验证项

| 项 | 方法 | 优先级 |
| --- | --- | --- |
| 单个 AI Pro 账号的 RP 月产能 | 连续 7 天跑标准 40k/900 轮次，记录 `/usage` 与周上限 | 高 |
| Argon 隐式缓存在 RP 的真实命中率 | 记录 `usage.cache_read_tokens` / 输入总量，按轮次分布 | 高 |
| 3.1 Pro 显式缓存的跨会话复用率 | 同角色卡多会话，统计命中与存储成本 | 高 |
| thinking low vs high 的质量差 | 100 轮盲评 + 成本 | 高 |
| Pro/Flash 分层的最优比例 | 10%/20%/30% 三组 A/B | 高 |
| 摘要周期（10/20/30 轮）对连贯性的影响 | 账本矛盾率 + 人工评分 | 中 |
| 3.8 Flash 跑 RP 的可用性 | 与 3.1 Pro 盲评对比 | 中 |
| 多账号订阅的封号率 | 小规模灰度（3~5 号，独立 IP/环境） | 中 |
| 12-01/01-01 涨价后的最优组合 | 用附录 A 脚本重算 | 高 |

---

## 附录 A. 成本计算器

```python
#!/usr/bin/env python3
"""RP 成本估算：单轮 / 日 / 月 / 会话。价格为 2026-11 快照，美元/1M token。"""

PRICES = {
    # name: (input, cached_input, output, threshold, input_hi, cached_hi, output_hi)
    "3.1-pro":      (2.00, 0.20, 12.00, 200_000, 4.00, 0.40, 18.00),
    "argon-2026":   (2.00, 0.10, 10.00, 10**9,  2.00, 0.10, 10.00),
    "argon-2026-12":(4.00, 0.20, 20.00, 10**9,  4.00, 0.20, 20.00),
    "3.8-flash":    (0.75, 0.075, 3.75, 10**9,  0.75, 0.075, 3.75),
    "3.8-flash-27": (1.50, 0.15, 7.50, 10**9,   1.50, 0.15, 7.50),
    "3.5-lite":     (0.30, 0.03, 2.50, 10**9,   0.30, 0.03, 2.50),
}


def turn_cost(model: str, input_tokens: int, output_tokens: int,
              cache_ratio: float = 0.9) -> float:
    i, c, o, th, i_hi, c_hi, o_hi = PRICES[model]
    hi = input_tokens > th
    pi, pc, po = (i_hi, c_hi, o_hi) if hi else (i, c, o)
    cached = input_tokens * cache_ratio
    fresh = input_tokens - cached
    return (fresh * pi + cached * pc) / 1e6 + output_tokens * po / 1e6


def session_cost(model: str, turns: int, prefix: int = 6_000,
                 per_turn: int = 700, output: int = 900,
                 summarize_every: int = 20, window_cap: int = 20_000,
                 cache_ratio: float = 0.9) -> float:
    total = 0.0
    history = 0
    for t in range(1, turns + 1):
        history = min(history + per_turn, window_cap)
        inp = prefix + history
        total += turn_cost(model, inp, output, cache_ratio)
        if summarize_every and t % summarize_every == 0:
            history = min(history, 2_000)  # 摘要后窗口重置
    return total


if __name__ == "__main__":
    for m in PRICES:
        one = turn_cost(m, 40_000, 900)
        day20k = one * 20_000
        print(f"{m:14s} 单轮 ${one:.4f}  20k轮/日 ${day20k:,.0f}  月 ${day20k*30:,.0f}")
    print("\n200 轮会话（摘要+缓存）:")
    for m in ("3.1-pro", "argon-2026", "3.8-flash"):
        print(f"  {m:14s} ${session_cost(m, 200):.2f}")
```

---

## 附录 B. 可直接用的评测脚本骨架

```python
import re, json
from collections import Counter

def four_gram_repetition(text: str) -> float:
    chars = re.sub(r"\s+", "", text)
    grams = [chars[i:i+4] for i in range(len(chars) - 3)]
    if not grams:
        return 0.0
    c = Counter(grams)
    repeated = sum(v - 1 for v in c.values() if v > 1)
    return repeated / len(grams)

def check_pov(text: str) -> int:
    # 粗略检测（仅适用于第三人称预设）：叙述里出现第二人称，通常意味着代写了玩家
    return len(re.findall(r"(?<![\u4e00-\u9fa5])你(?!们)", text))

def ledger_conflicts(ledger: dict, text: str) -> list[str]:
    hits = []
    for item in ledger.get("items", []):
        if item and item not in text and item in ledger.get("used_this_turn", []):
            hits.append(f"item missing: {item}")
    return hits
```

配套 judge prompt 见 §6.5；把结果写进 `eval_results.jsonl`，按模型/prompt 版本聚合。

---

## 附录 C. 检索关键词与来源

**检索关键词**

```
gemini 3.1 pro roleplay / gemini 3.1 pro creative writing regression
gemini 4 argon pricing / antigravity plans quota
antigravity cli headless --model / agy models
gemini explicit caching ttl / implicit caching 2050 tokens
gemini thinking_level low roleplay
antigravity oauth banned third-party gateway
sillytavern gemini 3.8 flash preset
```

**主要来源**（2026-11 访问）

- Gemini API 定价：<https://ai.google.dev/gemini-api/docs/pricing>
- 速率限制 / Tier / 花费上限：<https://ai.google.dev/gemini-api/docs/rate-limits>
- Gemini 3.1 Pro 模型卡：<https://ai.google.dev/gemini-api/docs/models/gemini-3.1-pro-preview>
- Gemini 4 Argon 与 long decode：<https://ai.google.dev/gemini-api/docs/long-decode>
- 上下文缓存（隐式/显式、TTL、最小 token）：<https://ai.google.dev/gemini-api/docs/caching>
- 安全设置与默认阈值：<https://ai.google.dev/gemini-api/docs/safety-settings>
- Batch / Flex / Priority：<https://ai.google.dev/gemini-api/docs/batch-api>、`flex-inference`、`priority-inference`
- Gemini CLI 配额（已被 Antigravity CLI 取代）：<https://github.com/google-gemini/gemini-cli/blob/main/docs/resources/quota-and-pricing.md>
- Antigravity 计划与配额：<https://antigravity.google/docs/plans>
- Antigravity 模型可用性：<https://antigravity.google/docs/models>
- Antigravity CLI headless：<https://antigravity.google/docs/cli/headless>
- Antigravity SDK Personas：<https://antigravity.google/docs/sdk/personas>
- 3.1 长文写作退化反馈：<https://discuss.ai.google.dev/t/feedback-creative-writing-with-gemini-3-1-issues-and-improvements/127438>
- Antigravity OAuth 第三方路由封号实证：<https://github.com/cheaper-inference/OmniRoute/issues/19266>
- RP 模型横向对比与 Gemini 选型：<https://arcanumrpgs.com/blog/best-llm-for-roleplay/>、<https://arcanumrpgs.com/blog/best-gemini-model-for-roleplay/>
- Antigravity 定价与配额解读：<https://usagebox.com/articles/google-antigravity-pricing-limits-2026>

---

## 附录 D. 术语

| 缩写 | 含义 |
| --- | --- |
| RP | Roleplay，多轮角色扮演/互动叙事 |
| canon | 不变设定层（规则、角色卡、世界观） |
| ledger | 状态账本（时间、地点、伤势、物品、关系） |
| TTL | Time To Live，缓存存活时间 |
| TPM / RPM | Tokens / Requests Per Minute |
| RPD | Requests Per Day |
| Flex | 五折、延迟不保证的同步推理档 |
| Priority | 贵 75%~100%、优先调度的推理档 |
| long decode | Argon 的单次超长输出（可到 1M token） |
