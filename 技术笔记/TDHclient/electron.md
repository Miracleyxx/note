# Electron 应用打包指南

## 1. 环境准备

### 1.1 安装 Node.js
1. 访问 [Node.js 官网](https://nodejs.org/)
2. 下载并安装最新的 LTS 版本
3. 安装时勾选"Automatically install the necessary tools"选项
4. 安装完成后，打开新的 PowerShell 窗口，验证安装：

```bash
node --version
npm --version
```

### 1.2 环境变量配置
如果命令未识别，需要手动配置环境变量：
1. 打开系统环境变量设置
2. 在 Path 中添加 Node.js 安装路径（通常是 `C:\Program Files\nodejs\`）
3. 重新打开 PowerShell 窗口

## 2. 项目配置

### 2.1 创建 package.json

```json
{
  "name": "snake-game",
  "productName": "snake-game",
  "version": "1.0.0",
  "description": "My Electron application description",
  "main": "src/main.js",
  "scripts": {
    "start": "electron .",
    "build": "electron-builder"
  },
  "author": {
    "name": "liyuc",
    "email": "liyuc@tdhnet.com.cn"
  },
  "license": "MIT",
  "devDependencies": {
    "electron": "^36.4.0",
    "electron-builder": "^26.0.12"
  },
  "build": {
    "appId": "com.snake.game",
    "productName": "Snake Game",
    "directories": {
      "output": "dist"
    },
    "files": [
      "src/**/*",
      "package.json"
    ],
    "win": {
      "target": [
        {
          "target": "nsis",
          "arch": ["x64"]
        }
      ]
    },
    "nsis": {
      "oneClick": false,
      "allowToChangeInstallationDirectory": true,
      "createDesktopShortcut": true,
      "createStartMenuShortcut": true,
      "shortcutName": "Snake Game",
      "menuCategory": true,
      "perMachine": false,
      "deleteAppDataOnUninstall": true,
      "uninstallDisplayName": "Snake Game",
      "artifactName": "${productName}-Setup-${version}.${ext}"
    }
  }
}
```

## 3. 构建流程

### 3.1 清理旧的构建文件

```bash
rm -r dist
```

### 3.2 执行构建

```bash
npm run build
```

## 4. 构建输出

构建完成后，在 `dist` 目录下会生成以下文件：
- `Snake Game-Setup-1.0.0.exe`：Windows 安装程序
- `win-unpacked/`：解压后的应用程序文件

## 5. 安装程序功能

### 5.1 基本功能
- 允许用户选择安装目录
- 创建桌面快捷方式
- 创建开始菜单快捷方式
- 显示在控制面板的程序列表中
- 支持卸载
- 卸载时删除应用程序数据

### 5.2 配置说明

#### 5.2.1 基本设置
- `appId`：应用程序唯一标识符
- `productName`：产品名称
- `directories.output`：输出目录
- `files`：需要打包的文件

#### 5.2.2 Windows 特定设置
- `target`：目标格式（nsis）
- `arch`：目标架构（x64）

#### 5.2.3 NSIS 安装程序设置
- `oneClick`：是否一键安装（false 允许自定义安装）
- `allowToChangeInstallationDirectory`：允许更改安装目录
- `createDesktopShortcut`：创建桌面快捷方式
- `createStartMenuShortcut`：创建开始菜单快捷方式
- `shortcutName`：快捷方式名称
- `menuCategory`：在开始菜单中创建分类
- `perMachine`：是否为所有用户安装
- `deleteAppDataOnUninstall`：卸载时删除应用数据
- `uninstallDisplayName`：卸载程序显示名称
- `artifactName`：安装包文件名格式

## 6. 注意事项

1. 确保 `src` 目录下有必要的源文件
2. 构建前确保所有依赖都已正确安装
3. 如果遇到构建错误，检查 `package.json` 配置是否正确
4. 构建完成后测试安装程序的所有功能

## 7. 常见问题解决

### 7.1 构建失败
- 检查 Node.js 和 npm 版本
- 确保所有依赖都已正确安装
- 检查 `package.json` 配置是否正确

### 7.2 安装程序问题
- 确保 `main` 字段指向正确的入口文件
- 检查 `files` 配置是否包含所有必要文件
- 验证应用程序是否能正常运行