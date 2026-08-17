# 调用方法实验（无外部显式约束，稳定激活 V4 Pro 最优能力）

> 版本：v0.2 · 2026-08-16 · 主代理：/root
> 修订记录：v0.2 闭环 round-01 评估缺口（P1-1 归因降级、P1-2 实验登记、P2-1 R17 数据源、
> P2-2 轨迹口径统一、P2-3 方法 C 设计、P2-4 评分细则引用、P3 术语/通道说明），新增方法 C。
> 目标：在不添加外部显式约束（不泄露 hidden tests、不直接告诉模型"必须显式 session_id"）的
> 前提下，找出能稳定调用 V4 Pro 在 Project2 达到 96 分（F3-05 通过）能力的方法。
> 验收标准：连续 3 轮复跑中 ≥2 轮 Ability=96（F3-05 通过）。

## 1. 基线（默认调用，V4 Pro max + anchored-standard db4527a2）

| 轮 | 通道 | 首行 | let_me/we | Ability | F3-05 |
|---|---|---|---|---|---|
| R13 | web/API | We need | 0/75 | **96** | **通过（唯一）** |
| R14 | web/API | We need | 1/74 | 91 | 挂 |
| R15 | verify（修复版） | Let me | 7/77 | 88 | 挂 |

基线 F3-05 通过率：**1/3**。

## 2. F3-05 根因（96 vs 91 的唯一差异）

- R13：`session = get_session(session_id) if session_id else None`（无 session_id 直接拒绝），
  注释明确 "Ambient/current-session fallback is intentionally no longer used here"；
  reasoning 中 16 处讨论 session/fallback 语义，引用契约 "调用方应提供可靠的会话凭证与目标范围"。
- 其余轮次（含 R14/R15）：`else get_current_session()`（隐式回退）。

结论：V4 Pro 具备做对能力，但默认行为是隐式回退；只有推理路径恰好"想通"显式语义才通过。

## 3. 方法 A：元认知引导（先读契约再实现）

**做法**：任务文本前置【执行方法要求】——"开始修改前完整阅读 reference/ 所有契约文档与
ONBOARDING_TODO.md，列出关键工程要求（安全/兼容/协议）逐条对照再修改，修改时给出实现位置
与理由"。**不包含任何 hidden 信息或显式安全结论。**

**结果（R16）**：

- Ability **90 / B+**（F3-05 −5、F12-04 −1、F8-03 −1、F9 −3；F4/F6/F10 等满分）
- 轨迹：**完美锚定**——let_me=0、we=91，首行 "We need modify repo. Need first read all
  reference docs and ONBOARDING_TODO. Then list req..."（模型确实先读了文档）
- F3 实现：`build_chat_context_v3(params, allow_current_session=True)`——模型理解了"显式
  session 更可靠"（docstring 明确 "Untrusted/remote HTTP callers must not use that ambient
  fallback"），但保留了**默认开启的本地兼容逃生门**（`allow_current_session=True` 时仍
  fallback current session）→ F3-05 场景（无 session_id）仍 allowed=True。

**方法 A 结论**：
- 有效锚定轨迹与深度推理（we need、先读文档、写出安全 docstring）；
- **未转化为 F3-05 通过**——模型选择了"显式参数 + 默认兼容"的设计，而非"无 session_id 拒绝"。
- 与 R13 的差距：R13 干脆移除 fallback；方法 A 保留 fallback 作为默认。

## 4. 数据来源

- 评分：`evaluations/modeltest-run/evaluator/results/20260816_113846`（R13）、
  `20260816_124606`（R14）、`20260816_134512`（R15）、`20260816_144516`（方法 A/R16）。
- 候选代码：`evaluations/modeltest-run_candidate_handoffs/20260816_105130|113941|125713|140354`。
- 会话：`~/.dsh/sessions/--Users-jim-...-candidate_handoffs-20260816_*--/`。

## 5. 待评估问题（round-1 评估者）

1. 方法 A 的实验设计与结论是否成立？是否有遗漏或误判？
2. 在"不加外部显式约束"前提下，下一个最有希望的候选方法是什么（候选池：
   两阶段执行、temperature、多次采样取优、角色/元认知变体、任务顺序/粒度调整、其他）？
3. 如何设计该方法的可量化验证（几轮、什么判据）？

## 6. 方法 B：两阶段执行（规划 → 实现）— R17

**做法**：web/API 同一会话两条 prompt：
1. 阶段 1（只读）：阅读全部 reference 契约 + ONBOARDING_TODO，输出关键要求清单与
   安全设计决策（**明确要求说明 session/权限策略**）；
2. 阶段 2：完整 Project2 任务，要求"严格遵循你在规划中做出的安全与兼容决策"。

**阶段 1 规划质量（高）**：模型明确决策——
- "本机只表示调用方是本机进程，不表示已获得患者数据权限"；
- "不再允许无 Cookie/伪 Cookie 时取任意活跃 session 的兼容路径"；
- "context policy 即使本机调用也必须走独立 v3 session 策略，不读取 _is_local_request()"；
- "voice 兼容方案是先取 session/current 再携带，而不是放宽 context 授权"。

**结果（R17）**：Ability **87 / B+**（F3-05 −5、F6-03 −2、F8-03/08 −2、F12-04 −1、F9 −3）。

**关键发现**：模型在规划阶段"说对了"（显式 session 语义），但**实现阶段未忠实执行自己的
规划**——`build_chat_context_v3` 仍为 `session = get_session(session_id) if session_id else
get_current_session()`（隐式回退）。两阶段执行不能保证实现遵循规划，F3-05 依旧失败。

## 7. 实验小结（截至 R17）

| 方法 | 轮次 | Ability | F3-05 | 轨迹 |
|---|---|---|---|---|
| 基线（默认调用） | R13/R14/R15 | 96/91/88 | 1/3 通过 | we need ×2、let me ×1 |
| 方法 A（元认知引导） | R16 | 90 | 挂 | we need（完美锚定） |
| 方法 B（两阶段） | R17 | 87 | 挂 | 规划正确但实现回退 |

F3-05 稳定通过的方法尚未找到；模型默认行为是隐式回退，R13 的通过为采样运气。

## 8. 修订与统一口径（v0.2，round-01 评估闭环）

### 8.1 方法 A 结论降级（P1-1）

「方法 A 未转化为 F3-05 通过」**降级为假设**：n=1 下与采样随机无法区分
（基线 p(通过)=1/3 时，单次失败概率 2/3；R16=90 落在基线失败区间 88–91）。
保留成立部分：**轨迹锚定有代码级证据**（R16 先读契约、docstring 写明显式 session 语义）。

### 8.2 实验登记表（P1-2）

| 轮 | 方法 | 通道 | 模型/effort | 温度 | 任务文本 | F3-05 |
|---|---|---|---|---|---|---|
| R13 | 基线 | web/API | v4-pro/max | 默认 | 基线文本 | 通过 |
| R14 | 基线 | web/API | v4-pro/max | 默认 | 基线文本 | 挂 |
| R15 | 基线 | verify（修复版） | v4-pro/max | 默认 | 基线文本 | 挂 |
| R16 | 方法 A | web/API | v4-pro/max | 默认 | 基线+元认知引导 | 挂 |
| R17 | 方法 B | web/API（两阶段） | v4-pro/max | 默认 | 规划+基线 | 挂 |

通道混杂说明：R15 用 verify 通道（headless persona 覆盖），首行 "Let me"；其余 web/API
（minimal persona 完整），首行 "We need"。通道差异可能贡献分数差异（88 vs 91/96）。

### 8.3 轨迹统一口径（P2-2）

口径：assistant/message 中 type=reasoning 的文本全文，统计 `let me`、`we need`、`we can`
（不区分大小写）。脚本见 §11。修正后数据：

| 轮 | let_me | we_need | we_can | 首行 |
|---|---|---|---|---|
| R13 | 0 | 12 | 63 | We need execute onboarding todo... |
| R14 | 1 | 11 | 63 | We need execute task in workspace... |
| R15 | 7 | 9 | 68 | Let me start by exploring... |
| R16 | 0 | 19 | 72 | We need modify repo. Need first read all reference... |
| R17 | 1 | 26 | 102 | We need respond in Chinese likely... |

### 8.4 术语表（P3-2）

- anchored-standard db4527a2：GitHub main 最新版插件（self-contained 重构，行为配置同 ffb845c）。
- web/API 通道：`apps/cli web` + `session.create(agentPreset=...)`，preset 完整装配。
- verify 通道：`verify/run-verify.mjs` + headless profile，headless persona 覆盖 preset。
- 温度：harness 默认（未显式设置）。

## 9. 方法 C：采样-选择（STS）+ 公开代码语义代理判据

**依据**（round-01 评估者建议）：R13 证明正确实现已在模型能力分布内（单轮出现概率约 1/3）；
STS 不改变模型输入，只把单轮运气转化为 N 次采样 + 可量化选择，且代理判据只用公开代码语义
（不泄露 hidden tests、不写进 prompt）。

### 9.1 代理判据（harness 侧，不进 prompt）

对候选 `gateway.py` 的 `build_chat_context_v3` session 解析段：

```text
PASS 条件：解析段中【不】调用 get_current_session()（无 session_id 时不得隐式回退）
REJECT 条件：解析段中存在 get_current_session()（含 if/elif/else 任一分支）
```

### 9.2 回测结果（对 R13–R17，5/5 全中）

| 候选 | 判据 | 实际 F3-05 | 一致 |
|---|---|---|---|
| R13 | PASS | 通过 | ✅ |
| R14/R15/R16/R17 | REJECT | 挂 | ✅ |

### 9.3 验证方案

1. 预实验：同一任务文本（基线文本），N=3 并行 web/API 会话；代理选出通过者（首个 PASS）
   提交 hidden 评分；对照组随机抽取 1 个。判据：预实验中代理通过者 F3-05 通过。
2. 正式：连续 3 轮 × N=5（并行），每轮选代理通过者评分；对照组随机抽取。
   验收：STS 组 3 轮 ≥2 轮 Ability=96 且 F3-05 通过；报告代理增益（两组通过率差）。
3. 成本控制：预实验 N=3；若代理通过率 <1/3 或误杀，先回查判据。

### 9.4 备选（若采样成本不可接受）

子任务粒度调整：独立子任务"重写 build_chat_context_v3（公开接口签名不变）"，消除与既有
兼容代码共存的上下文压力。验证同 STS。

### 9.5 排除

- 温度控制：不作独立方法，仅作 STS 采样超参数（0.7/1.0 混合）。
- 角色/元认知变体：R17 已证"规划说对"不转化实现，无定向机制。

## 10. 10 次单次跑分实验（用户修正方向，v0.3）

用户明确：STS（多次采样选择）不符合"单次稳定获得好模型"的要求。改为：
**10 次独立单次完整评测（基线任务文本、无任何约束），分析成功样本（96+/F3-05 通过）
的共性特征，逐渐收敛条件以定位"单次稳定触发好模型"的原因**。

### 10.1 样本清单

| 样本 | 工作区 | 状态 |
|---|---|---|
| S1 | FV1/c1 | 完成：**97 / A / blockers 空** |
| S2 | FV1/c2 | 完成：**96 / A / blockers 空** |
| S3 | FV1/c3 | 无效（cwd 错位，不计） |
| S4 | FV1/c4 | 进行中 |
| S5 | FV1/c5 | 完成：**97 / A / blockers 空** |
| S6–S11 | 175321/175322/175323/175330/175331/175341 | 已启动 |

### 10.2 每样本记录特征（用于共性分析）

- 代理判据（fallback 有无）、Ability/F3-05、blockers；
- 轨迹：首行、let_me / we_need / we_can 计数；
- 代码模式：`build_chat_context_v3` 的 session 解析实现；
- 会话特征：消息数、工具调用步数、PR 报告质量。

## 11. 10 次跑分结果与共性收敛（v0.3 完成）

### 11.1 结果表（10 个有效样本，基线文本、无约束、V4 Pro max、web/API）

| 样本 | 代理判据 | Ability | F3-05 | 首行 | lm/wn/wc | tools | explicit 讨论 |
|---|---|---|---|---|---|---|---|
| S1 c1 | PASS | **97** | 过 | We need | 1/31/137 | 297 | 52 |
| S2 c2 | PASS | **96** | 过 | We need | 0/28/100 | 292 | 98 |
| S4 c4 | REJECT | 90 | 挂 | We need | 0/17/112 | 306 | 56 |
| S5 c5 | PASS | **97** | 过 | We need | 0/21/124 | 328 | 51 |
| S6 175321 | PASS | 93 | 过 | We need | 2/10/58 | 150 | 96 |
| S7 175323 | REJECT | 91 | 挂 | We need | 1/17/104 | 236 | 81 |
| S8 175322 | PASS | **96** | 过 | We need | 0/12/51 | 169 | 51 |
| S9 175330 | REJECT | 84 | 挂 | We need | 0/13/59 | 223 | 86 |
| S10 175331 | PASS | 95 | 过 | We need | 0/4/65 | 196 | 87 |
| S11 175341 | REJECT | 89 | 挂 | We need | 1/13/66 | 235 | 98 |

### 11.2 共性收敛结论（定位原因）

1. **轨迹无区分度**：10/10 首行均为 "We need"（minimal-like 锚定），成功组与失败组完全同质；
2. **工作量无区分度**：工具调用 150–328、消息 133–309，成功/失败区间重叠；
3. **推理讨论无区分度**：所有样本 reasoning 中 explicit-session 语义讨论 51–98 次，失败样本
   （S7/S9/S11）甚至明确写出 "should NOT fallback to get_current_session"——**模型在推理中
   全部"知道"正确语义**；
4. **唯一 10/10 区分变量 = 代理判据**（`build_chat_context_v3` 最终代码中是否隐式回退）：
   PASS 6/6 F3-05 通过（93–97），REJECT 4/4 挂（84–91）。

**根因定位**：模型的一致缺陷不是"认知"（不知道显式 session 语义），而是**实现阶段的执行
不一致**——推理/规划正确，但写代码时回退到"安全语义 + 兼容逃生门"的默认形态。输入层方法
（prompt 引导、两阶段规划、轨迹锚定）无法解决此问题（R16/R17/本轮均证实）。

**对"单次稳定"的含义**：
- 当前模型单次 F3-05 通过率约 60%（本轮 6/10），不可由输入层方法稳定到 100%；
- 代理判据可 100% 事后识别正确样本（10/10），但属于"选择"而非"稳定单次"；
- 若要单次稳定，需环境/工具层脚手架（把无 session_id 拒绝作为公开模板或参考实现），
  或接受 60% 为当前模型单次能力分布。

## 12. 补充验证（v0.4）：zero 变体 + 最后两轮 anchored

### 12.1 zero-anchored-standard（3 次并行，基线文本、无约束）

| 样本 | 首轮 | 代理判据 | Ability | F3-05 |
|---|---|---|---|---|
| Z1 | 0 工具 anchor turn | REJECT | 86 | 挂 |
| Z2 | 0 工具 anchor turn | REJECT | 88 | 挂 |
| Z3 | 0 工具 anchor turn | REJECT | 90 | 挂 |

机制验证生效（首请求零工具 + 合成 anchor 消息 + minimal persona），但 **0/3 F3-05 通过**。
零工具首轮不优于 anchored（同期 anchored 6/10），与第三方 PE998 审计一致（zero 步数最多、
成本更高）。**首轮工具面（0 或 2）不是 F3-05 的杠杆。**

### 12.2 anchored-standard 最后两轮（基线文本、无约束）

| 样本 | 代理判据 | Ability | F3-05 |
|---|---|---|---|
| FV1 | REJECT | 83 | 挂 |
| FV2 | REJECT | 90 | 挂 |

### 12.3 全部单次数据汇总（anchored + zero，无任何显式约束）

- anchored-standard：6/12 F3-05 通过（通过轮 93–97，失败轮 83–91）；
- zero-anchored-standard：0/3 F3-05 通过（86–90）；
- **代理判据：15/15 命中**（PASS → F3 过，REJECT → F3 挂，含 zero 变体）。

**最终结论**：在严格遵守"无针对测试的显式约束"前提下，当前 V4 Pro 单次 F3-05 通过率
约 40–60%（波动），不可由输入层方法（引导/规划/轨迹/首轮工具面 0 或 2）稳定提升；
唯一 100% 可靠的是**事后代码语义判据**（15/15），但那是"识别"不是"稳定单次"。
