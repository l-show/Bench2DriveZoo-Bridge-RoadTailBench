# Bench2DriveZoo 项目总览（中文）

> 面向第一次接触本项目的同学：快速理解「文件关系」「UniAD如何运行」「环境与系统支持」。

## 1. 项目定位与三大模型

本仓库是 Bench2Drive 基准中三个自动驾驶模型的统一训练与评测实现：

- **BEVFormer**
- **UniAD**
- **VAD**

并覆盖两类评估：

- **开环评估（Open-loop）**：离线数据集上评估感知/预测/规划指标。
- **闭环评估（Closed-loop）**：在 CARLA + Bench2Drive leaderboard 工具中以 agent 方式跑车。

见仓库说明与文档入口：`README.md`、`docs/*.md`。

---

## 2. 文件与模块关系（从“入口”到“底层”）

可以按 6 层理解：

### A. 顶层入口与文档层

- `README.md`：项目简介、模型结果、文档跳转。
- `docs/INSTALL.md`：环境安装（Python/CUDA/Torch/CARLA）。
- `docs/DATA_PREP.md`：数据准备。
- `docs/TRAIN_EVAL.md`：三模型训练与开环评测命令。
- `docs/EVAL_IN_CARLA.md`：闭环评测接入 Bench2Drive 的方式。

### B. 模型“动物园”层：`adzoo/`

`adzoo` 目录按模型拆分为 3 套子工程：

- `adzoo/bevformer/`
- `adzoo/uniad/`
- `adzoo/vad/`

每套里通常包含：

- `configs/`：配置系统（网络结构、数据管线、训练策略）。
- `train.py` / `test.py`：训练和测试主程序。
- `dist_train.sh` / `dist_test.sh`（或 UniAD 自己命名）：分布式脚本包装。
- `analysis_tools/`、`misc/`：日志分析、可视化等工具。
- `data_converter/`：数据转换脚本。

### C. 通用框架层：`mmcv/`

这是本仓库“内置整合”的核心代码层，合并了多依赖生态能力（README 有说明）：

- `mmcv/models/`：backbone、neck、head、detector（含 `UniAD`、`VAD`、`BEVFormer` 检测器实现）。
- `mmcv/datasets/`：B2D 与 nuScenes 风格数据集定义、pipeline。
- `mmcv/runner/`：训练/评测流程控制。
- `mmcv/ops/`、`mmcv/layers/`：算子与 CUDA 扩展。

你可以把 `adzoo/*/configs` 看作“装配图”，把 `mmcv/` 看作“真正执行的发动机”。

### D. 闭环 agent 层：`team_code/`

面向 CARLA leaderboard 的驾驶体：

- `team_code/uniad_b2d_agent.py`
- `team_code/vad_b2d_agent.py`
- `team_code/pid_controller.py`
- `team_code/planner.py`

在闭环评测时，Bench2Drive leaderboard 会调用这里的 agent 入口。

### E. 资源与分析层

- `analysis/*.json`、`analysis/analysis.md`：结果与案例分析。
- `analysis/gifs/`：可视化失败/成功片段。
- `data/`：拆分文件、锚点等辅助资源。

### F. 打包层

- `setup.py`：`pip install -e .` 安装入口。
- `requirements.txt`：Python 依赖列表。

---

## 3. UniAD 是怎么“跑起来”的（两条路径）

## 3.1 开环评估/训练路径

以 stage2 base 为例：

1. 运行脚本（分布式包装）
   - 训练：`adzoo/uniad/uniad_dist_train.sh`
   - 评估：`adzoo/uniad/uniad_dist_eval.sh`

2. 脚本调用 Python 主程序
   - 训练走 `adzoo/uniad/train.py`
   - 评估走 `adzoo/uniad/test.py`

3. `test.py`/`train.py` 读取 config
   - 例如 `adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py`

4. config 再通过 `_base_` 继承
   - 数据基座：`adzoo/uniad/configs/_base_/datasets/nus-3d.py`
   - 运行时基座：`adzoo/uniad/configs/_base_/default_runtime.py`

5. 通过 `build_model`、`build_dataset` 组装模型与数据
   - 最终会进入 `mmcv/models/*` 与 `mmcv/datasets/*` 的具体实现。

> 简化理解：`*.sh -> test/train.py -> config -> mmcv 具体模块`。

## 3.2 闭环（CARLA）路径

闭环中，UniAD 不直接跑 `test.py`，而是走 agent：

1. leaderboard 加载 `team_code/uniad_b2d_agent.py`（`get_entry_point()` 返回 `UniadAgent`）。
2. `setup()` 里读取 `config_path + checkpoint`，并 `build_model + load_checkpoint`。
3. agent 声明多相机/IMU/GNSS/speed 传感器（`sensors()`）。
4. 运行时不断接收传感器数据，做预处理、前向推理。
5. 将网络输出与 PID/规划器结合，生成 `carla.VehicleControl`（转向/油门/刹车）。

所以你关心的“UniAD怎么运行”可以分成：

- **离线指标**：`adzoo/uniad/*.sh + train/test.py + configs`
- **在线驾驶**：`team_code/uniad_b2d_agent.py`

---

## 4. 你最应该优先读的关键文件（建议顺序）

1. `README.md`（全局定位）
2. `docs/INSTALL.md`（环境前提）
3. `docs/TRAIN_EVAL.md`（统一命令视图）
4. `docs/EVAL_IN_CARLA.md`（闭环接入方式）
5. `adzoo/uniad/configs/stage2_e2e/base_e2e_b2d.py`（UniAD 主配置）
6. `adzoo/uniad/test.py`（UniAD 开环推理流程）
7. `team_code/uniad_b2d_agent.py`（UniAD 闭环驾驶流程）
8. `mmcv/models/detectors/uniad_e2e.py`、`mmcv/models/detectors/uniad_track.py`（核心检测器）

---

## 5. 运行环境与系统支持（Windows / Ubuntu）

基于仓库文档可得：

- 官方安装流程明确依赖 **Python 3.8 + CUDA 11.8 + PyTorch cu118**。
- 闭环评测步骤下载的是 **CARLA_0.9.15 Linux 包**（`.tar.gz`），并把 Linux egg 路径写入 conda 环境。
- 命令脚本与流程以 bash、Linux 路径习惯为主。

因此：

- **Ubuntu/Linux：官方路径，直接支持（推荐）**。
- **Windows：仓库没有给出官方支持路径。**
  - 纯开环训练理论上可尝试在 Windows 适配，但会遇到 CUDA 扩展编译、脚本、依赖兼容问题。
  - 闭环 CARLA + leaderboard 这套在本仓库给出的流程是 Linux 优先。

**结论**：如果你要稳定复现三模型训练与尤其闭环评测，建议使用 **Ubuntu 20.04/22.04 + NVIDIA GPU** 环境。

---

## 6. 一句话总结构图

- `docs` 讲“怎么跑”
- `adzoo/*/configs` 讲“跑什么模型与参数”
- `adzoo/*/train|test.py` 讲“训练/评估流程”
- `mmcv/` 是“底层实现引擎”
- `team_code/*_agent.py` 是“闭环驾驶入口”

如果你愿意，我下一步可以继续给你一份 **UniAD 的逐函数调用链（从配置到 forward 到控制输出）**，按“能下断点调试”的粒度展开。
