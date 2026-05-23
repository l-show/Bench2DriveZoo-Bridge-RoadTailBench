# Bench2Drive 测试代码位置与关系

本文档按“闭环主链路”和“开环离线评测”整理测试相关代码。当前仓库是 `Bench2DriveZoo`，但闭环 benchmark 的评测器代码在 `Bench2Drive` 工程中。

## 1. 闭环测试入口关系

闭环运行时通常由 Bench2Drive leaderboard 启动，核心调用顺序如下：

```text
leaderboard_evaluator.py
  -> RouteIndexer 读取 routes XML
  -> 加载 CARLA town/weather/ego vehicle/scenario
  -> 动态加载 Bench2DriveZoo 的 team_code agent
  -> agent.setup(config+save_name)
  -> agent.sensors()
  -> AgentWrapper.setup_sensors()
  -> ScenarioManager.run_scenario()
  -> 每帧 agent.run_step()
  -> ego_vehicle.apply_control()
  -> scenario_tree.tick_once()
  -> criteria 产生 TrafficEvent
  -> StatisticsManager.compute_route_statistics()
  -> checkpoint JSON
```

## 2. Bench2DriveZoo 侧文件

### 2.1 UniAD 闭环 agent

文件：`team_code/uniad_b2d_agent.py`

关键函数：

- `get_entry_point()`：返回 agent 类名字符串，leaderboard 用它实例化 agent。
- `setup(path_to_conf_file)`：加载 config、checkpoint、模型、推理 pipeline、PID controller、保存路径等。
- `sensors()`：声明 CARLA 要创建的传感器。
- `tick(input_data)`：从 CARLA 输入中取相机、GPS、IMU、速度，并计算 route command。
- `run_step(input_data, timestamp)`：每个 CARLA tick 被调用一次，完成模型推理和控制输出。
- `save(tick_data)`：保存图像、BEV、PID metadata、`metric_info.json`。

UniAD 的闭环控制数据路径：

```text
CARLA sensor input_data
  -> tick()
  -> results 字典：img/lidar2img/can_bus/command 等
  -> inference_only_pipeline
  -> model(..., return_loss=False)
  -> output['planning']['result_planning']['sdc_traj']
  -> PIDController.control_pid()
  -> carla.VehicleControl
```

### 2.2 VAD 闭环 agent

文件：`team_code/vad_b2d_agent.py`

关键函数和 UniAD 基本一致。差异在于模型输出取法：

```text
output['pts_bbox']['ego_fut_preds']
  -> cumsum 得到未来轨迹
  -> 根据 command 选一条自车轨迹
  -> PIDController.control_pid()
  -> carla.VehicleControl
```

VAD 还维护 `prev_control_cache`，把历史 steer 放入模型输入特征 `ego_lcf_feat[8]`。

### 2.3 VAD 可视化 agent

文件：`team_code/vad_b2d_agent_visualize.py`

用途：

- 闭环控制逻辑接近 `vad_b2d_agent.py`。
- 更强调保存图像、BEV、可视化中间结果。
- 适合调试模型在 CARLA 闭环里的输入输出，但正式批量评测建议先用非 visualize agent。

### 2.4 RoutePlanner

文件：`team_code/planner.py`

核心职责：

- 把 leaderboard 传入的 global GPS route 转成本地平面坐标。
- 根据当前 GPS/位置弹出已经通过的 route 点。
- 返回下一个 near node 和 RoadOption command。

agent 中的使用方式：

```text
gps = input_data['GPS'][1][:2]
pos = gps_to_location(gps)
near_node, near_command = self._route_planner.run_step(pos)
```

这些输出随后进入模型输入中的 `command` 和 PID 的 `local_command_xy`。

### 2.5 PID 控制器

文件：`team_code/pid_controller.py`

核心函数：`PIDController.control_pid(waypoints, speed, target)`

输入：

- `waypoints`：模型预测的自车未来轨迹点。
- `speed`：speedometer 读数。
- `target`：route planner 给出的局部导航目标点。

输出：

- `steer`
- `throttle`
- `brake`
- `metadata`

控制逻辑：

- 根据相邻 waypoint 距离估计 `desired_speed`。
- 选择一个用于横向控制的 aim point。
- 如果导航 target 比模型 aim 更合理，则用 target 修正转向目标。
- turn PID 输出 steer。
- speed PID 输出 throttle。
- 当期望速度过低或当前速度明显高于期望速度时 brake。

## 3. Bench2Drive 侧文件

### 3.1 评测总入口

文件：`leaderboard/leaderboard/leaderboard_evaluator.py`

核心职责：

- 解析命令行参数，如 `--routes`、`--agent`、`--agent-config`、`--checkpoint`。
- 建立 CARLA client，加载 town。
- 根据 XML route 创建 route scenario。
- 创建 route record：`StatisticsManager.create_route_data()`。
- 实例化 agent 并传入 global plan：

```text
self.agent_instance = agent_class_obj(args.host, args.port, args.debug)
self.agent_instance.set_global_plan(self.route_scenario.gps_route, self.route_scenario.route)
args.agent_config = args.agent_config + '+' + save_name
self.agent_instance.setup(args.agent_config)
```

- 运行 scenario。
- 每条 route 结束后计算 route statistics。
- 所有 route 结束后计算 global statistics。

### 3.2 每帧闭环主循环

文件：`leaderboard/leaderboard/scenarios/scenario_manager.py`

关键函数：`_tick_scenario()`

每帧流程：

```text
world.tick(timeout)
timestamp = world.get_snapshot().timestamp
GameTime.on_carla_tick(timestamp)
CarlaDataProvider.on_carla_tick()
ego_action = self._agent_wrapper()
ego_vehicle.apply_control(ego_action)
scenario_tree.tick_once()
```

其中 `self._agent_wrapper()` 最终会调用 agent 的 `__call__()`，再调用 `run_step()`。

### 3.3 传感器创建和数据分发

文件：`leaderboard/leaderboard/autoagents/agent_wrapper.py`

关键职责：

- 读取 agent 的 `sensors()`。
- 检查 sensor 类型、数量、安装半径。
- 创建真实 CARLA sensor 或 pseudo sensor。
- 给每个 sensor 注册 callback：

```text
sensor.listen(CallBack(id_, type_, sensor, agent.sensor_interface))
```

文件：`leaderboard/leaderboard/envs/sensor_interface.py`

关键职责：

- `CallBack` 把 CARLA 原始 sensor 数据转换为 numpy/dict。
- `SensorInterface.update_sensor(tag, data, frame)` 把数据放入队列。
- `SensorInterface.get_data(frame)` 等待当前 frame 的所有传感器数据。

### 3.4 Agent 基类

文件：`leaderboard/leaderboard/autoagents/autonomous_agent.py`

关键函数：

- `set_global_plan()`：把 route 传给 agent。
- `__call__()`：读取 sensor data，调用 `run_step()`，返回 `VehicleControl`。
- `get_metric_info()`：读取 hero actor 的动力学状态，用于 smoothness。

`get_metric_info()` 输出字段：

- `acceleration`
- `angular_velocity`
- `forward_vector`
- `right_vector`
- `location`
- `rotation`

这些字段来自 CARLA actor API，而不是模型预测。

### 3.5 Route scenario 和 criteria

文件：`leaderboard/leaderboard/scenarios/route_scenario.py`

常驻 criteria：

```text
RouteCompletionTest
OutsideRouteLanesTest
CollisionTest
RunningRedLightTest
RunningStopTest
MinimumSpeedRouteTest
InRouteTest
ActorBlockedTest
```

在本机检查到的 Bench2Drive 版本中：

- leaderboard 版 `MinimumSpeedRouteTest(..., checkpoints=20)`。
- leaderboard 版 `ActorBlockedTest(min_speed=0.1, max_time=60.0)`。
- scenario_runner 备份/原始 route_scenario 中有 `checkpoints=4`、`max_time=180.0` 的版本；实际评测应以运行入口 import 的 `leaderboard/leaderboard/scenarios/route_scenario.py` 为准。

### 3.6 事件定义

文件：`scenario_runner/srunner/scenariomanager/traffic_events.py`

主要 `TrafficEventType`：

- `COLLISION_STATIC`
- `COLLISION_VEHICLE`
- `COLLISION_PEDESTRIAN`
- `ROUTE_DEVIATION`
- `ROUTE_COMPLETION`
- `TRAFFIC_LIGHT_INFRACTION`
- `STOP_INFRACTION`
- `OUTSIDE_ROUTE_LANES_INFRACTION`
- `VEHICLE_BLOCKED`
- `MIN_SPEED_INFRACTION`
- `YIELD_TO_EMERGENCY_VEHICLE`
- `SCENARIO_TIMEOUT`

### 3.7 实时事件检测

文件：`scenario_runner/srunner/scenariomanager/scenarioatomics/atomic_criteria.py`

重点类：

- `CollisionTest`：用 CARLA collision sensor 检测碰撞，并按静态物、车辆、行人分类。
- `RouteCompletionTest`：根据 ego 是否通过 route waypoint 计算完成百分比。
- `OutsideRouteLanesTest`：累计车辆在路线车道外/错误方向车道上的距离百分比。
- `InRouteTest`：判断 ego 是否偏离 route 超过允许距离，超过后终止 route。
- `RunningRedLightTest`：检测 ego 是否在红灯状态下穿越 stop line。
- `RunningStopTest`：检测 ego 是否在 Stop 标志影响区域内真正停下。
- `MinimumSpeedRouteTest`：按 route checkpoint 统计 ego 平均速度相对背景车平均速度的百分比。
- `ActorBlockedTest`：ego 低于最小速度太久则判定 blocked。
- `YieldToEmergencyVehicleTest`：场景级 criterion，用于急救车让行能力。
- `ScenarioTimeoutTest`：场景超时事件。

### 3.8 统计和 JSON

文件：`leaderboard/leaderboard/utils/statistics_manager.py`

核心职责：

- 读取所有 criteria 的 `node.events`。
- 按事件类型写入 `infractions`。
- 计算每条 route 的：
  - `score_route`
  - `score_penalty`
  - `score_composed`
  - `status`
  - `meta.duration_game`
  - `meta.duration_system`
- 计算全局均值、标准差、每公里 infractions。

输出 JSON 中每条 route 的典型结构：

```json
{
  "route_id": "...",
  "scenario_name": "...",
  "weather_id": "...",
  "save_name": "...",
  "status": "Completed/Perfect/Failed ...",
  "infractions": {
    "collisions_vehicle": [],
    "red_light": [],
    "stop_infraction": [],
    "outside_route_lanes": [],
    "min_speed_infractions": []
  },
  "scores": {
    "score_route": 100.0,
    "score_penalty": 1.0,
    "score_composed": 100.0
  }
}
```

## 4. 后处理工具

文件：`tools/merge_route_json.py`

- 合并多 route JSON。
- 默认按 220 条 route 计算 driving score 和 success rate。
- 如果不是 220 条，会打印 warning。

文件：`tools/ability_benchmark.py`

- 根据 route XML 中的 scenario type，把 route success 聚合到 5 类 ability。
- Traffic_Signs 有额外 junction completion 判断。

文件：`tools/efficiency_smoothness_benchmark.py`

- 读取 `merged.json` 和 `SAVE_PATH/<save_name>/metric_info.json`。
- 计算 Driving Efficiency 和 Driving Smoothness。

## 5. 开环测试代码

### 5.1 测试脚本

`adzoo/bevformer/test.py`

- BEVFormer 开环检测评测。
- 通常输出 bbox 相关指标，如 mAP、NDS。

`adzoo/uniad/test.py`

- UniAD 开环测试入口。
- 调用 `adzoo/uniad/test_utils.py` 做 bbox、occupancy、planning 等结果收集。

`adzoo/vad/test.py`

- VAD 开环测试入口。
- 调用 dataset 的 `evaluate()`。

### 5.2 Dataset evaluate

`mmcv/datasets/B2D_dataset.py`

- BEVFormer/检测类评测。
- 使用 nuScenes 风格 detection metrics。

`mmcv/datasets/B2D_e2e_dataset.py`

- UniAD e2e 数据集评测。
- 当前注释说明主要支持 detection 和 planning。

`mmcv/datasets/B2D_vad_dataset.py`

- VAD 数据集评测。
- 额外统计 motion prediction 指标和 planning 指标。

### 5.3 规划/占用指标

`mmcv/models/dense_heads/planning_head_plugin/planning_metrics.py`

- `UniADPlanningMetric`。
- 统计 `L2`、`obj_col`、`obj_box_col`。

`mmcv/models/dense_heads/planning_head_plugin/metric_stp3.py`

- ST-P3 风格 planning metric。
- 用 BEV occupancy 检测预测轨迹是否与目标占用区域碰撞。

`adzoo/uniad/test_utils.py`

- UniAD 开环测试时统计 occupancy IoU、panoptic metric、planning metric。

## 6. 修改/调试时应优先看的文件

如果你要改闭环 agent 输入：

- `team_code/uniad_b2d_agent.py`
- `team_code/vad_b2d_agent.py`
- `leaderboard/leaderboard/autoagents/agent_wrapper.py`
- `leaderboard/leaderboard/envs/sensor_interface.py`

如果你要改闭环控制：

- `team_code/pid_controller.py`
- `team_code/planner.py`
- agent 的 `run_step()`

如果你要改闭环指标：

- `scenario_runner/srunner/scenariomanager/scenarioatomics/atomic_criteria.py`
- `leaderboard/leaderboard/utils/statistics_manager.py`
- `tools/merge_route_json.py`
- `tools/ability_benchmark.py`
- `tools/efficiency_smoothness_benchmark.py`

如果你要换地图/路线：

- `leaderboard/data/*.xml`
- `leaderboard/leaderboard/utils/route_parser.py`
- `leaderboard/leaderboard/utils/route_indexer.py`
- `leaderboard/leaderboard/scenarios/route_scenario.py`

如果你只跑开环：

- `adzoo/*/test.py`
- `mmcv/datasets/B2D*.py`
- `mmcv/models/dense_heads/planning_head_plugin/*.py`
