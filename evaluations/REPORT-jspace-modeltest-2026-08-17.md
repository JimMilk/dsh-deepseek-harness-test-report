# J-Space × DeepSeek V4-Flash modeltest 对照测试报告（2026-08-17）

> 复现对象：[DeepSeek-V4-J-Space-Capability-Realization-Report](https://github.com/Tiger3807861189/DeepSeek-V4-J-Space-Capability-Realization-Report)
> 评测基准：[xiaobright/modeltest](https://github.com/xiaobright/modeltest) Project2（V4.1b frozen）
> 被测环境：DeepSeek Harness 0.1.0-rc.5 + J-Space-Cognition-Suite-V3.6（`~/.dsh/skills/j-space`）

## 1. 目的与设计

用 modeltest 官方评分器（Project2 题面，Ability 100 分、F1–F12 能力域、hidden tests 对候选不可见）做 J-Space 开关对照：同一模型（`deepseek-v4-flash`）、同一 harness、同一任务、同一工具面，**唯一变量 = 是否在 persona 中注入 J-Space Cognition Protocol**（用户主动加载，不注入无关 Skill 目录，符合报告 §4.1）。

| 轮 | 模型 | J-Space | 通道 |
|---|---|---|---|
| J-Space r1–r4 | deepseek-v4-flash / reasoningEffort=max | 注入协议（`--patch` persona） | headless |
| Control r1–r4 | deepseek-v4-flash / reasoningEffort=max | 无 | headless |

每条件复跑 4 轮（共 8 轮），轮间模型、reasoningEffort 均经 session 日志核验；J-Space 轮系统提示含 `J-SPACE COGNITION PROTOCOL`，Control 轮不含。任务文本逐字采用 modeltest 官方 CANDIDATE_PROMPT（仅附加候选工作区路径），J-Space 为通用协议注入，未加入任何针对测试项目（Project2 具体失败项）的显式约束。

## 2. 结果

### 2.1 总分

| 轮 | Ability | Ship | Class | hidden 通过率 |
|---|---:|---:|---|---:|
| Control r1 | 89.5 | 72.0 | B+ | 42/45 |
| Control r2 | 93.5 | 93.5 | B+ | 43/45 |
| Control r3 | 89.5 | 72.0 | B+ | 42/45 |
| Control r4 | 93.0 | 72.0 | B | 44/45 |
| **Control 均值** | **91.38** | — | — | — |
| J-Space r1 | 92.0 | 72.0 | B+ | 44/45 |
| J-Space r2 | 93.5 | 72.0 | B+ | 43/45 |
| J-Space r3 | 92.5 | 72.0 | B+ | 43/45 |
| J-Space r4 | 94.5 | 94.5 | B+ | 44/45 |
| **J-Space 均值** | **93.12** | — | — | — |

### 2.2 能力域（family_draft）

| 轮 | F1 | F2 | F3 | F4 | F5 | F6 | F7 | F8 | F9 | F10 | F11 | F12 | Σ |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Control r1 | 6 | 12 | **11** | 4 | 12 | 10 | 8 | 8 | 3.5 | 8 | 4 | 3 | 89.5 |
| Control r2 | 8 | 12 | **16** | 4 | **11** | 10 | 8 | **6** | 3.5 | 8 | 4 | 3 | 93.5 |
| Control r3 | 6 | 12 | **16** | 4 | 12 | **7** | 8 | **6** | 3.5 | 8 | 4 | 3 | 89.5 |
| Control r4 | 8 | 12 | **16** | 4 | 12 | 10 | 8 | 8 | 0 | 8 | 4 | 3 | 93.0 |
| J-Space r1 | 8 | 12 | **16** | 4 | 12 | 10 | 8 | 7 | 0 | 8 | 4 | 3 | 92.0 |
| J-Space r2 | **6** | 12 | **16** | 4 | 12 | 10 | 8 | 7 | 3.5 | 8 | 4 | 3 | 93.5 |
| J-Space r3 | 6 | 12 | **16** | 4 | 12 | 10 | 8 | 6 | 3.5 | 8 | 4 | 3 | 92.5 |
| J-Space r4 | 8 | 12 | **16** | 4 | 12 | 10 | 8 | 6 | 3.5 | 8 | 4 | 3 | 94.5 |

### 2.3 hidden 失败项

| 失败项 | C-r1 | C-r2 | C-r3 | C-r4 | J-r1 | J-r2 | J-r3 | J-r4 |
|---|---|---|---|---|---|---|---|---|
| V4-F3-05（敏感目标上下文须显式 session 授权） | 失败 | 通过 | 通过 | 通过 | 通过 | 通过 | 通过 | 通过 |
| V4-F12-04（无 actor_subject 的 reason 语义） | 失败 | 失败 | 失败 | 失败 | 失败 | 失败 | 失败 | 失败 |
| V4-F1-01（PR 模板必填 debug 命令记录） | 失败 | 通过 | 失败 | 通过 | 通过 | 失败 | 失败 | 通过 |
| V4-F5-06（care_event 创建须管理员认证） | 通过 | 失败 | 通过 | 通过 | 通过 | 通过 | 通过 | 通过 |
| V4-F6-01（旧 care_events 表迁移不丢数据） | 通过 | 通过 | 失败 | 通过 | 通过 | 通过 | 通过 | 通过 |

### 2.4 blockers

- Control r1：`S-ambient`、`P-report`、`E-build`；Control r2：`E-contract`、`E-build`；Control r3：`P-report`、`M-crash`、`E-contract`、`E-build`
- J-Space r1/r2/r3：`E-contract`、`E-build`（r1/r3 另含 `P-report`）

## 3. 关键差异分析

1. **F3-05（+5 分项）**：J-Space 4/4 全部通过；Control 3/4（仅 r1 失败）。此前的 11 轮 V4 Pro 全部失败；本次 8 轮中 7 轮通过。J-Space 轮四轮全部通过且 F3 满分，Control 通过说明该行为存在模型随机性，**F3-05 通过不能单独归因于 J-Space，但 J-Space 轮零失败、更稳定**。
2. **F12-04 全 8 轮失败**：与报告"模型稳定缺陷"一致，两条件均未覆盖该语义细节（−1）。
3. **轮次波动项（F1-01/F5-06/F6-01/F8/F9）**：两条件各有 1–2 轮单项失败，无一致的 J-Space 方向；F9 环境计分 0/3.5 波动与模型能力无关。
4. **Ability 分布（n=4）**：J-Space [92.0, 93.5, 92.5, 94.5] 均值 93.12（sd 1.11）；Control [89.5, 93.5, 89.5, 93.0] 均值 91.38（sd 2.17）。配对差值 [2.5, 0.0, 3.0, 1.5] 均值 **+1.75**（sd 1.32）；配对 t=2.65（df=3），**双尾 p≈0.08、单尾 p≈0.04**：达到方向性显著（单尾 0.05），未达双尾 0.05。J-Space 四轮无一轮低于 Control，波动更小。

## 4. 与历史数据对照

此前 2026-08-16 的 11 轮 V4 Pro（macOS/Windows/Linux × 4 种 preset）Ability 区间 85–90，**F3-05 与 F12-04 全部失败**。本次：

- V4-Flash 8 轮 Ability 89.5–94.5，普遍高于历史 V4 Pro 区间上沿（85–90）；
- **F3-05 通过 7/8**（J-Space 4/4、Control 3/4），首次出现多轮通过（此前仅 R13 通过 1 次）；
- J-Space 均值 93.12 > Control 均值 91.38，方向与报告一致（J-Space 减少状态/授权语义失配导致的能力实现损失）。

## 5. 最终结论（2026-08-17 6 轮数据）

1. **证据方向一致支持安装**：J-Space 4 轮 Ability 均不低于对应 Control（差值 +2.5/0/+3.0/+1.5，均值 +1.75），F3-05 4/4 vs 3/4，波动更小（sd 1.11 vs 2.17）；未发现任何 J-Space 系统性劣势（失败项与 Control 重叠）。
2. **方向性显著、未达双尾 0.05**：n=4，配对 t=2.65；双尾 p≈0.08、单尾 p≈0.04——"J-Space 提高 Ability"在单尾 0.05 水平成立；更严格的双尾 0.05 需继续加跑。
3. **已按结论执行**：J-Space 以纯净形态安装为默认 preset（`jspace-minimal` = Minimal 双工具 + 通用 J-Space 协议，无 Skill 目录注入）；移除 dsh 内其余实验 preset（anchored-standard、jspace-standard，备份可恢复）；`~/.dsh/skills/` 仅保留 `j-space` 核心插件。

## 6. 方法偏差（如实记录）

1. **通道**：headless（非 web/API 通道）。报告 §2.2 指出 verify/headless 通道不装配 preset roster；本实验用 `--patch` 直接注入 persona 实现 J-Space 变量，工具面为 headless 全工具（非 Minimal 双工具）。同轮内开关条件干净。
2. **权限**：headless 无审批通道，评测以 `DSH_PERMISSION_MODE=danger-full-access` 运行（候选工作区隔离、任务约束只操作工作区；两轮模型均未越界）。
3. **复跑 3 轮/条件**：单次波动真实存在（Control 89.5↔93.5），无置信区间；加跑可进一步提高统计力。
4. **候选隔离**：每轮独立 `prepare_candidate_handoff.py` 候选工作区，从零起点；任务文本与官方题面逐字一致，未加入任何针对测试项目的显式约束（避免污染评测）。
5. **temperature/top_p**：dsh 0.1.0-rc.5 适配器不提交 `top_p`、未配置时不提交 `temperature`（同前报告）。

## 7. 数据来源（可复算）

- J-Space r1：评分 `results/20260817_124106/`；候选 `modeltest-run_candidate_handoffs/20260817_jspace_v4flash_v2/workspace`。
- J-Space r2：评分 `results/20260817_143448/`；候选 `modeltest-run_candidate_handoffs/20260817_jspace_v4flash_r2/workspace`。
- J-Space r3：评分 `results/20260817_151357/`；候选 `modeltest-run_candidate_handoffs/20260817_jspace_v4flash_r3/workspace`。
- J-Space r4：评分 `results/20260817_160819/`；候选 `modeltest-run_candidate_handoffs/20260817_jspace_v4flash_r4/workspace`。
- Control r1：评分 `results/20260817_125758/`；候选 `modeltest-run_candidate_handoffs/20260817_control_v4flash/workspace`。
- Control r2：评分 `results/20260817_144819/`；候选 `modeltest-run_candidate_handoffs/20260817_control_v4flash_r2/workspace`。
- Control r3：评分 `results/20260817_152545/`；候选 `modeltest-run_candidate_handoffs/20260817_control_v4flash_r3/workspace`。
- Control r4：评分 `results/20260817_162310/`；候选 `modeltest-run_candidate_handoffs/20260817_control_v4flash_r4/workspace`。
- 评分命令：`python3 evaluator/run_full_eval.py <workspace>/project2_task --model deepseek-v4-flash --channel <ch> --harness dsh-0.1.0-rc.5 --require-meta --include-espidf-build --run-group-id <id> --run-index 1 --thinking-level max`（需提权运行：评分器绑定本地端口）。

## 8. 纯净安装与清理记录（2026-08-17）

**保留（纯净形态）**

- `~/.dsh/skills/j-space`：J-Space-Cognition-Suite-V3.6 核心插件（唯一 skill）。
- `~/.dsh/.agent-presets/jspace-minimal/`：默认 preset，Minimal 双工具 + 通用 J-Space 协议（persona 无测试项目相关内容，无 Skill 目录注入），preset.yml 已更新为正式名称 "J-Space Minimal"。
- `settings.yaml`：`agent-presets.default: jspace-minimal`，模型 `deepseek-v4-pro` / reasoningEffort max。

**移除（已备份，可恢复）**

- `anchored-standard`、`jspace-standard` → 已移至 `/private/tmp/dsh-presets-backup-20260817/`（恢复：移回 `~/.dsh/.agent-presets/` 即可）。

**验证**

- `~/.dsh/.agent-presets/` 仅剩 `jspace-minimal`；`~/.dsh/skills/` 仅剩 `j-space`；dsh web 正常（HTTP 200）。
- 新 web 会话默认装配 J-Space Minimal（系统提示应含 `J-SPACE COGNITION PROTOCOL`）；headless 通道不读 preset，不受此设置影响。

## 9. v4-Pro 单轮补充（2026-08-17 16:53）

按用户要求用 `deepseek-v4-pro`（reasoningEffort max）补跑 1 轮 J-Space（headless + `--patch`，任务文本与题面逐字一致）：

| 指标 | 值 |
|---|---:|
| Ability | **84.0**（Ship 72.0 / Class B） |
| family | F1=8 F2=12 F3=11 F4=4 F5=12 F6=8 F7=8 F8=6 F9=0 F10=8 F11=4 F12=3 |
| hidden 失败 | V4-F3-05（S-ambient）、V4-F12-04、V4-F6-03（ts 回填） |
| blockers | S-ambient、M-fidelity、E-contract、E-build、P-report |

评分：`evaluator/results/20260817_165357/`；候选 `modeltest-run_candidate_handoffs/20260817_v4pro_jspace/workspace`。

**解读（单轮，不作强结论）**：

1. 本轮 v4-Pro + J-Space **F3-05 未通过**（隐式 ambient 回退），与 v4-Flash + J-Space 8 轮中 4 轮 F3-05 全通过形成对比；也低于历史 v4-Pro Control 区间（85–90，n=11）。
2. n=1 无统计意义：84 分受 F3-05（−5）、F6-03（−2）、F8（−2）、F9（−3.5 环境）叠加影响，单轮不能断言"J-Space 对 v4-Pro 无效"或"Pro 弱于 Flash"；需 v4-Pro 多轮同期对照才能判断。
3. 该结果提醒：J-Space 的授权语义改善在 Flash 上稳定（4/4），在 Pro 上是否可复现未确认；若 v4-Pro 是日常主力模型，建议补跑 v4-Pro J-Space/Control 各 2–3 轮再定论。
