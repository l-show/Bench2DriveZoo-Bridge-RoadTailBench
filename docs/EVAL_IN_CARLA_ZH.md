# CARLA 闭环评测说明（中文增强版）

本文档对应 `docs/EVAL_IN_CARLA.md`，说明如何在 CARLA 中评测 UniAD 和 VAD。

## 1. 重要结论

原文开头说明：

```text
Please follow these steps to evaluate UniAD and VAD in CARLA.
```

也就是说，当前仓库的闭环评测主要面向：

- UniAD：`team_code/uniad_b2d_agent.py`
- VAD：`team_code/vad_b2d_agent.py`

当前仓库没有提供 BEVFormer 的闭环驾驶 agent。BEVFormer 可以做 open-loop 感知评测，但不能直接像 UniAD/VAD 一样控制 ego 车。

## 2. 准备工作

原文要求：

1. 按 `docs/INSTALL.md` 安装本仓库。
2. 从 Bench2Drive 官方仓库克隆 evaluation tools，并为其准备 CARLA。

Bench2Drive 地址：

```text
https://github.com/Thinklab-SJTU/Bench2Drive
```

建议目录结构：

```text
/data/project/
├── CARLA_0.9.15/
├── Bench2Drive/
└── Bench2DriveZoo/
```

设置环境变量：

```bash
export PROJECT_ROOT=/data/project
export CARLA_ROOT=$PROJECT_ROOT/CARLA_0.9.15
export B2D_ROOT=$PROJECT_ROOT/Bench2Drive
export B2DZOO_ROOT=$PROJECT_ROOT/Bench2DriveZoo
```

## 3. 将 Bench2DriveZoo 链接到 Bench2Drive

原文命令：

```bash
# Add your agent code
cd Bench2Drive/leaderboard
mkdir team_code
ln -s Bench2DriveZoo/team_code/* ./team_code    # link UniAD,VAD agents and utils
cd ..
ln -s Bench2DriveZoo  ./                        # link entire repo to Bench2Drive.
```

更稳妥的绝对路径写法：

```bash
cd $B2D_ROOT
ln -s $B2DZOO_ROOT Bench2DriveZoo

cd $B2D_ROOT/leaderboard
mkdir -p team_code
ln -s $B2DZOO_ROOT/team_code/* ./team_code/
```

检查：

```bash
ls -l $B2D_ROOT/Bench2DriveZoo
ls -l $B2D_ROOT/leaderboard/team_code
```

注意事项：

- `team_code/uniad_b2d_agent.py` 和 `team_code/vad_b2d_agent.py` 中有 `from Bench2DriveZoo...` 这样的 import，因此 `$B2D_ROOT/Bench2DriveZoo` 软链接很重要。
- 如果你不用软链接，也必须把 Bench2DriveZoo 加入 `PYTHONPATH`，但软链接更接近官方流程。

## 4. 设置 PYTHONPATH

建议运行闭环前设置：

```bash
export PYTHONPATH=$B2D_ROOT:$B2D_ROOT/leaderboard:$B2D_ROOT/scenario_runner:$PYTHONPATH
```

如果报错：

```text
No module named leaderboard
No module named scenario_runner
No module named Bench2DriveZoo
```

优先检查该变量。

## 5. 准备 checkpoint

闭环 UniAD/VAD 必须下载对应 checkpoint：

```text
Bench2DriveZoo/ckpts/uniad_base_b2d.pth
Bench2DriveZoo/ckpts/uniad_tiny_b2d.pth
Bench2DriveZoo/ckpts/vad_b2d_base.pth
```

下载地址见本文最后的 checkpoint 汇总。

## 6. TEAM_AGENT 和 TEAM_CONFIG

Bench2Drive leaderboard 通常通过 `TEAM_AGENT` 加载 agent，通过 `TEAM_CONFIG` 给 agent 传配置和权重。

### 6.1 UniAD

```bash
TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/uniad_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/uniad_base_b2d.pth+UniAD-Base"
```

`team_code/uniad_b2d_agent.py` 中解析逻辑：

```python
self.config_path = path_to_conf_file.split('+')[0]
self.ckpt_path = path_to_conf_file.split('+')[1]
```

因此必须使用：

```text
config路径+checkpoint路径+保存名
```

### 6.2 VAD

```bash
TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/vad_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/vad_b2d_base.pth+VAD-Base"
```

注意事项：

- 加号 `+` 是 agent 代码里硬编码的分隔符。
- 路径中不要包含额外的 `+`。
- 第三个字段通常作为保存名使用。

## 7. 运行评测

原文写法：

```text
Follow this to use evaluation tools of Bench2Drive.
```

即具体启动命令以 Bench2Drive 官方 evaluation tools 为准。一般需要配置：

```bash
ROUTES=...
TEAM_AGENT=...
TEAM_CONFIG=...
CHECKPOINT_ENDPOINT=...
SAVE_PATH=...
PORT=...
TM_PORT=...
GPU_RANK=...
```

示例 UniAD：

```bash
export IS_BENCH2DRIVE=True
export SAVE_PATH=$B2D_ROOT/eval_v1

ROUTES=$B2D_ROOT/leaderboard/data/bench2drive220.xml
TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/uniad_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/uniad_base_b2d.pth+UniAD-Base"
CHECKPOINT_ENDPOINT=$B2D_ROOT/eval_results/uniad_base_progress.json
```

示例 VAD：

```bash
export IS_BENCH2DRIVE=True
export SAVE_PATH=$B2D_ROOT/eval_v1

ROUTES=$B2D_ROOT/leaderboard/data/bench2drive220.xml
TEAM_AGENT=$B2D_ROOT/leaderboard/team_code/vad_b2d_agent.py
TEAM_CONFIG="$B2DZOO_ROOT/adzoo/vad/configs/VAD/VAD_base_e2e_b2d.py+$B2DZOO_ROOT/ckpts/vad_b2d_base.pth+VAD-Base"
CHECKPOINT_ENDPOINT=$B2D_ROOT/eval_results/vad_base_progress.json
```

注意：不同版本 Bench2Drive 的启动脚本参数可能不同，以你本地 Bench2Drive 的 `leaderboard/scripts/` 为准。

## 8. 指标输出

Bench2Drive evaluation tools 会输出 route 级别和汇总指标，通常包括：

- Driving Score
- Success Rate
- Route Completion
- Infraction 相关指标
- Ability benchmark
- Smoothness
- Efficiency

`team_code/uniad_b2d_agent.py` 和 `team_code/vad_b2d_agent.py` 中会保存：

```text
metric_info.json
```

这个文件对 smoothness 和 efficiency 指标很重要。

注意事项：

- 如果为了节省磁盘删除图像保存逻辑，不要删除 `metric_info.json` 保存逻辑。
- 完整评测需要所有 routes 都完成或有失败记录，否则合并指标可能不完整。
- 如果中断后重跑，保留 `CHECKPOINT_ENDPOINT` 可以帮助跳过已完成路线。

## 9. 常见问题

### 9.1 CARLA 无法启动

检查：

```bash
nvidia-smi
vulkaninfo
lsof -i:20000
```

尝试：

```bash
cd $CARLA_ROOT
./CarlaUE4.sh -RenderOffScreen -nosound -carla-rpc-port=20000
```

### 9.2 agent import 失败

检查：

```bash
ls -l $B2D_ROOT/Bench2DriveZoo
ls -l $B2D_ROOT/leaderboard/team_code
echo $PYTHONPATH
```

### 9.3 车辆不动

重点检查：

- checkpoint 路径是否正确。
- `TEAM_CONFIG` 是否为 `config+ckpt+name`。
- agent 是否成功加载模型到 GPU。
- `SAVE_PATH` 是否有 meta 输出。
- `metric_info.json` 是否生成。

### 9.4 BEVFormer 能不能闭环

当前仓库没有 BEVFormer 闭环 agent，因此不能直接闭环控制车。要闭环，需要自己实现 `team_code/bevformer_b2d_agent.py` 并补规划控制模块。

## 10. Checkpoint 下载位置

| 模型 | 文件 | 下载 |
|---|---|---|
| UniAD-Tiny | `uniad_tiny_b2d.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/uniad_tiny_b2d.pth；百度云: https://pan.baidu.com/s/1psr7AKYHD7CitZ30Bz-9sA?pwd=1234 |
| UniAD-Base | `uniad_base_b2d.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/uniad_base_b2d.pth；百度云: https://pan.baidu.com/s/11p9IUGqTax1f4W_qsdLCRw?pwd=1234 |
| VAD | `vad_b2d_base.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/vad_b2d_base.pth；百度云: https://pan.baidu.com/s/1rK7Z_D-JsA7kBJmEUcMMyg?pwd=1234 |

BEVFormer 权重也可下载，但用于 open-loop：

| 模型 | 文件 | 下载 |
|---|---|---|
| BEVFormer-Tiny | `bevformer_tiny_b2d.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/bevformer_tiny_b2d.pth；百度云: https://pan.baidu.com/s/1TWMs9YgKYm2DF5YfXF8i3g?pwd=1234 |
| BEVFormer-Base | `bevformer_base_b2d.pth` | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/bevformer_base_b2d.pth；百度云: https://pan.baidu.com/s/1Y4VkE1gc8BU0zJ4z2fmIkQ?pwd=1234 |

