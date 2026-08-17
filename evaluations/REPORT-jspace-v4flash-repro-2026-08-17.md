# J-Space × DeepSeek V4-Flash 复现实验报告（2026-08-17）

> 复现对象：[DeepSeek-V4-J-Space-Capability-Realization-Report](https://github.com/Tiger3807861189/DeepSeek-V4-J-Space-Capability-Realization-Report)
> 被测环境：DeepSeek Harness 0.1.0-rc.5（本地源码目录）+ J-Space-Cognition-Suite-V3.6（已装于 `~/.dsh/skills/j-space`）

## 1. 复现目标与协议

报告 §4.1 评测协议：DeepSeek Harness 组合 + `reasoning_effort = max`，唯一实验变量为是否加载 J-Space（用户主动加载、不注入无关 Skill 目录），单次运行记录。

本地已按协议配置并验证：

| 条件 | 值 | 验证证据 |
|---|---|---|
| 模型 | `deepseek-v4-flash` | 6 个会话 request/header 均为 `deepseek-v4-flash` |
| 推理强度 | `reasoningEffort: max` | 6 个会话 request/header 均为 `max` |
| J-Space 加载 | 用户主动加载（persona 注入协议文本，非自动 Skill 目录） | 实验组会话含 `J-SPACE COGNITION PROTOCOL`；对照组不含 |
| 对照变量 | 仅 persona 差异 | 工具集、模型、任务、验收脚本完全相同 |

**配置偏差（如实记录）**：

1. 本机 headless profile 不装配 agent-presets roster（源码注释确认），因此无法直接复现报告的 Minimal 双工具组合；实验使用 headless 基础工具集（含 read/write/edit/bash 等），J-Space 通过 `--patch` overlay 注入 persona。两组条件完全一致，仍满足"同 harness、同任务、同工具、唯一变量"的开关实验要求。
2. dsh 0.1.0-rc.5 的 DeepSeek 适配器只提交 `temperature`（未配置时不上传）且无 `top_p` 字段；本机会话未配置 `temperature=1.0`、`top_p=0.95`。报告已知官方思考模式可能忽略这两个参数，此偏差不影响 J-Space 因果判断。

## 2. 方法

3 个可本地客观验收的代理任务（覆盖报告 benchmark 类型），每任务 2 组各运行 1 次：

| 任务 | 对应报告类型 | 验收标准（脚本独立核验） |
|---|---|---|
| t1：从零构建 linecount CLI 项目（src/tests/README） | NL2Repo 型 | unittest 全过；CLI 输出与 `wc` 一致 |
| t2：数据管道（生成→处理→独立验证） | Toolathlon 型 | `verify.py` 输出 PASS；指标与独立重算一致 |
| t3：多阶段状态保持（constants→series→summary→check） | AutomationBench 型 | `check.py` 输出 ALL-OK；数值独立核验 |

任务目录每次运行前清空，保证从零起点。

## 3. 结果

### 3.1 任务质量（完成率）

| 任务 | Control | J-Space |
|---|---|---|
| t1 | 通过（5 测试 OK，CLI=wc） | 通过（5 测试 OK，CLI=wc） |
| t2 | 通过（PASS，独立 MATCH） | 通过（PASS，独立 MATCH） |
| t3 | 通过（ALL-OK，独立 OK） | 通过（ALL-OK，独立 OK，含篡改负向验证） |
| 完成率 | 3/3 | 3/3 |

### 3.2 效率（session 日志实测）

| 运行 | 工具调用 | 输入 token | 输出 token | 合计 token | 耗时(s) |
|---|---:|---:|---:|---:|---:|
| C-t1 | 9 | 4,632 | 10,648 | 15,280 | 77 |
| C-t2 | 8 | 831 | 3,125 | 3,956 | 22 |
| C-t3 | 9 | 10,683 | 5,728 | 16,411 | 44 |
| **C 合计** | **26** | **16,146** | **19,501** | **35,647** | **143** |
| J-t1 | 13 | 1,383 | 7,619 | 9,002 | 55 |
| J-t2 | 8 | 591 | 7,351 | 7,942 | 58 |
| J-t3 | 12 | 6,759 | 3,478 | 10,237 | 29 |
| **J 合计** | **33** | **8,733** | **18,448** | **27,181** | **142** |

**相对对比（J/C）**：

- 合计 token：0.76×（−24%）
- 总耗时：0.99×（持平）
- 工具调用：33 vs 26（J-Space 组更频繁检查/验证）

### 3.3 轨迹探针

- `we` 词频：两组均几乎为 0（C 共 1，J 共 1），无显著差异。
- `let me`：两组均常见（推理文本高频出现，如 "let me think"），与报告"Let me 偶尔出现并不自动构成失败"一致。
- 完成质量两组相同，因此词频差异不构成质量信号。

## 4. 结论

1. **协议条件已按报告配置**：v4-flash + reasoningEffort max + 唯一变量 J-Space 加载，全部经 session 日志验证。
2. **J-Space 方向与报告一致**：相同完成率下，J-Space 组 token 消耗 −24%（报告效率结果：得分/Token 2.21× 提升，方向一致）；未发现 J-Space 降低完成率。
3. **报告分数未复现**：报告 9 项 benchmark（HLE/NL2Repo/CyberGym 等）依赖官方数据集与实现，本地无对应评测环境，本报告只复现协议条件与因果方向，不声称复现具体分数。
4. **单次运行无置信区间**：与报告声明一致，结果按单次记录。

## 5. 变更清单

本次实验对本地 dsh 的持久变更：

- 新增 `~/.dsh/skills/j-space`（J-Space V3.6 skill，上轮安装，本轮复用）。
- 新增 `~/.dsh/.agent-presets/jspace-minimal/`（Minimal 双工具 + J-Space 协议 preset；headless 未使用，供 web/支持 preset 的入口使用）。
- `~/.dsh/.agent-presets/jspace-standard/agent.cordis.yml` 路径修正（`/private/tmp` → `~/.dsh/skills/j-space`，上轮完成）。
- 临时变更已恢复：`settings.yaml`（模型 v4-flash ↔ v4-pro、preset minimal ↔ jspace-minimal）已恢复原状。
- 测试临时文件：`/tmp/jspace-repro/`（任务目录、prompt、patch、分析脚本、验证 session 日志）。
