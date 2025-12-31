文档版本： 1.0

适用场景： 基于华为 Atlas 300V Pro 硬件环境部署 mts 转码应用程序

关键词： Docker, Atlas 300V, CANN, Systemd, ARM64

---

## 1. 概述

本文档详细阐述了在 ARM 架构服务器上，利用 Docker 容器化部署基于华为 Atlas 300V Pro 推理卡的 `mts` 转码程序的完整流程。文档涵盖环境准备、镜像构建、容器启动配置及注意事项。

---

## 2. 环境准备工作

在开始部署前，请确保宿主机满足以下条件，并准备好相应的软件包。

### 2.1 软件与依赖

- **基础镜像**：支持 systemd 的 ARM 架构基础镜像（本文示例：`ubuntu-systemctl-arm-build:20.04`）。
    
- **应用程序包**：`mts` 转码程序及其依赖（打包为 `.tar.gz`）。
    
- **华为 CANN 软件**：适用于 ARM64 的 NPU 运行时包（例如：`Ascend-cann-nnrt_8.3.RC1_linux-aarch64.run`）。
    

### 2.2 脚本与配置

需要准备以下辅助脚本以支持容器内的服务管理：

- `launch.sh`: 容器环境初始化或服务注册脚本。
    
- `rc-local.service`: Systemd 服务描述文件（配合 `launch.sh` 使用）。
    
- `version.txt`: 应用程序版本标识文件。
    

---

## 3. 构建 Docker 镜像

### 3.1 目录结构规划

建议创建一个干净的工作目录（如 `atlas_deployment`），并按照以下结构组织文件，以保持构建上下文清晰。


```plaintext
atlas_deployment/
├── Dockerfile                  # 镜像构建描述文件
├── Ascend-cann-nnrt_*.run      # CANN 软件包
├── launch.sh                   # 启动/注册脚本
├── rc-local.service            # Systemd 服务文件
└── media/                      # 资源子目录
    ├── mts_arm64.tar.gz        # 应用程序包
    └── version.txt             # 版本文件
```

### 3.2 编写 Dockerfile

在项目根目录下创建名为 `Dockerfile` 的文件。以下配置已针对 Atlas 环境进行了优化：

```dockerfile
  FROM ubuntu-systemctl:20.04
     MAINTAINER  lyc
     
     WORKDIR /app
     
     # 设置驱动路径环境变量
     ARG ASCEND_BASE=/usr/local/Ascend
     ENV LD_LIBRARY_PATH=\
     $ASCEND_BASE/driver/lib64:\
     $ASCEND_BASE/driver/lib64/common:\
     $ASCEND_BASE/driver/lib64/driver:\
     $LD_LIBRARY_PATH
     
     # 1 安装CANN
     ARG CANN_PKG=Ascend-cann-nnrt_*.run
     COPY ${CANN_PKG} .
     RUN chmod +x $CANN_PKG && \
      ./$CANN_PKG --quiet --install --install-path=$ASCEND_BASE --install-for-all
      
     ##安装vcs-file
     COPY media/mts_arm64.tar.gz .
     RUN tar -xvf mts_arm64.tar.gz -C /opt/ \
     && chmod a+x /opt/mts/start.sh \
     && chmod a+x /opt/mts/stop.sh \
     && chmod a+x /opt/mts/mts
     
     COPY media/version.txt /opt/mts/
     
     ####开机启动
     COPY launch.sh .
     COPY rc-local.service .
     RUN chmod a+x launch.sh \
     && sh launch.sh
     
     ##清理
     RUN rm mts_arm64.tar.gz launch.sh rc-local.service ${CANN_PKG}
     
     CMD /usr/bin/systemctl
```

### 3.3 执行镜像构建

1. **加载基础镜像**（如果基础镜像尚未导入）：
    
    ```
    docker load -i ubuntu-systemctl-arm64_20.04.tar
    ```
    
2. 构建业务镜像：
    
    在 atlas_deployment 目录下执行构建命令。
    
    Bash
    
    ```
    # 语法: docker build -f [Dockerfile文件名] -t [镜像名:标签] .
    docker build -f Dockerfile -t atlas-transcode:1.0.0 .
    ```
    

---

## 4. 运行 Docker 容器

### 4.1 启动命令

启动容器时需挂载 NPU 设备文件及驱动路径，以确保容器能直接访问 Atlas 硬件。

> ⚠️ 重要提示：
> 
> 一个物理 NPU 设备（如 /dev/davinci2）通常建议仅挂载到一个容器中，避免资源冲突。

Bash

```
docker run -idt \
  --net=host \
  --ipc=host \
  --name=atlas_node_1 \
  -h=atlas_node_1 \
  --privileged \
  \
  # --- NPU 设备映射 (根据实际设备号调整) ---
  --device=/dev/davinci2 \
  --device=/dev/davinci_manager \
  --device=/dev/hisi_hdc \
  --device=/dev/devmm_svm \
  \
  # --- 驱动与工具挂载 (必须与宿主机一致) ---
  -v /usr/local/Ascend/driver:/usr/local/Ascend/driver \
  -v /etc/ascend_install.info:/etc/ascend_install.info \
  -v /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info \
  -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
  -v /usr/local/dcmi:/usr/local/dcmi \
  \
  # --- 业务数据与日志挂载 ---
  -v /root/data/debs:/var/debs \
  -v /root/data/atlas1/conf:/opt/mts/conf \
  -v /root/data/atlas1/ssl:/opt/mts/ssl \
  -v /root/data/atlas1/log:/opt/mts/log \
  -v /root/data:/root/data \
  \
  atlas-transcode:1.0.0 \
  /usr/bin/systemctl
```

### 4.2 关键参数详解

|**参数类别**|**参数**|**说明**|
|---|---|---|
|**基础配置**|`--net=host`|使用宿主机网络栈，避免端口映射的性能损耗。|
||`--ipc=host`|共享宿主机 IPC 命名空间，便于高性能通信。|
||`--privileged`|开启特权模式，允许容器访问硬件设备。|
|**设备映射**|`--device`|将 `/dev/davinci*` 等 NPU 核心设备透传至容器。**请务必确认宿主机上实际存在的设备编号。**|
|**驱动挂载**|`-v .../driver`|**核心配置**。必须将宿主机的 Ascend 驱动目录挂载至容器相同路径，确保版本一致性。|
||`-v .../npu-smi`|挂载 NPU 管理工具，便于在容器内查询卡状态。|
|**业务挂载**|`-v .../conf`|挂载配置文件，实现配置持久化和容器外修改。|
||`-v .../log`|挂载日志目录，便于宿主机收集和监控日志。|
|**启动命令**|`/usr/bin/systemctl`|覆盖默认 CMD，确保容器作为类似虚拟机的系统运行，以支持后台服务管理。|

---

## 5. 验证与运维

### 5.1 进入容器

容器启动后，通过以下命令进入交互式终端进行验证：

Bash

```
docker exec -it atlas_node_1 /bin/bash
```

### 5.2 验证 NPU 状态

在容器内部执行以下命令，检查 NPU 是否被正确识别：

Bash

```
npu-smi info
```

_如果输出显示了 NPU 设备列表且状态正常（Health 栏显示 OK），则说明驱动挂载和设备透传成功。_

### 5.3 检查业务服务

检查 `mts` 服务是否已通过 systemd 自动启动：

Bash

```
ps -ef | grep mts
```

---

