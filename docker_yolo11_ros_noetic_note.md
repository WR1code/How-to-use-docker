# Docker 构建 YOLO11 + ROS Noetic + CUDA 环境笔记

本文记录在 Ubuntu 主机上使用 Docker 构建一个包含 **CUDA 12.8、cuDNN、ROS Noetic、Miniconda、YOLO11 Conda 环境** 的镜像过程，以及构建过程中遇到的 Docker 代理问题和解决方法。

---

## 1. 文件结构

假设项目目录为：

```bash
~/Downloads/uubuntu20.04docker
```

目录下至少包含：

```bash
Dockerfile
yolo11_env.tar.gz
```

其中：

* `Dockerfile`：用于构建 Docker 镜像的配置文件
* `yolo11_env.tar.gz`：已经打包好的 Conda 环境压缩包
* 该压缩包不是 Docker 镜像，一般不应该使用 `docker load` 导入，而应该在 `Dockerfile` 中通过 `ADD` 解压进容器

---

## 2. Dockerfile 内容说明

本次使用的基础镜像为：

```dockerfile
FROM nvidia/cuda:12.8.0-cudnn-devel-ubuntu20.04
```

该镜像提供：

* Ubuntu 20.04
* CUDA 12.8
* cuDNN
* NVIDIA GPU 支持环境

Dockerfile 主要完成以下工作：

1. 替换 Ubuntu apt 源为阿里云源
2. 安装基础工具：`curl`、`wget`、`git`、`vim`、`tmux`、`sudo` 等
3. 安装 ROS Noetic Desktop Full
4. 安装 `rosdepc`，用于替代普通 `rosdep` 解决国内网络问题
5. 安装 Miniconda
6. 解压 `yolo11_env.tar.gz` 到 Conda 环境目录
7. 执行 `conda-unpack` 修复环境中的绝对路径
8. 配置 `.bashrc`，进入容器后自动加载 ROS 和 Conda 环境

---

## 3. 构建 Docker 镜像

进入 Dockerfile 所在目录：

```bash
cd ~/Downloads/uubuntu20.04docker
```

确认文件存在：

```bash
ls -lh
```

应该能看到：

```bash
Dockerfile
yolo11_env.tar.gz
```

开始构建镜像：

```bash
docker build -t yolo11-noetic:cuda12.8 .
```

如果 Docker 需要 sudo 权限，则使用：

```bash
sudo docker build -t yolo11-noetic:cuda12.8 .
```

构建完成后查看镜像：

```bash
docker image ls
```

应该能看到类似：

```bash
REPOSITORY       TAG        IMAGE ID       CREATED        SIZE
yolo11-noetic    cuda12.8   xxxxxxxxxxxx   xx minutes ago xxGB
```

---

## 4. 启动容器

推荐使用以下命令启动容器：

```bash
docker run -it \
  --gpus all \
  --name yolo11_ros \
  --net=host \
  --ipc=host \
  -v ~/Downloads:/workspace \
  yolo11-noetic:cuda12.8
```

参数说明：

* `--gpus all`：允许容器使用 NVIDIA GPU
* `--name yolo11_ros`：给容器命名为 `yolo11_ros`
* `--net=host`：使用宿主机网络，方便 ROS 通信
* `--ipc=host`：共享 IPC，提高部分深度学习/ROS 程序兼容性
* `-v ~/Downloads:/workspace`：将宿主机的 `Downloads` 目录挂载到容器内 `/workspace`
* `yolo11-noetic:cuda12.8`：指定要运行的镜像

如果需要 sudo：

```bash
sudo docker run -it \
  --gpus all \
  --name yolo11_ros \
  --net=host \
  --ipc=host \
  -v ~/Downloads:/workspace \
  yolo11-noetic:cuda12.8
```

---

## 5. 进入容器后检查环境

进入容器后，终端一般会变成：

```bash
root@xxxx:/workspace#
```

检查 ROS：

```bash
roscore
```

如果 `roscore` 能正常启动，说明 ROS Noetic 环境正常。

检查 Conda 环境：

```bash
conda info --envs
```

应该能看到：

```bash
yolo11
```

激活环境：

```bash
conda activate yolo11
```

检查 Python：

```bash
python --version
```

检查 PyTorch 是否能使用 CUDA：

```bash
python -c "import torch; print(torch.cuda.is_available()); print(torch.version.cuda)"
```

如果输出：

```bash
True
```

说明容器内 GPU 可用。

---

## 6. 退出和重新进入容器

退出容器：

```bash
exit
```

下次重新进入已经创建好的容器，不要再 `docker run`，而是使用：

```bash
docker start -ai yolo11_ros
```

如果需要 sudo：

```bash
sudo docker start -ai yolo11_ros
```

---

## 7. 容器名字冲突问题

如果重新 `docker run` 时出现：

```bash
Conflict. The container name "/yolo11_ros" is already in use
```

说明已经存在同名容器。

可以直接进入旧容器：

```bash
docker start -ai yolo11_ros
```

如果确定不要旧容器，可以删除后重建：

```bash
docker rm yolo11_ros
```

然后重新运行：

```bash
docker run -it \
  --gpus all \
  --name yolo11_ros \
  --net=host \
  --ipc=host \
  -v ~/Downloads:/workspace \
  yolo11-noetic:cuda12.8
```

---

## 8. Docker 代理问题记录

构建时曾遇到如下错误：

```bash
ERROR: failed to build: failed to solve:
nvidia/cuda:12.8.0-cudnn-devel-ubuntu20.04:
failed to resolve source metadata for docker.io/nvidia/cuda:
failed to do request:
Head "https://registry-1.docker.io/v2/nvidia/cuda/manifests/12.8.0-cudnn-devel-ubuntu20.04":
proxyconnect tcp: dial tcp 127.0.0.1:10808:
connect: connection refused
```

原因是 Docker 正在使用旧代理端口：

```bash
127.0.0.1:10808
```

但是本机代理实际端口是：

```bash
127.0.0.1:7897
```

因此 Docker 拉取基础镜像失败。

---

## 9. 修改 Docker 代理端口

先查看 Docker 当前代理配置：

```bash
sudo systemctl show --property=Environment docker --no-pager
```

如果看到：

```bash
HTTP_PROXY=http://127.0.0.1:10808
HTTPS_PROXY=http://127.0.0.1:10808
```

说明 Docker 仍然在使用旧代理。

先创建 Docker systemd 配置目录：

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
```

删除旧配置：

```bash
sudo rm -f /etc/systemd/system/docker.service.d/*.conf
```

重新写入代理配置：

```bash
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf > /dev/null <<'EOF'
[Service]
Environment=
Environment="HTTP_PROXY=http://127.0.0.1:7897"
Environment="HTTPS_PROXY=http://127.0.0.1:7897"
Environment="ALL_PROXY=socks5://127.0.0.1:7897"
Environment="NO_PROXY=localhost,127.0.0.1,::1"
EOF
```

重新加载 systemd 并重启 Docker：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

检查是否修改成功：

```bash
sudo systemctl show --property=Environment docker --no-pager
```

正确结果应该包含：

```bash
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
ALL_PROXY=socks5://127.0.0.1:7897
```

确认本机代理端口正在监听：

```bash
ss -lntp | grep 7897
```

如果没有输出，说明代理软件没有正常启动，需要先打开 Clash、Clash Verge、mihomo、v2rayN 等代理软件。

---

## 10. 单独测试 Docker 拉取镜像

在重新构建之前，可以先测试 Docker 是否能拉取 CUDA 镜像：

```bash
docker pull nvidia/cuda:12.8.0-cudnn-devel-ubuntu20.04
```

如果可以正常拉取，再执行构建：

```bash
cd ~/Downloads/uubuntu20.04docker
docker build -t yolo11-noetic:cuda12.8 .
```

---

## 11. 清理错误导入的镜像

如果之前误执行了：

```bash
docker load -i yolo11_env.tar.gz
```

一般不会损坏文件。

可以查看本地镜像：

```bash
docker image ls
```

如果出现无用的 `<none>` 镜像，可以通过镜像 ID 删除：

```bash
docker rmi 镜像ID
```

例如：

```bash
docker rmi abc123456789
```

注意不要误删正在使用的镜像，例如：

```bash
yolo11-noetic:cuda12.8
```

---

## 12. 常用 Docker 命令汇总

查看镜像：

```bash
docker image ls
```

查看容器：

```bash
docker ps -a
```

启动已有容器：

```bash
docker start -ai yolo11_ros
```

停止容器：

```bash
docker stop yolo11_ros
```

删除容器：

```bash
docker rm yolo11_ros
```

删除镜像：

```bash
docker rmi 镜像ID
```

重新构建镜像：

```bash
docker build -t yolo11-noetic:cuda12.8 .
```

---

## 13. 总结

本次环境构建的核心流程是：

```bash
cd ~/Downloads/uubuntu20.04docker
docker build -t yolo11-noetic:cuda12.8 .
docker run -it --gpus all --name yolo11_ros --net=host --ipc=host -v ~/Downloads:/workspace yolo11-noetic:cuda12.8
```

构建失败的主要原因不是 Dockerfile 错误，而是 Docker 的代理端口配置错误。

解决方法是将 Docker systemd 代理配置中的旧端口 `10808` 改为当前代理端口 `7897`，然后重启 Docker 服务。

最终目标是得到一个可以同时使用：

* CUDA 12.8
* cuDNN
* ROS Noetic
* Conda
* YOLO11 环境
* NVIDIA GPU

的 Docker 开发环境。
