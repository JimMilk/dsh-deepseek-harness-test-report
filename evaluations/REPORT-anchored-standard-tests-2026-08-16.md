# DeepSeek V4 Pro × anchored-standard 多环境实测报告

> 日期：2026-08-16 · 作者：/root（Codex 工程 Agent）· 适用范围：modeltest Project2 题面
> **执行主体**：本报告由**搭载 DeepSeek-V4-Flash 的 Codex**（OpenAI Codex 桌面应用内的工程 Agent）
> 编写并驱动全部测试；被测对象为**搭载 DeepSeek-V4-Pro（reasoningEffort=max）的 DeepSeek Harness
> （0.1.0-rc.5）** 会话。即：**用搭载 v4-flash 的 Codex，给搭载 v4-pro 的 DSH 做测试。**
> **涉及项目**：插件项目 = [`xiaobright/dsh-anchored-standard`](https://github.com/xiaobright/dsh-anchored-standard)；
> 测试项目（基准） = [`xiaobright/modeltest`](https://github.com/xiaobright/modeltest)（Project2 题面）。
> 面向读者：dsh-anchored-standard 插件制作者（xiaobright）、V4 Pro 轨迹双路径观察提供者、
> 以及任何需要复算/对照本项目结论的工程师。
> 全部数据可复算：评分产物、会话日志、探针结果路径见 §11。

## 0. 摘要（30 秒版）

1. 在 **macOS / Windows 11 ARM64 VM / Linux（Lima Ubuntu aarch64）** 三个环境、4 种 preset
   （anchored-standard 旧版/新版/冻结快照、minimal、standard）、**共 11 轮完整 Project2 评测**
   （全部 DeepSeek V4 Pro + reasoningEffort=max + modeltest V4.1b 官方评分）下，Ability 区间
   **85–90**，无任何一轮接近插件 README 声称的 98/99。
2. **F3-05（−5）与 F12-04（−1）在全部 11 轮稳定失败**，与操作系统、preset、工具面、system
   prompt 无关。根因是模型对「敏感目标上下文必须显式 session 授权」语义的系统性实现缺陷：
   模型一致地实现 `session = get_session(session_id) if session_id else get_current_session()`
   （11 个候选工作区 9 个逐字相同），把 current session 当作隐式授权来源。
3. **reasoning 轨迹（we/let me）与分数无相关**：let_me=0 的轮次 87 分，let_me=167 的轮次 88 分。
   轨迹是 persona 路由表象，不是能力或分数原因。
4. **环境（OS）不改变分数**：三平台同水平带；F9（ESP-IDF 真实编译，−3）是唯一环境硬上限，
   macOS 任何模型封顶 97，叠加模型稳定缺陷（F3/F12 −6）后理论上限 91，实测 85–90。
5. 对插件制作者：**anchored-standard 的设计前提（首轮工具 schema 决定轨迹）在当前模型上不再稳定成立**；
   对观察提供者：**tools/tool_choice 双路径的「工具面→轨迹」部分可复现，但「长 prompt 覆盖」未复现，
   且 harness 层不支持 tool_choice 配置**。

## 1. 背景与问题

本报告记录 2026-08-15/16 对以下命题的独立验证：

- P1：`xiaobright/dsh-anchored-standard` 是否能在本地复现 README 声称的 Project2 98/99？
- P2：外部观察「DeepSeek V4 Pro 的 reasoning 风格由协议路径（tools/tool_choice）与语义路径
  （强 system prompt）双路径控制」是否成立？
- P3：插件作者文档「Pro 的首轮 model-visible tool schema 显著影响后续轨迹」在当前模型上是否成立？
- P4：环境（macOS / Windows / Linux、bash/read vs pwsh/read）对分数的影响。

评测基准：`xiaobright/modeltest`（V4.1b，frozen）。题面：Project2（护理/睡眠 gateway +
ESP32-S3 固件工程维护），Ability 100 分，F1–F12 能力域，hidden tests 对候选不可见。

## 2. 实验设置

### 2.1 模型与 API

- 模型：`deepseek-v4-pro`（harness catalog 对 DeepSeek 官方端点 `https://api.deepseek.com` 的
  wire id），`reasoningEffort=max`（serialize 为 `thinking: enabled, reasoning_effort: max`）。
- API key：`DEEPSEEK_API_KEY` 环境变量（start-dsh.sh 注入，未落盘至报告/记忆）。

### 2.2 Harness

- deepseek-harness `0.1.0-rc.5`，commit `47f9438`（与插件 README 声明的兼容版本一致）。
- 通道差异（关键）：
  - **verify 通道**：`verify/run-verify.mjs` 驱动 headless profile。该通道的 system prompt 由
    headless bundle 提供（"You are a coding agent powered by the {{model}} model..."），**preset 的
    minimal persona 不生效**；且该通道对官方 minimal preset 的工具隔离不生效（首请求 26 工具）。
  - **web/API 通道**：`apps/cli web` + `session.create(agentPreset=...)`。该通道 preset 完整装配：
    persona、工具目录、上下文注入均按 preset 生效。**只有此通道能测出真正的 minimal 与
    anchored-standard 完整行为**（R5–R9、R11）。

### 2.3 Preset 版本

- anchored-standard 旧版：`f57a1bd`（首次测试时 GitHub main）。
- anchored-standard 新版：`ffb845c`（重装时最新 main，#29 dev-tool-search scoped schemas）。
- anchored-standard 冻结快照：`modeltest/tools/deepseek-harness-presets/anchored-standard/`
  （作者 98/99 双跑同款，3 文件；Windows 首轮 `pwsh/read`）。
- minimal / standard：harness 官方 shipped preset（`apps/cli/config/agent-presets/`）。

## 3. 完整评测数据（11 轮）

### 3.1 总表

| 轮 | 时间(UTC+8) | 环境 | preset | 通道 | 首轮工具 | 轨迹 let_me/we | Ability | Ship/Class | 失败项 |
|---|---|---|---|---|---|---|---|---|---|
| R1 | 08-15 19:39–20:05 | macOS | anchored 旧版 f57a1bd | verify | bash+str_replace_editor | 178/3 | 87 | 87/B+ | F3-05,F6-03/05,F12-04,F9 |
| R2 | 08-15 20:19–21:13 | macOS | anchored 新版 ffb845c | verify | bash+str_replace_editor | **0/83** | 87 | 87/B+ | F1-05,F3-05,F4-01,F12-04,F9 |
| R3 | 08-15 21:32–22:00 | macOS | anchored 冻结快照 | verify | bash+read | 49/34 | 87 | 87/B+ | F3-05,F6-03,F8-05/08,F12-04,F9 |
| R4 | 08-15 22:15–22:43 | macOS | anchored 冻结快照 | verify | bash+read | **1/46** | **90** | 90/B+ | F3-05,F8-08,F12-04,F9 |
| R5 | 08-15 23:06–23:48 | macOS | minimal | **web** | bash+str_replace_editor | 8/69 | 89 | 89/B+ | F3-05,F6-03,F12-04,F9 |
| R6 | 08-16 00:07–00:26 | macOS | standard | web | 26 工具全量 | **167/0** | 88 | 88/B+ | F3-05,F8-03/05/08,F12-04,F9 |
| R7 | 08-16 00:37–01:13 | macOS | anchored 三折1（preset 内置 minimal） | web | bash+read | 45/53 | 89 | 89/B+ | F3-05,F8-05/08,F12-04,F9 |
| R8 | 08-16 01:21–01:53 | macOS | anchored 三折2 | web | bash+read | 35/33 | 89 | 89/B+ | F3-05,F8-05/08,F12-04,F9 |
| R9 | 08-16 01:54–02:27 | macOS | anchored 三折3 | web | bash+read | 23/45 | 87 | 87/B+ | F3-05,F4-01,F12-04,F9 |
| R10 | 08-15(VM 时) | **Windows 11 ARM64 VM** | anchored 冻结快照 | verify | **pwsh+read** | 首行 Let me | 87 | 87/B+* | F1,F3-05,F8-05/08,F12-04,F9 |
| R11 | 08-16 03:38–04:20 | **Linux Lima Ubuntu aarch64** | minimal | **web** | bash+str_replace_editor | 未取得(会话未落盘) | 85 | 85/B+ | F3-05,F4-01,F6-03,F12-04,F9 |

*R10 的 Ship/Class 受 VM 无 git 的 `--no-diff` 伪影影响（T-tamper/P-report 为检查脚本 ERROR，
非模型篡改）；Ability 不受影响。

### 3.2 数据一致性核验

11 轮 `family_draft` 求和与 `ability_draft` 完全一致（脚本核对，见 §11 汇总 JSON）。
每轮除 `F9-01`（skipped_env）与 `F11-01`（heuristic，实际满分 4/4）外，失败项均为真实扣分。

### 3.3 能力域得分矩阵（family_draft）

| 轮 | F1 | F2 | F3 | F4 | F5 | F6 | F7 | F8 | F9 | F10 | F11 | F12 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| R1 | 8 | 12 | 11 | 4 | 12 | 6 | 8 | 8 | 3 | 8 | 4 | 3 |
| R2 | 8 | 12 | 11 | 0 | 12 | 10 | 8 | 8 | 3 | 8 | 4 | 3 |
| R3 | 8 | 12 | 11 | 4 | 12 | 8 | 8 | 6 | 3 | 8 | 4 | 3 |
| R4 | 8 | 12 | 11 | 4 | 12 | 10 | 8 | 7 | 3 | 8 | 4 | 3 |
| R5 | 8 | 12 | 11 | 4 | 12 | 8 | 8 | 8 | 3 | 8 | 4 | 3 |
| R6 | 8 | 12 | 11 | 4 | 12 | 10 | 8 | 5 | 3 | 8 | 4 | 3 |
| R7 | 8 | 12 | 11 | 4 | 12 | 10 | 8 | 6 | 3 | 8 | 4 | 3 |
| R8 | 8 | 12 | 11 | 4 | 12 | 10 | 8 | 6 | 3 | 8 | 4 | 3 |
| R9 | 8 | 12 | 11 | 0 | 12 | 10 | 8 | 8 | 3 | 8 | 4 | 3 |
| R10 | 6 | 12 | 11 | 4 | 12 | 10 | 8 | 6 | 3 | 8 | 4 | 3 |
| R11 | 8 | 12 | 11 | 0 | 12 | 8 | 8 | 8 | 3 | 8 | 4 | 3 |

## 4. 轨迹（reasoning 风格）分析

### 4.1 词频统计（assistant/message 的 reasoning 文本）

- R2/R4 为 minimal-like（let_me 0/1，we 83/46）；R6 为 standard-like（let_me 167，we 0）；
  R1/R3/R7/R8/R9 为混合。
- **同配置跨轮波动**：R3 vs R4（同为冻结快照 + verify 通道）let_me 49 vs 1；R7–R9
  （同为 web 通道 anchored 三折）let_me 45/35/23——**首轮工具 schema 对完整会话轨迹不具备决定性**。
- **轨迹与分数无单调关系**：let_me=0（R2）87、let_me=1（R4）90、let_me=167（R6）88。

### 4.2 微探针矩阵（trigger_probe，每组合 2 次，V4 Pro max）

| 工具面 | 结果 | 与作者 2026-08-14 观测对比 |
|---|---|---|
| minimal bash+read | 2/2 minimal-like | 一致 |
| standard 25 工具 | **2/2 minimal-like** | **不一致**（作者：Let me / standard-like） |
| bash only | 1 minimal-like + 1 ambiguous | 基本一致 |
| bash+read | 2/2 minimal-like | 一致 |
| **bash+glob** | **1 standard-like + 1 minimal-like** | 分界弱复现（作者：Let me） |
| bash+edit / write | minimal-like | 一致 |

结论：当前模型（2026-08-16）整体强烈倾向 minimal-like 轨迹，即使完整 Standard 工具面也如此；
`glob` 作为触发分界只弱复现。作者 2026-08-14 的「Standard 25 → Let me」在当前权重上不再成立。

### 4.3 对「双路径观察」的验证（api-probe，7 组合 × 3 次）

| 组合 | we_need/let_me 结果 |
|---|---|
| 无 tools、无 system | 3/3 we_need |
| tools: [] | 3/3 we_need |
| tools + tool_choice=auto（bash only） | 2/3 let_me |
| tools + tool_choice=none | 3/3 we_need |
| 强 system prompt + tools auto | 1/3 we_need、2/3 let_me |
| 强 system prompt、无 tools | 3/3 we_need |
| 普通 system prompt + tools auto | 3/3 we_need |

协议路径（tools 空/none → we need；tools+auto → let_me 倾向）**部分成立**，但不稳定
（standard 25 + auto 在微探针中 2/2 we_need）。语义路径（「长 prompt 覆盖协议」）**未复现**：
本实验构造的强 prompt 未覆盖；而 web standard 的完整 system + 全工具在完整会话中反而稳定
let_me——轨迹由 system 内容与完整会话共同决定，不是「长度」或「协议字段」单方决定。

## 5. 稳定缺陷深挖

### 5.1 F3-05（ambient 泄漏，−5，全 11 轮失败）

测试语义（`test_context_policy.py:183`）：DB 存在有效 staff 会话，请求
`build_chat_context_v3({"target_subject_id": ["sub_patient_a"]})` **不传 session_id**，
断言 `policy.allowed == False`——敏感目标必须显式 session 授权，不得静默套用 current session。

模型实现（11 个候选工作区中 9 个逐字相同，其余 2 轮语义相同）：

```python
session = get_session(session_id) if session_id else get_current_session()
```

无 session_id 时 fallback 到 current session → staff 当前会话被视为授权 → `allowed=True` → 失败。
即使 R5/R9 两轮代码写了 "Never fall back to an ambient session..." 注释，else 分支仍是
`get_current_session()`——模型理解了「id 失效不回退」，没理解「无 id 也不回退」。

推断动机：任务指南 F10 要求本机 worker 路径（`/api/v3/context/chat` 等）可用，worker 调用通常
无 session_id；模型为兼容 worker 实现了隐式授权，未区分「worker 隐式调用」与「敏感目标数据请求」。

**结论：不是 modeltest 缺陷**（语义合理、可复现、作者 98/99 轮次通过）；**不是 OS 环境问题**
（三平台同模式）；是**当前模型对显式授权语义的稳定实现缺陷**。

### 5.2 F12-04（reason 语义，−1，全 11 轮失败）

无 actor_subject_id 的 session 在策略 reason 中返回 `not_authenticated`，期望
`not_authorized_for_target`（session 存在但无法代表目标授权）。模型 11 轮全部未做对。

### 5.3 轮次波动项（非稳定缺陷）

- F4 voice bridge：R1/R3/R5/R7/R8 过，R2/R9/R11 挂（模型实现变化）。
- F6 DB 迁移：R2/R4/R6/R7/R8/R9/R10 满分，R1/R3/R5/R11 挂（ts 回填/排序）。
- F8 ESP 静态契约：多数过，R3/R6/R7/R8/R10 挂 1–3 分（MQTT marker/网络配置依赖）。

## 6. 环境因素与分数上限推导

- F9（ESP-IDF 真实编译，6 分）：macOS/Linux/Windows VM 均无 ESP-IDF v6.0.1 工具链 →
  `skipped_env` 3/6。**这是唯一环境硬上限**：本机任何模型 Ability ≤ 97。
- 叠加模型稳定缺陷（F3 −5、F12 −1）→ **理论上限 91 = 100 − 3 − 5 − 1**。
- 实测 85–90 = 91 再减去轮次波动（F1/F4/F6/F8 单轮 −2 到 −4）。
- **Windows VM（R10）未补 F9**：ARM64 架构无 xtensa toolchain、未装 ESP-IDF，F9 仍 3/6；
  因此 87 与 macOS 同水平带，印证「OS 不改变分数」。
- 作者 98/99 的复现前提（Windows EIM + 完整 ESP-IDF + 当时模型权重）在本测试中均未满足；
  其中「模型权重」是最大未知变量（见 §7）。

## 7. 插件制作者（xiaobright）视角

1. **设计验证**：anchored-standard 的「minimal persona + 首轮窄工具 + promote 全量」组合在
   web/API 通道可完整装配（R7–R9 验证 system 为 "You are a helpful software engineer assistant."、
   首轮 bash+read、promote 后 25 工具）。冻结快照版在 Windows 上首轮确为 pwsh+read（R10）。
2. **当前模型行为漂移**：插件前提「首轮工具 schema 决定轨迹」在当前 V4 Pro 权重上不再稳定
   （微探针 standard 25 也 minimal-like；完整会话同配置跨轮 let_me 0–178 波动）。98/99 的
   trajectory（355 blocks 1 let_me）在 2026-08-16 无法复现，首行普遍为 "Let me..."。
3. **通道注意**：verify/headless 通道会覆盖 preset persona（headless bundle 的 system-prompt row），
   且对官方 minimal 工具隔离不生效——**测试 anchored-standard 的完整行为必须走 web/API 装配**，
   否则首轮 persona 并非 minimal，轨迹结论会失真。
4. **分数瓶颈不在 preset**：F3-05/F12-04 是模型稳定缺陷，与 preset 无关；建议向模型层反馈
   （显式授权语义），而非继续调工具目录。
5. **小问题**：README 兼容性声明（0.1.0-rc.5 / 47f9438）与实测一致；`verify/run-verify.mjs`
   的 `stopAfterFirstAssistant` 取消逻辑在本地实测为无条件取消（仅测首请求），需加条件守卫
   才能跑完整任务（本报告测试已本地修复）。

## 8. 观察提供者视角

1. 「tools 不存在/空/`tool_choice=none` → we need；tools 非空 + auto → let me」在 API 微探针
   中部分成立（无 tools/空/none 稳定 we_need；bash-only+auto 2/3 let_me），**但 non-deterministic**
   （standard 25 + auto 反例；bash+glob 1/2）。
2. 「长 prompt 覆盖协议」未复现：本实验构造的强 prompt 未覆盖；web standard 完整 system 反而
   let_me。**「长度」不是变量，内容才是**——需要可复现的《某长prompt》原文才能进一步验证。
3. **harness 层无法配置 tool_choice**（`GenerateOptions` 无该字段，adapter serialize 只写
   tools），只能改工具目录——协议路径在 harness 内的可操作空间有限。
4. 「we/let me 是 persona 路由差异而非能力差异」：本报告 11 轮数据支持该判断的推论
   （轨迹与分数无关），但「同一权重不同激活路径」无法从黑盒 API 观测证实/证伪（与作者
   trigger 文档的立场一致：不能声称识别了隐藏路由）。

## 9. 限制与未做

- R10/R11 会话文件未落盘（verify/rc.6 bundles 持久化差异），轨迹仅部分取得；评分数据完整。
- 未复现 modeltest 98/99：未在装好 ESP-IDF 的 Windows x64 EIM 环境实测（本机无此环境）。
- 未使用作者 2026-08-14 的同一模型快照（API 无版本 pin，模型权重可能已变）。
- 未做 F3-05 修复反馈迭代（避免评测污染；若要测「模型+反馈」上限需另立口径）。
- 全部结论限于 Project2 题面，不构成跨任务普适结论（modeltest 自身立场）。

## 10. 结论

1. anchored-standard 在当前 V4 Pro（2026-08-16）上实测 85–90，未复现 98/99；差距主要来自
   模型稳定缺陷（F3/F12）与环境 F9 上限，而非 preset。
2. reasoning 轨迹与分数无关；「we need 高分」在当前模型上既不充分也不必要。
3. 双路径观察部分成立但非确定性；语义路径需要原文才能进一步验证。
4. 环境（OS）不改变分数；唯一环境变量是 ESP-IDF 工具链（F9）。

## 11. 数据来源（可复算）

- 评分产物：`evaluations/modeltest-run/evaluator/results/20260815_200511 ~ 20260816_042005`
  （R1–R9、R11）；R10 在 Windows VM `C:\Users\newmaple\modeltest\evaluator\results\20260815_192122`。
- 汇总 JSON：`/tmp/all-results-summary.json`、`/tmp/trajectory-summary.json`。
- 探针：`evaluations/modeltest-run/evaluator/trigger_probe/results/*.json`（14 次微探针）。
- 会话：macOS `~/.dsh/sessions/`（R1–R9）；Windows VM / Linux 会话未落盘（见 §9）。
- 候选工作区：`evaluations/modeltest-run_candidate_handoffs/2026*`（各轮 diff）。
- 复算命令：`python evaluator/run_full_eval.py <workspace>/project2_task`；
  `node src/cli.mjs --run --model deepseek-v4-pro --tools <tools.json>`（trigger_probe）。

---

*报告内全部数字经脚本核验（family 求和 = Ability、失败项与 hidden_summary 一致）；如发现出入，
以 §11 原始数据为准。*
