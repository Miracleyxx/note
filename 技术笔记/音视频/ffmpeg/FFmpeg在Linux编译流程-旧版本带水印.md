## FFmpeg在Linux编译流程

## 安装`freetype`

###  1.下载

```Bash
wget https://bigsearcher.com/mirrors/nongnu/freetype/freetype-2.10.0.tar.bz2 
```

###  2.解压

```Bash
tar jxf  freetype-2.10.0.tar.bz2
cd freetype-2.10.0
```

###  3.安装

 设置安装的目录为`/usr/local/freetype`

```Bash
./configure --prefix=/usr/local/freetype
sudo make && sudo make install
```

###  4.配置环境变量

1. ####  打开文件`vim /etc/profile`

```Bash
export PKG_CONFIG_PATH="/usr/local/freetype/lib/pkgconfig:$PKG_CONFIG_PATH"
```

 **`source /etc/profile`** 使环境变量生效

1. ####  编辑`vim /etc/ld.so.conf.d/ffmpeg.conf`在添加下面一行内容：

```C++
/usr/local/freetype
```

保存文件，执行`ldconfig`使配置生效。

## 安装`libxml2`

###  1.下载

```C++
wget http://xmlsoft.org/sources/libxml2-2.9.10.tar.gz
```

###  2.解压

```C++
tar -xzf libxml2-2.9.10.tar.gz
```

###  3.安装

 设置安装的目录为**`/usr/local/libxml2`**

```C++
//x86
./configure --prefix=/usr/local/libxml2 --without-python --without-zlib

sudo make && sudo make install
```

###  4.配置环境变量

1. ####  打开文件`vim /etc/profile`

```C++
export PKG_CONFIG_PATH="/usr/local/libxml2/lib/pkgconfig:$PKG_CONFIG_PATH"
```

 **`source /etc/profile`** 使环境变量生效

1. ####  编辑`vim /etc/ld.so.conf.d/ffmpeg.conf`在添加下面一行内容：

```C++
/usr/local/libxml2/lib
```

保存文件，执行`ldconfig`使配置生效。

## 安装`fontconfig`

###  1.下载

```C++
wget https://www.freedesktop.org/software/fontconfig/release/fontconfig-2.9.92.tar.gz
```

###  2.解压

```C++
tar -xzf fontconfig-2.9.92.tar.gz
```

###  3.安装

 设置安装的目录为`/usr/local/``fontconfig`

```C++
./configure --enable-libxml2 --with-freetype-config=/usr/local/freetype/include/freetype2/freetype/config --prefix=/usr/local/fontconfig
sudo make && sudo make install
```

###  4.配置环境变量cd 

1. ####  打开文件`vim /etc/profile`

```C++
export PKG_CONFIG_PATH="/usr/local/fontconfig/lib/pkgconfig:$PKG_CONFIG_PATH"
```

 **`source /etc/profile`** 使环境变量生效

1. ####  编辑`vim /etc/ld.so.conf.d/ffmpeg.conf`在添加下面一行内容：

```C++
/usr/local/fontconfig/lib
```

保存文件，执行`ldconfig`使配置生效。

## 安装`fribidi`

###  1.下载

```C++
 wget https://codeload.github.com/fribidi/fribidi/zip/master
```

###  2.解压

```C++
unzip master
cd fribidi-master/
apt install libtool
apt install autoconf
apt install automake
./autogen.sh
```

###  3.安装

 设置安装的目录为`/usr/local/fribidi`

```C++
./configure --prefix=/usr/local/fribidi
sudo make && sudo make install
```

###  4.配置环境变量

1. ####  打开文件`vim /etc/profile`

```C++
export PKG_CONFIG_PATH="/usr/local/fribidi/lib/pkgconfig:$PKG_CONFIG_PATH"
```

 **`source /etc/profile`** 使环境变量生效

1. ####  编辑`vim /etc/ld.so.conf.d/ffmpeg.conf`在添加下面一行内容：

```C++
/usr/local/fribidi/lib
```

保存文件，执行`ldconfig`使配置生效。

## 安装`FFmpeg`

### 1. 下载

```C++
git clone http://172.16.20.107/songgan/ffmpeg-4.4.1.git
```

### 2.环境变量

```C++
export PKG_CONFIG_PATH=/usr/local/freetype/lib/pkgconfig:$PKG_CONFIG_PATH
export PKG_CONFIG_PATH=/usr/local/libxml2/lib/pkgconfig:$PKG_CONFIG_PATH
export PKG_CONFIG_PATH=/usr/local/fontconfig/lib/pkgconfig:$PKG_CONFIG_PATH
export PKG_CONFIG_PATH=/usr/local/fribidi/lib/pkgconfig:$PKG_CONFIG_PATH
```

**`source /etc/profile`** 使环境变量生效

### 3.配置`FFmpeg`

```C++
//x86架构
./configure --enable-shared --enable-decoder=h264 --enable-parser=h264 --enable-libfreetype --enable-libfontconfig --enable-libfribidi --arch=x86_64 --prefix=/usr/local/ffmpeg
//arm架构
./configure --enable-shared --enable-decoder=h264 --enable-parser=h264 --enable-libfreetype --enable-libfontconfig --enable-libfribidi --arch=aarch64 --prefix=/usr/local/ffmpeg
```

### 4.安装

```C++
sudo make  && sudo make install
```

### 5.测试

```C++
ffmpeg -i input.mp4 -vf "drawtext=fontfile=/path/to/font.ttf: text='中文水印': x=(w-text_w)/2: y=(h-text_h)/2: fontsize=200: fontcolor=white" output.mp4
```