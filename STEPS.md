以下是根据您的指正，将 `setuptools` 和 `wheel` 的预下载与预安装步骤补充到正确位置后的完整文档。

---

# 断网环境复现YOLOv8训练环境操作总结（最终完整版）

## 一、总体思路

1. **在联网机器上**：根据环境A的`pip freeze`结果，下载所有Python包（wheel）、系统RPM包、模型文件等。
2. **将文件拷贝到断网机器**：通过物理介质（U盘/移动硬盘）传输。
3. **在断网容器B中**：离线安装RPM包和Python包，处理版本冲突，最终验证训练环境。

---

## 二、在联网机器上的操作

### 1. 提取环境A的依赖清单

在环境A的容器内执行：
```bash
pip freeze > requirements.txt
```

### 2. 清理依赖清单，去掉本地路径引用

```bash
# 过滤掉所有以 file:// 开头的行（这些是CANN镜像自带的本地wheel，不需要重新安装）
grep -v 'file://' requirements.txt > requirements_clean.txt

# 查看剩余内容，确认只有PyPI包
cat requirements_clean.txt
```

### 3. 处理numpy版本冲突，准备下载清单

由于`opencv-python-headless==5.0.0.93`要求`numpy>=2`，而最终需要`numpy==1.26.4`，因此分两步下载：

```bash
# 创建一个用于下载的临时清单，去掉 numpy==1.26.4 这一行，让pip自动解析出numpy 2.x
grep -v '^numpy==' requirements_clean.txt > requirements_for_download.txt
```

### 4. 下载所有Python包（除numpy外）

```bash
# 在联网机器上创建目录
mkdir -p ./packages

# 下载所有包（指定平台、Python版本，只下载二进制wheel）
pip download -r requirements_for_download.txt -d ./packages \
  -i https://pypi.tuna.tsinghua.edu.cn/simple \
  -f https://mirrors.huaweicloud.com/ascend/torch_npu/
```

> 注意：如果某些包没有aarch64的wheel，可能需要调整版本。本次操作中`contourpy`和`opencv-python-headless`都成功下载了wheel。

### 5. 单独下载numpy 1.26.4的wheel

```bash
pip download numpy==1.26.4 -d ./packages \
  --platform manylinux2014_aarch64 \
  --python-version 3.11 \
  --only-binary=:all: \
  -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 6. 额外下载 setuptools 和 wheel（用于Pillow源码编译）

```bash
pip download setuptools wheel -d ./packages \
  --platform manylinux2014_aarch64 \
  --python-version 3.11 \
  --only-binary=:all: \
  -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 7. 下载系统RPM依赖（用于Pillow编译）

```bash
# 安装dnf下载插件
dnf install -y dnf-plugins-core --setopt=sslverify=0

# 创建RPM目录
mkdir -p /tmp/offline_rpms

# 下载所有RPM包及其依赖（必须加 --resolve 和 --alldeps）
dnf download --resolve --alldeps --destdir /tmp/offline_rpms \
  zlib-devel libjpeg-turbo-devel libpng-devel freetype-devel libtiff-devel \
  --setopt=sslverify=0
```

### 8. 下载预训练模型和项目文件

```bash
# 下载yolov8n.pt
wget https://github.com/ultralytics/assets/releases/download/v8.2.0/yolov8n.pt

# 打包项目代码（假设项目在 /models/.../Yolov8_for_PyTorch）
tar -czvf yolo_project.tar.gz -C /models/y00546703-personal-use/retest0818 Yolov8_for_PyTorch
```

### 9. 整理传输文件

将以下内容拷贝到物理介质：
- `packages/` 目录（所有Python wheel，含 numpy 1.26.4、numpy 2.x、setuptools、wheel）
- `offline_rpms/` 目录（所有RPM包）
- `requirements_for_download.txt` 和 `requirements_clean.txt`
- `yolov8n.pt`
- `yolo_project.tar.gz`

---

## 三、在断网容器B中的操作

### 1. 启动断网容器

```bash
docker run -d \
  --name yolo_offline \
  --network none \
  --shm-size=32g \
  --privileged \
  --device=/dev/davinci_manager \
  --device=/dev/hisi_hdc \
  --device=/dev/devmm_svm \
  --device=/dev/davinci0 \
  --device=/dev/davinci1 \
  --device=/dev/davinci2 \
  --device=/dev/davinci3 \
  --device=/dev/davinci4 \
  --device=/dev/davinci5 \
  --device=/dev/davinci6 \
  --device=/dev/davinci7 \
  -v /usr/local/Ascend/driver:/usr/local/Ascend/driver:ro \
  -v /usr/local/sbin:/usr/local/sbin:ro \
  -v /你的传输目录:/mnt/transfer:rw \
  swr.cn-south-1.myhuaweicloud.com/ascendhub/cann:9.0.0-910b-openeuler24.03-py3.11-devel \
  sleep infinity
```

进入容器：
```bash
docker exec -it yolo_offline /bin/bash
```

### 2. 安装系统RPM依赖

```bash
cd /mnt/transfer/offline_rpms

# 使用dnf本地安装（推荐）
dnf install -y --disablerepo=* ./*.rpm

# 如果报依赖错误，尝试使用rpm强制安装（不推荐，但可应急）
# rpm -ivh --nodeps ./*.rpm
```

验证：
```bash
rpm -qa | grep -E "zlib-devel|libjpeg-turbo-devel|libpng-devel|freetype-devel|libtiff-devel"
```

### 3. 处理Python环境

#### 3.1 卸载已有的numpy

```bash
pip uninstall numpy -y
```

#### 3.2 先安装 setuptools 和 wheel（关键步骤）

```bash
cd /mnt/transfer
pip install --no-index --find-links=./packages setuptools wheel
```

这一步确保后续 Pillow 从源码编译时，pip 的构建环境能使用本地的 setuptools，避免因断网导致构建依赖安装失败。

#### 3.3 安装除numpy外的所有Python包

```bash
pip install --no-index --find-links=./packages -r requirements_for_download.txt
```

此时pip会自动安装numpy 2.x以满足opencv等依赖。

#### 3.4 强制降级numpy到1.26.4

```bash
pip install --no-index --find-links=./packages --force-reinstall numpy==1.26.4
```

#### 3.5 处理OpenCV完整版冲突

安装过程中可能同时装上了`opencv-python`（完整版）和`opencv-python-headless`，导致`libxcb.so.1`错误。

```bash
# 卸载完整版
pip uninstall opencv-python -y

# 强制重装headless版本
pip install --no-index --find-links=./packages --force-reinstall opencv-python-headless==5.0.0.93
```

验证：
```bash
python3 -c "import cv2; print('cv2:', cv2.__version__)"
```

### 4. 验证环境

```bash
python3 -c "import numpy; print('numpy:', numpy.__version__)"
python3 -c "import torch; import torch_npu; print('torch:', torch.__version__); print('npu:', torch.npu.is_available())"
python3 -c "import cv2; print('cv2:', cv2.__version__)"
```

预期输出：
- numpy: 1.26.4
- torch: 2.1.0
- npu: True
- cv2: 5.0.0

### 5. 准备项目并运行训练

```bash
# 解压项目文件
cd /models/y00546703-personal-use/retest0818
tar -xzvf /mnt/transfer/yolo_project.tar.gz

# 进入项目目录
cd Yolov8_for_PyTorch

# 如果本地有ultralytics文件夹，重命名（与A机器一致）
mv ultralytics ultralytics_local_backup 2>/dev/null

# 加载CANN环境变量
source /usr/local/Ascend/ascend-toolkit/set_env.sh

# 单卡1个epoch快速验证
export ASCEND_RT_VISIBLE_DEVICES=7
yolo detect train data=mydownload.yaml model=yolov8n.pt epochs=1 imgsz=640 batch=16 device=npu:0
```

---

## 四、关键问题与解决

| 问题 | 解决方法 |
| :--- | :--- |
| `pip download` 报 `file://` 本地路径不存在 | 用 `grep -v 'file://'` 过滤掉这些行 |
| numpy与opencv版本冲突 | 分两步下载：先下载除numpy外的所有包（自动选numpy 2.x），再单独下载numpy 1.26.4；安装时先装所有包，再强制降级numpy |
| `dnf download` 缺少依赖 | 必须同时使用 `--resolve` 和 `--alldeps` 参数 |
| Pillow编译缺少zlib | 提前下载并安装 `zlib-devel` 等RPM包 |
| `setuptools` 缺失导致构建失败 | **在联网机器上下载 `setuptools` 和 `wheel`，并在断网容器中优先安装**（见第二章第6步、第三章第3.2步） |
| OpenCV完整版导致`libxcb.so.1`错误 | 卸载`opencv-python`，强制重装`opencv-python-headless` |
| `yolo` 命令无法识别`npu:` | 升级`ultralytics`，并重命名本地`ultralytics`文件夹 |

---

## 五、最终结果

在断网容器B中，成功复现了与环境A一致的YOLOv8训练环境，并可以正常启动训练。所有依赖均通过离线方式安装，未使用任何网络连接。

---

**备注**：以上命令中的路径需根据实际挂载目录调整。核心原则是：**联网机器负责下载所有依赖，断网机器负责离线安装**，并通过分步处理版本冲突来确保环境一致性。
