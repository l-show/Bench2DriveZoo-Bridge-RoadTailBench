# 代码迁移指南（中文增强版）

本文档对应 `docs/CONVERT_GUIDE.md`，说明如果你要把基于 nuScenes 或其他数据集的方法迁移到 Bench2Drive，需要注意哪些地方。对于你后续把 Bench2Drive 换成 RoadTailBench，也可以参考本文。

## 1. 模型代码迁移

原文说明：本仓库把多个 MMCV 相关依赖集成到了 `mmcv/` 目录，不再安装原始独立库。

含义：

- 不要默认依赖外部 `mmcv-full`、`mmdet`、`mmdet3d`、`mmseg` 的完整安装。
- 本仓库使用本地 `mmcv/` 目录作为统一框架。
- 新模型、新 head、新 dataset、新 loss 应优先放到本仓库 `mmcv/` 对应目录，并完成注册。

常见放置位置：

```text
mmcv/models/detectors/       detector 或 end-to-end 模型
mmcv/models/dense_heads/     head
mmcv/models/modules/         transformer/attention/module
mmcv/models/losses/          loss
mmcv/datasets/               dataset
mmcv/datasets/pipelines/     data pipeline
mmcv/core/                   bbox/eval/postprocess 等核心功能
```

注意事项：

- 如果从外部项目复制代码，先检查 import 是否依赖外部 `mmdet3d` 或 `mmcv.ops`。
- 如果本地 `mmcv/` 没有对应模块，需要补齐。
- 添加新类后，确认使用了注册器，例如 `@DETECTORS.register_module()`、`@DATASETS.register_module()`。

## 2. 脚本和配置迁移

原文说明：每个方法的 config 和 script 可以放到 `adzoo/` 下，便于管理。

本仓库结构：

```text
adzoo/
├── bevformer/
├── uniad/
└── vad/
```

如果你新增方法，例如 `my_model`，建议：

```text
adzoo/my_model/
├── configs/
├── train.py
├── test.py
├── dist_train.sh
├── dist_test.sh
├── apis/
├── analysis_tools/
└── misc/
```

注意事项：

- config 中的 `type` 名称必须和注册类名一致。
- 脚本中 `PYTHONPATH` 要能找到本仓库。
- 如果使用 plugin 机制，确认 `plugin=True` 和 `plugin_dir` 指向正确。

## 3. Bench2Drive config 细节

原文列出三点：

1. Bench2Drive 的 name-to-class mapping 和 evaluation settings 已经直接写在 config 里。
2. nuScenes 有 10 类，而 Bench2Drive 使用 9 类。
3. UniAD/VAD 在 nuScenes 上使用 3 个 command，而 Bench2Drive 使用从 CARLA 得到的 6 个 command。

### 3.1 类别映射

Bench2Drive config 中会定义车辆、行人、交通灯、交通标志等类别映射。例如：

```python
NameMapping = {
    ...
}
class_names = [...]
```

如果迁移到 RoadTailBench 或自定义数据：

- 保持类别集合和 checkpoint 训练时一致，最稳。
- 如果新增类别，需要重新训练或至少修改 head 的 `num_classes`，不能直接用旧 checkpoint 严格加载。
- 如果只是闭环测试，不训练模型，CARLA 里的 actor blueprint 最好仍映射到原 Bench2Drive 类别。

### 3.2 command 数量

Bench2Drive 使用 CARLA route command，通常是 6 类指令。VAD agent 中会构造：

```python
command_onehot = np.zeros(6)
command_onehot[command] = 1
results['ego_fut_cmd'] = command_onehot
```

注意事项：

- 如果你的 RoadTailBench 路线命令类型不兼容，需要先适配 leaderboard route command，而不是直接改模型。
- 改 command 维度会影响 VAD/UniAD 模型输入，旧 checkpoint 可能不兼容。

## 4. 数据集迁移

原文强调：Bench2Drive 坐标系与 BEVFormer/UniAD/VAD 原始使用的数据坐标系差异很大。

关键脚本：

```text
mmcv/datasets/prepare_B2D.py
```

它负责把：

- world coordinate
- ego coordinate
- sensor coordinate
- vehicle coordinate
- bounding box coordinate
- sensor extrinsics

转换成模型期望的坐标约定。

注意事项：

- 坐标转换是迁移最容易出错的地方。
- 如果 box 朝向、轨迹方向、相机外参错了，模型评测可能完全失效。
- 不要只改路径就认为完成了数据迁移。

### 4.1 时间频率差异

原文说明：

- nuScenes keyframe 是 2Hz。
- Bench2Drive 是 10Hz，每帧都有标注。
- 为复现 UniAD/VAD，窗口长度设为 0.5s，窗口 shift 为 0.1s。

含义：

- 当前帧可以更密集地滑动选取。
- 未来轨迹和历史轨迹需要按 0.5s 时间间隔组织。
- 如果 RoadTailBench 采集频率不同，需要重新检查轨迹采样逻辑。

### 4.2 地图数据

原文说明：Bench2Drive 存储 vectorized maps，可参考代码提取一定范围内的地图元素。

如果迁移 RoadTailBench：

- CARLA 闭环只需要 CARLA 地图和 route/scenario 即可先跑。
- open-loop 数据集训练/评测则需要准备地图信息，并适配 `b2d_map_infos.pkl` 生成逻辑。
- RoadTailBench 地图元素类型最好能映射到现有 `map_classes`。

## 5. Team agent 迁移

原文说明：闭环评测需要设置传感器，从 CARLA 获取数据，计算模型所需输入，再把模型输出转换为 `carla.VehicleControl`。

当前已有 agent：

```text
team_code/uniad_b2d_agent.py
team_code/vad_b2d_agent.py
```

它们完成了：

- 注册 6 个 RGB 相机。
- 注册 IMU/GNSS/speed。
- 读取全局路线。
- 坐标转换。
- 构造模型输入。
- 调用模型。
- 从模型输出取未来轨迹。
- 使用 PID 输出 `steer/throttle/brake`。

如果换 RoadTailBench，优先检查：

- CARLA 地图能否返回正确 GNSS。
- route XML 中 waypoints 是否在道路上。
- `RoutePlanner` 是否能正常返回 `command_near_xy`。
- agent 中 `gps_to_location()` 是否适合你的地图 geo reference。
- 传感器外参是否和模型训练时一致。通常不要改传感器配置，否则输入分布会变。

## 6. RoadTailBench 迁移建议

如果只是闭环测试，不训练模型，推荐最小改动路径：

1. 让 CARLA 0.9.15 能加载 RoadTailBench 地图。
2. 在 Bench2Drive evaluation tools 中新增 RoadTailBench route XML。
3. 先不加复杂 scenario，跑一条空路线。
4. 使用原 UniAD/VAD agent 和原 checkpoint。
5. 检查车辆是否能沿路线行驶。
6. 再逐步加入 RoadTailBench 的交通参与者和场景。
7. 最后修改指标脚本，使其适配 RoadTailBench route 数量和场景类别。

如果还要 open-loop 数据集：

1. 采集 RoadTailBench 数据。
2. 转成类似 Bench2Drive 的目录和标注格式。
3. 修改 `prepare_B2D.py`。
4. 生成新的 `infos/*.pkl`。
5. 修改 config 的 `data_root` 和 `ann_file`。
6. 重新评测，必要时重新训练。

## 7. 不建议的做法

- 不建议直接把 RoadTailBench 数据路径替换到 config 里就跑训练。
- 不建议修改传感器外参后继续使用原 checkpoint 期待同等效果。
- 不建议修改类别数后强行加载旧 checkpoint。
- 不建议在没有验证 CARLA 地图、route、GNSS 坐标的情况下调模型代码。

## 8. 推荐检查顺序

```text
CARLA 能加载地图
→ route XML 能启动
→ ego 车能生成
→ agent sensors 正常
→ route planner 输出正常
→ 模型 checkpoint 加载正常
→ 模型输出轨迹正常
→ PID 控制正常
→ 指标 JSON 正常
```

