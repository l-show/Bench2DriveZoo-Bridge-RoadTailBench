# tcp/admlp 分支代码结构与模型说明

本文档总结 `tcp/admlp` 分支的用途、TCP/ADMLP 分别是什么、与 `uniad/vad` 分支的关系，以及训练、开环评测、闭环 CARLA 评测的代码链路。

## 1. 这个分支是干什么的

`tcp/admlp` 分支是 Bench2DriveZoo 中面向 **TCP** 和 **ADMLP** 两个轻量驾驶 baseline 的分支。它和 `uniad/vad` 分支不是同一套代码体系：

- `uniad/vad` 分支包含 BEVFormer、UniAD、VAD，以及整合后的 `mmcv/` 大框架。
- `tcp/admlp` 分支只包含 TCP、ADMLP 两个模型子目录、两个 CARLA agent、数据预处理脚本和少量工具。

该分支支持：

- 数据预处理：把 Bench2Drive 采集数据转成 `.npy` 训练文件。
- 训练：使用 PyTorch Lightning 训练 TCP/ADMLP。
- Open-loop 评测：在 `.npy` 验证集上计算轨迹 L2。
- Closed-loop 评测：在 Bench2Drive/CARLA 中加载 agent 控制 ego 车。

## 2. TCP 和 ADMLP 是什么

### 2.1 TCP

TCP 是一个轻量端到端驾驶模型，输入包含：

- 前向多相机拼接图像。
- 当前速度。
- route target point。
- route command one-hot。

输出包含：

- 未来轨迹 `pred_wp`。
- 离散控制动作 `action_index`。
- 速度预测 `pred_speed`。
- value/feature 分支，用于模仿教师或辅助监督。

闭环时 TCP 可以有三种控制模式：

- `only_traj`：只使用预测轨迹 + PID 控制。
- `only_ctrl`：只使用模型预测的离散控制动作。
- `merge_ctrl_traj`：融合轨迹 PID 和离散控制。

代码入口：

```text
TCP/model.py
team_code/tcp_b2d_agent.py
```

### 2.2 ADMLP

ADMLP 是一个更轻量的 MLP 轨迹预测模型。它基本不依赖图像感知，主要使用历史状态和导航命令：

- 历史轨迹点。
- 历史朝向。
- 当前速度。
- 当前加速度。
- route command one-hot。

输出：

- 未来 6 帧轨迹，每帧包含 `(x, y, theta)`。

闭环时 ADMLP 使用预测轨迹 + PID 输出控制量。

代码入口：

```text
ADMLP/model.py
team_code/admlp_b2d_agent.py
```

### 2.3 它们是不是模型

是。TCP 和 ADMLP 都是自动驾驶模型/驾驶策略 baseline。

但它们和 UniAD/VAD 这类大模型不同：

- TCP：图像 + 状态输入，输出轨迹和控制，是轻量端到端 baseline。
- ADMLP：状态/轨迹输入，输出未来轨迹，是更简单的 MLP baseline。
- UniAD/VAD：包含感知、地图、运动预测、规划等更复杂模块。

## 3. 分支目录结构

```text
tcp/admlp
├── README.md
├── requirement.txt
├── TCP/
│   ├── config.py
│   ├── data.py
│   ├── model.py
│   ├── resnet.py
│   ├── train.py
│   ├── train.sh
│   ├── test.py
│   └── TCP-only-traj.json
├── ADMLP/
│   ├── config.py
│   ├── data.py
│   ├── model.py
│   ├── train.py
│   ├── train.sh
│   └── test.py
├── team_code/
│   ├── tcp_b2d_agent.py
│   ├── admlp_b2d_agent.py
│   └── planner.py
└── tools/
    ├── gen_tcp_data.py
    └── gen_admlp_data.py
```

## 4. 依赖关系

`requirement.txt` 内容很少：

```text
lightning
pillow
torchvision
imgaug
matplotlib
```

实际运行还需要：

- PyTorch。
- NumPy。
- tqdm。
- OpenCV。
- CARLA Python API。
- Bench2Drive leaderboard。
- scipy。

因为 `team_code/*_agent.py` 中有：

```python
import carla
from leaderboard.autoagents import autonomous_agent
from scipy.optimize import fsolve
```

所以闭环评测依旧需要 Bench2Drive/CARLA 环境。

## 5. TCP 代码关系

### 5.1 配置

文件：

```text
TCP/config.py
```

关键参数：

```python
seq_len = 1
pred_len = 4
train_data = './tcp_bench2drive-train.npy'
val_data = './tcp_bench2drive-val.npy'
input_resolution = 256
lr = 1e-4
```

控制器参数：

```python
turn_KP / turn_KI / turn_KD
speed_KP / speed_KI / speed_KD
max_throttle
brake_speed
brake_ratio
clip_delta
```

注意：

- `root_dir_all = "bench2drive-base/"` 需要按你的数据路径修改。
- `train_data` 和 `val_data` 是 `tools/gen_tcp_data.py` 生成的 `.npy`。

### 5.2 数据集

文件：

```text
TCP/data.py
```

`CARLA_Data` 会读取 `tcp_bench2drive-train.npy` 或 `tcp_bench2drive-val.npy`，并从其中取：

- `front_img`
- `input_x`
- `input_y`
- `input_theta`
- `speed`
- `future_x`
- `future_y`
- `future_theta`
- `feature`
- `value`
- `action`
- `action_index`
- `target_command`

图像处理逻辑：

1. 读取 `rgb_front`。
2. 自动替换路径读取 `rgb_front_left` 和 `rgb_front_right`。
3. 裁剪三路图像。
4. 横向拼接。
5. resize 到 `(256, 900)`。
6. ToTensor + ImageNet Normalize。

### 5.3 模型

文件：

```text
TCP/model.py
```

核心结构：

- `resnet34(pretrained=True)` 作为视觉 backbone。
- `measurements` MLP 编码速度、target point、command。
- `join_traj` 分支预测未来轨迹。
- `join_ctrl` 分支预测离散动作。
- `decoder_traj`：GRUCell 自回归生成未来 waypoint。
- `decoder_ctrl`：GRUCell 生成未来控制相关特征。
- `action_head`：39 类离散动作分类。

输入：

```python
img          # 拼接后的前视图像
state        # speed + target_point + command one-hot
target_point # route planner 目标点
```

输出：

```python
outputs['pred_speed']
outputs['pred_wp']
outputs['action_index']
outputs['future_action_index']
outputs['pred_value_traj']
outputs['pred_value_ctrl']
```

### 5.4 控制输出

TCP 有两条控制路径：

1. 离散控制路径：

```python
process_action(pred, command, speed, target_point)
```

它从 `action_index` 选一个离散动作，映射到：

```text
throttle, steer, brake, reverse
```

2. 轨迹 PID 路径：

```python
control_pid(pred['pred_wp'], velocity, target_point)
```

它根据未来 waypoint 估计期望速度和转向，输出：

```text
steer, throttle, brake
```

## 6. ADMLP 代码关系

### 6.1 配置

文件：

```text
ADMLP/config.py
```

关键参数：

```python
train_data = 'admlp_bench2drive-train.npy'
val_data = 'admlp_bench2drive-val.npy'
```

同时包含和 TCP 类似的 PID 参数。

### 6.2 数据集

文件：

```text
ADMLP/data.py
```

读取：

- `input_x`
- `input_y`
- `input_theta`
- `input_speed`
- `input_speed_acc`
- `input_command`
- `future_x`
- `future_y`
- `future_theta`

构造输入：

```python
data['input'] = torch.cat((
    hist_waypoints,
    hist_thetas,
    speed,
    speed_acc,
    command
))
```

输入维度为 22，对应 `ADMLP/model.py`：

```python
nn.Linear(22, 512)
```

### 6.3 模型

文件：

```text
ADMLP/model.py
```

结构非常简单：

```python
self.plan_head = nn.Sequential(
    nn.Linear(22, 512),
    nn.ReLU(inplace=True),
    nn.Linear(512, 512),
    nn.ReLU(inplace=True),
    nn.Linear(512, 6 * 3)
)
```

输出 reshape 为：

```python
bs, 6, 3
```

含义是未来 6 个点，每个点为：

```text
x, y, theta
```

闭环控制只使用轨迹点，再通过 PID 转成车辆控制。

## 7. 数据预处理

### 7.1 TCP 数据生成

文件：

```text
tools/gen_tcp_data.py
```

原 README 命令：

```bash
python tools/gen_tcp_data.py
```

但代码里有硬编码：

```python
path = 'YOUR_PATH'
```

你必须改成 Bench2Drive 采集数据目录，例如：

```python
path = '/data/bench2drive/v1'
```

它会读取每条 route 下：

```text
anno/*.json.gz
expert_assessment/*.npz
camera/rgb_front/*.jpg
camera/rgb_front_left/*.jpg
camera/rgb_front_right/*.jpg
```

输出：

```text
tcp_bench2drive-train.npy
tcp_bench2drive-val.npy
```

注意：

- `TRAIN = True` 时生成 train。
- `TRAIN = False` 时生成 val。
- 需要分别跑两次，或手动改代码生成两个文件。
- TCP 依赖 `expert_assessment/*.npz`，里面提供 value、feature、action_index 等教师信息。

### 7.2 ADMLP 数据生成

文件：

```text
tools/gen_admlp_data.py
```

同样需要改：

```python
path = 'YOUR_PATH'
```

它读取：

```text
anno/*.json.gz
```

输出：

```text
admlp_bench2drive-train.npy
admlp_bench2drive-val.npy
```

ADMLP 不依赖图像，也不依赖 `expert_assessment`，所以比 TCP 更轻。

## 8. 训练流程

### 8.1 TCP 训练

README 命令：

```bash
export PYTHONPATH=$PYTHONPATH:PATH_TO_TCP
python TCP/train.py --gpus NUM_OF_GPUS
```

或：

```bash
bash TCP/train.sh
```

但 `TCP/train.sh` 里有占位符：

```bash
export PYTHONPATH=$PYTHONPATH:YOUR_PATH/Bench2DriveZoo/TCP
```

需要改成实际路径。

训练调用链：

```text
TCP/train.py
→ GlobalConfig
→ TCP.data.CARLA_Data
→ TCP.model.TCP
→ TCP_planner.training_step
→ Lightning Trainer
→ log/TCP/best_*.ckpt
```

### 8.2 ADMLP 训练

README 命令：

```bash
export PYTHONPATH=$PYTHONPATH:PATH_TO_ADMLP
python ADMLP/train.py --gpus NUM_OF_GPUS
```

或：

```bash
bash ADMLP/train.sh
```

注意：当前 `ADMLP/train.py` 中有一个明显问题：

```python
self.model = ADMLP()
```

但 `ADMLP/model.py` 中定义是：

```python
class ADMLP(nn.Module):
    def __init__(self, config):
```

因此训练脚本应该改为：

```python
config = GlobalConfig()
self.model = ADMLP(config)
```

否则直接运行可能报缺少 `config` 参数。

## 9. Open-loop 评测

### 9.1 TCP

文件：

```text
TCP/test.py
```

原代码中 checkpoint 路径是占位符：

```python
ckpt = torch.load('CKPY_PATH', map_location="cuda")
```

需要改成实际 checkpoint，例如：

```python
ckpt = torch.load('tcp_b2d.ckpt', map_location="cuda")
```

评测指标：

- 0.5s L2。
- 1.0s L2。
- 1.5s L2。
- 2.0s L2。
- 最后打印平均值。

### 9.2 ADMLP

文件：

```text
ADMLP/test.py
```

默认 checkpoint：

```python
admlp_b2d.ckpt
```

如果文件名不同，需要修改。

评测同样计算多个时间点 L2。

## 10. Closed-loop CARLA 评测

### 10.1 TCP agent

文件：

```text
team_code/tcp_b2d_agent.py
```

入口：

```python
def get_entry_point():
    return 'TCPAgent'
```

`setup()` 解析 checkpoint：

```python
if IS_BENCH2DRIVE:
    self.save_name = path_to_conf_file.split('+')[-1]
    self.config_path = path_to_conf_file.split('+')[0]
else:
    self.config_path = path_to_conf_file
```

注意：TCP/ADMLP 分支的 agent 和 UniAD/VAD 不同，`path_to_conf_file` 不是 `config+ckpt+name`，而是：

```text
checkpoint路径+保存名
```

示例：

```bash
TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/tcp_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/tcp_b2d.ckpt+TCP-only-traj"
export PLANNER_TYPE=only_traj
```

TCP agent 需要设置：

```bash
export PLANNER_TYPE=only_traj
```

可选：

```bash
export PLANNER_TYPE=only_ctrl
export PLANNER_TYPE=merge_ctrl_traj
```

当前 `README.md` 中提供的评测 JSON 是：

```text
TCP/TCP-only-traj.json
```

说明官方至少报告过 `only_traj` 模式。

### 10.2 ADMLP agent

文件：

```text
team_code/admlp_b2d_agent.py
```

入口：

```python
def get_entry_point():
    return 'ADMLPAgent'
```

同样使用：

```text
checkpoint路径+保存名
```

示例：

```bash
TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/admlp_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/admlp_b2d.ckpt+ADMLP"
```

ADMLP agent 中虽然读取了 `PLANNER_TYPE`，但实际只使用 `only_traj` 风格的预测轨迹 + PID。

### 10.3 传感器差异

TCP agent 注册：

- `CAM_FRONT`
- `CAM_FRONT_LEFT`
- `CAM_FRONT_RIGHT`
- `IMU`
- `GPS`
- `SPEED`
- 可选 `bev`

ADMLP agent 注册：

- `CAM_FRONT`
- `IMU`
- `GPS`
- `SPEED`
- 可选 `bev`

注意：ADMLP 虽然注册了前视相机，但模型输入没有使用图像；相机主要用于保存可视化。

## 11. 与 uniad/vad 分支的区别

| 对比项 | tcp/admlp 分支 | uniad/vad 分支 |
|---|---|---|
| 模型 | TCP、ADMLP | BEVFormer、UniAD、VAD |
| 框架 | 轻量 PyTorch/PyTorch Lightning | 本地整合 mmcv/mmdet/mmdet3d 风格框架 |
| 数据格式 | `.npy` 聚合文件 | `data/infos/*.pkl` |
| 闭环 agent | TCP/ADMLP agent | UniAD/VAD agent |
| BEVFormer | 无 | 有 open-loop 感知模型 |
| 模型复杂度 | 轻量 baseline | 大型端到端或感知模型 |
| 是否直接闭环 | TCP/ADMLP 都可闭环 | UniAD/VAD 可闭环，BEVFormer 无现成闭环 agent |

## 12. Checkpoint 下载

README 中给出：

TCP：

- Hugging Face: `https://huggingface.co/rethinklab/Bench2DriveZoo/tree/main`
- 百度云: `https://pan.baidu.com/s/1CgYscY2esIJLRepkO3FBvQ?pwd=1234`

ADMLP：

- Hugging Face: `https://huggingface.co/rethinklab/Bench2DriveZoo/tree/main`
- 百度云: `https://pan.baidu.com/s/1RefJxk0B4kYcnf63Vi-ISA?pwd=1234`

注意：

- README 没写具体文件名，但代码默认常见命名可能是 `tcp_b2d.ckpt`、`admlp_b2d.ckpt`。
- `TCP/test.py` 中 checkpoint 路径是 `CKPY_PATH`，必须手动改。
- `ADMLP/test.py` 默认使用 `admlp_b2d.ckpt`。
- 闭环 agent 中直接把 `TEAM_CONFIG` 第一段当 checkpoint 路径。

## 13. 复现建议

如果只是想理解和跑通：

1. 先下载 TCP/ADMLP checkpoint。
2. 只跑 closed-loop，不训练。
3. TCP 使用 `PLANNER_TYPE=only_traj`。
4. ADMLP 直接用 `admlp_b2d_agent.py`。
5. 保存 `metric_info.json`，用于 smoothness/efficiency。

如果要训练：

1. 先用 `tools/gen_admlp_data.py` 跑 ADMLP 数据，因为它不依赖图像和 expert feature。
2. 修复 `ADMLP/train.py` 中 `ADMLP(config)` 的问题。
3. 再尝试 TCP，因为 TCP 数据依赖图像和 `expert_assessment`。

## 14. 已发现的代码注意事项

### 14.1 README 软链接命令可能有误

README 中：

```bash
cd Bench2Drive/
ln -s Bench2DriveZoo/team_code/*  ./ # link entire repo to Bench2Drive
```

这行注释和命令不一致。更合理的是：

```bash
cd Bench2Drive
ln -s /path/to/Bench2DriveZoo ./Bench2DriveZoo

cd Bench2Drive/leaderboard
mkdir -p team_code
ln -s /path/to/Bench2DriveZoo/team_code/* ./team_code/
```

### 14.2 ADMLP/train.py 构造模型少传 config

当前：

```python
self.model = ADMLP()
```

应改：

```python
self.model = ADMLP(GlobalConfig())
```

或把 config 传进 `ADMLP_planner`。

### 14.3 数据生成脚本有硬编码 YOUR_PATH

需要改：

```python
path = 'YOUR_PATH'
```

否则无法找到数据。

### 14.4 TCP/test.py 有 CKPY_PATH 占位符

需要改：

```python
ckpt = torch.load('CKPY_PATH', map_location="cuda")
```

为实际 checkpoint 路径。

### 14.5 TCP 闭环必须设置 PLANNER_TYPE

`team_code/tcp_b2d_agent.py` 中：

```python
if not PLANNER_TYPE:
    raise 'please set PLANNER_TYPE'
```

运行前必须设置：

```bash
export PLANNER_TYPE=only_traj
```

## 15. 最小闭环运行参数示例

TCP：

```bash
export IS_BENCH2DRIVE=True
export SAVE_PATH=$B2D_ROOT/eval_tcp
export PLANNER_TYPE=only_traj
export PYTHONPATH=$B2D_ROOT:$B2D_ROOT/leaderboard:$B2D_ROOT/scenario_runner:$B2DZOO_ROOT:$PYTHONPATH

TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/tcp_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/tcp_b2d.ckpt+TCP-only-traj"
```

ADMLP：

```bash
export IS_BENCH2DRIVE=True
export SAVE_PATH=$B2D_ROOT/eval_admlp
export PYTHONPATH=$B2D_ROOT:$B2D_ROOT/leaderboard:$B2D_ROOT/scenario_runner:$B2DZOO_ROOT:$PYTHONPATH

TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/admlp_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/admlp_b2d.ckpt+ADMLP"
```

具体 `run_evaluation.sh` 参数以你本地 Bench2Drive evaluation tools 为准。

