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
