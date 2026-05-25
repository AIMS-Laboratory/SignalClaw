# SignalClaw Idea Discovery - Phase 1 Literature Landscape

**Date**: 2026-05-25
**Direction**: 基于 Pi-Light 做可落地的 SignalClaw，输出两套边缘可执行 skills：信号周期规划、下一相位规划。
**Hard constraint**: SUMO 仿真必须模拟真实落地闭环；同一交通输入不能回放探索不同动作空间，不能按常规 MTSC/RL 做反事实动作试验。

## 结论摘要

**2026-05-26 correction**: `traffic_full_20260521_112701.sql` 最多只能用于构造 SUMO 场景，不能直接作为学习数据。下文中凡涉及“从真实业务日志学习/归纳”的表述应理解为已废弃；当前有效主线是 `idea-stage/IDEA_REPORT.md` 中的 SUMO-only no-fork 版本。

SignalClaw 不应直接沿用 Pi-Light 的 MCTS 强化学习搜索。Pi-Light 的优势在于可解释 DSL、轻量程序策略和边缘设备可部署性；但其训练方式依赖对候选程序反复跑 episode 并用全局 travel time 评价，这违反当前落地约束。修订后的研究主线是：用 SQL 只构造 SUMO 场景分布，在 SUMO 中执行 no-fork 单时间线评估，并在独立场景样本上筛选两级边缘程序策略。

最优先方向建议修订为 **No-Fork SUMO Program Synthesis for Edge Signal Skills**：不学习 Q 值，不从 SQL 学习动作标签，不对同一 SUMO 输入分叉探索；只从 SUMO 单时间线运行结果中筛选两类 DSL 程序：

1. **CyclePlanSkill**：按周期生成各相位 duration / offset / 绿波协调参数。
2. **NextPhaseSkill**：在当前相位即将结束时，根据当前路口 + one-hot 邻接路口状态，选择合法下一相位或合法延长/切换动作。

## 项目内证据

### 真实 SQL 数据

`traffic_full_20260521_112701.sql` 是 5GB 真实业务库导出，核心表结构显示系统已经有实际落地闭环：

- `crossing`: 路口、真实业务 ID、SUMO ID、AI 开关、下发模式、固定周期字段。
- `crossing_phase`: 相位 ID、真实相位值、SUMO 相位值、最小/最大绿灯、车道数、微调权重。
- `crossing_base_rule`: 基础相位配时 JSON。
- `cycle_time`: 周期开始时间、运行方式，包含真控/仿真控/SUMO 模式。
- `batch_plan`: 批次计划，记录计划执行时间、相位、持续时长、执行状态。
- `run_timing_adjustment`: base_duration、ai_duration、micro_duration、actual_duration。该表只能帮助理解真实系统的场景构造范围，不能作为监督信号。
- `phase_green_event`: 真实绿灯开始事件。
- `traffic_flow_record`: 按 crossing/batch/camera_phase 记录 area1_car、area2_car。
- `prediction_record`: 按方向/周期记录 wait_start_peak、green_start_peak、red_max_peak、predicted_value。
- `wave/wave_node`: 绿波/红波和 offset 结构，适合作为邻接协同约束。

这些表现在只允许支持 **SUMO 场景构造 + 场景真实性约束**，不允许作为离线学习标签，也不支持同一状态多动作试验。

### SUMO 场景

`sumo_scenarios/chengdu` 当前只有 1 小时：

- `chengdu.sumocfg`: begin=0, end=3599.75。
- `chengdu.net.xml`: 46 个 traffic light controller。
- `chengdu.rou.xml`: 9898 个车辆。

该场景可用于搭建 TraCI 闭环和指标管线，但目前与真实 SQL 数据不匹配，不能作为“训练环境”。它应该先做两件事：

1. 按真实路口/相位/周期/流量记录校准路口映射与 detector 输出。
2. 按真实时间顺序回放已观测流量与策略输出，只评价策略，不做分叉探索。

### Pi-Light 代码

Pi-Light 的可复用部分：

- `agent/pi_light/program.py`: DSL 程序结构，包含条件、指令、if/else、复杂度约束。
- `agent/pi_light/pi_light.py`: 对 lane link 计算 value，再聚合到 phase value，输出 argmax 相位。
- 轻量程序形式适合转成边缘设备可执行代码。

Pi-Light 的不可直接复用部分：

- `agent/pi_light/MCTS.py`: 每个候选程序通过 `run_an_episode` 在环境中反复评估 travel time，并用 MCTS/贝叶斯优化搜索参数。
- 这本质上是对候选动作/程序做仿真试验，不满足“同一交通输入不能回放探索不同动作空间”。

## SUMO/TraCI 落地约束

Obsidian SUMO 材料显示，TraCI 支持读取和控制信号灯：

- 可读取 traffic light id、当前 state、当前 phase、phase duration、controlled lanes、controlled links、next switch 等。
- 可用 `setPhase`、`setPhaseDuration`、`setProgramLogic`、`setProgram` 修改信号控制。
- 对落地式控制，推荐用长绿相位 program + TraCI 决定当前绿灯持续时间，SUMO 自动处理黄灯/过渡；若直接 `setRedYellowGreenState`，脚本必须自己处理全部相位和过渡。

SignalClaw 在 SUMO 中应实现为：

- 单一时间线。
- 每个 tick 读取当前观测。
- 策略只输出一次动作。
- SUMO 执行动作并推进。
- 日志记录状态、动作、结果。
- 禁止对同一状态 fork 多个动作 episode。

## 文献景观

### 相关但不适合作为主线

1. 常规 RL/MARL-TSC 仍以在线/仿真交互为主。2025 年综述指出 RL-TSC 主流覆盖单/多智能体、奖励设计、仿真平台和协调机制，但实际部署仍面临安全、泛化、可解释和工程约束问题。
2. Pi-Light 解决可解释和边缘部署，但仍是 programmatic RL，用 MCTS 搜索程序，训练评价依赖反复仿真。
3. 安全 RL、约束 RL、action masking 等文献可提供约束表达，但仍大多需要环境交互或可重复仿真评估。
4. Max-Pressure + RL 能提供交通工程先验，但 RL 部分同样容易滑回动作探索。

### 更接近本项目的空白

1. **Interpretable program synthesis for TSC without action exploration**：Pi-Light/GPLight/SymLight 都强调可解释策略，但大多仍靠仿真搜索/进化。真实落地日志下的 counterfactual-free 程序归纳较少。
2. **SUMO-only no-fork program synthesis**：用 SQL 构造 SUMO 场景分布，但策略学习和评价只来自 SUMO 单时间线结果，避免真实日志监督学习。
3. **Two-skill hierarchical signal control**：把周期规划和下一相位规划拆成两个边缘 skill，既符合业务表结构，也符合信号控制工程流程。
4. **Neighbor one-hot coordination without communication-heavy MARL**：邻接路口 one-hot 状态可进入 DSL 条件和特征，但不引入神经消息传递或多智能体探索。

## 推荐问题定义

**Problem Anchor**: 在不能对同一交通状态进行动作空间探索的真实信号控制场景中，如何从历史业务日志与一次性 SUMO 数字孪生回放中学习两套可解释、可审计、可边缘部署的信号控制 skills，并通过邻接 one-hot 状态自发形成局部协同？

## Phase 2 备选 idea 方向

### Idea A - No-Fork SUMO Program Synthesis

从 SUMO 独立场景运行结果筛选两级 DSL：

- CyclePlanSkill 从 SUMO 场景状态输出 `{phase_id: duration}` / offset。
- NextPhaseSkill 从 SUMO 当前状态输出合法下一相位/延长动作。
- 每个候选策略只使用独立 route realization；禁止同一 SUMO 输入复用。
- 目标是跨场景 travel time、queue、throughput、违例率的稳健统计，而不是真实日志模仿。

### Idea B - SUMO-Only Pi-Light Reparameterization

保留 Pi-Light lane-link priority 聚合形式，但不做 MCTS episode 搜索：

- 用 SQL 只生成 SUMO 场景；候选 Pi-Light 风格程序只能在独立 SUMO scenarios 上评估。
- 用跨场景置信区间淘汰明显差的程序。
- 输出边缘可执行代码。

### Idea C - Green-Wave Aware Dual Skill

利用 `wave/wave_node` 的 offset 结构，将邻接 one-hot 和 offset 偏差作为 DSL 输入：

- 周期 skill 负责绿波协调与 offset 调整。
- 下一相位 skill 只做局部合法修正。
- 适合强调协同配置，但依赖真实路口拓扑和 SUMO 映射质量。

## Phase 1 Checkpoint

建议进入 Phase 2 时只生成围绕 Idea A 的 6-8 个变体，不再发散到常规 RL/MARL。若你确认，我下一步会进入 idea generation/filtering，并重点筛掉任何需要“同一输入多动作回放探索”的方案。

## Sources

- Pi-Light AAAI 2024: https://ojs.aaai.org/index.php/AAAI/article/view/30103
- Traffic Signal Control via RL review, Infrastructures 2025: https://www.mdpi.com/2412-3811/10/5/114
- GPLight arXiv 2024: https://arxiv.org/abs/2403.17328
- GPLight+ arXiv 2025: https://arxiv.org/abs/2508.16090
- SymLight arXiv 2025: https://arxiv.org/abs/2511.05790
- Local SUMO notes: `sumo_docs/TraCI/Traffic_Lights_Value_Retrieval.md`, `sumo_docs/TraCI/Change_Traffic_Lights_State.md`, `sumo_docs/Simulation/Traffic_Lights.md`, `sumo_docs/Tutorials/TraCI4Traffic_Lights.md`
- Local project files: `traffic_full_20260521_112701.sql`, `sumo_scenarios/chengdu`, `pi_light_code`, `pi_light_original_paper/30103-Article_Text-34157-1-2-20240324_3_.md`
