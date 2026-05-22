# Bench2DriveZoo 闭环复现与 RoadTailBench 替换教程

本文档针对你的目标：在 Ubuntu 服务器上使用已下载的 CARLA 0.9.15 和 UE4.26，复现 Bench2DriveZoo 中自带模型的闭环测试，让算法控制 ego 车并输出指标；后续再把测试环境从 Bench2Drive 地图/路线替换为自己的 RoadTailBench 地图/路线。

## 0. 先回答几个关键问题

### 0.1 这个仓库自带 checkpoint 吗？

仓库本身通常不直接包含大模型权重文件，需要你手动下载到 `ckpts/` 目录。`README.md` 里给出了下载链接：

- `uniad_base_b2d.pth`
- `uniad_tiny_b2d.pth`
- `vad_b2d_base.pth`
- `bevformer_base_b2d.pth`
- `bevformer_tiny_b2d.pth`
- `resnet50-19c8e357.pth`
- `r101_dcn_fcos3d_pretrain.pth`

你不需要自己训练模型，但需要下载对应 checkpoint。

### 0.2 BEVFormer、UniAD、VAD 都能闭环控制车吗？

以当前仓库代码为准：

- UniAD：可以闭环控制车。对应文件是 `team_code/uniad_b2d_agent.py`。
- VAD：可以闭环控制车。对应文件是 `team_code/vad_b2d_agent.py`。
- BEVFormer：当前仓库没有提供 `bevformer_b2d_agent.py` 这类闭环 agent。BEVFormer 在本仓库主要用于 open-loop 感知检测评测，不是直接输出规划轨迹的驾驶 agent。

因此第一阶段闭环复现建议只跑：

```text
UniAD closed-loop
VAD closed-loop
```

BEVFormer 可以先跑 open-loop 检测评测。若你一定要 BEVFormer 闭环控制，需要额外写一个 agent，把 BEVFormer 的感知输出接入一个规划/控制模块；这个仓库目前没有现成实现。

### 0.3 “用他的数据集去闭环测试”是什么意思？

闭环测试不是直接读取离线 `data/bench2drive/v1` 训练数据，而是在 CARLA 里加载 Bench2Drive 评测路线和场景，让 agent 实时开车。

闭环测试主要依赖：

- CARLA 0.9.15。
- Bench2Drive evaluation tools 仓库。
- Bench2Drive 提供的 routes、scenarios、town maps。
- 本仓库里的 `team_code/*_agent.py`。
- 本仓库里的模型 config 和 checkpoint。

离线数据集主要用于训练和 open-loop 评测；闭环评测主要用 CARLA 地图、路线 XML、场景配置和 leaderboard evaluator。

## 1. 推荐目录布局

建议在服务器上使用下面的目录结构，避免 Python import 和软链接混乱：

```text
/data/project/
├── CARLA_0.9.15/
│   ├── CarlaUE4.sh
│   ├── PythonAPI/
│   └── ImportAssets.sh
├── Bench2Drive/
│   ├── leaderboard/
│   ├── scenario_runner/
│   ├── tools/
│   └── ...
└── Bench2DriveZoo/
    ├── adzoo/
    ├── team_code/
    ├── mmcv/
    ├── ckpts/
    └── ...
```

本文后续用下面的环境变量表示这些路径：

```bash
export PROJECT_ROOT=/data/project
export CARLA_ROOT=/data/project/CARLA_0.9.15
export B2D_ROOT=/data/project/Bench2Drive
export B2DZOO_ROOT=/data/project/Bench2DriveZoo
```

如果你的路径不同，统一替换即可。

## 2. Ubuntu 环境配置

### 2.1 创建 conda 环境

本仓库官方文档要求 Python 3.8：

```bash
conda create -n b2d_zoo python=3.8 -y
conda activate b2d_zoo
```

### 2.2 安装 CUDA toolkit 和 PyTorch

官方推荐 CUDA 11.8：

```bash
conda install -c "nvidia/label/cuda-11.8.0" cuda-toolkit -y
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

建议确认：

```bash
python - <<'PY'
import torch
print(torch.__version__)
print(torch.cuda.is_available())
print(torch.version.cuda)
PY
```

期望：

```text
torch.cuda.is_available() = True
CUDA version = 11.8 或兼容版本
```

### 2.3 设置 GCC 和 CUDA 环境变量

官方推荐 GCC 9.4。服务器上可以先检查：

```bash
gcc --version
g++ --version
nvcc --version
```

如果系统默认不是 GCC 9，可以安装并切换：

```bash
sudo apt update
sudo apt install gcc-9 g++-9 -y
export CC=/usr/bin/gcc-9
export CXX=/usr/bin/g++-9
```

设置 CUDA：

```bash
export CUDA_HOME=/usr/local/cuda-11.8
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
```

如果你使用的是 conda 安装的 `cuda-toolkit`，`CUDA_HOME` 可能需要指向 conda 环境：

```bash
export CUDA_HOME=$CONDA_PREFIX
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib:$LD_LIBRARY_PATH
```

### 2.4 安装基础依赖

```bash
pip install ninja packaging
cd $B2DZOO_ROOT
pip install -r requirements.txt
pip install -v -e .
```

`pip install -v -e .` 会编译本仓库里的部分 C++/CUDA 扩展。这里是最容易出问题的步骤。

常见问题：

- `nvcc not found`：检查 `CUDA_HOME` 和 `PATH`。
- GCC 版本过高或过低：切换到 GCC 9。
- CUDA/PyTorch 版本不一致：优先保持 PyTorch cu118 和 CUDA 11.8。
- 编译内存不足：减少并行编译，或在空闲机器上编译。

## 3. CARLA 0.9.15 检查

你已经下载了 CARLA 0.9.15 和 UE4.26。先确认 CARLA 能单独启动。

### 3.1 配置 CARLA Python egg

官方文档使用 Linux egg：

```bash
echo "$CARLA_ROOT/PythonAPI/carla/dist/carla-0.9.15-py3.7-linux-x86_64.egg" \
  >> $CONDA_PREFIX/lib/python3.8/site-packages/carla.pth
```

Python 3.8 通常也能使用这个 py3.7 egg。

测试：

```bash
python - <<'PY'
import carla
print(carla.__file__)
PY
```

### 3.2 单独启动 CARLA

先不要跑 leaderboard，直接启动 CARLA：

```bash
cd $CARLA_ROOT
./CarlaUE4.sh -RenderOffScreen -nosound -carla-rpc-port=20000
```

如果是多 GPU 服务器，可指定图形卡：

```bash
./CarlaUE4.sh -RenderOffScreen -nosound -carla-rpc-port=20000 -graphicsadapter=0
```

常见检查：

```bash
vulkaninfo | head -n 20
nvidia-smi
lsof -i:20000
```

如果 CARLA 一启动就退出，优先排查 Vulkan、显卡驱动、DISPLAY/离屏渲染、端口冲突。

## 4. 准备 Bench2Drive evaluation tools

闭环评测需要另一个仓库：Bench2Drive evaluation tools。它负责 routes、scenarios、CARLA 启动、leaderboard 评测、指标合并。

```bash
cd $PROJECT_ROOT
git clone https://github.com/Thinklab-SJTU/Bench2Drive.git
cd $B2D_ROOT
```

如果你已经有 Bench2Drive，则跳过 clone。

需要确认这些目录存在：

```text
Bench2Drive/
├── leaderboard/
├── scenario_runner/
├── tools/
└── docs/
```

Bench2Drive 官方说明中，闭环工具会自动启动 CARLA；你也可以单独启动 CARLA 做调试。

## 5. 链接 Bench2DriveZoo 到 Bench2Drive

在 Bench2Drive 下软链接本仓库，使 agent 里的 import 能找到 `Bench2DriveZoo`。

```bash
cd $B2D_ROOT
ln -s $B2DZOO_ROOT Bench2DriveZoo
```

把 agent 链接到 leaderboard：

```bash
cd $B2D_ROOT/leaderboard
mkdir -p team_code
ln -s $B2DZOO_ROOT/team_code/* ./team_code/
```

确认：

```bash
ls -l $B2D_ROOT/Bench2DriveZoo
ls -l $B2D_ROOT/leaderboard/team_code
```

如果遇到：

```text
No module named 'Bench2DriveZoo'
```

通常是因为：

- 没有在 `$B2D_ROOT` 下创建 `Bench2DriveZoo` 软链接。
- 当前工作目录不是 `$B2D_ROOT`。
- `PYTHONPATH` 没有包含 `$B2D_ROOT`。

建议在启动脚本里加：

```bash
export PYTHONPATH=$B2D_ROOT:$B2D_ROOT/leaderboard:$B2D_ROOT/scenario_runner:$PYTHONPATH
```

## 6. 下载模型 checkpoint

在 Bench2DriveZoo 目录下创建 `ckpts/`：

```bash
mkdir -p $B2DZOO_ROOT/ckpts
```

从 `README.md` 的 Hugging Face 或百度云链接下载这些文件：

UniAD：

```text
$B2DZOO_ROOT/ckpts/uniad_base_b2d.pth
$B2DZOO_ROOT/ckpts/uniad_tiny_b2d.pth
```

VAD：

```text
$B2DZOO_ROOT/ckpts/vad_b2d_base.pth
```

BEVFormer open-loop：

```text
$B2DZOO_ROOT/ckpts/bevformer_base_b2d.pth
$B2DZOO_ROOT/ckpts/bevformer_tiny_b2d.pth
```

backbone 预训练权重：

```text
$B2DZOO_ROOT/ckpts/resnet50-19c8e357.pth
$B2DZOO_ROOT/ckpts/r101_dcn_fcos3d_pretrain.pth
```

确认：

```bash
ls -lh $B2DZOO_ROOT/ckpts
```

## 7. 闭环测试 UniAD

### 7.1 UniAD agent 参数格式

`team_code/uniad_b2d_agent.py` 的 `setup()` 里用下面方式解析参数：

```python
self.config_path = path_to_conf_file.split('+')[0]
self.ckpt_path = path_to_conf_file.split('+')[1]
if IS_BENCH2DRIVE:
    self.save_name = path_to_conf_file.split('+')[-1]
```

所以 `TEAM_CONFIG` 应该写成：

```text
配置文件路径+checkpoint路径+保存名称
```

例如：

```bash
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/uniad_base_b2d.pth+UniAD-Base"
```

### 7.2 写 UniAD debug 启动脚本

建议先跑 debug 或少量 routes，不要一上来跑完整 220 routes。

在 `$B2D_ROOT/leaderboard/scripts/` 下新建：

```bash
vim $B2D_ROOT/leaderboard/scripts/run_uniad_b2d_debug.sh
```

写入：

```bash
#!/usr/bin/env bash
set -e

export PROJECT_ROOT=/data/project
export CARLA_ROOT=$PROJECT_ROOT/CARLA_0.9.15
export B2D_ROOT=$PROJECT_ROOT/Bench2Drive
export B2DZOO_ROOT=$PROJECT_ROOT/Bench2DriveZoo

export PYTHONPATH=$B2D_ROOT:$B2D_ROOT/leaderboard:$B2D_ROOT/scenario_runner:$PYTHONPATH
export IS_BENCH2DRIVE=True

PORT=20000
TM_PORT=50000
GPU_RANK=0

# Bench2Drive 官方完整评测通常是 leaderboard/data/bench2drive220.xml。
# 初次复现建议先找一个 debug/mini route。如果你的 Bench2Drive 只有 bench2drive220，就先用它，但可配合 TASK_LIST 或 debug 脚本缩小范围。
ROUTES=$B2D_ROOT/leaderboard/data/bench2drive220.xml

TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/uniad_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/uniad_base_b2d.pth+UniAD-Base"

CHECKPOINT_ENDPOINT=$B2D_ROOT/eval_results/uniad_base_progress.json
SAVE_PATH=$B2D_ROOT/eval_v1
PLANNER_TYPE=null

mkdir -p $B2D_ROOT/eval_results
mkdir -p $SAVE_PATH

bash $B2D_ROOT/leaderboard/scripts/run_evaluation.sh \
  $PORT \
  $TM_PORT \
  1 \
  $ROUTES \
  $TEAM_AGENT \
  "$TEAM_CONFIG" \
  $CHECKPOINT_ENDPOINT \
  $SAVE_PATH \
  $PLANNER_TYPE \
  $GPU_RANK
```

赋权并运行：

```bash
chmod +x $B2D_ROOT/leaderboard/scripts/run_uniad_b2d_debug.sh
cd $B2D_ROOT
bash leaderboard/scripts/run_uniad_b2d_debug.sh
```

如果你使用的是 Bench2Drive 官方的 `run_evaluation_debug.sh`，则改里面的：

```bash
TEAM_AGENT=leaderboard/team_code/uniad_b2d_agent.py
TEAM_CONFIG="Bench2DriveZoo/adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py+Bench2DriveZoo/ckpts/uniad_base_b2d.pth+UniAD-Base"
GPU_RANK=0
```

## 8. 闭环测试 VAD

### 8.1 VAD agent 参数格式

`team_code/vad_b2d_agent.py` 和 UniAD 类似：

```python
self.config_path = path_to_conf_file.split('+')[0]
self.ckpt_path = path_to_conf_file.split('+')[1]
```

所以 `TEAM_CONFIG`：

```bash
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/vad_b2d_base.pth+VAD-Base"
```

### 8.2 写 VAD debug 启动脚本

新建：

```bash
vim $B2D_ROOT/leaderboard/scripts/run_vad_b2d_debug.sh
```

写入：

```bash
#!/usr/bin/env bash
set -e

export PROJECT_ROOT=/data/project
export CARLA_ROOT=$PROJECT_ROOT/CARLA_0.9.15
export B2D_ROOT=$PROJECT_ROOT/Bench2Drive
export B2DZOO_ROOT=$PROJECT_ROOT/Bench2DriveZoo

export PYTHONPATH=$B2D_ROOT:$B2D_ROOT/leaderboard:$B2D_ROOT/scenario_runner:$PYTHONPATH
export IS_BENCH2DRIVE=True

PORT=20010
TM_PORT=50010
GPU_RANK=0

ROUTES=$B2D_ROOT/leaderboard/data/bench2drive220.xml
TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/vad_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/vad_b2d_base.pth+VAD-Base"

CHECKPOINT_ENDPOINT=$B2D_ROOT/eval_results/vad_base_progress.json
SAVE_PATH=$B2D_ROOT/eval_v1
PLANNER_TYPE=null

mkdir -p $B2D_ROOT/eval_results
mkdir -p $SAVE_PATH

bash $B2D_ROOT/leaderboard/scripts/run_evaluation.sh \
  $PORT \
  $TM_PORT \
  1 \
  $ROUTES \
  $TEAM_AGENT \
  "$TEAM_CONFIG" \
  $CHECKPOINT_ENDPOINT \
  $SAVE_PATH \
  $PLANNER_TYPE \
  $GPU_RANK
```

运行：

```bash
chmod +x $B2D_ROOT/leaderboard/scripts/run_vad_b2d_debug.sh
cd $B2D_ROOT
bash leaderboard/scripts/run_vad_b2d_debug.sh
```

## 9. 完整 220 routes 评测与指标输出

Bench2Drive 官方说明中，完整评测通常要求 220 条 routes 都有结果。失败或 crash 的 route 也应该被记录，否则整体指标会不准。

### 9.1 单进程跑完整 routes

单进程最简单，但耗时长：

```bash
cd $B2D_ROOT
bash leaderboard/scripts/run_uniad_b2d_debug.sh
```

如果中途 crash，`CHECKPOINT_ENDPOINT` 会记录进度。重新运行同一个脚本时，leaderboard 通常会基于 checkpoint 跳过已完成路线。

### 9.2 多进程/多 GPU

Bench2Drive 官方提供了类似：

```bash
bash leaderboard/scripts/run_evaluation_multi_uniad.sh
```

你需要根据自己机器改：

```bash
TASK_NUM=...
GPU_RANK_LIST=...
TASK_LIST=...
TEAM_AGENT=...
TEAM_CONFIG=...
```

多进程时尤其注意：

- 每个进程使用不同 `PORT` 和 `TM_PORT`。
- 每个进程使用不同 `CHECKPOINT_ENDPOINT`。
- 每个进程最好使用独立输出目录，避免互相覆盖。
- CARLA 启动慢的机器需要加大脚本里的 `sleep`。

### 9.3 合并结果和计算指标

Bench2Drive 官方 README 说明：

```bash
cd $B2D_ROOT
python tools/merge_route_json.py -f your_json_folder/
python tools/ability_benchmark.py -r merge.json
python tools/efficiency_smoothness_benchmark.py -f merge.json -m your_metric_folder/
```

建议目录结构：

```text
Bench2Drive/
├── eval_results/
│   ├── uniad_base_progress.json
│   ├── route_result_*.json
│   └── merge.json
└── eval_v1/
    ├── UniAD-Base/
    │   ├── metric_info.json
    │   ├── rgb_front/
    │   ├── bev/
    │   └── meta/
    └── VAD-Base/
```

注意：

- `team_code/uniad_b2d_agent.py` 和 `team_code/vad_b2d_agent.py` 已经会保存 `metric_info.json`，用于 smoothness 和 efficiency。
- 如果你关闭保存传感器图像以节省空间，要保留 `metric_info.json` 的保存逻辑。
- 完整指标要求 route 数量完整。缺 route 会被当成 0 分或导致统计不准。

## 10. BEVFormer 怎么测试

当前仓库没有 BEVFormer 闭环 agent。你可以先做 open-loop 感知评测：

```bash
cd $B2DZOO_ROOT
./adzoo/bevformer/dist_test.sh \
  ./adzoo/bevformer/configs/bevformer/bevformer_base_b2d.py \
  ./ckpts/bevformer_base_b2d.pth \
  1
```

如果你未来要让 BEVFormer 闭环控制车，需要新增类似：

```text
team_code/bevformer_b2d_agent.py
```

并补齐：

1. `sensors()`：注册多相机、IMU、GNSS、speed。
2. `run_step()`：构造 BEVFormer 所需输入。
3. 感知输出解析：读取 3D detection 结果。
4. 规划模块：基于检测结果、地图和 route command 生成未来轨迹。
5. 控制模块：使用 PID 或 MPC 输出 `carla.VehicleControl`。

也就是说，BEVFormer 本身不是端到端规划模型，不能像 UniAD/VAD 一样直接读取轨迹输出控制车。

## 11. 从 Bench2Drive 地图切换到 RoadTailBench 的总体思路

替换为 RoadTailBench 不是只改一个路径。闭环测试涉及四层：

```text
CARLA 地图资产
→ leaderboard routes/scenarios
→ agent 坐标/传感器/route command 适配
→ 指标统计和结果保存
```

如果后续还要 open-loop 或训练，则还要额外处理：

```text
数据采集
→ 标注格式
→ mmcv/datasets/prepare_B2D.py 适配
→ data/infos/*.pkl
→ config 中 data_root / classes / map / routes 适配
```

你当前目标是闭环测试，所以优先做 CARLA 地图资产和 leaderboard route/scenario。

## 12. RoadTailBench 第一步：让 CARLA 能加载自定义地图

你的 RoadTailBench 地图需要被 CARLA 0.9.15 识别为一个 Town/map。

### 12.1 地图导入

通常有两种方式：

1. 已经打包好的 CARLA package：

```text
RoadTailBench.tar.gz
```

放入：

```bash
$CARLA_ROOT/Import/
```

然后：

```bash
cd $CARLA_ROOT
bash ImportAssets.sh
```

2. UE4.26 工程中自己 cooked/package：

需要保证最终生成的 `.umap`、OpenDRIVE、导航、静态资源能被 CARLA 加载。

### 12.2 检查地图能否加载

启动 CARLA 后，用 Python 测试：

```bash
python - <<'PY'
import carla
client = carla.Client('127.0.0.1', 20000)
client.set_timeout(20.0)
print(client.get_available_maps())
PY
```

你应该能看到类似：

```text
/Game/Carla/Maps/RoadTailBench
```

然后测试加载：

```bash
python - <<'PY'
import carla
client = carla.Client('127.0.0.1', 20000)
client.set_timeout(60.0)
world = client.load_world('RoadTailBench')
print(world.get_map().name)
print(len(world.get_map().get_spawn_points()))
PY
```

如果 spawn points 为 0，leaderboard 可能无法正常生成 ego 初始位置，需要修地图的 OpenDRIVE/spawn points。

## 13. RoadTailBench 第二步：制作 route XML

Bench2Drive/leaderboard 通过 route XML 指定：

- 使用哪个 town/map。
- ego 车从哪里出发。
- 经过哪些 waypoints。
- route 对应哪些 scenario。

你需要在：

```text
$B2D_ROOT/leaderboard/data/
```

新增例如：

```text
roadtailbench_eval.xml
```

典型 route XML 结构类似：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<routes>
  <route id="0" town="RoadTailBench">
    <weather id="ClearNoon"/>
    <waypoints>
      <position x="0.0" y="0.0" z="0.0" yaw="0.0"/>
      <position x="20.0" y="0.0" z="0.0" yaw="0.0"/>
      <position x="60.0" y="10.0" z="0.0" yaw="10.0"/>
    </waypoints>
    <scenarios>
    </scenarios>
  </route>
</routes>
```

实际字段要以你当前 Bench2Drive 的 route XML 格式为准。建议复制：

```text
$B2D_ROOT/leaderboard/data/bench2drive220.xml
```

然后只改：

- `town`
- route id
- waypoint 坐标
- scenario 配置

### 13.1 获取 RoadTailBench waypoint 坐标

可以写脚本从 CARLA map 获取 spawn points：

```bash
python - <<'PY'
import carla
client = carla.Client('127.0.0.1', 20000)
client.set_timeout(30.0)
world = client.load_world('RoadTailBench')
spawn_points = world.get_map().get_spawn_points()
for i, sp in enumerate(spawn_points[:50]):
    loc = sp.location
    rot = sp.rotation
    print(i, loc.x, loc.y, loc.z, rot.yaw)
PY
```

根据输出挑选起点、中间点、终点，填入 route XML。

## 14. RoadTailBench 第三步：制作 scenario 配置

如果你只是先验证 agent 能在 RoadTailBench 上开起来，可以先做无交互场景：

```xml
<scenarios>
</scenarios>
```

如果要加入 cut-in、行人、障碍物等交互场景，需要参考 Bench2Drive 原有 scenario 配置。一般涉及：

- scenario type。
- trigger point。
- actor transform。
- actor blueprint。
- 行为参数。

建议流程：

1. 先无 scenario 跑通一条 route。
2. 再加入一个最简单的静态障碍物或车辆。
3. 最后批量生成 RoadTailBench 的完整场景集。

## 15. RoadTailBench 第四步：改启动脚本 routes

把原来的：

```bash
ROUTES=$B2D_ROOT/leaderboard/data/bench2drive220.xml
```

改成：

```bash
ROUTES=$B2D_ROOT/leaderboard/data/roadtailbench_eval.xml
```

其他参数可以先保持不变：

```bash
TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/uniad_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/uniad_base_b2d.pth+UniAD-RoadTailBench"
```

或者 VAD：

```bash
TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/vad_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/vad_b2d_base.pth+VAD-RoadTailBench"
```

## 16. RoadTailBench 第五步：检查 agent 坐标和 route command

UniAD/VAD agent 中有这些关键坐标逻辑：

- `gps_to_location()`
- `_init()` 里根据 global plan 估计 `lat_ref/lon_ref`
- `RoutePlanner(4.0, 50.0, lat_ref=..., lon_ref=...)`
- `command_near`
- `local_command_xy`
- `can_bus`
- `lidar2global`

如果 RoadTailBench 的 OpenDRIVE/GPS geo reference 正确，通常不需要改。

如果出现车辆目标点方向明显错误、local command 跳变、路线距离异常，需要检查：

1. RoadTailBench OpenDRIVE 是否包含正确 geoReference。
2. CARLA 返回的 GNSS 和 world coordinate 是否可互相转换。
3. `team_code/planner.py` 中的 `gps_to_location()` 是否适合该地图。
4. `_global_plan` 和 `_global_plan_world_coord` 是否正常。

必要时可以在 `team_code/uniad_b2d_agent.py` 或 `team_code/vad_b2d_agent.py` 的 `run_step()` 里临时打印：

```python
print('pos', tick_data['pos'], 'near', tick_data['command_near_xy'], 'cmd', tick_data['command_near'], flush=True)
print('local_command_xy', local_command_xy, flush=True)
```

调试完成后再删掉或用环境变量控制打印。

## 17. RoadTailBench 第六步：指标统计适配

Bench2Drive 官方指标包括：

- Driving Score。
- Success Rate。
- Multi-ability results。
- Efficiency。
- Smoothness。

如果你的 RoadTailBench route 数量不是 220，不能直接照搬完整 Bench2Drive 220 routes 的假设。官方工具 `merge_route_json.py` 可能默认按 220 routes 处理。

你需要检查并可能修改：

```text
$B2D_ROOT/tools/merge_route_json.py
$B2D_ROOT/tools/ability_benchmark.py
$B2D_ROOT/tools/efficiency_smoothness_benchmark.py
```

重点检查：

- 是否硬编码 route 总数 220。
- 是否按 Bench2Drive ability/scenario 类型统计。
- 是否要求特定 route id。
- 是否要求特定 scenario name。

如果 RoadTailBench 只是内部测试，建议先输出：

```text
每条 route 是否完成
每条 route 的 infractions
平均 route completion
平均 driving score
平均速度
急加速/急刹/急转等 smoothness
```

之后再设计 RoadTailBench 自己的 ability taxonomy。

## 18. 推荐复现路线

### 阶段 A：只验证环境

1. `python -c "import torch; print(torch.cuda.is_available())"`
2. `python -c "import carla; print(carla.__file__)"`
3. 单独启动 CARLA。
4. `pip install -v -e .` 编译 Bench2DriveZoo。

### 阶段 B：跑通 UniAD 单 route

1. 下载 `uniad_base_b2d.pth`。
2. 链接 `Bench2DriveZoo` 到 `Bench2Drive/`。
3. 链接 `team_code`。
4. 运行 `run_uniad_b2d_debug.sh`。
5. 确认输出 JSON 和 `metric_info.json`。

### 阶段 C：跑通 VAD 单 route

1. 下载 `vad_b2d_base.pth`。
2. 运行 `run_vad_b2d_debug.sh`。
3. 对比 UniAD 和 VAD 的路线完成情况。

### 阶段 D：完整 Bench2Drive routes

1. 使用官方 `bench2drive220.xml`。
2. 使用多进程脚本。
3. 合并 JSON。
4. 计算 driving score、success rate、ability、smoothness、efficiency。

### 阶段 E：RoadTailBench

1. CARLA 能加载 RoadTailBench。
2. 制作 1 条无 scenario route。
3. 跑 UniAD/VAD。
4. 检查 route planner 和控制输出。
5. 增加 scenarios。
6. 修改指标脚本中的 route 数量和 ability 分类。

## 19. 常见问题排查

### 19.1 No module named Bench2DriveZoo

处理：

```bash
cd $B2D_ROOT
ln -s $B2DZOO_ROOT Bench2DriveZoo
export PYTHONPATH=$B2D_ROOT:$PYTHONPATH
```

### 19.2 No module named leaderboard 或 scenario_runner

处理：

```bash
export PYTHONPATH=$B2D_ROOT:$B2D_ROOT/leaderboard:$B2D_ROOT/scenario_runner:$PYTHONPATH
```

### 19.3 CARLA 端口冲突

检查：

```bash
lsof -i:20000
lsof -i:50000
```

换端口：

```bash
PORT=20100
TM_PORT=50100
```

### 19.4 CARLA 残留进程

使用 Bench2Drive 工具：

```bash
bash $B2D_ROOT/tools/clean_carla.sh
```

或手动：

```bash
pkill -f CarlaUE4
pkill -f leaderboard_evaluator
```

### 19.5 CUDA out of memory

处理：

- 换更小模型，如 UniAD-Tiny。
- 单进程单 GPU。
- 确保没有其他任务占 GPU。
- 减少同时运行的 CARLA/agent 数量。

### 19.6 路线开始后车辆不动

检查：

- checkpoint 是否正确加载。
- `TEAM_CONFIG` 是否是 `config+ckpt+name`。
- sensor 数据是否正常。
- `command_near` 是否异常。
- 模型输出轨迹是否全 0 或 NaN。
- PID 输入速度和目标点是否正常。

### 19.7 RoadTailBench 地图加载但 route 失败

检查：

- route XML 的 `town` 名称是否与 CARLA map 名称匹配。
- waypoint 是否在道路上。
- OpenDRIVE 是否有 lane 信息。
- spawn point 是否可用。
- scenario trigger point 是否合理。

## 20. 需要改代码的位置总结

### 20.1 复现 Bench2Drive 官方闭环

通常不需要改模型代码，只需要：

- 新增/修改启动脚本：
  - `$B2D_ROOT/leaderboard/scripts/run_uniad_b2d_debug.sh`
  - `$B2D_ROOT/leaderboard/scripts/run_vad_b2d_debug.sh`
- 创建软链接：
  - `$B2D_ROOT/Bench2DriveZoo -> $B2DZOO_ROOT`
  - `$B2D_ROOT/leaderboard/team_code/* -> $B2DZOO_ROOT/team_code/*`
- 下载 checkpoint 到：
  - `$B2DZOO_ROOT/ckpts/`

### 20.2 替换 RoadTailBench 地图

最少需要改/新增：

- CARLA 地图资产：
  - `$CARLA_ROOT/Import/`
  - `ImportAssets.sh`
- leaderboard route：
  - `$B2D_ROOT/leaderboard/data/roadtailbench_eval.xml`
- 启动脚本：
  - `ROUTES=$B2D_ROOT/leaderboard/data/roadtailbench_eval.xml`
- 指标脚本：
  - `$B2D_ROOT/tools/merge_route_json.py`
  - `$B2D_ROOT/tools/ability_benchmark.py`
  - `$B2D_ROOT/tools/efficiency_smoothness_benchmark.py`

可能需要改：

- `team_code/planner.py`
- `team_code/uniad_b2d_agent.py`
- `team_code/vad_b2d_agent.py`

只有当 RoadTailBench 的 GPS/world 坐标转换、route command 或传感器配置与 Bench2Drive 不兼容时，才需要改 agent。

## 21. 最小可执行清单

完成下面清单后，就能开始第一次 UniAD/VAD 闭环复现：

```text
[ ] Ubuntu 服务器 GPU 驱动正常
[ ] CARLA 0.9.15 能单独启动
[ ] conda b2d_zoo Python 3.8
[ ] PyTorch cu118 可用
[ ] carla Python egg 可 import
[ ] Bench2DriveZoo pip install -v -e . 成功
[ ] Bench2Drive evaluation tools 已准备
[ ] Bench2Drive/Bench2DriveZoo 软链接存在
[ ] leaderboard/team_code 软链接存在
[ ] ckpts/uniad_base_b2d.pth 已下载
[ ] ckpts/vad_b2d_base.pth 已下载
[ ] run_uniad_b2d_debug.sh 已配置
[ ] run_vad_b2d_debug.sh 已配置
[ ] 单 route/debug eval 能输出 JSON
[ ] merge_route_json.py 能合并结果
```

