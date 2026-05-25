# SignalClaw Research Idea Report - SUMO-Only Revision

**Direction**: 基于 Pi-Light 做可落地 SignalClaw，输出两套边缘可执行 skills：信号周期规划、下一相位规划。
**Revised**: 2026-05-26
**Key correction**: `traffic_full_20260521_112701.sql` 只能用于构造 SUMO 场景，不能直接从中学习。学习与验证只能依赖 SUMO。
**Hard constraint**: 同一交通输入不能回放探索不同动作空间；每个 SUMO route realization / 仿真时间线只能服务一个策略执行轨迹。

## Revised Landscape Summary

上一版报告把真实 SQL 当成可监督学习日志，这是错误方向。现在真实 SQL 的作用必须降级为场景构造材料：用于路口、相位、检测器、需求分布、时段流量和 SUMO 映射校准，而不是训练标签、奖励标签或专家动作标签。

SignalClaw 仍可继承 Pi-Light 的关键优点：可解释 DSL、轻量程序策略、边缘可部署。但不能继承 Pi-Light 的训练方式。Pi-Light 的 MCTS 对候选程序反复跑 episode，本质上会让同一环境输入被多个候选策略比较；在落地约束下不可作为主线。

因此修订后的核心问题是：**只依赖 SUMO，但 SUMO 必须像真实部署一样单时间线运行；如何在不能对同一交通输入做反事实动作探索的前提下，生成两套可解释、可边缘部署的信号控制 skills？**

## Recommended Ideas

### Idea 1: No-Fork SUMO Program Synthesis - RECOMMENDED

- **Summary**: 用 SQL 构造 SUMO 场景分布；每个候选 DSL 程序只在独立 SUMO 场景实例上执行一次单时间线评估；通过跨场景统计稳健性筛选程序。
- **Hypothesis**: 即使不能在同一输入上比较多个动作，只要 SUMO 场景分布足够覆盖真实需求，程序搜索仍可通过独立场景样本获得稳定的策略排序。
- **CyclePlanSkill**: 输出周期级 `{phase_id: duration}`、cycle length、offset 修正。
- **NextPhaseSkill**: 输出合法下一相位、保持当前相位或延长绿灯；必须遵守 min/max green、黄灯、相位序列与安全约束。
- **Minimum experiment**:
  1. 从 SQL 抽取路口/相位/流量分布，只生成 SUMO scenario，不抽取训练标签。
  2. 构建 scenario sampler，每个 sample 生成唯一 route realization。
  3. 为每个候选程序分配不重复 scenario，执行单时间线 SUMO 回放。
  4. 以跨 scenario 的 travel time、queue、throughput、违例率做稳健排序。
- **Expected outcome**: 相比 fixed-time、max-pressure、SOTL 等 SUMO 内基线，找到更短且可解释的 dual-skill 程序；失败时能明确说明 no-fork 评估噪声是否太大。
- **Novelty**: 8/10。Pi-Light/GPLight/SymLight 多依赖可重复仿真搜索；这里的差异是 no-fork、SUMO-only、部署式评估协议。
- **Feasibility**: 中。核心难点是场景采样、公平统计和禁止同输入复用。
- **Risk**: MEDIUM-HIGH。
- **Pilot result**: SKIPPED。需要先实现 no-fork scenario registry 和 SUMO replay harness。

### Idea 2: Scenario-Conditioned Dual Skill Library - RECOMMENDED BACKUP

- **Summary**: 不训练单一全局策略，而是在 SUMO 场景分布上生成一个小型可解释 skill library；边缘端根据当前场景类别选择一个程序。
- **Hypothesis**: 在 no-fork 约束下，学习一个全局最优程序很难；但按 demand regime、路口拓扑、相位数和邻接 one-hot 切分后，小型程序库更稳定。
- **Minimum experiment**: 用 SUMO scenario metadata 聚类，不用 SQL 标签；每个簇独立搜索程序，在未见过的独立 SUMO scenarios 上评估泛化。
- **Expected outcome**: 程序库比单一程序在异质路口上更稳健，同时仍可边缘部署。
- **Novelty**: 7/10。
- **Feasibility**: 中。
- **Risk**: MEDIUM。
- **Pilot result**: SKIPPED。

### Idea 3: SUMO-Only Pi-Light Reparameterization

- **Summary**: 保留 Pi-Light lane-link priority 结构，但废弃 MCTS 同场景比较；只在独立 SUMO scenarios 上做参数/结构筛选。
- **Hypothesis**: Pi-Light 的 pressure-like DSL 结构足够强，真正需要改的是评估协议而不是控制表达式。
- **Minimum experiment**: 从 Pi-Light DSL 生成短程序候选；每个候选只拿到不同 SUMO scenario；按跨 scenario 置信区间淘汰明显差的候选。
- **Expected outcome**: 得到最小改造版本，便于快速落地。
- **Novelty**: 6/10。更像工程改造，作为备份合适。
- **Feasibility**: 高。
- **Risk**: MEDIUM。
- **Pilot result**: SKIPPED。

### Idea 4: Neighbor One-Hot Coordination Under SUMO-Only Evaluation

- **Summary**: 邻接路口 one-hot 不从 SQL 学习，而从 SUMO 网络拓扑、TLS 控制器和实时相位状态构造，作为 DSL 输入。
- **Hypothesis**: one-hot 邻接状态能让局部程序捕捉绿波、队列外溢和相邻相位冲突，而不需要 MARL 通信网络。
- **Minimum experiment**: 在 SUMO no-fork scenario set 上比较有/无 neighbor one-hot 的 dual skill；禁止在同一 route realization 上比较两者，使用独立 scenario blocks。
- **Expected outcome**: 若邻接 one-hot 有效，应在走廊 throughput、queue spillback 指标上改善。
- **Novelty**: 7/10。
- **Feasibility**: 中。
- **Risk**: MEDIUM。
- **Pilot result**: SKIPPED。

### Idea 5: No-Fork Evaluation Protocol as the Main Contribution

- **Summary**: 把“不能同输入多动作回放”的落地约束形式化成 SUMO-TSC 评估协议，并用它重新评估 Pi-Light-like、fixed-time、max-pressure、SOTL 和 SignalClaw。
- **Hypothesis**: 许多 TSC 方法在可重复仿真反事实评估下表现好，但在 no-fork 部署式评估下排序会变；该协议本身有论文价值。
- **Minimum experiment**: 建立 scenario registry、single-use route realization、policy assignment log、state/action hash audit；报告不同评估协议下的排名变化。
- **Expected outcome**: 即使 SignalClaw 方法增益有限，评估协议也能形成清晰贡献。
- **Novelty**: 8/10。
- **Feasibility**: 高。
- **Risk**: LOW-MEDIUM。
- **Pilot result**: SKIPPED。

## Eliminated or Revised Ideas

| Previous idea | Status | Reason |
|---|---|---|
| Counterfactual-Free Dual-Skill DSL from real logs | REVISED | 不能从 SQL 直接学习，改为 SUMO-only no-fork program synthesis。 |
| Log-Constrained Pi-Light Distillation | ELIMINATED | “log-constrained” 依赖真实 SQL 动作/时长标签，不允许。 |
| Offline imitation from SQL | ELIMINATED | 直接违反“SQL 不能学习”。 |
| Offline Q-learning / CQL | ELIMINATED | 依赖未执行动作价值估计，也不符合 no-fork。 |
| CoLight/MARL | ELIMINATED | 常规动作探索，不符合落地约束。 |
| Pi-Light MCTS on same SUMO scenario | ELIMINATED | 反复 episode 比较候选程序，违反同输入不可多动作探索。 |

## Required Infrastructure Before Any Pilot

1. **Scenario constructor**: SQL -> SUMO only，输出 routes、detectors、TLS mapping、time windows；不输出训练标签。
2. **Scenario registry**: 每个 route realization 有唯一 ID，记录是否已被某个 policy 使用。
3. **No-fork replay harness**: 一个 scenario realization 只能执行一个 policy；禁止同一输入复用。
4. **Policy assignment log**: 记录 `scenario_id, policy_id, seed, start_time, action_log_hash, metrics`。
5. **Audit checks**: 自动检测相同 route 文件、相同 seed、相同 state hash 是否被多个候选策略复用。

## Revised Pilot Plan

| Pilot | Estimate | Success metric |
|---|---:|---|
| P1 SQL-to-SUMO scenario constructor audit | 2-4h | 能生成不含训练标签的 SUMO scenario，且相位/TLS 映射可解释 |
| P2 no-fork scenario registry | 1-2h | 同一 route realization 不能被两个 policy 使用 |
| P3 fixed-time / max-pressure / SOTL no-fork baseline | 2-4h | 能在独立 scenario blocks 上产生可比指标 |
| P4 tiny DSL candidate search | 2-6h | 候选程序全部使用独立 scenarios，无输入复用 |

## Current Recommendation

下一阶段不要做 novelty-check 之前的“真实日志学习”验证；应先把 proposal 改成 SUMO-only 版本，并把第一批实验定义为基础设施 pilot：

1. SQL-to-SUMO scene construction only。
2. SUMO no-fork replay harness。
3. Baseline controllers under no-fork protocol。
4. Minimal Pi-Light-like DSL under independent-scenario evaluation。

## Sources

- Pi-Light AAAI 2024: https://ojs.aaai.org/index.php/AAAI/article/view/30103
- Traffic Signal Control via RL review, Infrastructures 2025: https://www.mdpi.com/2412-3811/10/5/114
- GPLight arXiv 2024: https://arxiv.org/abs/2403.17328
- GPLight+ arXiv 2025: https://arxiv.org/abs/2508.16090
- SymLight arXiv 2025: https://arxiv.org/abs/2511.05790
- Local constraint update: `idea-stage/CONSTRAINT_UPDATE_20260526_000957.md`
