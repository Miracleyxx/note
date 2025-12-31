
---

# 💿 Linux 本地 ISO 软件源配置手册 (openEuler/CentOS/RHEL)

> ℹ️ **场景定义**：在**离线（Air-gapped）**或弱网服务器环境下，利用系统安装镜像（ISO）作为本地 `yum/dnf` 仓库，解决软件依赖安装问题。

## 📋 准备工作 (Prerequisites)

- **权限**：`root` 用户权限。
    
- **介质**：存有 ISO 镜像的 U 盘。
    
- **目标架构**：将 ISO 挂载至 `/mnt/iso` $\rightarrow$ 配置 Repo 指向该路径。
    

---

## 🛠️ 第一步：挂载 ISO 镜像 (Mount ISO)

请根据您的 U 盘制作方式，选择 **方案 A** 或 **方案 B**。

### 【方案 A】Ventoy 启动盘 (ISO 文件模式)

**场景特征**：U 盘被视为移动硬盘，ISO 文件作为一个普通文件存在于分区中。这是目前运维最常用的方式。

1. 识别设备分区

输入 lsblk 查看磁盘树形结构：

```shell
lsblk
# 输出示例：
# sdb      8:16   1  58G  0 disk
# ├─sdb1   8:17   1  58G  0 part  <-- 📦 大容量数据分区 (存放 ISO 的位置)
# └─sdb2   8:18   1  32M  0 part  <-- Ventoy 引导分区
```

2. 挂载 U 盘 (一级挂载)

首先需要将 U 盘的文件系统挂载到系统上，才能访问里面的文件。

```shell
mkdir -p /mnt/usb
# 挂载数据分区 (通常是 sdb1)
mount /dev/sdb1 /mnt/usb
```

> ⚠️ **注意**：如果报错 `unknown filesystem type 'exfat'`，说明系统缺少 exFAT 驱动。请参阅文末故障排查。

**3. 挂载 ISO 文件 (二级 Loop 挂载)**

这是最关键的一步：利用 `loop` 设备将文件模拟为块设备。

```shell
# 1. 创建最终挂载点
mkdir -p /mnt/iso

# 2. 挂载 ISO (Loop 模式)
# 💡 技巧：输入路径时按 <Tab> 键自动补全文件名，避免手打出错
mount -o loop /mnt/usb/openEuler-24.03-LTS-Everything-x86_64-dvd.iso /mnt/iso
```

> **成功标志**：系统提示 `mount: /mnt/iso: WARNING: device is write-protected, mounting read-only`。

---

### 【方案 B】传统刻录盘 (Rufus/UltraISO/dd)

**场景特征**：U 盘已被格式化为 ISO9660 光盘结构，文件系统直接展开，看不见 `.iso` 文件。

1. 识别设备

输入 lsblk：

```shell
lsblk
# 输出示例：
# sdb      8:16   1   8G  0 disk
# └─sdb1   8:17   1   8G  0 part  <-- 💿 识别为光盘结构
```

2. 直接挂载设备

无需寻找文件，直接将分区挂载即可。

```shell
mkdir -p /mnt/iso
mount /dev/sdb1 /mnt/iso
```

_(注：若 `sdb1` 挂载失败，尝试直接挂载整盘 `mount /dev/sdb /mnt/iso`)_

---

## ⚙️ 第二步：配置 Yum/DNF 源 (Configure Repo)

无论采用哪种挂载方案，此时 `/mnt/iso` 下应包含 `Packages` 或 `repodata` 文件夹。

1. 备份现有源 (Disable Online Repos)

防止在离线环境下 dnf 尝试连接互联网导致超时报错。

Bash

```shell
mkdir -p /etc/yum.repos.d/backup
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup/
```

**2. 创建本地源配置**

```shell
vi /etc/yum.repos.d/local.repo
```

3. 写入配置内容

粘贴以下内容（注意 baseurl 的三个斜杠）：

|**参数**|**值**|**说明**|
|---|---|---|
|`[local-iso]`|-|仓库 ID，必须唯一|
|`name`|`Local ISO`|仓库描述|
|`baseurl`|**`file:///mnt/iso`**|**核心参数**：指向挂载点|
|`enabled`|`1`|启用此仓库|
|`gpgcheck`|`0`|**离线推荐**：关闭签名检查，避免导入 Key 的繁琐|

---

## ✅ 第三步：验证与缓存 (Verify)

**1. 重建缓存**

```shell
dnf clean all
dnf makecache
```

_预期结果：终端输出中包含 `Local ISO`，且最后显示 `Metadata cache created`。_

**2. 安装测试**

```shell
dnf install vim -y
# 或者检查仓库列表
dnf repolist
```

---

## 🔧 常见故障排查 (Troubleshooting)

|**错误现象 (Symptom)**|**根本原因 (Root Cause)**|**解决方案 (Solution)**|
|---|---|---|
|`unknown filesystem type 'exfat'`|Ventoy 默认使用 exFAT 格式，而服务器精简系统（Minimal Install）通常不带该驱动。|**方案1 (推荐)**: 找台 Windows 电脑，将 U 盘数据区格式化为 **NTFS**，Linux 通常支持或易于挂载 NTFS。<br><br>  <br><br>**方案2**: 使用更小的 FAT32 U 盘转存 ISO。|
|`Failed to download metadata`|1. ISO 未挂载成功（目录为空）。<br><br>  <br><br>2. `baseurl` 路径拼写错误。|1. 运行 `ls /mnt/iso` 确保能看到文件。<br><br>  <br><br>2. 检查 repo 文件中是否为 `file:///` (3个斜杠)。|
|**重启后失效**|`mount` 命令是临时的，重启后内存中的挂载关系丢失。|**建议**: 既然是临时维护，**每次重启后手动运行 mount 命令即可**。<br><br>  <br><br>⚠️ **警告**: 不要将 U 盘挂载写入 `/etc/fstab`，否则拔掉 U 盘重启会导致服务器无法开机（进入救援模式）。|

---
