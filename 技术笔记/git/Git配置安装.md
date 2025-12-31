## Linxu下安装

使用Linux，可以用本地包管理系统来安装。

```shll
$ apt-get install git-core
```

## 安装与初始化

### Git配置

使用git的第一件事情就是设置你的名字和email，这些就是你在提交commit时的签名。

```shell
$ git config -- gloable user.name "XXXXXX"

$ git config -- global user.email "XXXX@XX.com"
```

执行完成上述命令后，在主目录下面生成 `.gitconfig`文件，内容如下:

```plaintext
[user]
		name = XXX
		email = XXX@XXX.com
```