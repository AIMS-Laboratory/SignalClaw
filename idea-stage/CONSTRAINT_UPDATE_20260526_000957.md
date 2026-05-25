# Constraint Update - SUMO-Only Learning

**Timestamp**: 20260526_000957

## User Correction

`traffic_full_20260521_112701.sql` 最多只能用于构造 SUMO 场景，不能直接从中进行学习。SignalClaw 的学习与验证只能依赖 SUMO。

## Implication

此前 Phase 1/2 中“从真实业务日志监督学习/归纳 skill”的表述需要废弃。真实 SQL 中的 `run_timing_adjustment`、`phase_green_event`、`prediction_record`、`traffic_flow_record` 等表只能用于：

- 构造或校准 SUMO 场景的路口、相位、需求、检测器和时段分布。
- 给 SUMO 路网/route/detector 生成提供约束。
- 作为场景真实性来源，而不是训练标签、奖励标签或专家动作标签。

## Revised Boundary

SignalClaw 现在必须满足三个边界：

1. **SUMO-only**: skill 学习、筛选、验证都只能来自 SUMO 运行结果。
2. **No same-input action replay**: 同一个交通输入/route realization/仿真时间线不能被分叉给多个动作或多个候选策略。
3. **Deployable dual skills**: 输出仍然是两套边缘可执行代码：
   - `CyclePlanSkill`
   - `NextPhaseSkill`

## Revised Research Direction

主线从“真实日志程序归纳”改为：

**No-Fork SUMO Program Synthesis for Edge Signal Skills**

用 SQL 仅构造场景分布；每个候选程序只在独立 SUMO 场景实例上做单时间线部署式评估。不能用 common random numbers 比较候选策略，也不能对同一仿真输入回放多个动作。策略搜索必须接受这种高噪声评估设定，并用跨场景统计稳健性而非同场景反事实收益来筛选程序。
