## 准备安装用户和运行用户

1. **创建非root用户**

```shell
groupadd HwHiAiUser
useradd -g HwHiAiUser -d /home/HwHiAiUser -m HwHiAiUser -s /bin/bash
```

2. **设置密码**

```shell
passwd HwHiAiUser
```

## 安装驱动所需依赖

**可通过镜像中自带的source安装依赖**[[本地 ISO 软件源配置]]

1. **驱动编译工具**

```shell
yum install -y make dkms gcc kernel-headers-$(uname -r) kernel-devel-$(uname -r)
```

2. **lspci和ifconfig命令工具**

```shell
yum install -y net-tools pciutils
```

## 安装驱动

```shell
chmod +x Ascend-hdk-<chip_type>-npu-driver_<version>_linux-<arch>.run
./Ascend-hdk-<chip_type>-npu-driver_<version>_linux-<arch>.run --full --install-for-all

#输入如下信息提示正确
[root@localhost ~]# ./Ascend-hdk-310p-npu-driver_25.3.rc1_linux-aarch64.run --full --install-for-all
Verifying archive integrity...  100%   SHA256 checksums are OK. All good.
Uncompressing ASCEND DRIVER RUN PACKAGE  100%  
[Driver] [2025-12-24 14:54:18] [INFO]Start time: 2025-12-24 14:54:18
[Driver] [2025-12-24 14:54:18] [INFO]LogFile: /var/log/ascend_seclog/ascend_install.log
[Driver] [2025-12-24 14:54:18] [INFO]OperationLogFile: /var/log/ascend_seclog/operation.log
[Driver] [2025-12-24 14:54:18] [WARNING]Do not power off or restart the system during the installation/upgrade
[Driver] [2025-12-24 14:54:18] [INFO]set username and usergroup, HwHiAiUser:HwHiAiUser
[Driver] [2025-12-24 14:54:19] [INFO]driver install type: DKMS
[Driver] [2025-12-24 14:54:19] [INFO]upgradePercentage:10%
[Driver] [2025-12-24 14:54:25] [INFO]upgradePercentage:30%
[Driver] [2025-12-24 14:54:25] [INFO]upgradePercentage:40%
[Driver] [2025-12-24 14:54:56] [INFO]upgradePercentage:90%
[Driver] [2025-12-24 14:54:58] [INFO]Waiting for device startup...
[Driver] [2025-12-24 14:54:58] [INFO]Device startup success
[Driver] [2025-12-24 14:55:08] [INFO]upgradePercentage:100%
[Driver] [2025-12-24 14:55:18] [INFO]Driver package installed successfully! The new version takes effect immediately. 
[Driver] [2025-12-24 14:55:19] [INFO]End time: 2025-12-24 14:55:19
```

## 安装固件

```shell
chmod +x Ascend-hdk-<chip_type>-npu-firmware_<version>.run
./Ascend-hdk-310p-npu-firmware_7.8.0.2.212.run --full

#输入如下信息提示正确
[root@localhost ~]# ./Ascend-hdk-310p-npu-firmware_7.8.0.2.212.run --full
Verifying archive integrity...  100%   SHA256 checksums are OK. All good.
Uncompressing ASCEND-HDK-310P-NPU FIRMWARE RUN PACKAGE  100%  
[Firmware] [2025-12-24 14:57:03] [INFO]Start time: 2025-12-24 14:57:03
[Firmware] [2025-12-24 14:57:03] [INFO]LogFile: /var/log/ascend_seclog/ascend_install.log
[Firmware] [2025-12-24 14:57:03] [INFO]OperationLogFile: /var/log/ascend_seclog/operation.log
[Firmware] [2025-12-24 14:57:04] [WARNING]Do not power off or restart the system during the installation/upgrade
[Firmware] [2025-12-24 14:57:05] [INFO]upgradePercentage: 0%
[Firmware] [2025-12-24 14:57:15] [INFO]upgradePercentage: 90%
[Firmware] [2025-12-24 14:57:16] [INFO]upgradePercentage: 100%
[Firmware] [2025-12-24 14:57:16] [INFO]The firmware of [1] chips are successfully upgraded.
[Firmware] [2025-12-24 14:57:16] [INFO]Firmware package installed successfully! Reboot now or after driver installation for the installation/upgrade to take effect.
[Firmware] [2025-12-24 14:57:16] [INFO]End time: 2025-12-24 14:57:16

```

## 检测驱动

1. **查看驱动是否加载成功**

```shell
npu-smi info

#输入如下信息驱动正确加载
[root@localhost ~]# npu-smi info
+--------------------------------------------------------------------------------------------------------+
| npu-smi 25.3.rc1                                 Version: 25.3.rc1                                     |
+-------------------------------+-----------------+------------------------------------------------------+
| NPU     Name                  | Health          | Power(W)     Temp(C)           Hugepages-Usage(page) |
| Chip    Device                | Bus-Id          | AICore(%)    Memory-Usage(MB)                        |
+===============================+=================+======================================================+
| 6       310P3                 | OK              | NA           50                0     / 0             |
| 0       0                     | 0000:84:00.0    | 0            1839 / 21525                            |
+===============================+=================+======================================================+
+-------------------------------+-----------------+------------------------------------------------------+
| NPU     Chip                  | Process id      | Process name             | Process memory(MB)        |
+===============================+=================+======================================================+
| No running processes found in NPU 6                                                                    |
+===============================+=================+======================================================+

```

2.  重启生效 (重要)

固件安装完成后，系统通常会提示需要重启以生效。

```shell
#安装固件时提示Reboot now or after driver installation for the installation/upgrade to take effect.需重启系统
1. reboot
2.重启之后使用npu-smi info重新验证
```

## 安装CANN软件
### NNRT离线引擎包

1. **安装** **`NNRT`** **离线推理引擎包**

```shell
chmod +x Ascend-cann-nnrt_<version>_linux-<arch>.run
./Ascend-cann-nnrt_<version>_linux-<arch>.run --install

# 安装完成后，若显示如下信息，则说明软件安装成功：
xxx install success
# xxx表示安装的实际软件包名。

# 如果用户未指定安装路径，则软件会安装到默认路径下，默认安装路径如下。root用户：“/usr/local/Ascend”，非root用户：“${HOME}/Ascend”，${HOME}为当前用户目录。
```

2. **配置环境变量**

```sh
# 非root用户安装后的默认路径为例
source ${HOME}/Ascend/nnrt/set_env.sh
#root用户安装后的默认路径为例
source /usr/local/Ascend/nnrt/set_env.sh
```

3. **检查版本信息**

```shell
# 进入软件包安装信息文件目录，请用户根据实际安装路径替换。<arch>表示CPU架构（aarch64或x86_64），root安装类似
cd ${HOME}/Ascend/nnrt/latest/<arch>-linux
# 执行命令，查看version字段提供的版本信息。
cat ascend_nnrt_install.info
```

### ToolKit**开发套件包**
1. **安装** **`ToolKit`** **开发套件包**

```shell
# 安装依赖
sudo apt-get install -y python3 python3-pip
# 执行权限
chmod +x Ascend-cann-toolkit_<version>_linux-<arch>.run
# 安装软件包
./Ascend-cann-toolkit_<version>_linux-<arch>.run --install

# 安装完成后，若显示如下信息，则说明软件安装成功：
xxx install success
# xxx表示安装的实际软件包名。

# 如果用户未指定安装路径，则软件会安装到默认路径下，默认安装路径如下。root用户：“/usr/local/Ascend”，非root用户：“${HOME}/Ascend”，${HOME}为当前用户目录。
```

2. **配置环境变量**

```sh
# 非root用户安装后的默认路径为例
source ${HOME}/Ascend/ascend-toolkit/set_env.sh
#root用户安装后的默认路径为例
source /usr/local/Ascend/ascend-toolkit/set_env.sh
```

3. **检查版本信息**

```shell
# 进入软件包安装信息文件目录，请用户根据实际安装路径替换。<arch>表示CPU架构（aarch64或x86_64），root安装类似
cd ${HOME}/Ascend/ascend-toolkit/latest/<arch>-linux
# 执行命令，查看version字段提供的版本信息。
cat ascend_ascend-toolkit_install.info