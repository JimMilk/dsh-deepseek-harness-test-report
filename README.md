# DeepSeek V4 Pro × dsh-anchored-standard 多环境实测报告

## 项目说明

- **插件项目**：[`xiaobright/dsh-anchored-standard`](https://github.com/xiaobright/dsh-anchored-standard)
  （DeepSeek Harness 两阶段锚定 preset）
- **测试项目**：[`xiaobright/modeltest`](https://github.com/xiaobright/modeltest)
  （Project2 工程维护评测，V4.1b frozen）
- **执行主体**：本测试由搭载 **DeepSeek-V4-Flash** 的 Codex 工程 Agent 编写并驱动；
  被测对象为搭载 **DeepSeek-V4-Pro（reasoningEffort=max）** 的 DeepSeek Harness（0.1.0-rc.5）会话。
  即：**用搭载 v4-flash 的 Codex，给搭载 v4-pro 的 DSH 做测试。**

## 内容

- 完整报告：[evaluations/REPORT-anchored-standard-tests-2026-08-16.md](evaluations/REPORT-anchored-standard-tests-2026-08-16.md)
- 评分汇总数据：[reports-data/all-results-summary.json](reports-data/all-results-summary.json)
- 轨迹统计数据：[reports-data/trajectory-summary.json](reports-data/trajectory-summary.json)

## 核心结论（摘要）

1. macOS / Windows 11 ARM64 VM / Linux（Lima Ubuntu）三环境、4 种 preset、11 轮完整
   Project2 评测（V4 Pro + max + modeltest V4.1b 官方评分），Ability 区间 **85–90**，
   未复现插件 README 的 98/99。
2. **F3-05（−5）与 F12-04（−1）全部 11 轮稳定失败**：模型一致地把无显式 session_id 的
   敏感目标请求回退到 current session 隐式授权，是模型层稳定缺陷，与 OS/preset/轨迹无关。
3. reasoning 轨迹（we/let me）与分数无相关；「首轮工具 schema 决定轨迹」在当前模型上不再稳定。
4. 环境（OS）不改变分数；F9（ESP-IDF 真实编译）是唯一环境硬上限（macOS 封顶 97，
   叠加模型缺陷后理论上限 91）。

详见完整报告（含全部数据来源与复算命令）。

---

# J-Space × DeepSeek V4 测试（2026-08-17）

## 项目说明

- **插件项目**：[`Tiger3807861189/J-Space-Cognition-Suite-V3.6`](https://github.com/Tiger3807861189/J-Space-Cognition-Suite-V3.6)
  （J-Space 推理时认知控制套件，已纯净安装为 dsh 默认 preset `jspace-minimal`）
- **测试项目**：[`xiaobright/modeltest`](https://github.com/xiaobright/modeltest)
  （Project2 工程维护评测，V4.1b frozen）
- **执行主体**：Codex 工程 Agent 编写并驱动；被测对象为 DeepSeek Harness（0.1.0-rc.5）会话
  （v4-Flash 8 轮 + v4-Pro 1 轮，均 reasoningEffort=max）。

## 内容

- 总汇总：[evaluations/SUMMARY-jspace-tests-2026-08-17.md](evaluations/SUMMARY-jspace-tests-2026-08-17.md)
- modeltest 对照报告：[evaluations/REPORT-jspace-modeltest-2026-08-17.md](evaluations/REPORT-jspace-modeltest-2026-08-17.md)
- V4-Flash 复现实验：[evaluations/REPORT-jspace-v4flash-repro-2026-08-17.md](evaluations/REPORT-jspace-v4flash-repro-2026-08-17.md)
- 9 轮评分摘要：[reports-data/jspace-modeltest-results-summary.json](reports-data/jspace-modeltest-results-summary.json)

## 核心结论（摘要）

1. **J-Space 对 v4-Flash 有方向性显著提升**：modeltest Ability 均值 93.12 vs 91.38（+1.75），
   配对 t=2.65（n=4，单尾 p≈0.04）；关键项 **F3-05（显式 session 授权）J-Space 4/4 通过、
   Control 3/4**；无一轮低于对照、波动更小。
2. **F12-04 两条件 8 轮全挂**（已知模型稳定缺陷）；F9（ESP-IDF 真机构建）受本机无工具链限制。
3. **v4-Pro + J-Space 单轮 84.0**（F3-05 未通过），n=1 不作结论。
4. 实验保持题面纯净：任务文本与官方 CANDIDATE_PROMPT 逐字一致，未加入针对测试项目的显式约束。

## 环境与配置

- dsh 0.1.0-rc.5；默认模型 `deepseek-v4-flash` / reasoningEffort max；默认 preset `jspace-minimal`。
- `~/.dsh/.agent-presets/` 仅 `jspace-minimal`；`~/.dsh/skills/` 仅 `j-space`。
