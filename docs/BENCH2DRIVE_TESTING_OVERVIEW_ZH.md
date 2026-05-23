# Bench2Drive 测试体系总览

本文档面向复现 Bench2DriveZoo 的自动驾驶闭环测试。它把当前仓库 `Bench2DriveZoo` 和外部评测工程 `Bench2Drive` 放在一起解释，因为真正的闭环指标并不完全在本仓库内。

## 1. 两个工程的分工

`Bench2DriveZoo` 是模型仓库，核心职责是：

- 提供 BEVFormer、UniAD、VAD 等模型代码、配置、训练/开环评测脚本。
- 提供 CARLA agent 适配代码，把 UniAD/VAD 的预测轨迹转成 `carla.VehicleControl`。
- 在闭环运行时保存 `metric_info.json`、摄像头图像、BEV 图和 PID 控制元数据。

`Bench2Drive` 是闭环 benchmark/leaderboard 工程，核心职责是：

- 启动 CARLA、加载 route、天气、交通参与者和场景。
- 每个 simulation tick 调用模型 agent 的 `run_step()`。
- 把 agent 返回的 `VehicleControl` 应用到 ego vehicle。
- 用 CARLA 实时状态触发碰撞、红灯、Stop、路线完成率、偏离路线、阻塞等测试事件。
- 汇总 route JSON、全局 driving score、success rate、ability、efficiency、smoothness。

因此，闭环测试链路可以理解为：

```text
Bench2Drive route XML
  -> Bench2Drive leaderboard_evaluator
  -> CARLA world + ego vehicle + sensors + scenarios
  -> Bench2DriveZoo team_code/*.py agent
  -> UniAD/VAD checkpoint 推理
  -> PIDController 输出 VehicleControl
  -> CARLA 执行动作
  -> scenario_runner criteria 实时记录 TrafficEvent
  -> StatisticsManager 写 route 结果 JSON
  -> tools/*.py 合并和计算总指标
```

## 2. 闭环与开环的区别

闭环测试：

- 运行 CARLA。
- 模型每帧从 CARLA 传感器拿实时数据。
- 模型输出控制量，车辆运动会反过来影响下一帧输入。
- 指标来自车辆真实驾驶结果，例如是否撞车、是否闯红灯、路线完成率、是否堵住。
- 主要代码在 `Bench2Drive` 的 `leaderboard/`、`scenario_runner/` 和 `tools/`，模型侧代码在本仓库 `team_code/`。

开环测试：

- 不运行 CARLA 闭环控制车辆。
- 使用已采集的 Bench2Drive 数据集样本做离线推理。
- 预测不会反作用于下一帧输入。
- 指标是检测、跟踪、运动预测、占用、规划轨迹 L2/碰撞等离线指标。
- 主要代码在本仓库 `adzoo/*/test.py`、`mmcv/datasets/B2D*.py`、`mmcv/models/dense_heads/planning_head_plugin/`。

## 3. 本仓库中和闭环测试最相关的文件

`team_code/uniad_b2d_agent.py`

- UniAD 闭环 agent。
- 定义 `get_entry_point()`，供 Bench2Drive leaderboard 动态加载。
- 定义 `sensors()`，声明 6 个 RGB 相机、IMU、GNSS、speedometer；设置 `SAVE_PATH` 时还会加顶视 BEV 相机。
- `run_step()` 每帧把传感器输入转成 UniAD 所需格式，调用模型推理，取 `planning.result_planning.sdc_traj`，再用 PID 转为 `VehicleControl`。
- 保存 `metric_info.json`，供 efficiency/smoothness 后处理使用。

`team_code/vad_b2d_agent.py`

- VAD 闭环 agent。
- 输入传感器和 UniAD 基本一致。
- `run_step()` 取 `pts_bbox.ego_fut_preds` 的自车未来轨迹，根据导航 command 选择对应轨迹，再用 PID 控制车辆。
- 保存每帧轨迹、控制、metric 信息。

`team_code/vad_b2d_agent_visualize.py`

- VAD 可视化版本 agent。
- 逻辑类似 VAD agent，但更偏向保存/可视化中间结果。

`team_code/planner.py`

- `RoutePlanner`。
- 接收 Bench2Drive 传入的 global GPS route。
- 每帧根据当前 GPS/位置找到最近的未来 route node 和导航 command。
- agent 用它构造模型输入中的 `command` 和局部目标点。

`team_code/pid_controller.py`

- `PIDController.control_pid()`。
- 输入模型预测的未来 waypoints、当前 speed、局部导航目标点。
- 输出 `steer`、`throttle`、`brake`。
- 这是 UniAD/VAD 从“规划轨迹”变成“闭环可控车”的关键。

## 4. Bench2Drive 中和闭环测试最相关的文件

以下文件位于外部 Bench2Drive 工程，例如本机检查到的 `G:\Bench2Drive`。

`leaderboard/leaderboard/leaderboard_evaluator.py`

- 闭环评测入口。
- 加载 route、town、weather、scenario。
- 动态 import agent 文件，调用 `get_entry_point()` 获得 agent 类。
- 调用 `agent.set_global_plan()` 和 `agent.setup()`。
- 读取 `agent.sensors()` 并交给 `AgentWrapper` 创建 CARLA 传感器。
- 每条 route 结束后调用 `StatisticsManager.compute_route_statistics()`。

`leaderboard/leaderboard/scenarios/scenario_manager.py`

- 闭环 tick 主循环。
- 每个 tick：
  - `world.tick()`
  - 更新 `GameTime` 和 `CarlaDataProvider`
  - 通过 wrapper 调用 agent
  - `ego_vehicle.apply_control(ego_action)`
  - tick 场景树和 criteria
  - 必要时更新 live statistics
- 当前版本有 `tick_count > 4000` 的超时保护。

`leaderboard/leaderboard/autoagents/agent_wrapper.py`

- 根据 agent 的 `sensors()` 创建 CARLA 传感器。
- 负责 sensor 类型、数量、安装半径合法性检查。
- 把 CARLA sensor callback 接入 agent 的 `SensorInterface`。

`leaderboard/leaderboard/envs/sensor_interface.py`

- 传感器数据队列。
- `CallBack` 把 CARLA Image/LiDAR/Radar/GNSS/IMU 转成 numpy 或 dict。
- `SensorInterface.get_data(frame)` 等待同一 frame 的所有传感器数据，并传给 agent。

`leaderboard/leaderboard/autoagents/autonomous_agent.py`

- 所有 agent 的基类。
- `__call__()` 中调用 `sensor_interface.get_data(GameTime.get_frame())`，再调用具体 agent 的 `run_step(input_data, timestamp)`。
- `get_metric_info()` 从 CARLA hero actor 读取 acceleration、angular_velocity、forward_vector、right_vector、location、rotation。

`leaderboard/leaderboard/scenarios/route_scenario.py`

- 构建 route scenario 和测试 criteria。
- 常驻 criteria 包括路线完成率、是否出路线车道、碰撞、红灯、Stop、最低速度、路线偏离、车辆阻塞。

`scenario_runner/srunner/scenariomanager/scenarioatomics/atomic_criteria.py`

- 所有闭环事件的实时检测逻辑。
- 例如 `CollisionTest`、`RunningRedLightTest`、`RunningStopTest`、`RouteCompletionTest`、`OutsideRouteLanesTest`。

`leaderboard/leaderboard/utils/statistics_manager.py`

- 把 criteria 产生的 `TrafficEvent` 转成 route JSON。
- 计算 route completion、infraction penalty、driving score。
- 计算全局均值和每公里 infractions。

`tools/merge_route_json.py`

- 合并多条 route 的 JSON。
- 计算 Bench2Drive 口径的 driving score 和 success rate。
- Bench2Drive 默认总 route 数是 220，缺失 route 会导致指标不准确。

`tools/ability_benchmark.py`

- 按场景类型聚合 Overtaking、Merging、Emergency_Brake、Give_Way、Traffic_Signs 五类能力指标。

`tools/efficiency_smoothness_benchmark.py`

- 读取合并后的 route JSON 和每条 route 的 `metric_info.json`。
- 计算 Driving Efficiency 和 Driving Smoothness。

## 5. 推荐阅读顺序

1. 先读 `docs/BENCH2DRIVE_TESTING_CODE_MAP_ZH.md`，明确测试代码位置和调用关系。
2. 再读 `docs/BENCH2DRIVE_CLOSED_LOOP_DATAFLOW_ZH.md`，理解 CARLA 每帧如何把数据送进模型、模型如何控制车。
3. 再读 `docs/BENCH2DRIVE_METRICS_ZH.md`，对照代码理解所有闭环指标公式。
4. 最后回到 `docs/EVAL_IN_CARLA_ZH.md` 和 `bridge/CLOSED_LOOP_REPRO_AND_ROADTAIL_GUIDE_ZH.md`，执行复现命令和地图替换。

## 6. 复现时最容易混淆的点

- `Bench2DriveZoo` 不是完整闭环评测器，它需要配合 `Bench2Drive` leaderboard 运行。
- BEVFormer 在本仓库主要是感知/开环检测模型；当前主分支闭环 agent 明确提供的是 UniAD 和 VAD。
- UniAD/VAD 可以闭环控制车，是因为 `team_code/*_agent.py` 加了 route planner 和 PID controller。
- Driving Score、Success Rate、Ability 等 leaderboard 指标不是在 `adzoo/*/test.py` 里算的，而是在 `Bench2Drive/tools/` 和 `StatisticsManager` 里算的。
- 如果没有设置 `SAVE_PATH`，闭环主指标仍可从 route JSON 得到，但 smoothness/efficiency 所需的 `metric_info.json` 不完整或不存在。
