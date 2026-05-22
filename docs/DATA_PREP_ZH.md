# Bench2Drive 数据准备说明（中文增强版）

本文档对应 `docs/DATA_PREP.md`，说明如何准备 Bench2Drive 离线数据，并生成本仓库训练/open-loop 评测所需的 info 文件。

## 1. 下载 Bench2Drive 数据

原文说明：从 Bench2Drive 官方仓库下载数据：

```text
https://github.com/Thinklab-SJTU/Bench2Drive
```

注意：

- Bench2DriveZoo 本仓库通常不包含完整数据集。
- 数据体积较大，建议放在大容量磁盘或服务器数据盘。
- 不同版本数据目录可能略有差异，必要时使用软链接 `ln -s` 或修改 config/data path。

## 2. 期望数据目录结构

原文要求数据结构大致如下：

```text
Bench2DriveZoo
├── data/
│   ├── bench2drive/
│   │   ├── v1/                         # Bench2Drive base 数据
│   │   │   ├── Accident_Town03_Route101_Weather23/
│   │   │   ├── Accident_Town03_Route102_Weather20/
│   │   │   └── ...
│   │   └── maps/                       # Town 地图文件
│   │       ├── Town01_HD_map.npz
│   │       ├── Town02_HD_map.npz
│   │       └── ...
│   ├── others/
│   │   └── b2d_motion_anchor_infos_mode6.pkl
│   └── splits/
│       └── bench2drive_base_train_val_split.json
```

本仓库已经包含：

```text
data/others/b2d_motion_anchor_infos_mode6.pkl
data/splits/bench2drive_base_train_val_split.json
```

你需要额外准备：

```text
data/bench2drive/v1/
data/bench2drive/maps/
```

如果你的数据放在其他位置，例如 `/data/datasets/bench2drive`，可以软链接：

```bash
cd Bench2DriveZoo/data
ln -s /data/datasets/bench2drive bench2drive
```

最终应满足：

```bash
ls data/bench2drive/v1
ls data/bench2drive/maps
```

## 3. 生成 Bench2Drive data info

原文命令：

```bash
cd mmcv/datasets
python prepare_B2D.py --workers 16
```

该命令会在 `data/infos` 下生成：

```text
b2d_infos_train.pkl
b2d_infos_val.pkl
b2d_map_infos.pkl
```

注意事项：

- 命令应从仓库根目录或正确相对路径运行。原文进入 `mmcv/datasets` 后运行，脚本内部路径默认按仓库结构查找。
- `--workers 16` 是并行进程数。服务器 CPU 核心少或内存不足时可以减小，例如 `--workers 4`。
- 原文说明：Base set 1000 clips，16 workers 大约需要 1 小时。
- 如果路径不对，优先检查 `data/bench2drive` 是否存在、是否软链接正确。

检查生成结果：

```bash
ls -lh data/infos
```

如果你从 `mmcv/datasets` 目录运行，检查相对路径可能需要：

```bash
ls -lh ../../data/infos
```

## 4. 数据划分逻辑

原文说明：该命令默认使用 `data/splits/bench2drive_base_train_val_split.json` 中列出的路线作为验证集，其余路线作为训练集。

这意味着：

- `b2d_infos_train.pkl`：训练集 info。
- `b2d_infos_val.pkl`：验证/open-loop 测试 info。
- `b2d_map_infos.pkl`：地图相关 info。

注意事项：

- 如果你修改了 split json，需要重新运行 `prepare_B2D.py`。
- 如果你要换成自己的 RoadTailBench 离线数据，不能只改 split，还需要适配 `prepare_B2D.py` 中的路径、坐标、标注字段和地图格式。

## 5. 准备完成后的完整结构

原文给出的最终结构整理如下：

```text
Bench2DriveZoo
├── adzoo/
│   ├── bevformer/
│   ├── uniad/
│   └── vad/
├── ckpts/
│   ├── r101_dcn_fcos3d_pretrain.pth
│   ├── resnet50-19c8e357.pth
│   ├── bevformer_base_b2d.pth
│   ├── uniad_base_b2d.pth
│   └── ...
├── data/
│   ├── bench2drive/
│   │   ├── v1/
│   │   └── maps/
│   ├── infos/
│   │   ├── b2d_infos_train.pkl
│   │   ├── b2d_infos_val.pkl
│   │   └── b2d_map_infos.pkl
│   ├── others/
│   │   └── b2d_motion_anchor_infos_mode6.pkl
│   └── splits/
│       └── bench2drive_base_train_val_split.json
├── docs/
├── mmcv/
└── team_code/
```

## 6. 离线数据和闭环评测的区别

这一点容易混淆：

- `docs/DATA_PREP.md` 准备的是离线数据，用于训练和 open-loop 评测。
- CARLA closed-loop 评测不直接读取 `data/bench2drive/v1` 中的图片序列，而是在 CARLA 中实时生成传感器数据。
- closed-loop 主要依赖 Bench2Drive evaluation tools 中的 routes、scenarios 和 CARLA 地图资产。

所以如果你只想先跑闭环 UniAD/VAD，重点是：

```text
CARLA 0.9.15
Bench2Drive evaluation tools
team_code/uniad_b2d_agent.py 或 vad_b2d_agent.py
对应 checkpoint
```

如果你还要跑 open-loop 或训练，才必须准备 `data/bench2drive/v1` 并生成 `data/infos/*.pkl`。

## 7. 常见问题

### 7.1 找不到 data/bench2drive

处理：

```bash
cd Bench2DriveZoo/data
ln -s /your/real/bench2drive/path bench2drive
```

确保：

```bash
ls data/bench2drive/v1
ls data/bench2drive/maps
```

### 7.2 prepare_B2D.py 路径错误

检查当前工作目录和脚本中的相对路径。建议先从仓库根目录运行：

```bash
cd Bench2DriveZoo
python mmcv/datasets/prepare_B2D.py --workers 16
```

如果脚本要求从 `mmcv/datasets` 运行，则按原文方式。

### 7.3 生成速度很慢

可调大 `--workers`，但会增加 CPU 和内存占用。服务器资源不足时反而可能更慢或被杀进程。

### 7.4 RoadTailBench 离线数据适配

如果后续要把自己的 RoadTailBench 数据做成 open-loop 数据集，需要重点改：

- `mmcv/datasets/prepare_B2D.py`
- `mmcv/datasets/B2D_dataset.py`
- `mmcv/datasets/B2D_e2e_dataset.py`
- `mmcv/datasets/B2D_vad_dataset.py`
- 各模型 config 中的 `data_root`、`ann_file_train`、`ann_file_val`、类别映射、地图信息。

