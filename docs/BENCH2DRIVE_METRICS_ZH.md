# Bench2Drive 测试指标与公式

本文档整理 Bench2Drive 闭环 benchmark 的主要指标、公式和代码来源，并补充 Bench2DriveZoo 开环评测指标的位置。闭环指标以本机检查到的 `G:\Bench2Drive` 代码为依据。

## 1. 闭环 route 级指标

代码来源：

- `leaderboard/leaderboard/utils/statistics_manager.py`
- `scenario_runner/srunner/scenariomanager/scenarioatomics/atomic_criteria.py`
- `scenario_runner/srunner/scenariomanager/traffic_events.py`

每条 route 最终包含三个核心分数：

```text
score_route
score_penalty
score_composed
```

### 1.1 Route Completion

字段：`scores.score_route`

事件：`TrafficEventType.ROUTE_COMPLETION`

代码：`RouteCompletionTest`

含义：ego vehicle 沿 route waypoint 前进的完成百分比，范围约为 `[0, 100]`。

计算方式：

1. 对 route 中所有 waypoint 累计里程。
2. 每帧找 ego 是否已经通过后续 waypoint。
3. 当前 waypoint 对应累计里程占总里程的比例就是完成率。

公式：

```text
score_route = 100 * accumulated_distance_to_current_route_index / total_route_distance
```

完成 route 的判定：

```text
score_route > 99
并且 ego 到终点距离 < 10 m
```

最终 `StatisticsManager` 会把 `ROUTE_COMPLETION` 事件中的 `route_completed` 写为 `score_route`。

### 1.2 Infraction Penalty

字段：`scores.score_penalty`

代码：`StatisticsManager.compute_route_statistics()`

初始值：

```text
score_penalty = 1.0
```

每发生一个固定惩罚事件，乘上对应系数：

| 事件 | JSON 字段 | 乘法系数 |
|---|---|---:|
| 撞行人 `COLLISION_PEDESTRIAN` | `collisions_pedestrian` | 0.5 |
| 撞车 `COLLISION_VEHICLE` | `collisions_vehicle` | 0.6 |
| 撞静态物/道路设施 `COLLISION_STATIC` | `collisions_layout` | 0.65 |
| 闯红灯 `TRAFFIC_LIGHT_INFRACTION` | `red_light` | 0.7 |
| Stop 未停车 `STOP_INFRACTION` | `stop_infraction` | 0.8 |
| 场景超时 `SCENARIO_TIMEOUT` | `scenario_timeouts` | 0.7 |
| 未给急救车让行 `YIELD_TO_EMERGENCY_VEHICLE` | `yield_emergency_vehicle_infractions` | 0.7 |

固定惩罚公式：

```text
score_penalty = Π penalty_value(event_i)
```

如果同一类事件发生多次，会重复乘。例如两次闯红灯：

```text
score_penalty = 1.0 * 0.7 * 0.7 = 0.49
```

### 1.3 百分比型惩罚

代码：`PENALTY_PERC_DICT`

当前代码中包括：

| 事件 | JSON 字段 | 配置 |
|---|---|---|
| `OUTSIDE_ROUTE_LANES_INFRACTION` | `outside_route_lanes` | `[0, 'increases']` |
| `MIN_SPEED_INFRACTION` | `min_speed_infractions` | `[0.7, 'unused']` |

百分比型通用公式在 `StatisticsManager.set_score_penalty()` 中：

如果类型是 `increases`：

```text
score_penalty *= 1 - (1 - penalty_value) * event_percentage / 100
```

`outside_route_lanes` 的 `penalty_value = 0`，所以：

```text
score_penalty *= 1 - outside_route_percentage / 100
```

如果类型是 `decreases`：

```text
score_penalty *= 1 - (1 - penalty_value) * (1 - event_percentage / 100)
```

但是当前代码中 `MIN_SPEED_INFRACTION` 的类型是 `unused`，也就是最低速度事件会被记录，但不再扣 driving score。这也符合 Bench2Drive README 中“移除 min speed penalty”的说明。

### 1.4 Driving Score

字段：`scores.score_composed`

代码：

```text
route_record.scores['score_composed'] = max(score_route * score_penalty, 0.0)
```

公式：

```text
DrivingScore_route = RouteCompletion_route * InfractionPenalty_route
```

注意：

- `score_route` 是百分制，例如 80。
- `score_penalty` 是乘法惩罚系数，例如 0.7。
- 所以 `score_composed` 仍是百分制，例如 56。

### 1.5 Route Status

字段：`status`

代码：`StatisticsManager.compute_route_statistics()`

逻辑：

```text
如果 target_reached:
    如果 num_infractions > 0: status = 'Completed'
    否则: status = 'Perfect'
否则:
    status = 'Failed'
    如果有 failure_message，则附加失败原因
```

常见失败原因：

- `Agent deviated from the route`
- `Agent got blocked`
- `Agent timed out`
- `Simulation crashed`
- `Agent crashed`

## 2. 闭环事件指标

### 2.1 Collision

代码：`CollisionTest`

CARLA 来源：`sensor.other.collision`

去重规则：

- `COLLISION_RADIUS = 5 m`：距离太近的连续碰撞视作同一次。
- `MAX_ID_TIME = 5 s`：同一个 actor 5 秒内重复碰撞忽略。
- `EPSILON = 0.1`：ego 速度太低时不认为是 ego 责任碰撞。

分类：

```text
static/traffic 且不是 sidewalk -> COLLISION_STATIC
vehicle -> COLLISION_VEHICLE
walker -> COLLISION_PEDESTRIAN
```

### 2.2 Red Light Infraction

代码：`RunningRedLightTest`

核心判定：

- 只检查距离 ego 15 m 内的红灯。
- 找到该红灯影响的 lane waypoint。
- 用车辆尾部线段和 stop line 线段是否相交判断是否越线。
- 如果红灯状态为 Red 且越线，则记录 `TRAFFIC_LIGHT_INFRACTION`。

### 2.3 Stop Sign Infraction

代码：`RunningStopTest`

关键阈值：

```text
PROXIMITY_THRESHOLD = 4.0 m
SPEED_THRESHOLD = 0.1 m/s
WAYPOINT_STEP = 0.5 m
```

核心判定：

- 车辆进入 Stop 标志影响区域后，必须出现速度 `< 0.1 m/s` 的状态。
- 如果离开影响区域前没有停过，则记录 `STOP_INFRACTION`。

### 2.4 Outside Route Lanes

代码：`OutsideRouteLanesTest`

关键阈值：

```text
ALLOWED_OUT_DISTANCE = 0.5 m
MAX_VEHICLE_ANGLE = 120 deg
MAX_WAYPOINT_ANGLE = 150 deg
WINDOWS_SIZE = 3
```

含义：

- 统计 ego 已行驶 route 中，有多少距离处在路线车道外或错误方向车道。

公式：

```text
outside_route_percentage = wrong_distance / total_distance * 100
```

惩罚：

```text
score_penalty *= 1 - outside_route_percentage / 100
```

### 2.5 Route Deviation

代码：`InRouteTest`

关键阈值：

```text
offroad_max = 30 m
MAX_ROUTE_PERCENTAGE = 30 %
WINDOWS_SIZE = 5
```

核心判定：

- 如果 ego 到最近 route waypoint 的距离超过 `offroad_max`，或累计偏离距离占 route 总长超过 30%，触发 `ROUTE_DEVIATION`。
- 该 criterion 设置了 `terminate_on_failure=True`，触发后会提前结束该 route。

### 2.6 Vehicle Blocked

代码：`ActorBlockedTest`

leaderboard 版本阈值：

```text
min_speed = 0.1 m/s
max_time = 60.0 s
```

含义：

- 如果 ego 长时间速度低于 0.1 m/s，则认为 agent blocked。
- 该 criterion 会提前终止 route。

### 2.7 Minimum Speed

代码：`MinimumSpeedRouteTest`

leaderboard 版本：

```text
checkpoints = 20
RATIO = 1
```

每个 checkpoint 统计：

```text
checkpoint_value = ego_average_speed / background_traffic_average_speed * 100
```

记录格式：

```text
"Average speed is {checkpoint_value}% of the surrounding traffic's one"
```

注意：

- 当前 `StatisticsManager` 中 `MIN_SPEED_INFRACTION` 是 `unused`。
- 它会进入 `min_speed_infractions`，但不扣 `score_composed`。
- 后处理的 Driving Efficiency 会读取这个百分比。

### 2.8 Scenario Timeout

代码：`ScenarioTimeoutTest`

含义：

- 场景级 criterion。
- 如果对应 scenario 的 blackboard 标记超时，记录 `SCENARIO_TIMEOUT`。
- 对 `score_penalty` 乘以 0.7。

### 2.9 Yield To Emergency Vehicle

代码：`YieldToEmergencyVehicleTest`

含义：

- 用于急救车相关场景。
- 如果急救车最终没有处于 ego 前方，记录 `YIELD_TO_EMERGENCY_VEHICLE`。
- 对 `score_penalty` 乘以 0.7。

## 3. 全局闭环指标

代码来源：

- `leaderboard/leaderboard/utils/statistics_manager.py`
- `tools/merge_route_json.py`

### 3.1 Leaderboard Global Mean

`StatisticsManager.compute_global_statistics()` 会计算：

```text
mean_score_composed = Σ score_composed_i / total_routes
mean_score_route = Σ score_route_i / total_routes
mean_score_penalty = Σ score_penalty_i / total_routes
```

还会计算各分数标准差：

```text
std = sqrt(Σ(score_i - mean)^2 / (total_routes - 1))
```

### 3.2 Infractions Per Kilometer

代码中先计算实际完成里程：

```text
km_driven = Σ(route_length_i / 1000 * score_route_i / 100)
km_driven = max(km_driven, 0.001)
```

除 `outside_route_lanes` 外，大多数 infraction 以每公里次数统计：

```text
infraction_per_km = total_infraction_count / km_driven
```

`outside_route_lanes` 特殊处理为累计公里数，不再除以 `km_driven`。

### 3.3 Bench2Drive Driving Score

文件：`tools/merge_route_json.py`

Bench2Drive 后处理使用固定 220 条 route 作为分母：

```text
DrivingScore_B2D = Σ score_composed_i / 220
```

如果实际 JSON 不满 220 条，脚本会提示：

```text
All metrics (Driving Score, Success Rate, Ability) are inaccurate
```

### 3.4 Success Rate

文件：`tools/merge_route_json.py`

成功 route 条件：

```text
status 是 Completed 或 Perfect
并且 infractions 中除 min_speed_infractions 外全部为空
```

公式：

```text
SuccessRate = success_num / 220
```

注意：

- `min_speed_infractions` 不影响 success。
- 如果只跑 dev10 或部分 route，这个脚本仍除以 220，因此不能直接代表完整 benchmark 成绩。

## 4. Ability 指标

文件：`tools/ability_benchmark.py`

Ability 分为 5 类：

- `Overtaking`
- `Merging`
- `Emergency_Brake`
- `Give_Way`
- `Traffic_Signs`

普通 ability 成功率：

```text
Ability_k = successful_routes_in_ability_k / total_routes_in_ability_k
```

普通 route 是否成功：

```text
status 是 Completed 或 Perfect
并且除 min_speed_infractions 外没有任何 infraction
```

`Traffic_Signs` 有额外逻辑：

1. 对 route XML 中的 keypoints 用 CARLA `GlobalRoutePlanner` 插值。
2. 找到第一个 junction waypoint。
3. 计算 junction completion：

```text
junction_completion = (count_to_first_junction + 8) / len(waypoint_route)
```

4. 如果：

```text
record_completion > junction_completion
且 stop_infraction 为空
且 red_light_infraction 为空
```

则该 route 在 `Traffic_Signs` ability 上算成功。

最终平均能力：

```text
Ability_mean = (Overtaking + Merging + Emergency_Brake + Give_Way + Traffic_Signs) / 5
```

## 5. Driving Efficiency

文件：`tools/efficiency_smoothness_benchmark.py`

数据来源：

- route JSON 中 `infractions.min_speed_infractions`。
- 每条 min speed infraction message 形如：

```text
Average speed is X% of the surrounding traffic's one
```

脚本会用正则提取百分比 `X%`。

每条 route 的效率值：

```text
driving_efficiency_route = mean(valid_min_speed_percentages)
```

全局输出：

```text
Driving Efficiency = mean(driving_efficiency_route)
```

代码细节：

- 百分比大于 1000 的异常值会被跳过。
- 只有存在 `min_speed_infractions` 的 route 会进入 `driving_efficiency` 列表。
- 如果没有任何 `min_speed_infractions`，原脚本会遇到空列表除法，需要自行防护。

## 6. Driving Smoothness

文件：`tools/efficiency_smoothness_benchmark.py`

数据来源：

- `SAVE_PATH/<save_name>/metric_info.json`
- 字段来自 CARLA hero actor：
  - `acceleration`
  - `angular_velocity`
  - `forward_vector`
  - `right_vector`
  - `location`
  - `rotation`

平顺性阈值：

| 子指标 | 阈值 |
|---|---:|
| jerk magnitude | `|jerk| < 8.37 m/s^3` |
| lateral acceleration | `|lat_acc| < 4.89 m/s^2` |
| longitudinal acceleration | `-4.05 < lon_acc < 2.40 m/s^2` |
| yaw acceleration | `|yaw_acc| < 1.93 rad/s^2` |
| longitudinal jerk | `|lon_jerk| < 4.13 m/s^3` |
| yaw rate | `|yaw_rate| < 0.95 rad/s` |

计算步骤：

1. 从 `metric_info.json` 读取每帧动力学数据。
2. 用 `forward_vector` 和 `right_vector` 分解加速度：

```text
lon_acc = dot(acceleration_xy, forward_vector_xy)
lat_acc = dot(acceleration_xy, right_vector_xy)
mag_acc = sqrt(ax^2 + ay^2)
```

3. 对信号使用 Savitzky-Golay filter 平滑。
4. 用时间间隔 `0.1 s` 计算 jerk。
5. 判断上述 6 个条件是否全部满足。
6. 默认每 20 帧分段计算：

```text
smoothness_route = number_of_comfort_segments / number_of_valid_segments
```

全局输出：

```text
Driving Smoothness = mean(smoothness_route)
```

## 7. 开环检测指标

代码来源：

- `mmcv/datasets/B2D_dataset.py`
- `mmcv/datasets/B2D_e2e_dataset.py`
- `mmcv/datasets/eval_utils/`
- `mmcv/core/evaluation/`

BEVFormer/UniAD/VAD 的 detection 评测基本沿用 nuScenes detection protocol，包括：

- `mAP`
- `NDS`
- `mATE`
- `mASE`
- `mAOE`
- `mAVE`
- `mAAE`

常见含义：

- `mAP`：多类别平均精度，按中心距离阈值匹配。
- `NDS`：nuScenes Detection Score，综合 mAP 和多个 TP error。
- `mATE`：平移误差。
- `mASE`：尺度误差。
- `mAOE`：朝向误差。
- `mAVE`：速度误差。
- `mAAE`：属性误差。

这些是开环感知指标，不直接等同于 CARLA 闭环驾驶成绩。

## 8. 开环运动预测指标

代码来源：`mmcv/datasets/B2D_vad_dataset.py`

VAD dataset evaluate 中统计：

- `EPA`
- `ADE`
- `FDE`
- `MR`

类别：

- `car`
- `pedestrian`

公式：

```text
EPA_cls = (hit_cls - alpha * fp_cls) / gt_cls
alpha = 0.5
ADE_cls = total_ADE_cls / cnt_ADE_cls
FDE_cls = total_FDE_cls / cnt_FDE_cls
MR_cls = total_MR_cls / cnt_FDE_cls
```

含义：

- `ADE`：Average Displacement Error，预测轨迹平均位移误差。
- `FDE`：Final Displacement Error，最终时刻位移误差。
- `MR`：Miss Rate，未命中率。
- `EPA`：带 false positive 惩罚的 endpoint/trajectory 类指标，具体 hit/fp 的判定在模型输出的 `metric_results` 中累计。

## 9. 开环规划指标

代码来源：

- `mmcv/models/dense_heads/planning_head_plugin/planning_metrics.py`
- `mmcv/models/dense_heads/planning_head_plugin/metric_stp3.py`
- `adzoo/uniad/test_utils.py`
- `mmcv/datasets/B2D_vad_dataset.py`

### 9.1 L2

UniADPlanningMetric 中：

```text
L2_t = sqrt(((pred_xy_t - gt_xy_t)^2 * gt_mask_t).sum())
```

最终：

```text
L2_t_mean = Σ L2_t / total_samples
```

注意：仓库原文档说明 UniAD 和 VAD 对 Planning L2 的报告口径不同：

- UniAD 原始代码按每个未来时刻计算，例如 0.5s、1.0s、1.5s。
- VAD 按时间段平均，例如 0-0.5s、0-1.0s、0-1.5s。
- 最终论文/表格报告会把 UniAD 的 L2 转换成 VAD 口径。

### 9.2 Object Collision

`UniADPlanningMetric.evaluate_coll()` 统计：

- `obj_col`
- `obj_box_col`

含义：

- `obj_col`：预测轨迹中心点是否落入未来占用 segmentation。
- `obj_box_col`：把 ego 车体矩形投影到 BEV 后，是否和未来占用 segmentation 相交。

为避免把 ground truth 本身不可避免的碰撞计入，代码会先计算 `gt_box_coll`，并过滤 GT 已碰撞位置。

### 9.3 ST-P3 风格规划碰撞

`metric_stp3.py` 使用 BEV occupancy map：

- ego 尺寸：`width = 1.85`、`length = 4.084`。
- BEV 范围：`[-50, 50] m`，分辨率 `0.5 m`，得到约 `200 x 200` 网格。
- 把预测轨迹和 ego box 映射到 BEV 像素，检查是否与车辆/行人 occupancy 相交。

## 10. 开环 occupancy/map 指标

代码来源：`adzoo/uniad/test_utils.py`

UniAD 开环测试中，如果模型有 occupancy head：

- 对 `30x30` 和 `100x100` 范围分别统计。
- 使用 `IntersectionOverUnion` 计算 IoU。
- 使用 `PanopticMetric` 计算 panoptic 相关指标。

代码中：

```text
EVALUATION_RANGES = {
  '30x30': (70, 130),
  '100x100': (0, 200)
}
```

这些指标用于离线 perception/prediction/planning 评估，不参与 Bench2Drive 闭环 driving score。

## 11. 指标来源总结

闭环主指标来源：

- CARLA 实时 actor 状态。
- CARLA collision sensor。
- CARLA traffic light/stop sign actor。
- Route XML 插值后的路线 waypoint。
- scenario_runner criteria 生成的 `TrafficEvent`。
- `StatisticsManager` 汇总。

闭环后处理指标来源：

- route JSON。
- agent 保存的 `metric_info.json`。

开环指标来源：

- Bench2Drive 数据集标注。
- 模型离线预测结果。
- nuScenes 风格检测评测器。
- UniAD/VAD 自带 motion/planning/occupancy metric。
