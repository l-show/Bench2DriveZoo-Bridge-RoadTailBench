# 更新日志：UniAD Agent 固定路径版本替换

本文记录当前 Bench2DriveZoo 目录下对 UniAD 闭环 agent 做的文件调整：

- 将原始 `team_code/uniad_b2d_agent.py` 备份为 `team_code/uniad_b2d_agent原始文件.py`
- 新增新的 `team_code/uniad_b2d_agent.py`
- 移除仓库根目录下原来的 `uniad_b2d_agent_固定路径.py`

经对比，新的 `team_code/uniad_b2d_agent.py` 与被移除的 `uniad_b2d_agent_固定路径.py` 内容一致。因此，本次调整可以理解为：把“固定 checkpoint 路径的 UniAD 调试版本”移动到标准 agent 入口位置，使 leaderboard 启动脚本继续通过 `team_code/uniad_b2d_agent.py` 加载 UniAD。

## 1. 新增与改名文件用途

| 文件 | 用途 |
|---|---|
| `team_code/uniad_b2d_agent原始文件.py` | 原始 UniAD agent 的备份文件，用于保留官方/原始实现，方便回滚或对照差异。 |
| `team_code/uniad_b2d_agent.py` | 当前实际被 leaderboard 加载的 UniAD agent。该版本加入固定 checkpoint 路径、BEV 传感器、速度日志和限速阈值调整。 |

leaderboard evaluator 通过 `--agent` 参数动态加载 agent 文件。若启动脚本仍指定：

```bash
--agent=/path/to/Bench2DriveZoo/team_code/uniad_b2d_agent.py
```

则后续闭环评测会使用新的固定路径版本，而不是备份的原始版本。

## 2. 新版脚本功能变化

新版 `team_code/uniad_b2d_agent.py` 保留了原始 UniAD agent 的核心闭环流程：

1. 通过 `get_entry_point()` 返回 `UniadAgent`。
2. 在 `setup()` 中读取配置、构建 UniAD 模型并加载 checkpoint。
3. 在 `sensors()` 中声明 6 路相机、IMU、GPS、Speedometer 等传感器。
4. 每帧通过 `tick()` 读取传感器数据并构造模型输入。
5. 通过 UniAD 模型输出规划轨迹。
6. 使用 PID controller 将规划轨迹转成 `carla.VehicleControl`。
7. 可选保存图像、BEV、meta 和 `metric_info.json`。

与原始文件相比，新版主要增加或修改了以下能力。

### 2.1 固定 checkpoint 加载路径

新版在 `setup()` 中强制指定 UniAD base checkpoint：

```python
cfg.load_from = '/home/hqj/bench2drive-work/Bench2DriveZoo/ckpts/uniad_base_b2d.pth'
self.ckpt_path = '/home/hqj/bench2drive-work/Bench2DriveZoo/ckpts/uniad_base_b2d.pth'
```

并打印：

```python
print(f"Force loading checkpoint from: {self.ckpt_path}")
```

这意味着即使命令行 `--agent-config` 中传入了其他 checkpoint，当前版本也会优先使用代码中写死的路径。

用途：

- 避免启动脚本中 `agent-config` 不完整或 checkpoint 路径错误导致模型加载失败。
- 便于在固定机器环境中快速调试 UniAD base 模型。

风险：

- 该路径是本机绝对路径，换机器或换目录后需要手动修改。
- `--agent-config` 的 checkpoint 段会被覆盖，不再完全由命令行控制。

### 2.2 新增默认 BEV 顶视相机

新版在 `sensors()` 默认传感器列表中新增了一个 id 为 `bev` 的 RGB 相机：

```python
{
    'type': 'sensor.camera.rgb',
    'x': 0.0, 'y': 0.0, 'z': 2.5,
    'roll': 0.0, 'pitch': -90.0, 'yaw': 0.0,
    'width': 512, 'height': 512, 'fov': 50,
    'id': 'bev'
}
```

原始版本中，`bev` 传感器只会在 `IS_BENCH2DRIVE` 环境变量存在时额外添加。新版将一个低高度顶视相机直接加入默认传感器列表，保证 `tick()` 中读取：

```python
bev = cv2.cvtColor(input_data['bev'][1][:, :, :3], cv2.COLOR_BGR2RGB)
```

时通常不会因为缺少 `bev` 输入而报错。

注意：如果同时设置了 `IS_BENCH2DRIVE`，代码后面仍会再追加一个 id 同为 `bev` 的高空顶视相机。不同版本 leaderboard 对重复 sensor id 的处理可能不同，运行异常时应重点检查这里。

### 2.3 调整高速限速阈值

原始版本在速度超过 `5 m/s` 时停止继续给油：

```python
if tick_data['speed'] > 5:
    throttle_traj = 0
```

新版改为：

```python
if tick_data['speed'] > 110 / 3.6:
    throttle_traj = 0
```

即阈值从约 `18 km/h` 提高到 `110 km/h`。这会让车辆在更高速度范围内仍允许 throttle 输出，更适合需要高速行驶或较长路线的闭环测试。

### 2.4 增加每帧中文运行日志

新版在 `run_step()` 中记录当前仿真时间，并在每帧返回控制量前打印：

```python
print(
    f"[主车 UniAD] SimTime: {current_sim_time:.3f}s | "
    f"Step: {self.step} | "
    f"Hero(自己)速度: {hero_speed_kmh:.2f} km/h | "
    f"Command: {tick_data['command_near']}"
)
```

日志内容包括：

| 字段 | 说明 |
|---|---|
| `SimTime` | 当前 CARLA 仿真时间戳。 |
| `Step` | agent 内部帧计数。 |
| `Hero(自己)速度` | ego 车速度，单位从 m/s 转为 km/h。 |
| `Command` | RoutePlanner 给出的近端导航命令。 |

该日志适合调试闭环运行状态，例如确认车辆是否持续推进、速度是否异常、导航 command 是否更新。

## 3. 参数说明

### 3.1 `path_to_conf_file`

`setup()` 接收 leaderboard 传入的 `path_to_conf_file`，通常来自 evaluator 的：

```bash
--agent-config=...
```

代码仍按 `+` 分隔：

```python
self.config_path = path_to_conf_file.split('+')[0]
self.ckpt_path = path_to_conf_file.split('+')[1]
```

因此理论格式仍是：

```text
配置文件路径+checkpoint路径+保存名
```

示例：

```bash
--agent-config=/home/hqj/Bench2DriveZoo-Bridge-RoadTailBench/adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py+/home/hqj/bench2drive-work/Bench2DriveZoo/ckpts/uniad_base_b2d.pth+UniAD-Base
```

但新版随后会把 `self.ckpt_path` 覆盖为固定路径：

```text
/home/hqj/bench2drive-work/Bench2DriveZoo/ckpts/uniad_base_b2d.pth
```

所以当前版本中，`agent-config` 的第二段 checkpoint 主要用于满足 `split('+')[1]` 不报错，实际加载路径以代码内固定路径为准。

### 3.2 关键环境变量

| 变量 | 作用 |
|---|---|
| `SAVE_PATH` | 若存在，则保存相机图像、BEV、meta 和 `metric_info.json`。 |
| `IS_BENCH2DRIVE` | 若存在，则 `save_name` 使用 `agent-config` 的最后一段，并追加 Bench2Drive 风格的高空 `bev` 传感器。 |
| `ROUTES` | 保存数据时可用于标识路线，当前代码中相关 route stem 逻辑被注释，实际保存目录使用 `save_name`。 |

### 3.3 传感器输入

新版 agent 期望每帧至少存在以下输入：

| id | 类型 | 用途 |
|---|---|---|
| `CAM_FRONT` | RGB camera | 前视图像。 |
| `CAM_FRONT_LEFT` | RGB camera | 前左图像。 |
| `CAM_FRONT_RIGHT` | RGB camera | 前右图像。 |
| `CAM_BACK` | RGB camera | 后视图像。 |
| `CAM_BACK_LEFT` | RGB camera | 后左图像。 |
| `CAM_BACK_RIGHT` | RGB camera | 后右图像。 |
| `IMU` | IMU | compass、加速度、角速度。 |
| `GPS` | GNSS | 当前位置经纬度。 |
| `SPEED` | speedometer | ego 前向速度。 |
| `bev` | RGB camera | 顶视图保存与调试。 |

## 4. 文件作用说明

### 4.1 `team_code/uniad_b2d_agent.py`

这是当前 UniAD 闭环评测的主入口文件。leaderboard 加载它后，会调用：

```python
get_entry_point()
```

并实例化返回的 `UniadAgent` 类。闭环运行中，它负责：

- 声明 CARLA 传感器；
- 读取每帧传感器数据；
- 将 CARLA 数据整理成 UniAD 推理所需格式；
- 调用 UniAD 模型预测规划轨迹；
- 使用 PID 控制器输出车辆转向、油门和刹车；
- 保存调试数据和评测指标；
- 打印主车速度和导航命令日志。

### 4.2 `team_code/uniad_b2d_agent原始文件.py`

这是原始 agent 的保留版本。它的价值是：

- 作为回滚备份；
- 用于和当前固定路径版本对比；
- 在需要恢复命令行 checkpoint 控制时，可参考其加载逻辑。

### 4.3 `uniad_b2d_agent_固定路径.py`

这是仓库根目录下原来的固定路径调试版本，当前已被移除。新的 `team_code/uniad_b2d_agent.py` 与该历史文件内容一致，相当于把固定路径调试版放到了 leaderboard 默认会加载的标准 agent 路径下。

## 5. 示例运行命令

### 5.1 通过 Bench2Drive leaderboard 脚本运行

假设 Bench2Drive 与 Bench2DriveZoo 目录如下：

```text
/home/hqj/Bench2Drive-Bridge-RoadTailBench
/home/hqj/Bench2DriveZoo-Bridge-RoadTailBench
```

启动 CARLA：

```bash
cd /home/hqj/carla
./CarlaUE4.sh -RenderOffScreen -nosound -carla-rpc-port=2000
```

运行 UniAD：

```bash
cd /home/hqj/Bench2Drive-Bridge-RoadTailBench/leaderboard
bash run_uniad.sh
```

### 5.2 直接指定新版 agent 运行

```bash
cd /home/hqj/Bench2Drive-Bridge-RoadTailBench

python3 leaderboard/leaderboard/leaderboard_evaluator.py \
  --routes=leaderboard/data/routes_rtb007.xml \
  --routes-subset=0 \
  --repetitions=1 \
  --track=SENSORS \
  --checkpoint=leaderboard/results.json \
  --agent=/home/hqj/Bench2DriveZoo-Bridge-RoadTailBench/team_code/uniad_b2d_agent.py \
  --agent-config=/home/hqj/Bench2DriveZoo-Bridge-RoadTailBench/adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py+/home/hqj/bench2drive-work/Bench2DriveZoo/ckpts/uniad_base_b2d.pth+UniAD-Base-RTB007 \
  --debug=1 \
  --port=2000 \
  --traffic-manager-port=8000
```

注意：即使上述 `--agent-config` 中写了 checkpoint，当前新版 agent 仍会强制加载：

```text
/home/hqj/bench2drive-work/Bench2DriveZoo/ckpts/uniad_base_b2d.pth
```

### 5.3 保存调试数据

如需保存相机、BEV、控制 meta 和 `metric_info.json`，可设置：

```bash
export SAVE_PATH=/home/hqj/Bench2Drive-Bridge-RoadTailBench/eval_debug
export IS_BENCH2DRIVE=True
```

然后再启动 evaluator。保存目录会使用 `agent-config` 最后一段作为 `save_name`。

## 6. 使用注意事项

1. 当前 `team_code/uniad_b2d_agent.py` 含有本机绝对 checkpoint 路径，迁移环境时必须检查该路径是否存在。
2. `--agent-config` 仍需包含至少一个 `+checkpoint` 段，否则 `path_to_conf_file.split('+')[1]` 会越界。
3. 若设置了 `IS_BENCH2DRIVE`，可能出现两个 id 同为 `bev` 的传感器定义，运行异常时优先检查 sensor id 是否重复。
4. 新版每帧都会打印中文状态日志，长时间评测时终端输出会明显增多。
5. 原始版本已保存在 `team_code/uniad_b2d_agent原始文件.py`，需要恢复原始行为时可将其改回 `team_code/uniad_b2d_agent.py`。
