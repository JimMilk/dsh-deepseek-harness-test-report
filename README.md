# DeepSeek Harness 第三方实测报告

公开 DeepSeek Harness（dsh）在 DeepSeek V4 系列模型（`deepseek-v4-pro` /
`deepseek-v4-flash`，`reasoningEffort=max`）上的第三方评测报告与原始数据。
测试基准统一为 [`xiaobright/modeltest`](https://github.com/xiaobright/modeltest)
Project2 工程维护评测（V4.1b frozen，官方评分器）。

## 执行主体

本仓库全部测试由**配置了 `deepseek-v4-flash` 的 Codex 工程 Agent** 编写并驱动；
被测对象为搭载 DeepSeek V4 系列模型（`deepseek-v4-pro` / `deepseek-v4-flash`，
`reasoningEffort=max`）的 DeepSeek Harness（0.1.0-rc.5）会话。
即：**用配置了 v4-flash 的 Codex，给 DSH 做测试。**

## 报告索引

| 报告 | 日期 | 内容 |
|---|---|---|
| [J-Space × DeepSeek V4 总汇总](evaluations/SUMMARY-jspace-tests-2026-08-17.md) | 2026-08-17 | 9 轮 modeltest 对照 + 3 任务复现实验 |
| [J-Space modeltest 对照报告](evaluations/REPORT-jspace-modeltest-2026-08-17.md) | 2026-08-17 | v4-Flash 8 轮对照 + v4-Pro 单轮 |
| [J-Space v4-Flash 复现实验](evaluations/REPORT-jspace-v4flash-repro-2026-08-17.md) | 2026-08-17 | 3 任务开关实验（token/耗时/工具调用） |
| [anchored-standard 多环境实测](evaluations/REPORT-anchored-standard-tests-2026-08-16.md) | 2026-08-16 | 3 环境、4 preset、11 轮完整评测（V4 Pro） |
| [调用方法实验](evaluations/METHOD-EXPERIMENTS.md) | 2026-08-16 | R13–R17 基线/方法 A/B + 10 次采样共性收敛 |

## 核心结论（摘要）

### J-Space × DeepSeek V4（2026-08-17）

1. **J-Space 对 v4-Flash 有方向性显著提升**：modeltest Ability 均值 93.12 vs 91.38
   （+1.75），配对 t=2.65（n=4，单尾 p≈0.04）；关键项 **F3-05（显式 session 授权）
   J-Space 4/4 通过、Control 3/4**；无一轮低于对照、波动更小（sd 1.11 vs 2.17）。
2. **F12-04 两条件 8 轮全挂**（已知模型稳定缺陷）；F9（ESP-IDF 真机构建）受本机无工具链限制。
3. **v4-Pro + J-Space 单轮 84.0**（F3-05 未通过），n=1 不作结论，需多轮对照确认。
4. 实验保持题面纯净：任务文本与官方 CANDIDATE_PROMPT 逐字一致，未加入针对测试项目的显式约束。

### anchored-standard × V4 Pro 多环境（2026-08-16）

1. macOS / Windows 11 ARM64 VM / Linux（Lima Ubuntu）三环境、4 种 preset、11 轮完整
   Project2 评测，Ability 区间 **85–90**，未复现插件 README 的 98/99。
2. **F3-05（−5）与 F12-04（−1）全部 11 轮稳定失败**：模型一致地把无显式 session_id 的
   敏感目标请求回退到 current session 隐式授权，是模型层稳定缺陷，与 OS/preset/轨迹无关。
3. reasoning 轨迹（we/let me）与分数无相关；「首轮工具 schema 决定轨迹」在当前模型上不再稳定。
4. 环境（OS）不改变分数；F9（ESP-IDF 真实编译）是唯一环境硬上限（macOS 封顶 97，
   叠加模型缺陷后理论上限 91）。

### 调用方法实验（2026-08-16，补充）

- 基线 96/91/88（F3-05 通过 1/3）；方法 A 元认知引导 90、方法 B 两阶段执行 87，均未稳定通过 F3-05。
- 10 次单次采样：代理判据（代码中是否隐式回退 `get_current_session()`）15/15 命中
  PASS→F3-05 过（93–97）、REJECT→挂（84–91）；轨迹/工作量/推理讨论均无区分度。
- 结论：当前 V4 Pro 单次 F3-05 通过率约 40–60%，输入层方法（引导/规划/轨迹）无法稳定提升；
  唯一 100% 可靠的是事后代码语义判据，属"识别"而非"稳定单次"。

## 数据文件

| 文件 | 说明 |
|---|---|
| [reports-data/all-results-summary.json](reports-data/all-results-summary.json) | anchored-standard 11 轮 score_draft 汇总（ability/family/blockers/失败项） |
| [reports-data/trajectory-summary.json](reports-data/trajectory-summary.json) | macOS 各轮会话 reasoning 词频（let_me/we）与首行 |
| [reports-data/jspace-modeltest-results-summary.json](reports-data/jspace-modeltest-results-summary.json) | J-Space 9 轮评分摘要（含 family 明细与 hidden 失败项） |
| [reports-data/README.md](reports-data/README.md) | 数据文件说明与来源指引 |

## 环境与配置

- dsh 0.1.0-rc.5；默认模型 `deepseek-v4-flash` / `reasoningEffort=max`。
- 默认 preset `jspace-minimal`（Minimal 双工具 + 通用 J-Space 协议，纯净安装）；
  `~/.dsh/.agent-presets/` 仅 `jspace-minimal`，`~/.dsh/skills/` 仅 `j-space`。
- anchored-standard、jspace-standard 实验 preset 已备份移除；调用方法实验为历史存档。

## 复算

- modeltest 评分：`python3 evaluator/run_full_eval.py <workspace>/project2_task --model <m>
  --channel <ch> --harness dsh-0.1.0-rc.5 --require-meta --include-espidf-build
  --run-group-id <id> --run-index <n> --thinking-level max`（评分器绑定本地端口，需提权）。
- 轨迹统计口径：assistant/message 中 type=reasoning 的文本全文，统计 `let me` / `we need` /
  `we can`（不区分大小写），脚本见各报告「数据来源」章节。
