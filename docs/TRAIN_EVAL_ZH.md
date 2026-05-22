# 训练与 Open-loop 评测说明（中文增强版）

本文档对应 `docs/TRAIN_EVAL.md`，说明如何训练和验证 BEVFormer、UniAD、VAD。你如果只是复现已有模型，不想训练，只需要关注 open-loop eval 命令和 checkpoint 下载。

## 1. 总体说明

本仓库支持三类模型：

- BEVFormer
- UniAD
- VAD

训练/评测入口都在 `adzoo/` 下：

```text
adzoo/bevformer/
adzoo/uniad/
adzoo/vad/
```

Open-loop 评测需要：

- 已安装环境。
- 已准备 `data/infos/*.pkl`。
- 已下载对应 checkpoint。

Closed-loop CARLA 评测不走本文命令，见 `docs/EVAL_IN_CARLA_ZH.md`。

## 2. BEVFormer

### 2.1 训练

原文命令：

```bash
# train BEVFormer base
./adzoo/bevformer/dist_train.sh ./adzoo/bevformer/configs/bevformer/bevformer_base_b2d.py 4

# train BEVFormer tiny
./adzoo/bevformer/dist_train.sh ./adzoo/bevformer/configs/bevformer/bevformer_tiny_b2d.py 4
```

参数说明：

- 第一个参数是 config。
- 第二个参数是 GPU 数量。

注意事项：

- 训练前需要完整 Bench2Drive 离线数据和 `data/infos/*.pkl`。
- `bevformer_base_b2d.py` 中有 `load_from = 'ckpts/r101_dcn_fcos3d_pretrain.pth'`，训练前需要下载该预训练权重。
- 多 GPU 训练依赖 `torch.distributed.launch`。

### 2.2 Open-loop 评测

原文命令：

```bash
# eval BEVFormer base
./adzoo/bevformer/dist_test.sh ./adzoo/bevformer/configs/bevformer/bevformer_base_b2d.py ./ckpts/bevformer_base_b2d.pth 1

# test BEVFormer tiny
./adzoo/bevformer/dist_test.sh ./adzoo/bevformer/configs/bevformer/bevformer_tiny_b2d.py ./ckpts/bevformer_tiny_b2d.pth 1
```

注意事项：

- BEVFormer 在当前仓库中主要是 open-loop 感知模型，没有现成闭环控制 agent。
- `bevformer_base_b2d.pth` 和 `bevformer_tiny_b2d.pth` 需要从 README 链接下载到 `ckpts/`。

## 3. UniAD

UniAD 分两阶段训练：

- Stage1：track/map。
- Stage2：end-to-end，包括规划相关模块。

如果只复现，不训练，可以直接下载 stage2 checkpoint 跑 eval 或 closed-loop。

### 3.1 训练 Stage1

原文命令：

```bash
# train UniAD base
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage1_track_map/base_track_map_b2d.py 4

# train UniAD tiny
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage1_track_map/tiny_track_map_b2d.py 4
```

注意事项：

- Stage1 训练结果通常用于 Stage2 初始化。
- 复现已有结果时不建议从头训练，成本较高。

### 3.2 训练 Stage2

原文命令：

```bash
# train UniAD base
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py 1

# train UniAD tiny
./adzoo/uniad/uniad_dist_train.sh ./adzoo/uniad/configs/stage2_e2e/tiny_e2e_b2d.py 1
```

注意事项：

- Stage2 config 中包含 tracking、map、motion、occupancy、planning 等模块。
- UniAD 训练显存要求高，base 版本更重。
- 如果只做闭环复现，直接下载 `uniad_base_b2d.pth` 或 `uniad_tiny_b2d.pth`。

### 3.3 Open-loop 评测

原文中 UniAD base 命令疑似少写了 checkpoint 参数。更合理的写法是：

```bash
# eval UniAD base
./adzoo/uniad/uniad_dist_eval.sh ./adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py ./ckpts/uniad_base_b2d.pth 1

# eval UniAD tiny
./adzoo/uniad/uniad_dist_eval.sh ./adzoo/uniad/configs/stage2_e2e/tiny_e2e_b2d.py ./ckpts/uniad_tiny_b2d.pth 1
```

原因：`adzoo/uniad/uniad_dist_eval.sh` 中参数定义为：

```bash
CFG=$1
CKPT=$2
GPUS=$3
```

因此必须传入 checkpoint。

## 4. VAD

### 4.1 训练

原文命令：

```bash
./adzoo/vad/dist_train.sh ./adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py ./ckpts/vad_b2d_base.pth 1
```

注意事项：

- 这个命令里的第二个参数在脚本中不是典型的 `load_from`，需要看具体 `dist_train.sh` 和 `train.py` 的解析逻辑。
- 如果你不训练，跳过本步骤。

### 4.2 Open-loop 评测

原文命令：

```bash
./adzoo/vad/dist_test.sh ./adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py ./ckpts/vad_b2d_base.pth 1
```

注意事项：

- `vad_b2d_base.pth` 需要下载到 `ckpts/`。
- VAD 有现成 closed-loop agent：`team_code/vad_b2d_agent.py`。

## 5. Planning L2 指标差异

原文说明：

UniAD 和 VAD 使用不同的 Planning L2 计算定义：

- UniAD：在每个时间点计算 L2，例如 0.5s、1.0s、1.5s。
- VAD：计算每个时间段内的平均值，例如 0s-0.5s、0s-1.0s、0s-1.5s。

仓库保留了各自原始代码逻辑，但报告时会把 UniAD 的 Planning L2 转换为 VAD 的定义。

注意事项：

- 比较 UniAD 和 VAD 时，不要只看原始代码中的 L2 计算函数，要看最终报告口径。
- open-loop 规划指标和 closed-loop driving score 不是同一类指标。closed-loop 更看重实际驾驶完成度、碰撞、违规等。

## 6. Checkpoint 下载位置

模型权重在 README 中提供：

| 模型 | 文件 | 下载 |
|---|---|---|
| UniAD-Tiny | `uniad_tiny_b2d.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/uniad_tiny_b2d.pth；百度云: https://pan.baidu.com/s/1psr7AKYHD7CitZ30Bz-9sA?pwd=1234 |
| UniAD-Base | `uniad_base_b2d.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/uniad_base_b2d.pth；百度云: https://pan.baidu.com/s/11p9IUGqTax1f4W_qsdLCRw?pwd=1234 |
| VAD | `vad_b2d_base.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/vad_b2d_base.pth；百度云: https://pan.baidu.com/s/1rK7Z_D-JsA7kBJmEUcMMyg?pwd=1234 |
| BEVFormer-Tiny | `bevformer_tiny_b2d.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/bevformer_tiny_b2d.pth；百度云: https://pan.baidu.com/s/1TWMs9YgKYm2DF5YfXF8i3g?pwd=1234 |
| BEVFormer-Base | `bevformer_base_b2d.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/bevformer_base_b2d.pth；百度云: https://pan.baidu.com/s/1Y4VkE1gc8BU0zJ4z2fmIkQ?pwd=1234 |

下载后放入：

```text
Bench2DriveZoo/ckpts/
```

## 7. 复现建议

如果你的目标是先复现结果，不建议从训练开始。推荐顺序：

1. 配好环境。
2. 下载 checkpoint。
3. 准备 open-loop 数据 info。
4. 先跑 VAD 或 BEVFormer 的单卡 open-loop eval。
5. 再跑 UniAD open-loop eval。
6. 最后接 CARLA closed-loop。

