# Bench2Drive CARLA 闭环数据流

本文档解释闭环测试中 CARLA 如何把实时数据传给模型 agent，agent 如何控制车辆，以及指标如何从实时状态中产生。

## 1. 闭环 tick 的主时序

核心代码：`Bench2Drive/leaderboard/leaderboard/scenarios/scenario_manager.py`

每个 simulation tick 的顺序：

```text
1. CARLA world.tick()
2. 读取 world snapshot timestamp
3. GameTime.on_carla_tick(timestamp)
4. CarlaDataProvider.on_carla_tick()
5. AgentWrapper 调用 agent.__call__()
6. agent.__call__() 从 SensorInterface 取当前 frame 输入
7. agent.run_step(input_data, timestamp)
8. agent 返回 carla.VehicleControl
9. ego_vehicle.apply_control(ego_action)
10. scenario_tree.tick_once()
11. criteria 检查实时驾驶事件
12. route 完成/失败/超时时，StatisticsManager 汇总结果
```

这里最重要的是：模型输出的控制量会真正作用到 CARLA 车辆上，车辆下一帧的位置、速度、传感器图像都会变化，所以这是闭环，不是离线 replay。

## 2. Agent 被如何加载

入口代码：`Bench2Drive/leaderboard/leaderboard/leaderboard_evaluator.py`

leaderboard 通过命令行参数指定 agent 文件，例如：

```bash
--agent /path/to/Bench2DriveZoo/team_code/uniad_b2d_agent.py
```

加载逻辑：

1. 动态 import agent Python 文件。
2. 调用文件中的 `get_entry_point()`。
3. 根据返回的类名实例化 agent。
4. 调用：

```text
agent.set_global_plan(gps_route, world_route)
agent.setup(agent_config + '+' + save_name)
agent.sensors()
```

`save_name` 会被附加到 `agent_config` 后面，Bench2DriveZoo agent 用它创建保存目录，并让 route JSON 中的 `save_name` 和 `SAVE_PATH/<save_name>/metric_info.json` 对齐。

## 3. 传感器声明

UniAD/VAD agent 文件：

- `team_code/uniad_b2d_agent.py`
- `team_code/vad_b2d_agent.py`

`sensors()` 返回列表。主要传感器：

| id | 类型 | 用途 |
|---|---|---|
| `CAM_FRONT` | `sensor.camera.rgb` | 前视相机 |
| `CAM_FRONT_LEFT` | `sensor.camera.rgb` | 前左相机 |
| `CAM_FRONT_RIGHT` | `sensor.camera.rgb` | 前右相机 |
| `CAM_BACK` | `sensor.camera.rgb` | 后视相机 |
| `CAM_BACK_LEFT` | `sensor.camera.rgb` | 后左相机 |
| `CAM_BACK_RIGHT` | `sensor.camera.rgb` | 后右相机 |
| `IMU` | `sensor.other.imu` | 加速度、角速度、compass |
| `GPS` | `sensor.other.gnss` | 经纬度 |
| `SPEED` | `sensor.speedometer` | 伪传感器，前向速度 |
| `bev` | `sensor.camera.rgb` | 仅在 `SAVE_PATH`/Bench2Drive 保存模式下添加，用于顶视保存 |

相机分辨率通常是：

```text
1600 x 900
```

agent 内部会把相机图像 JPEG 压缩到较低质量再解码，目的是模拟数据处理并节省中间表示。

## 4. 传感器创建与回调

核心代码：

- `Bench2Drive/leaderboard/leaderboard/autoagents/agent_wrapper.py`
- `Bench2Drive/leaderboard/leaderboard/envs/sensor_interface.py`

流程：

```text
agent.sensors()
  -> AgentWrapper.setup_sensors(vehicle)
  -> world.spawn_actor(sensor_blueprint, sensor_transform, attach_to=ego)
  -> sensor.listen(CallBack(id, type, sensor, agent.sensor_interface))
```

不同 sensor 的解析：

- CARLA Image -> numpy array，形状约为 `(H, W, 4)`。
- LiDAR -> numpy array，形状约为 `(N, 4)`。
- Radar -> numpy array，形状约为 `(N, 4)`。
- GNSS -> `[latitude, longitude, altitude]`。
- IMU -> `[acc_x, acc_y, acc_z, gyro_x, gyro_y, gyro_z, compass]`。
- Speedometer pseudo sensor -> `{'speed': forward_speed}`。

`SensorInterface.get_data(frame)` 会等待同一 frame 的所有 sensor 数据，然后返回：

```python
{
    "CAM_FRONT": (frame, image_array),
    "GPS": (frame, np.array([lat, lon, alt])),
    "IMU": (frame, np.array([... compass])),
    "SPEED": (frame, {"speed": speed}),
    ...
}
```

## 5. UniAD/VAD 每帧如何处理输入

agent 侧核心函数：`tick(input_data)`

主要处理：

```text
1. step += 1
2. 读取 6 路相机图像
3. 读取 GPS、speed、IMU compass、acceleration、angular_velocity
4. gps_to_location() 把经纬度转成本地 XY
5. RoutePlanner.run_step(pos) 得到 near route node 和 command
6. 组装 tick_data
```

`tick_data` 主要字段：

```python
{
    "imgs": {...},
    "gps": gps,
    "pos": pos,
    "speed": speed,
    "compass": compass,
    "bev": bev,
    "acceleration": acceleration,
    "angular_velocity": angular_velocity,
    "command_near": near_command,
    "command_near_xy": near_node
}
```

如果 compass 是 NaN，agent 会把 compass 置为 0，并把加速度/角速度置零，避免模型输入崩溃。

## 6. RoutePlanner 如何提供导航 command

代码：`team_code/planner.py`

输入：

- leaderboard 传入的 global GPS route。
- 当前 ego GPS 转换后的本地坐标。

逻辑：

- 路线保存在 deque 中。
- 每帧查找 ego 附近已经经过的 route 点。
- 弹出已经经过的点。
- 返回下一个目标点和 RoadOption command。

agent 中会把 command 转成模型需要的整数/one-hot：

```text
if command < 0:
    command = 4
command -= 1
```

并计算局部目标点：

```text
command_near_xy = near_node - current_position
local_command_xy = rotation_matrix @ command_near_xy
```

## 7. 模型推理输入如何构造

UniAD/VAD 都会构造类似 nuScenes/Bench2Drive 开环格式的 `results` 字典，字段包括：

- `img`
- `lidar2img`
- `lidar2cam`
- `can_bus`
- `command`
- `l2g_r_mat`
- `l2g_t`
- `img_shape`
- `ori_shape`
- `pad_shape`
- `box_type_3d`

`can_bus` 主要包含：

```text
position x/y
quaternion rotation
speed
acceleration
angular velocity
ego yaw
ego yaw degree
```

然后：

```text
results = inference_only_pipeline(results)
input_data_batch = mm_collate_to_batch_form([results], samples_per_gpu=1)
model(input_data_batch, return_loss=False, rescale=True)
```

## 8. UniAD 如何输出控制

代码：`team_code/uniad_b2d_agent.py`

模型输出：

```python
out_truck = output_data_batch[0]["planning"]["result_planning"]["sdc_traj"][0]
```

含义：UniAD planning head 预测的自车未来轨迹。

控制转换：

```text
out_truck
  -> PIDController.control_pid(out_truck, speed, local_command_xy)
  -> steer/throttle/brake
  -> carla.VehicleControl
```

额外限制：

- `brake < 0.05` 时置 0。
- `throttle > brake` 时 brake 置 0。
- `speed > 5` 时 throttle 置 0。
- throttle clip 到 `[0, 0.75]`。
- steer clip 到 `[-1, 1]`。

## 9. VAD 如何输出控制

代码：`team_code/vad_b2d_agent.py`

模型输出：

```python
all_out_truck_d1 = output_data_batch[0]["pts_bbox"]["ego_fut_preds"]
all_out_truck = np.cumsum(all_out_truck_d1, axis=1)
out_truck = all_out_truck[command]
```

含义：

- VAD 输出多个 command 对应的自车未来增量轨迹。
- `cumsum` 转成绝对未来轨迹。
- 根据 route planner 的 command 选择一条轨迹。

控制转换和 UniAD 一样：

```text
out_truck
  -> PIDController.control_pid()
  -> carla.VehicleControl
```

VAD 还会把历史控制中的 steer 放入 `ego_lcf_feat`，作为模型输入的一部分。

## 10. PIDController 如何从轨迹变成油门刹车转向

代码：`team_code/pid_controller.py`

输入：

```text
waypoints: 模型预测未来轨迹
speed: 当前速度
target: route planner 局部目标点
```

期望速度：

```text
desired_speed = mean_norm_between_future_waypoints * 2.0
```

横向 aim point：

- 遍历相邻 waypoint 中点。
- 选择距离 `aim_dist = 4.0 m` 最接近的 waypoint。
- 如果 route target 角度更可靠，则使用 target 作为 aim。

转向角误差：

```text
angle = (pi/2 - atan2(aim_y, aim_x)) converted to degrees / 90
```

刹车条件：

```text
brake = desired_speed < brake_speed
        or speed / desired_speed > brake_ratio
```

默认阈值：

```text
brake_speed = 0.4
brake_ratio = 1.1
max_throttle = 0.75
clip_delta = 0.25
```

输出：

```python
carla.VehicleControl(
    steer=clip(steer, -1, 1),
    throttle=clip(throttle, 0, 0.75),
    brake=clip(brake, 0, 1)
)
```

## 11. CARLA 如何执行控制

代码：`ScenarioManager._tick_scenario()`

```python
ego_action = self._agent_wrapper()
self.ego_vehicles[0].apply_control(ego_action)
```

`ego_action` 是 agent 返回的 `carla.VehicleControl`。

应用控制后，下一帧 `world.tick()` 会推进物理仿真，车辆位置、速度、姿态和传感器观测都会更新。

## 12. 指标如何从 CARLA 实时获得

指标不是 agent 自己算的，而是 scenario criteria 从 CARLA 实时状态中触发事件。

### 12.1 碰撞

来源：

```text
CARLA sensor.other.collision
```

事件：

```text
COLLISION_STATIC
COLLISION_VEHICLE
COLLISION_PEDESTRIAN
```

### 12.2 红灯

来源：

- CARLA traffic light actors。
- ego transform。
- ego bounding box。
- HD map waypoint/lane。

判定：

- 红灯状态为 Red。
- ego 处于该红灯影响 lane。
- 车辆尾部线段穿越 stop line。

### 12.3 Stop sign

来源：

- CARLA stop sign actors。
- ego transform 和速度。
- HD map waypoint。

判定：

- ego 进入 stop trigger area。
- 离开前没有出现速度 `< 0.1 m/s`。

### 12.4 Route completion

来源：

- route waypoint。
- ego location。

判定：

- ego 是否通过 route waypoint。
- 累计通过距离占 route 总距离百分比。

### 12.5 Route deviation

来源：

- route waypoint。
- ego location。

判定：

- ego 到 route 的横向距离是否超过阈值。
- 偏离距离比例是否超过阈值。

### 12.6 Outside route lanes

来源：

- ego location。
- map waypoint road/lane。
- route waypoint。

判定：

- ego 是否离开 route lane。
- ego 是否进入错误方向 lane。
- 累计错误距离占已完成路线距离百分比。

### 12.7 Blocked

来源：

- ego velocity。
- simulation game time。

判定：

- 速度低于 0.1 m/s 持续超过 60 秒。

### 12.8 Minimum speed

来源：

- ego velocity。
- background vehicles velocity。
- route checkpoint。

判定：

- 每个 checkpoint 统计 ego 平均速度和背景车平均速度的比例。
- 记录 `MIN_SPEED_INFRACTION`，但当前版本不扣 driving score。

## 13. `metric_info.json` 的来源和用途

`metric_info.json` 由 Bench2DriveZoo agent 保存：

- `team_code/uniad_b2d_agent.py`
- `team_code/vad_b2d_agent.py`

每帧：

```python
metric_info = self.get_metric_info()
self.metric_info[self.step] = metric_info
```

`get_metric_info()` 实际定义在 Bench2Drive 的 agent 基类：

```text
leaderboard/leaderboard/autoagents/autonomous_agent.py
```

读取的是 CARLA hero actor：

```text
hero_actor.get_acceleration()
hero_actor.get_angular_velocity()
hero_actor.get_transform().get_forward_vector()
hero_actor.get_transform().get_right_vector()
hero_actor.get_transform().location
hero_actor.get_transform().rotation
```

用途：

- `tools/efficiency_smoothness_benchmark.py` 用它计算 Driving Smoothness。
- 也可用于调试车辆实际运动是否抖动、急刹、异常旋转。

注意：

- `metric_info.json` 依赖 `SAVE_PATH`。
- 如果不保存该文件，Driving Smoothness 无法按官方脚本计算。

## 14. route JSON 的来源和用途

route JSON 由 `StatisticsManager` 写入，通常通过 `--checkpoint` 指定。

每条 route 包括：

- `route_id`
- `scenario_name`
- `weather_id`
- `save_name`
- `status`
- `scores`
- `infractions`
- `meta`

后处理流程：

```text
多个 route json
  -> tools/merge_route_json.py
  -> merged.json
  -> tools/ability_benchmark.py
  -> *_ability.json
  -> tools/efficiency_smoothness_benchmark.py
  -> Driving Efficiency / Driving Smoothness
```

## 15. 调试闭环数据流的建议

如果 agent 没有启动：

- 检查 `get_entry_point()` 返回的类名是否正确。
- 检查 `--agent` 指向是否是当前分支的 agent 文件。
- 检查 `setup()` 中 checkpoint/config 路径。

如果传感器无数据：

- 检查 `sensors()` 中 sensor id 是否和 `tick()` 中读取的 key 一致。
- 检查 sensor 数量和类型是否超过 `agent_wrapper.py` 的限制。
- 检查 CARLA 是否同步 tick 正常。

如果能推理但车不动：

- 检查模型输出轨迹是否全零或 NaN。
- 检查 `PIDController.control_pid()` 中 `desired_speed`。
- 检查 `brake` 是否一直为 True。
- 检查 `local_command_xy` 是否方向异常。

如果主指标有但 smoothness 缺失：

- 检查是否设置 `SAVE_PATH`。
- 检查 `SAVE_PATH/<save_name>/metric_info.json` 是否存在。
- 检查 route JSON 中的 `save_name` 是否和保存目录一致。

如果换自定义地图后指标异常：

- 先确认 route XML 的 town 名称能被 CARLA 加载。
- 确认地图中 traffic light/stop sign trigger volume 正常。
- 确认 GlobalRoutePlanner 能沿 keypoints 插值。
- 确认 ego route 不穿过非 driving lane，否则 outside route/deviation 会异常升高。
