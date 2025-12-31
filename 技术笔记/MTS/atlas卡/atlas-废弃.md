# 华为 Atlas 300V Pro 推理卡 Docker 部署流程

本文档旨在整理和说明使用 Docker 部署基于华为 Atlas 300V Pro 推理卡的应用程序（以`mts`转码程序为例）的流程，重点结合了 Dockerfile 构建和容器运行的详细步骤及解释。

---

## 一、 准备工作

在开始部署之前，请确保准备好以下各项：

1.  **应用程序**: `mts` 转码程序及其所有依赖文件。
2.  **昇腾运行库 (Ascend Runtime)**: 包含 `libascendcl.so` 等核心库的压缩包 (`ascend.tar.gz`)，通常可从 CANN Toolkit (昇腾开发套件) 中获取。
3.  **基础 Docker 镜像**: 一个适用于目标环境（ARM 架构、支持 systemd）的基础镜像。本文档后续示例基于 `ubuntu-systemctl-arm-build:20.04`。
4.  **相关脚本和配置文件**:
    * `version.txt` (用于 mts)
    * `launch.sh` (容器启动脚本)
    * `rc-local.service` (systemd 服务文件，如果 `launch.sh` 需要)

---

## 二、 创建 Docker 环境与镜像

1. **创建工作目录**

   * 在你的工作环境中创建一个用于存放 Dockerfile 和相关资源的项目目录。

   ```bash
   cd /home/ # 或者你选择的其他路径
   mkdir atlas_deployment
   cd atlas_deployment
   ```

2. **准备资源文件**

   * 在项目目录下创建一个 `media` 子目录。
   * 将以下文件放入 `media` 目录中：
     * `ascend.tar.gz` (包含昇腾运行库)
     * `mts.tar.gz` (包含 mts 程序和依赖)
     * `version.txt`
   * 将 `launch.sh` 和 `rc-local.service` 文件直接放在项目目录下（与 `Dockerfile` 同级）。

3. **编写 Dockerfile**

   * 在项目目录下创建 `Dockerfile` 文件，内容如下。**请仔细阅读注释，特别是关于 Ascend 库路径，并根据你的实际文件进行修改。**

     ```dockerfile
     FROM ubuntu-systemctl-arm-build:20.04
       
     # 2. 设置维护者信息
     MAINTAINER lyc
     
     WORKDIR /app
     
     ENV LANG C.UTF-8
     
     # --- 安装 Ascend 库 ---
     COPY media/ascend.tar.gz .
     
     RUN echo "--> Installing Ascend Libraries..." \
         && mkdir -p /opt/Ascend \
         && tar -xvf ascend.tar.gz -C /opt/Ascend --strip-components=1 \
         && echo "/opt/Ascend" > /etc/ld.so.conf.d/ascend.conf \
         && echo "/usr/local/Ascend/driver/lib64/common" >> /etc/ld.so.conf.d/ascend.conf \
         && echo "/usr/local/Ascend/driver/lib64/driver" >> /etc/ld.so.conf.d/ascend.conf \
         && echo "/usr/local/Ascend/driver/lib64" >> /etc/ld.so.conf.d/ascend.conf \
         && ldconfig \
         && rm ascend.tar.gz \
         && echo "--> Ascend Libraries installed and configured."
     
     
     ##安装mts
     COPY media/mts.tar.gz .
     RUN tar -xvf mts.tar.gz -C /opt/ \
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
     RUN rm mts.tar.gz launch.sh rc-local.service
     
     CMD /usr/bin/systemctl
     ```

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
     
     

4. **(可选) 加载基础镜像**

   * 如果你的基础镜像 `ubuntu-systemctl-arm-build:20.04` 是通过 `.tar` 文件提供的（如原文档所示 `ubuntu-systemctl-arm64_20.04.tar`），你需要先加载它：

   ```bash
   # docker load -i ubuntu-systemctl-arm64_20.04.tar # 确认实际的tar文件名
   ```

5. **构建 Docker 镜像**

   * 在包含 `Dockerfile` 和 `media` 目录的项目目录下，执行以下命令：

   ```bash
   docker build -t ubuntu-systemctl-arm-build_ascend:20.04 .
   ```

   * `-t` 参数为新构建的镜像设置名称和标签。
   * `.` 表示使用当前目录作为构建上下文。

---

## 三、 运行 Docker 容器

1. **运行容器命令详解**

   * 使用以下命令创建并启动容器：
   * **注意：一个硬件设备只能挂载到一个docker运行的容器上**
   
   ```bash
   docker run --net=host --name=atlas2 -h=atlas2 \
     --device=/dev/davinci0 \
     --device=/dev/davinci2 \
     --device=/dev/davinci_manager \
     --device=/dev/hisi_hdc \
     -v /usr/local/Ascend/driver:/usr/local/Ascend/driver \
     -v /root/data/debs:/var/debs \
     -v /var/log/npu/:/var/log/npu/ \
     -idt ubuntu-systemctl-arm-build_ascend:20.04 \
     /usr/bin/systemctl
     
   #无换行符  
   docker run --net=host --name=atlas2 -h=atlas2  --device=/dev/davinci0  --device=/dev/davinci2  --device=/dev/davinci_manager        --device=/dev/hisi_hdc  -v /usr/local/Ascend/driver:/usr/local/Ascend/driver  -v /root/data/debs:/var/debs    -idt ubuntu-systemctl-arm-build_ascend:20.04    /usr/bin/systemctl
     
     
   docker run -idt --net=host --ipc=host --name=atlas1 -h=atlas1 --privileged --device=/dev/davinci2 --device=/dev/davinci_manager --device=/dev/hisi_hdc --device=/dev/devmm_svm -v /usr/local/Ascend/driver:/usr/local/Ascend/driver -v /etc/ascend_install.info:/etc/ascend_install.info -v /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi -v /usr/local/dcmi:/usr/local/dcmi -v /root/data/debs:/var/debs -v /root/data/atlas1/conf:/opt/mts/conf -v /root/data/atlas1/ssl:/opt/mts/ssl -v /root/data/atlas1/log:/opt/mts/log  -v /root/data:/root/data -idt atlas:1.0.1 /usr/bin/systemctl
   ```
   
   * **参数解释**:
     * `docker run`: 创建并启动一个新容器。
     * `--net=host`: 使容器共享宿主机的网络栈，容器内端口直接暴露在宿主机上。
     * `--name=atlas2`: 将容器命名为 "atlas2"。
     * `-h=atlas2`: 设置容器内部的主机名为 "atlas2"。
     * `--device=/dev/davinci0`, `--device=/dev/davinci2`, `--device=/dev/davinci_manager`, `--device=/dev/hisi_hdc`: 将宿主机的 NPU 相关设备节点映射到容器内部，允许容器访问物理 Atlas 卡。请根据实际存在的设备调整。
     * `-v /usr/local/Ascend/driver:/usr/local/Ascend/driver`: 将宿主机的驱动目录挂载到容器内部。这可能与 Dockerfile 中安装库的方式有交互，请确保两者兼容或按需调整。
     * `-v /root/data/debs:/var/debs`: 挂载宿主机目录到容器，用于传递数据或文件。
     * `-v /var/log/npu/:/var/log/npu/`: 挂载宿主机日志目录到容器，用于持久化 NPU 日志。
     * `-idt`: `-i` (交互式), `-d` (后台运行), `-t` (分配伪终端) 的组合。容器将在后台运行，但分配了终端以便后续连接。
     * `ubuntu-systemctl-arm-build_ascend:20.04`: 指定要使用的镜像。
     * `/usr/bin/systemctl`: **覆盖** Dockerfile 中的 `CMD`，将 `systemctl` 作为容器的主进程 (PID 1)。这通常用于在容器内运行完整的 systemd 初始化系统。
   
3. **(可选) 拷贝 NPU 工具**

   * 如果需要将宿主机的 `npu-smi` 工具拷贝到正在运行的容器 `atlas2` 中：

   ```bash
   # 假设 npu-smi 在宿主机的 /usr/local/sbin/ 目录下
   docker cp /usr/local/sbin/npu-smi atlas2:/usr/local/sbin/
   ```

4. **进入容器**

   * 要进入正在运行的容器 `atlas2` 的交互式 Shell：

   ```bash
   docker exec -it atlas2 /bin/bash
   ```

---

## 四、 Atlas 卡常用命令

* **查看驱动版本**:

  ```bash
  # 这个命令通常在宿主机或挂载了驱动目录的容器内执行
  /usr/local/Ascend/driver/tools/upgrade-tool --device_index -1 --system_version
  # 查看32卡的性能信息l's'c'p
  npu-smi info watch -i 32 -c 0 -d 5 -s ptaicmb
  # -i 0 表示监控设备 ID 0
  watch -n 1 npu-smi info -t usages -i 0
  #查看具体型号
  npu-smi info -t product -i 6
  ```
