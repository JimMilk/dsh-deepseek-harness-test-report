# J-Space × DeepSeek V4 测试总汇总（2026-08-17）

> 汇总两份专项报告：[modeltest 对照测试](REPORT-jspace-modeltest-2026-08-17.md)（9 轮官方评分）与 [V4-Flash 复现实验](REPORT-jspace-v4flash-repro-2026-08-17.md)（3 任务开关实验）。
> 数据摘要：[reports-data/jspace-modeltest-results-summary.json](../reports-data/jspace-modeltest-results-summary.json)。
> 评测基准：modeltest Project2（V4.1b frozen）；插件：J-Space-Cognition-Suite-V3.6；Harness：dsh 0.1.0-rc.5。

## 1. 结论（30 秒版）

1. **J-Space 对 v4-Flash 有方向性显著提升**：modeltest Ability 均值 93.12 vs 91.38（+1.75），配对 t=2.65（n=4，单尾 p≈0.04、双尾 p≈0.08）；关键项 **F3-05（显式 session 授权）J-Space 4/4 通过、Control 3/4**；四轮无一轮低于对照，波动更小。
2. **无负面信号**：J-Space 的失败项（F12-04/F1-01/F8/F9 环境）与 Control 重叠，未发现系统性劣势。
3. **v4-Pro 单轮未复现**：v4-Pro + J-Space 单轮 84.0（F3-05 失败），n=1 不作结论；若 Pro 为日常主力需补跑多轮。
4. **已按结论纯净安装**：`jspace-minimal`（Minimal 双工具 + 通用 J-Space 协议）为 dsh 默认 preset；`~/.dsh/skills/` 仅保留 `j-space`；实验 preset（anchored-standard、jspace-standard）已备份移除。

## 2. modeltest 官方评分（9 轮）

| 轮 | 模型 | Ability | 关键失败项 |
|---|---:|---:|---|
| J-Space r1 | v4-Flash | 92.0 | F12-04 |
| J-Space r2 | v4-Flash | 93.5 | F12-04、F1-01 |
| J-Space r3 | v4-Flash | 92.5 | F12-04、F1-01 |
| J-Space r4 | v4-Flash | **94.5** | F12-04 |
| Control r1 | v4-Flash | 89.5 | **F3-05**、F12-04、F1-01 |
| Control r2 | v4-Flash | 93.5 | F5-06、F12-04 |
| Control r3 | v4-Flash | 89.5 | F12-04、F6-01、F1-01 |
| Control r4 | v4-Flash | 93.0 | F12-04 |
| J-Space | v4-Pro | 84.0 | **F3-05**、F12-04、F6-03 |

- v4-Flash：J-Space 均值 93.12（sd 1.11）vs Control 91.38（sd 2.17）；配对差值 [+2.5, 0, +3.0, +1.5]。
- **F3-05**：J-Space 4/4 vs Control 3/4（历史 11 轮 V4-Pro Control 全挂）；**F12-04**：两条件 8 轮全挂（已知模型稳定缺陷）。
- F9（ESP-IDF 真机构建）受本机无工具链限制，两条件同为环境扣分。

## 3. V4-Flash 复现实验（3 任务开关）

任务：NL2Repo 型（CLI 项目构建）、Toolathlon 型（数据管道独立验证）、AutomationBench 型（多阶段状态保持），每任务 Control/J-Space 各 1 次，外部脚本独立验收：

- 完成率：两组均 3/3；J-Space 组合计 token −24%（35,647 → 27,181），耗时持平（143s vs 142s），工具调用更多（33 vs 26，验证更频繁）。
- 与报告效率结论（得分/Token 2.21×）方向一致。

## 4. 方法约束（防污染）

- 任务文本逐字采用 modeltest 官方 CANDIDATE_PROMPT（仅附加候选工作区路径），未加入任何针对测试项目（F3/session/PR 模板等）的显式约束。
- J-Space 变量仅为通用协议 persona 注入（`--patch`）；每轮独立候选工作区、从零起点。
- 单次/多轮均为单运行记录，无置信区间；v4-Pro 仅 1 轮，不作结论。

## 5. 环境与配置现状

- dsh 0.1.0-rc.5；默认模型 `deepseek-v4-flash` / reasoningEffort max；默认 preset `jspace-minimal`（纯净安装）。
- headless 通道不装配 preset roster，测试以 `--patch` 注入协议、`danger-full-access` 运行（候选工作区隔离）。
- 测试痕迹已清理：候选工作区（3.1G）删除，评分产物保留于 `modeltest-run/evaluator/results/20260817_*`。

## 6. 复算

- 评分：`python3 evaluator/run_full_eval.py <workspace>/project2_task --model <m> --channel <ch> --harness dsh-0.1.0-rc.5 --require-meta --include-espidf-build --run-group-id <id> --run-index <n> --thinking-level max`（需提权：评分器绑定本地端口）。
- 摘要 JSON：`reports-data/jspace-modeltest-results-summary.json`。
