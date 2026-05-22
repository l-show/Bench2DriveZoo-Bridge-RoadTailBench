# 环境安装说明（中文增强版）

本文档对应 `docs/INSTALL.md`，在原文基础上补充 Ubuntu 服务器复现时的注意事项。Bench2DriveZoo 对环境版本比较敏感，建议严格按本文版本配置。

## 1. 创建 Conda 环境

原文要求严格使用 Python 3.8：

```bash
conda create -n b2d_zoo python=3.8
conda activate b2d_zoo
```

注意事项：

- 不建议使用 Python 3.9/3.10/3.11。该项目包含较多旧版 OpenMMLab 风格代码和 CUDA 扩展，Python 版本偏离后容易出现编译或 import 问题。
- 后续所有安装、编译和运行命令都应在 `b2d_zoo` 环境里执行。
- 如果服务器已有多个 conda，先确认 `which python` 和 `which pip` 指向当前环境。

检查：

```bash
which python
python --version
which pip
```

## 2. 安装 CUDA Toolkit

原文命令：

```bash
conda install -c "nvidia/label/cuda-11.8.0" cuda-toolkit
```

注意事项：

- 官方推荐 CUDA 11.8。
- 服务器 NVIDIA 驱动版本需要支持 CUDA 11.8。
- 如果系统已经安装 `/usr/local/cuda-11.8`，也可以使用系统 CUDA，但要保证 PyTorch、CUDA、编译器版本匹配。

检查：

```bash
nvidia-smi
nvcc --version
```

如果 `nvcc` 不存在，但你使用的是 conda `cuda-toolkit`，可以设置：

```bash
export CUDA_HOME=$CONDA_PREFIX
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib:$LD_LIBRARY_PATH
```

如果你使用系统 CUDA：

```bash
export CUDA_HOME=/usr/local/cuda-11.8
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
```

## 3. 安装 PyTorch

原文命令：

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

安装后检查：

```bash
python - <<'PY'
import torch
print("torch:", torch.__version__)
print("cuda available:", torch.cuda.is_available())
print("torch cuda:", torch.version.cuda)
print("gpu count:", torch.cuda.device_count())
PY
```

期望：

```text
cuda available: True
torch cuda: 11.8
```

注意事项：

- 如果 `torch.cuda.is_available()` 是 `False`，先不要继续安装本仓库，优先修复 CUDA/PyTorch。
- 不建议混装 conda 的 PyTorch 和 pip 的 PyTorch。
- 如果你在服务器无 root 权限，仍然可以用 conda CUDA toolkit + pip PyTorch。

## 4. 设置编译环境变量

原文提示：

```bash
# cuda 11.8 and GCC 9.4 is strongly recommended. Otherwise, it might encounter errors.
export PATH=YOUR_GCC_PATH/bin:$PATH
export CUDA_HOME=YOUR_CUDA_PATH/
```

推荐使用 GCC/G++ 9：

```bash
gcc --version
g++ --version
```

如果系统不是 GCC 9，可安装：

```bash
sudo apt update
sudo apt install gcc-9 g++-9 -y
```

然后在当前 shell 中设置：

```bash
export CC=/usr/bin/gcc-9
export CXX=/usr/bin/g++-9
export PATH=/usr/bin:$PATH
```

注意事项：

- 本项目包含 `mmcv/ops`、`mmcv/layers` 等 C++/CUDA 扩展，编译器版本不匹配会导致 `pip install -e .` 失败。
- Ubuntu 22.04 默认 GCC 可能较新，建议明确指定 GCC 9。
- 如果你用集群模块系统，例如 `module load cuda/11.8 gcc/9.4`，以集群规则为准。

## 5. 安装 ninja 和 packaging

原文命令：

```bash
pip install ninja packaging
```

注意事项：

- `ninja` 可以加速 C++/CUDA 扩展编译。
- 如果编译失败，先确认 `ninja --version` 可用。

## 6. 安装本仓库

在仓库根目录执行：

```bash
pip install -v -e .
```

注意事项：

- 这个步骤会编译本地扩展，耗时可能较长。
- 如果失败，优先看日志中第一个 C++/CUDA 编译错误。
- 不建议跳过 `-v`，详细日志有助于定位问题。
- 如果你后续修改了 C++/CUDA 扩展，可能需要重新执行该命令。

常见错误：

```text
nvcc fatal: Unsupported gpu architecture
```

可能原因是 CUDA 版本和 GPU 架构不匹配，或编译参数不支持当前显卡。可检查 PyTorch 支持的 CUDA 架构和服务器 GPU 型号。

```text
fatal error: cuda_runtime_api.h: No such file or directory
```

通常是 `CUDA_HOME` 未设置或 CUDA toolkit 不完整。

```text
gcc: error: unrecognized command-line option
```

通常是 GCC 版本不合适。

## 7. 准备预训练权重和 checkpoint

原文要求创建 `ckpts` 目录：

```bash
mkdir ckpts
```

建议路径：

```text
Bench2DriveZoo/ckpts/
```

### 7.1 Backbone/预训练权重

这些权重主要用于训练或部分配置初始化：

| 文件 | 用途 | 下载 |
|---|---|---|
| `resnet50-19c8e357.pth` | VAD 图像 backbone 预训练权重 | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/resnet50-19c8e357.pth；百度云: https://pan.baidu.com/s/1LlSrbYvghnv3lOlX1uLU5g?pwd=1234 |
| `r101_dcn_fcos3d_pretrain.pth` | BEVFormer 相关预训练权重 | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/r101_dcn_fcos3d_pretrain.pth；百度云: https://pan.baidu.com/s/1o7owaQ5G66xqq2S0TldwXQ?pwd=1234 |

### 7.2 模型 checkpoint

这些是你不训练、直接复现测试时最重要的权重：

| 模型 | 文件名 | 用途 | 下载 |
|---|---|---|---|
| UniAD-Tiny | `uniad_tiny_b2d.pth` | UniAD 闭环/开环测试 | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/uniad_tiny_b2d.pth；百度云: https://pan.baidu.com/s/1psr7AKYHD7CitZ30Bz-9sA?pwd=1234 |
| UniAD-Base | `uniad_base_b2d.pth` | UniAD 闭环/开环测试 | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/uniad_base_b2d.pth；百度云: https://pan.baidu.com/s/11p9IUGqTax1f4W_qsdLCRw?pwd=1234 |
| VAD | `vad_b2d_base.pth` | VAD 闭环/开环测试 | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/vad_b2d_base.pth；百度云: https://pan.baidu.com/s/1rK7Z_D-JsA7kBJmEUcMMyg?pwd=1234 |
| BEVFormer-Tiny | `bevformer_tiny_b2d.pth` | BEVFormer open-loop 感知评测 | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/bevformer_tiny_b2d.pth；百度云: https://pan.baidu.com/s/1TWMs9YgKYm2DF5YfXF8i3g?pwd=1234 |
| BEVFormer-Base | `bevformer_base_b2d.pth` | BEVFormer open-loop 感知评测 | Hugging Face: https://huggingface.co/rethinklab/Bench2DriveZoo/blob/main/bevformer_base_b2d.pth；百度云: https://pan.baidu.com/s/1Y4VkE1gc8BU0zJ4z2fmIkQ?pwd=1234 |

下载完成后确认：

```bash
ls -lh ckpts
```

注意事项：

- checkpoint 文件名应尽量保持和文档命令一致。
- 闭环复现 UniAD/VAD 时，必须下载对应 checkpoint。
- 当前仓库只有 UniAD 和 VAD 提供 CARLA 闭环 agent；BEVFormer checkpoint 用于 open-loop 感知评测，不是直接控制车辆的闭环 agent。

## 8. 安装 CARLA 用于闭环评测

原文命令：

```bash
mkdir carla
cd carla
wget https://carla-releases.s3.us-east-005.backblazeb2.com/Linux/CARLA_0.9.15.tar.gz
tar -xvf CARLA_0.9.15.tar.gz
cd Import && wget https://carla-releases.s3.us-east-005.backblazeb2.com/Linux/AdditionalMaps_0.9.15.tar.gz
cd .. && bash ImportAssets.sh
export CARLA_ROOT=YOUR_CARLA_PATH
```

如果你已经下载好了 CARLA 0.9.15，可以跳过下载和解压，只设置：

```bash
export CARLA_ROOT=/path/to/CARLA_0.9.15
```

把 CARLA Python egg 写入当前 conda 环境：

```bash
echo "$CARLA_ROOT/PythonAPI/carla/dist/carla-0.9.15-py3.7-linux-x86_64.egg" \
  >> $CONDA_PREFIX/lib/python3.8/site-packages/carla.pth
```

检查：

```bash
python - <<'PY'
import carla
print(carla.__file__)
PY
```

注意事项：

- 文档使用的是 Linux 版 CARLA，不是 Windows 版。
- 虽然 egg 文件名是 `py3.7`，官方说明中 Python 3.8 也可用。
- 闭环评测不仅需要 CARLA，还需要 Bench2Drive evaluation tools。
- 服务器无显示器时通常需要 `-RenderOffScreen`。

单独启动 CARLA 测试：

```bash
cd $CARLA_ROOT
./CarlaUE4.sh -RenderOffScreen -nosound -carla-rpc-port=20000
```

## 9. 安装后的最小检查清单

```bash
conda activate b2d_zoo
python --version
python -c "import torch; print(torch.cuda.is_available())"
python -c "import carla; print(carla.__file__)"
cd /path/to/Bench2DriveZoo
python -c "import mmcv; print(mmcv.__file__)"
ls -lh ckpts
```

如果这些检查都正常，再继续数据准备、open-loop 评测或 CARLA closed-loop 评测。

