# Bench2DriveZoo 代码结构与运行流程中文文档

本文档面向第一次接触本项目的读者，重点说明仓库文件之间的关系、重要文件功能、UniAD/VAD/BEVFormer 三类模型的运行方式，以及项目运行环境对 Windows 和 Ubuntu 的支持情况。

## 1. 项目定位

Bench2DriveZoo 是将 BEVFormer、UniAD、VAD 三个自动驾驶模型适配到 Bench2Drive 数据集和评测协议上的项目。它覆盖两类主要流程：

- Open-loop 离线训练/评测：读取 Bench2Drive 离线数据，训练模型或计算验证指标。
- Closed-loop CARLA 闭环评测：把模型封装为 CARLA leaderboard agent，在仿真中实时接收传感器输入并输出车辆控制量。

项目入口文档：

- `README.md`：项目介绍、模型结果、预训练权重链接、文档入口。
- `docs/INSTALL.md`：环境安装。
- `docs/DATA_PREP.md`：Bench2Drive 数据准备。
- `docs/TRAIN_EVAL.md`：三类模型的训练和 open-loop 评测命令。
- `docs/EVAL_IN_CARLA.md`：闭环 CARLA 评测接入方式。

## 2. 总体目录关系

```text
Bench2DriveZoo
├── README.md
├── requirements.txt
├── setup.py
├── docs/
├── adzoo/
│   ├── bevformer/
│   ├── uniad/
│   └── vad/
├── mmcv/
├── team_code/
├── data/
├── analysis/
└── assets/
```

可以按下面的层次理解整个仓库：

```text
docs/                 说明如何安装、准备数据、训练、评测
adzoo/*/configs/      定义模型结构、数据集、pipeline、优化器、训练策略
adzoo/*/train.py      open-loop 训练入口
adzoo/*/test.py       open-loop 评测入口
mmcv/                 合并后的底层框架、模型、数据集、算子、runner
team_code/*_agent.py  closed-loop CARLA agent 入口
data/                 数据 split、motion anchor、后续生成的 infos
analysis/             评测结果 JSON、失败案例 GIF 和分析
```

一句话概括：

- `docs/` 说明“怎么跑”。
- `adzoo/*/configs/` 说明“跑什么模型、用什么参数”。
- `adzoo/*/train.py` 和 `adzoo/*/test.py` 负责训练/评测流程。
- `mmcv/` 是模型、数据集、runner、算子等底层实现。
- `team_code/` 是 CARLA 闭环评测的驾驶 agent。

## 3. 重要文件和目录功能

### 3.1 项目级文件

- `README.md`：项目主页说明，包含 Bench2DriveZoo 定位、模型结果、预训练模型下载链接和文档索引。
- `requirements.txt`：Python 依赖列表。
- `setup.py`：本仓库安装入口，执行 `pip install -v -e .` 时使用。该文件也负责本地扩展、包信息和相关模块注册。
- `LICENSE`：开源许可证。

### 3.2 docs 文档目录

- `docs/INSTALL.md`：安装环境，明确要求 Python 3.8、CUDA 11.8、PyTorch cu118，并推荐 GCC 9.4。
- `docs/DATA_PREP.md`：说明 Bench2Drive 原始数据目录结构，以及如何运行 `mmcv/datasets/prepare_B2D.py` 生成训练/评测所需的 `b2d_infos_train.pkl`、`b2d_infos_val.pkl`、`b2d_map_infos.pkl`。
- `docs/TRAIN_EVAL.md`：BEVFormer、UniAD、VAD 的训练和 open-loop 评测命令。
- `docs/EVAL_IN_CARLA.md`：将本仓库链接到 Bench2Drive leaderboard，并进行 CARLA 闭环评测。
- `docs/CONVERT_GUIDE.md`：将基于 nuScenes 或其他数据集的代码迁移到 Bench2Drive 时需要注意的事项。

### 3.3 adzoo 模型脚本层

`adzoo/` 可以理解为不同模型的子工程集合：

```text
adzoo/
├── bevformer/
├── uniad/
└── vad/
```

每个模型目录通常包含：

- `configs/`：模型结构、数据集、训练策略、pipeline 配置。
- `train.py`：训练入口。
- `test.py`：open-loop 测试入口。
- `dist_train.sh` / `dist_test.sh` 或类似脚本：分布式训练/评测命令包装。
- `analysis_tools/`：日志分析、可视化、benchmark 等工具。
- `data_converter/`：数据转换工具。
- `misc/`：打印配置、浏览数据集、结果可视化等辅助工具。

### 3.4 mmcv 底层框架层

本仓库没有直接依赖原始独立的 mmcv/mmdet/mmdet3d/mmseg 包，而是把大量相关能力合并进本地 `mmcv/` 目录。它是三类模型真正执行的底层实现。

重要子目录：

- `mmcv/models/`：模型主体、backbone、neck、head、transformer、loss 等。
- `mmcv/models/detectors/`：检测器/端到端模型入口。
- `mmcv/models/dense_heads/`：BEVFormerHead、VADHead、tracking head、motion head、planning head、occ head 等。
- `mmcv/models/modules/`：transformer、attention、encoder、decoder 等模块。
- `mmcv/datasets/`：Bench2Drive 和 nuScenes 风格数据集、pipeline、评测工具。
- `mmcv/runner/`：训练 runner、hook、checkpoint、optimizer hook。
- `mmcv/ops/` 和 `mmcv/layers/`：C++/CUDA 自定义算子。
- `mmcv/core/`：bbox、evaluation、visualization、post-processing 等通用核心逻辑。

关键模型文件：

- `mmcv/models/detectors/bevformer.py`：BEVFormer detector。
- `mmcv/models/detectors/bevformerV2.py`：BEVFormerV2 detector。
- `mmcv/models/detectors/bevformer_fp16.py`：BEVFormer fp16 版本。
- `mmcv/models/detectors/uniad_e2e.py`：UniAD end-to-end 总体模型。
- `mmcv/models/detectors/uniad_track.py`：UniAD tracking 基类和 track 流程。
- `mmcv/models/detectors/VAD.py`：VAD detector。

关键数据集文件：

- `mmcv/datasets/B2D_dataset.py`：BEVFormer 使用的 Bench2Drive 数据集，注册名为 `B2D_Dataset`。
- `mmcv/datasets/B2D_e2e_dataset.py`：UniAD 使用的端到端数据集，注册名为 `B2D_E2E_Dataset`。
- `mmcv/datasets/B2D_vad_dataset.py`：VAD 使用的数据集，注册名为 `B2D_VAD_Dataset`。
- `mmcv/datasets/prepare_B2D.py`：将 Bench2Drive 原始数据转换为 info pkl 的脚本。

### 3.5 team_code 闭环 agent 层

`team_code/` 面向 Bench2Drive/CARLA leaderboard。闭环评测时，leaderboard 不直接运行 `adzoo/*/test.py`，而是加载这里的 agent。

重要文件：

- `team_code/uniad_b2d_agent.py`：UniAD 闭环评测 agent。
- `team_code/vad_b2d_agent.py`：VAD 闭环评测 agent。
- `team_code/vad_b2d_agent_visualize.py`：带可视化保存逻辑的 VAD agent。
- `team_code/planner.py`：根据全局路线和 GPS 找当前目标路点。
- `team_code/pid_controller.py`：把模型预测轨迹转换为方向盘、油门、刹车。

### 3.6 data、analysis、assets

- `data/splits/bench2drive_base_train_val_split.json`：Bench2Drive base 数据集划分。
- `data/others/b2d_motion_anchor_infos_mode6.pkl`：UniAD motion head 使用的 motion anchor 信息。
- `analysis/*.json`：模型闭环评测结果。
- `analysis/analysis.md`：失败案例分析。
- `analysis/gifs/`：成功/失败案例可视化 GIF。
- `assets/`：README 中使用的项目图片。

## 4. 数据准备与数据流

### 4.1 预期数据目录

根据 `docs/DATA_PREP.md`，准备完成后的数据结构大致如下：

```text
Bench2DriveZoo
├── data/
│   ├── bench2drive/
│   │   ├── v1/
│   │   │   ├── Accident_Town03_Route101_Weather23/
│   │   │   ├── Accident_Town03_Route102_Weather20/
│   │   │   └── ...
│   │   └── maps/
│   │       ├── Town01_HD_map.npz
│   │       ├── Town02_HD_map.npz
│   │       └── ...
│   ├── infos/
│   │   ├── b2d_infos_train.pkl
│   │   ├── b2d_infos_val.pkl
│   │   └── b2d_map_infos.pkl
│   ├── others/
│   │   └── b2d_motion_anchor_infos_mode6.pkl
│   └── splits/
│       └── bench2drive_base_train_val_split.json
└── ckpts/
    ├── resnet50-19c8e357.pth
    ├── r101_dcn_fcos3d_pretrain.pth
    ├── bevformer_base_b2d.pth
    ├── uniad_base_b2d.pth
    └── ...
```

### 4.2 数据转换流程

原始 Bench2Drive 数据不能直接被三个模型使用，需要先转换为本项目的数据 info 格式：

```bash
cd mmcv/datasets
python prepare_B2D.py --workers 16
```

该命令会生成：

- `data/infos/b2d_infos_train.pkl`
- `data/infos/b2d_infos_val.pkl`
- `data/infos/b2d_map_infos.pkl`

整体数据流：

```text
Bench2Drive 原始数据
→ data/bench2drive/v1/*
→ data/bench2drive/maps/*
→ mmcv/datasets/prepare_B2D.py
→ data/infos/*.pkl
→ B2D_Dataset / B2D_E2E_Dataset / B2D_VAD_Dataset
→ pipeline
→ dataloader
→ model.forward_train 或 model.forward_test
→ dataset.evaluate
```

### 4.3 三个模型对应的数据集

```text
BEVFormer → B2D_Dataset     → 3D detection / BEV perception
UniAD     → B2D_E2E_Dataset → tracking + map + motion + occupancy + planning
VAD       → B2D_VAD_Dataset → detection + map + motion/planning
```

## 5. BEVFormer 运行流程

BEVFormer 在本项目中主要是感知模型，重点输出 3D 检测结果。

核心配置文件：

- `adzoo/bevformer/configs/bevformer/bevformer_base_b2d.py`
- `adzoo/bevformer/configs/bevformer/bevformer_tiny_b2d.py`

训练命令示例：

```bash
./adzoo/bevformer/dist_train.sh ./adzoo/bevformer/configs/bevformer/bevformer_base_b2d.py 4
./adzoo/bevformer/dist_train.sh ./adzoo/bevformer/configs/bevformer/bevformer_tiny_b2d.py 4
```

Open-loop 评测命令示例：

```bash
./adzoo/bevformer/dist_test.sh ./adzoo/bevformer/configs/bevformer/bevformer_base_b2d.py ./ckpts/bevformer_base_b2d.pth 1
./adzoo/bevformer/dist_test.sh ./adzoo/bevformer/configs/bevformer/bevformer_tiny_b2d.py ./ckpts/bevformer_tiny_b2d.pth 1
```

代码调用链：

```text
adzoo/bevformer/dist_train.sh 或 dist_test.sh
→ adzoo/bevformer/train.py 或 test.py
→ Config.fromfile(config)
→ build_dataset(cfg.data.train/test)
→ build_model(cfg.model)
→ load_checkpoint
→ custom_train_model 或 custom_multi_gpu_test
→ mmcv/models/detectors/bevformer.py
→ BEVFormerHead / transformer / dataset.evaluate
```

关键配置项：

- `model.type = 'BEVFormer'`
- `dataset_type = 'B2D_Dataset'`
- `data_root = 'data/bench2drive'`
- `queue_length = 4`
- `point_cloud_range = [-51.2, -51.2, -5.0, 51.2, 51.2, 3.0]`

## 6. UniAD 运行流程

UniAD 是 end-to-end 自动驾驶模型，包含 tracking、map、motion、occupancy、planning 等模块。

核心配置文件：

- `adzoo/uniad/configs/stage1_track_map/base_track_map_b2d.py`
- `adzoo/uniad/configs/stage1_track_map/tiny_track_map_b2d.py`
- `adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py`
- `adzoo/uniad/configs/stage2_e2e/tiny_e2e_b2d.py`

### 6.1 Open-loop 训练/评测流程

训练 stage1：

```bash
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage1_track_map/base_track_map_b2d.py 4
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage1_track_map/tiny_track_map_b2d.py 4
```

训练 stage2：

```bash
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py 1
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage2_e2e/tiny_e2e_b2d.py 1
```

Open-loop 评测：

```bash
./adzoo/uniad/uniad_dist_eval.sh ./adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py ./ckpts/uniad_base_b2d.pth 1
./adzoo/uniad/uniad_dist_eval.sh ./adzoo/uniad/configs/stage2_e2e/tiny_e2e_b2d.py ./ckpts/uniad_tiny_b2d.pth 1
```

调用链：

```text
adzoo/uniad/uniad_dist_train.sh 或 uniad_dist_eval.sh
→ adzoo/uniad/train.py 或 test.py
→ Config.fromfile(config)
→ build_dataset(cfg.data.*)
→ build_model(cfg.model)
→ cfg.model.type = 'UniAD'
→ mmcv/models/detectors/uniad_e2e.py
→ UniAD.forward / forward_train / forward_test
→ track/map/motion/occ/planning heads
```

关键配置项：

- `model.type = 'UniAD'`
- `dataset_type = 'B2D_E2E_Dataset'`
- `queue_length = 3`
- `data_root = 'data/bench2drive'`
- `model.motion_head.anchor_info_path` 会使用 `data/others/b2d_motion_anchor_infos_mode6.pkl`

### 6.2 Closed-loop CARLA 流程

闭环评测时，UniAD 不直接运行 `adzoo/uniad/test.py`，而是由 Bench2Drive leaderboard 加载 `team_code/uniad_b2d_agent.py`。

调用链：

```text
Bench2Drive leaderboard
→ team_code/uniad_b2d_agent.py:get_entry_point()
→ UniadAgent.setup(config+checkpoint)
→ build_model + load_checkpoint + model.cuda().eval()
→ sensors() 注册传感器
→ run_step(input_data, timestamp)
→ tick() 整理图像/GPS/速度/IMU/route command
→ 构造 lidar2img、lidar2cam、can_bus、command、l2g_r_mat、l2g_t
→ cfg.inference_only_pipeline 预处理
→ self.model(input_data_batch, return_loss=False)
→ 读取 planning result: sdc_traj
→ PIDController.control_pid()
→ carla.VehicleControl
```

UniAD agent 注册的主要传感器：

- 6 个 RGB 相机：前、前左、前右、后、后左、后右。
- IMU。
- GNSS。
- speedometer。
- 如果 `IS_BENCH2DRIVE` 环境变量存在，还会添加一个 BEV 顶视相机用于保存/分析。

模型输出转控制量的关键逻辑：

```text
output_data_batch[0]['planning']['result_planning']['sdc_traj']
→ PIDController.control_pid()
→ steer / throttle / brake
→ carla.VehicleControl
```

## 7. VAD 运行流程

VAD 也是面向规划的端到端模型，但它的输出结构和 UniAD 不同。

核心配置文件：

- `adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py`

### 7.1 Open-loop 训练/评测流程

训练：

```bash
./adzoo/vad/dist_train.sh ./adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py ./ckpts/vad_b2d_base.pth 1
```

Open-loop 评测：

```bash
./adzoo/vad/dist_test.sh ./adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py ./ckpts/vad_b2d_base.pth 1
```

调用链：

```text
adzoo/vad/dist_train.sh 或 dist_test.sh
→ adzoo/vad/train.py 或 test.py
→ Config.fromfile(config)
→ build_dataset(cfg.data.*)
→ build_model(cfg.model)
→ cfg.model.type = 'VAD'
→ mmcv/models/detectors/VAD.py
→ VAD.forward / forward_train / forward_test
→ VADHead
```

关键配置项：

- `model.type = 'VAD'`
- `dataset_type = 'B2D_VAD_Dataset'`
- `queue_length = 4`
- `map_classes = ['Broken', 'Solid', 'SolidSolid', 'Center', 'TrafficLight', 'StopSign']`
- `point_cloud_range = [-15.0, -30.0, -2.0, 15.0, 30.0, 2.0]`

### 7.2 Closed-loop CARLA 流程

调用链：

```text
Bench2Drive leaderboard
→ team_code/vad_b2d_agent.py:get_entry_point()
→ VadAgent.setup(config+checkpoint)
→ build_model + load_checkpoint + model.cuda().eval()
→ sensors() 注册传感器
→ run_step()
→ 构造多相机图像、can_bus、command、ego_fut_cmd
→ self.model(input_data_batch, return_loss=False)
→ 读取 output_data_batch[0]['pts_bbox']['ego_fut_preds']
→ cumsum 得到未来轨迹
→ 根据 route command 选择一条轨迹
→ PIDController.control_pid()
→ carla.VehicleControl
```

VAD 和 UniAD 的闭环主要差异：

```text
UniAD:
output_data_batch[0]['planning']['result_planning']['sdc_traj']

VAD:
output_data_batch[0]['pts_bbox']['ego_fut_preds']
→ np.cumsum(...)
→ all_out_truck[command]
```

## 8. planner 和 PID 控制器

### 8.1 RoutePlanner

文件：`team_code/planner.py`

功能：

- 接收 leaderboard 给出的全局路线。
- 将 GPS 经纬度转换到局部平面坐标。
- 根据当前车辆位置，找到前方最近的目标路点和导航指令。

核心方法：

- `set_route(global_plan, gps=True)`：设置全局路线。
- `run_step(gps)`：根据当前 GPS/位置返回下一目标点和 command。
- `gps_to_location(gps)`：经纬度到局部坐标转换。

### 8.2 PIDController

文件：`team_code/pid_controller.py`

功能：

- 输入模型预测的未来轨迹、当前速度、route target。
- 输出 `steer`、`throttle`、`brake`。

核心逻辑：

- 根据预测轨迹点之间的距离估计期望速度。
- 选择一个 aim point 作为转向目标。
- 根据 route target 修正转向目标，减少异常轨迹点造成的误差。
- 使用 turn PID 输出方向盘。
- 使用 speed PID 输出油门。
- 根据速度和期望速度决定是否刹车。

最终返回：

```text
steer, throttle, brake, metadata
```

agent 再将其写入：

```python
control = carla.VehicleControl()
control.steer = ...
control.throttle = ...
control.brake = ...
```

## 9. 配置文件阅读方法

以 `adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py` 为例，优先看这些部分：

- `point_cloud_range`：感知/规划空间范围。
- `class_names`：Bench2Drive 类别。
- `queue_length`：时序输入帧数量。
- `model`：模型结构总定义。
- `data`：train/val/test 数据集配置。
- `train_pipeline`：训练预处理流程。
- `test_pipeline`：测试预处理流程。
- `inference_only_pipeline`：CARLA agent 在线推理预处理流程。
- `optimizer`：优化器。
- `runner`：训练轮数和 runner 类型。

三个模型的配置差异：

- BEVFormer 配置更偏 3D 检测/感知。
- UniAD 配置包含 tracking、map、motion、occupancy、planning 多个 head。
- VAD 配置包含 detection、map、trajectory/planning 相关 head 和 loss，并有 `ego_fut_cmd` 导航命令输入。

## 10. 训练和评测入口总结

### 10.1 BEVFormer

```bash
# train base
./adzoo/bevformer/dist_train.sh ./adzoo/bevformer/configs/bevformer/bevformer_base_b2d.py 4

# eval base
./adzoo/bevformer/dist_test.sh ./adzoo/bevformer/configs/bevformer/bevformer_base_b2d.py ./ckpts/bevformer_base_b2d.pth 1
```

### 10.2 UniAD

```bash
# train stage1 base
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage1_track_map/base_track_map_b2d.py 4

# train stage2 base
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py 1

# eval stage2 base
./adzoo/uniad/uniad_dist_eval.sh ./adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py ./ckpts/uniad_base_b2d.pth 1
```

### 10.3 VAD

```bash
# train
./adzoo/vad/dist_train.sh ./adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py ./ckpts/vad_b2d_base.pth 1

# eval
./adzoo/vad/dist_test.sh ./adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py ./ckpts/vad_b2d_base.pth 1
```

## 11. 闭环 CARLA 评测接入

根据 `docs/EVAL_IN_CARLA.md`，闭环评测需要先安装本仓库，然后克隆 Bench2Drive evaluation tools，并把本仓库链接到 Bench2Drive leaderboard。

大致步骤：

```bash
cd Bench2Drive/leaderboard
mkdir team_code
ln -s Bench2DriveZoo/team_code/* ./team_code
cd ..
ln -s Bench2DriveZoo ./
```

然后按照 Bench2Drive 官方 evaluation tools 的方式运行闭环评测。

闭环评测的核心不是 `adzoo/*/test.py`，而是：

```text
team_code/uniad_b2d_agent.py
team_code/vad_b2d_agent.py
```

leaderboard 会调用 agent 的：

- `get_entry_point()`
- `setup(path_to_conf_file)`
- `sensors()`
- `run_step(input_data, timestamp)`
- `destroy()`

## 12. 运行环境与系统支持

### 12.1 官方推荐环境

根据 `docs/INSTALL.md`，官方路径是 Linux 风格环境：

```text
Ubuntu 20.04/22.04
Python 3.8
CUDA 11.8
PyTorch cu118
GCC 9.4
NVIDIA GPU
CARLA 0.9.15 Linux package
```

安装关键命令：

```bash
conda create -n b2d_zoo python=3.8
conda activate b2d_zoo
conda install -c "nvidia/label/cuda-11.8.0" cuda-toolkit
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
pip install ninja packaging
pip install -v -e .
```

CARLA 部分使用的是 Linux 包：

```bash
wget https://carla-releases.s3.us-east-005.backblazeb2.com/Linux/CARLA_0.9.15.tar.gz
tar -xvf CARLA_0.9.15.tar.gz
```

并且文档中写入的是 Linux egg：

```text
carla-0.9.15-py3.7-linux-x86_64.egg
```

### 12.2 Ubuntu 支持情况

Ubuntu/Linux 是官方推荐和实际支持路径。训练、open-loop 评测、CARLA 闭环评测都围绕 Linux 命令、Linux CARLA 包和 Linux CUDA 编译链设计。

结论：

```text
Ubuntu/Linux：推荐，适合完整复现训练、评测和闭环 CARLA。
```

### 12.3 Windows 支持情况

本仓库没有提供 Windows 原生支持流程，不建议直接在 Windows 原生环境完整运行。

主要原因：

- 脚本大量使用 `.sh`、`export`、`ln -s`、bash 语法。
- CARLA 安装文档使用 Linux 包和 Linux egg。
- `mmcv/ops`、`mmcv/layers` 中包含大量 C++/CUDA 扩展，Windows 编译链适配难度较高。
- 文档推荐 GCC 9.4，而 Windows 通常使用 MSVC 编译链。
- 闭环评测依赖 CARLA + leaderboard，项目文档没有 Windows 路径。

结论：

```text
Windows 原生：不推荐，仓库没有官方支持流程。
WSL2：open-loop 训练/评测理论上可能适配，但 CUDA、编译扩展、CARLA 图形和 leaderboard 仍需额外处理。
Ubuntu/Linux：最稳妥。
```

## 13. 建议阅读顺序

如果你是第一次接触项目，建议按下面顺序阅读：

1. `README.md`
2. `docs/INSTALL.md`
3. `docs/DATA_PREP.md`
4. `docs/TRAIN_EVAL.md`
5. `docs/EVAL_IN_CARLA.md`
6. `adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py`
7. `team_code/uniad_b2d_agent.py`
8. `team_code/vad_b2d_agent.py`
9. `mmcv/models/detectors/uniad_e2e.py`
10. `mmcv/models/detectors/VAD.py`
11. `mmcv/models/detectors/bevformer.py`
12. `mmcv/datasets/B2D_e2e_dataset.py`
13. `mmcv/datasets/B2D_vad_dataset.py`
14. `mmcv/datasets/B2D_dataset.py`

## 14. 新手调试建议

建议先不要直接跑完整训练。更稳妥的顺序是：

1. 先确认环境能 `pip install -v -e .`。
2. 确认 CUDA、PyTorch、编译扩展可用。
3. 准备 Bench2Drive 数据并生成 `data/infos/*.pkl`。
4. 用下载的 checkpoint 跑单卡 open-loop eval。
5. 确认 open-loop 结果能跑通后，再接入 CARLA closed-loop。
6. 研究闭环 agent 时，优先看 `run_step()`，因为它串起了传感器、预处理、模型推理和控制输出。

## 15. 最核心的运行链路速记

Open-loop：

```text
*.sh
→ train.py / test.py
→ config
→ build_dataset
→ build_model
→ model.forward_train / forward_test
→ evaluate
```

Closed-loop：

```text
Bench2Drive leaderboard
→ team_code/*_agent.py
→ setup()
→ sensors()
→ run_step()
→ model inference
→ predicted trajectory
→ PIDController
→ carla.VehicleControl
```

三模型对应关系：

```text
BEVFormer:
config → B2D_Dataset → BEVFormer → BEVFormerHead → 3D detection

UniAD:
config → B2D_E2E_Dataset → UniAD → track/map/motion/occ/planning → sdc_traj

VAD:
config → B2D_VAD_Dataset → VAD → VADHead → ego_fut_preds → selected trajectory
```
