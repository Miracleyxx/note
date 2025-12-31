
---

# 📝 [Linux 包管理：APT 与 DPKG 核心指南]

> 💡 **一句话摘要**：`apt` 是处理依赖与下载的智能管家（高层），`dpkg` 是处理本地 `.deb` 文件与元数据的底层工头（底层），二者配合维持系统健康。

## 🧩 核心概念 (Core Concepts)

- **`apt` (Advanced Package Tool)**：**高层管理工具**。从镜像源下载包，自动解决复杂的依赖关系（Dependency Hell）。
    
- **`dpkg` (Debian Package)**：**底层操作工具**。仅针对本地 `.deb` 文件进行安装/卸载，**不处理依赖**。
    
- **关键路径**：
    
    - **配置源**：`/etc/apt/sources.list` 及 `/etc/apt/sources.list.d/`
        
    - **包状态库**：`/var/lib/dpkg/status` (dpkg 维护)
        
    - **本地缓存**：`/var/cache/apt/archives/` (下载的 `.deb` 包)
        

## 📍 核心命令矩阵 (Command Matrix)

_(按操作逻辑分类，高频命令已加粗)_

### 1. 生命周期管理 (APT)

- **同步元数据**：`sudo apt update` (操作前必做)
    
- **安装/升级**：
    
    - **`sudo apt install <pkg>`**：安装软件包（`-y` 自动确认）。
        
    - `sudo apt upgrade`：保守升级（不删除旧包）。
        
    - `sudo apt full-upgrade`：激进升级（允许移除冲突包，用于内核/版本升级）。
        
- **清理/移除**：
    
    - **`sudo apt autoremove`**：清除不再需要的孤立依赖。
        
    - `sudo apt clean`：清空本地 `.deb` 缓存（释放磁盘空间）。
        

### 2. 状态查询 (Query)

- **查已安装**：`apt list --installed` 或 `dpkg -l`
    
- **查文件归属**：`dpkg -S <path/to/file>` (反查文件属于哪个包)
    
- **查包详情**：`apt show <pkg>` (版本、依赖) 或 `apt policy <pkg>` (安装源优先级)
    
- **锁定版本**：`sudo apt-mark hold <pkg>` (禁止自动升级)
    

### 3. 灾难救援 (Rescue)

> 当出现“依赖破坏”或“半安装”状态时，按以下顺序执行：

1. **配置残留**：`sudo dpkg --configure -a`
    
2. **修复依赖**：`sudo apt --fix-broken install` (或 `-f`)
    
3. **强制重装**：`sudo apt install --reinstall <pkg>`
    

## ⚖️ 对比分析 (Comparison)

### 1. 移除策略对比

|**命令**|**删除二进制文件**|**删除配置文件**|**适用场景**|
|---|---|---|---|
|**`remove`**|✅ 是|❌ 否|暂时卸载，保留配置以便重装|
|**`purge`**|✅ 是|✅ 是|彻底清理软件残留|

### 2. 本地安装流程对比

|**场景**|**操作流**|**备注**|
|---|---|---|
|**仓库安装**|`sudo apt install pkg`|自动全搞定|
|**本地 .deb**|`sudo dpkg -i file.deb` $\rightarrow$ `sudo apt -f install`|若 `dpkg` 报错缺失依赖，必须用 `apt -f` 修复|

## 🛠 常见故障排除 (Troubleshooting)

- **错误 1：无法获取锁 (Could not get lock)**
    
    - _原因_：后台有 apt/dpkg 进程正在运行，或异常退出导致锁文件未释放。
        
    - _解决_：
        
        1. `ps aux | grep apt` 检查是否有进程，有则等待或 `kill`。
            
        2. **慎用**：确认无进程后，删除锁文件 `sudo rm /var/lib/dpkg/lock-frontend`。
            
        3. 执行 `sudo dpkg --configure -a` 修复数据库。
            
- **错误 2：404 Not Found**
    
    - _原因_：软件源配置过时（发行版代号错误）或仓库已下线。
        
    - _解决_：检查 `/etc/apt/sources.list` 中的代号（如 `jammy`, `bullseye`）是否匹配当前系统版本。
        
- **错误 3：公钥错误 (NO_PUBKEY)**
    
    - _解决_：`gpg --keyserver keyserver.ubuntu.com --recv-keys <KEYID>` (导入公钥)。
        

## 🚀 行动指南 / 总结 (Takeaways)

> **金句**：日常维护用 `apt`，手动修包用 `dpkg`，遇到报错先 `fix-broken`。

- ✅ **安全检查**：在执行 `full-upgrade` 或涉及系统核心的 `remove` 前，务必看清提示列表。
    
- ✅ **脚本技巧**：自动化脚本中使用 `DEBIAN_FRONTEND=noninteractive apt install -y` 避免弹窗卡死。
    
- ✅ **空间释放**：磁盘告急时，第一件事运行 `sudo apt clean`，通常能释放数 GB 空间。
    

---