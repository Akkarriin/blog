---
date: '2026-01-20T22:19:41+08:00'
draft: true
title: '在 Ubuntu 24 NAS上用 ALAS + MAA + Redroid 挂机明日方舟'
tags: ["NAS", "Arknights", "MAA", "Docker", "Linux", "Redroid"]
categories: ["折腾"]
author: "safawf"
---

之前 Nas 上一直靠破解n卡的 vgpu 功能开 windows 虚拟机挂机明日方舟，最近沉迷视频编解码买了块 intel 的 a380,但是换成a380后好像不能搞 vgpu 了，只能想办法找找有没有 linux 下也可以完美挂机的方案，踩的一些坑总结如下。

## 硬件与系统环境

*   **CPU**: AMD Ryzen 5800
*   **GPU**: Intel Arc A380 
*   **OS**: Ubuntu 24.04 LTS
*   **容器管理**: Dockge

> **关于系统的选择**：
> 其他 Linux 发行版理论上都可以，Redroid 的文档里面给了很多系统的使用方法。Debian 也行。但我推荐 Ubuntu，因为在处理 Redroid 的 `binder` 设备绑定时，Ubuntu 比较省心。Debian 需要三个三个地绑定 binder devices，我不只一个游戏要挂机，就用的 Ubuntu。

## 一、Redroid 容器配置 

我选用的镜像是这个

*   **镜像**: `erstt/redroid:11.0.0_ndk_magisk_litegapps_ChromeOS`

在我这个配置下只有这个镜像运行的不错，赛马娘台服也能跑，但是学马仕跑不了，运行明日方舟有奇怪的bug,启动的时候会黑屏卡住。

**解决方案**：
完全靠运气或者手速。黑屏的时候在彻底卡住闪退之前，按一下安卓的“任务栏键”，然后再点回游戏，有概率能刷出画面。一旦进入主界面，后续挂机就一切正常了。

## 二、ALAS 与 MAA 部署

我所有 Compose 文件都用 **Dockge** 进行部署管理。选用 ALAS 不用 maa cli 的原因是我不会用，突然发现这个居然不仅有 web ui,还可以跑 maa 和 fgo 的脚本。项目地址是[AzurLaneAutoScript](https://github.com/LmeSzinc/AzurLaneAutoScript)
1.  **安装 ALAS**：
    根据 ALAS 文档 docker 部署的教程，直接 Clone 到 `/opt/stack` 目录，然后重命名为 `azurlaneautoscript`，这样方便 Dockge 识别和管理。

2.  **准备 MAA **：
    下载 MAA 的 Linux 版本，解压到 NAS 的某个目录（例如 `/home/yourname/MAA`）。
    *   **注意**：建议先进入 MAA 文件夹运行一下 `./maa`，根据提示更新资源或初始化之类的，确保 MAA 本身能正常工作。

## 三、修改 Dockerfile 

ALAS 原版的 `Dockerfile` 我死活构建不起来。

参考了项目 Issue 中大佬 **CJH-James** 的方案，修改了一份 `Dockerfile`。

请修改 `azurlaneautoscript/deploy/docker/Dockerfile` 内容如下：

```dockerfile
# 将基础镜像改为 Ubuntu 24.04 以支持 GLIBC 2.38/2.39
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# System dependencies
# 重点：添加了 libatomic1 等必要依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    wget git adb libgomp1 libatomic1 openssh-client ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Miniconda (Miniconda3-py313_25.9.1-3)
RUN wget -q https://repo.anaconda.com/miniconda/Miniconda3-py313_25.9.1-3-Linux-x86_64.sh \
    -O /tmp/miniconda.sh \
    && bash /tmp/miniconda.sh -b -p /opt/conda \
    && rm /tmp/miniconda.sh

# Add conda to PATH
ENV PATH="/opt/conda/bin:${PATH}"

# 切换为 conda-forge 源，移除 defaults
RUN conda config --system --remove channels defaults \
    && conda config --system --add channels conda-forge \
    && conda config --system --set channel_priority strict

RUN conda install -y mamba

WORKDIR /app/AzurLaneAutoScript

COPY requirements.txt /tmp/requirements.txt

# Install PyAV according to requirements.txt
# 单独安装 PyAV，锁定 Python 3.7
RUN AV_VERSION=$(grep -E '^av==' /tmp/requirements.txt | cut -d '=' -f 3) \
    && echo "Installing av == $AV_VERSION" \
    && mamba install -y "python=3.7" "av==$AV_VERSION" \
    && conda clean --all --yes

# Install other pip dependencies (skip av)
RUN grep -v '^av==' /tmp/requirements.txt > /tmp/requirements_no_av.txt \
    && pip install --no-cache-dir -r /tmp/requirements_no_av.txt \
    && rm /tmp/requirements.txt /tmp/requirements_no_av.txt

# 环境检查（Debug 用）
RUN echo "[ENV_CHECK_START]========================================" \
    && echo "Final Environment Verification Checks (After all installations):" \
    && echo "1. Python Version Check (should be 3.7.x):" \
    && python --version \
    && echo "2. Pip Path Check (should be /opt/conda/bin/pip):" \
    && which pip \
    && echo "3. Complete Package List (Conda and Pip):" \
    && conda list \
    && echo "[ENV_CHECK_END]=========================================="

CMD ["python", "gui.py"]
```
然后在 Dockge 里面点启动就行。

## 五、ALAS 内部配置

容器启动成功后，进入 ALAS 的 WebUI，还需要做几步修改。

### 1. 修改 MAA 路径
在 ALAS 设置中，找到 MAA 路径设置。
*   填写容器内的路径：`/app/MAA`

### 2. ADB 连接与触控方案
*   **模拟器地址**：填入你 NAS 的 IP + Redroid 端口（例如 `192.168.1.xxx:5555`）。
*   **触控方案**：须选择 **ADB Input**，其他方案我运行不起来。

### 3. 修复 ADB 连接失败 
按照默认配置，启动任务时控制台会报错：

```text
INFO     13:38:39.742 │ [Message.ConnectionInfo] 192.168.1.xxx:5555 ConnectFailed
CRITICAL 13:38:39.744 │ Request human takeover
```

**原因**：ALAS 默认寻找的 adb 路径和我们容器内安装的路径不一致，或者权限有问题。

**解决方法**：
需要修改 ALAS 的配置文件 `deploy.yaml`。
1.  找到文件路径：`/azurlaneautoscript/config/deploy.yaml`
2.  找到 `AdbExecutable:` 这一项。
3.  将其修改为系统自带的 ADB 路径：
    ```yaml
    AdbExecutable: /usr/bin/adb
    ```

保存文件后重启 ALAS 容器，再次连接，应该就可以了。
目前跑了三四天左右，还是很稳定的。只要不关机就这样一直挂着应该也没什么问题。

---