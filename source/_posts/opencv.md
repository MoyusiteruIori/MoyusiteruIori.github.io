---
title: OpenCV环境配置
date: 2022-03-27 14:06:58
tags:
  - OpenCV
  - C++
categories: OpenCV
keywords: 'C++, OpenCV'
description: 在 Ubuntu 18.04 上配置 C++ OpenCV 开发环境
cover: cover.png
---
懒癌拖了半年之后终于还是花了半天时间给博客搬了下家，之前的穷人阿里云 + wordpress + sakura 这种憨批组合实在卡到变形，搞得人根本没有点开的欲望。现在搭了新的，之后先把最重要的东西搬运过来。因为配 OpenCV 环境这件事经常要搞，这一篇优先搬过来。

## 安装准备

### 工具和依赖

首先需要安装cmake，pkgconfig和一些依赖库（其中很多并不是必须的）。（其实这些都没什么可说的，到处都有教程，但为了完整性还是写一下吧。

```shell
sudo apt-get install cmake
sudo apt-get install pkg-config
sudo apt-get install build-essential libgtk2.0-dev libavcodec-dev libavformat-dev libjpeg.dev libpng-dev libtiff-dev libswscale-dev
sudo apt-get install libjasper-dev
```

安装`libjasper-dev`时会有无法定位的问题，使用

```shell
sudo add-apt-repository "deb http://security.ubuntu.com/ubuntu xenial-security main" sudo apt update sudo apt upgrade sudo apt install libjasper1 libjasper-dev
```

来解决。

### 下载解压OpenCV源码

接下来是下载OpenCV，直接 https://opencv.org/releases/ 就可以

<div align=center> 

{% asset_img dlcv.png 下载源码 %}

</div>


选择Sources下载即可。这里是从github下载源码，似乎需要科学上网才行，也挺慢的。

下载好后是个压缩包在Downloads文件夹里，解压后会得到一个文件夹，想要重命名一下或者挪到其他目录的可以自己搞一下，应该没什么影响，反正之后库是安装在其他地方的。解压源码之后需要编译源码，源码提供了CMakeLists.txt文件，所以我们cmake取得makefile，然后make编译源码。

解压后的文件夹是这样（我的保存路径是桌面上的opencvsouce文件夹）：

<div align=center> 

{% asset_img opencv/stsrc.png 解压 %}

</div>

## 安装

 

### Cmake

解压好之后是常规的cmake操作，在CMakeLists.txt所在的目录创建一个build文件夹并进入

<div align=center> 

{% asset_img cdbuild.png build %}

</div>

然后使用cmake命令生成makefile

```shell
 cmake -D WITH_TBB=ON -D WITH_EIGEN=ON -D OPENCV_GENERATE_PKGCONFIG=ON -D BUILD_DOCS=ON -D BUILD_TESTS=OFF -D BUILD_PERF_TESTS=OFF -D BUILD_EXAMPLES=OFF -D WITH_OPENCL=OFF -D WITH_CUDA=OFF -D BUILD_opencv_gpu=OFF -D BUILD_opencv_gpuarithm=OFF -D BUILD_opencv_gpubgsegm=O -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local/OpenCV/OpenCV4.5.3 ..
```

这里解释一下两个地方：

第一个是`-D OPENCV_GENERATE_PKGCONFIG=ON`，这里要求cmake后自动生成opencv4.pc配置文件，这在后面的配置中需要用到，OpenCV4.x的版本中该选项默认为OFF，如果不手动打开需要后续手动编写opencv4.pc，很麻烦。

最后的`..`一定不要落下，它表示CMakeLists.txt在build的上一级目录。

`CMAKE_INSTALL_PREFIX=`之后跟的是OpenCV的安装路径，默认为/usr/local，我这里另外建立了一个。

之后执行命令，见到`Configuring done` `Generating done`即表示cmake成功。

### 安装OpenCV

cmake成功后

```shell
sudo make install
```

直接编译安装即可，这个过程也很慢。

 

## 配置环境变量

使用gedit

```shell
sudo gedit /etc/ld.so.conf.d/opencv.conf
```

打开的应该是一个空文件，我们向文件中添加安装OpenCV时安装的库的目录，我的是/usr/local/OpenCV/OpenCV4.5.3/lib，

然后

```shell
sudo ldconfig
```

更新。

接下来使用pkg-config进行配置，管理OpenCV：

```shell
sudo gedit /etc/bash.bashrc
```

打开文件后，在末尾添加：

PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/OpenCV/OpenCV4.5.3/lib/pkgconfig

export PKG_CONFIG_PATH

这里还是要注意，PATH后的路径是自己的OpenCV库安装的路径中保存opencv4.pc的pkgconfig，然后

```shell
source /etc/bash.bashrc 
```

更新即可。

这里我们停下检查一下环境变量是否添加成功，

`pkg-config --list-all`可以查看pkgconfig管理的所有包，不过这里大概会有一大堆看不过来的包，一个个找有没有opencv很困难，所以我们查找一下：

```shell
pkg-config --list-all|grep opencv
```

如果出现了OpenCV的variable和description，像这样：

```shell
opencv4  OpenCV - Open Source Computer Vision Library
```

那么到这一步为止的配置就都成功了。

## 在vscode中测试OpenCV库

OpenCV给出了一些样例代码，在

../opencv-4.5.3/samples/cpp中，选择drawing.cpp作为示例，把代码复制到vscode中

<div align=center> 

{% asset_img includeerr.png include错误 %}

</div>

可以看到首行已经打红色波浪线了。

尝试编译一下（虽然这显然是不可能通过的）：

```shell
g++ cvtest.cpp
```

意料之中的：

<div align=center> 

{% asset_img includeerr1.png include错误 %}

</div>

OpenCV是一个第三方库，没有给g++指定参数，g++当然不知道去哪找到这个头文件，但我们自己知道头文件的位置，因此手动为g++指定编译参数：

```shell
g++ -I/usr/local/OpenCV/OpenCV4.5.3/include/opencv4 cvtest.cpp
```

这样还是没有通过：

<div align=center> 

{% asset_img linkerr.png 链接错误 %}

</div>

虽然仍然不能通过，但错误发生了改变，这次可以判断错误发生在链接阶段，可以猜测编译是可以通过的。

给g++指定参数，仅进行编译：

```shell
g++ -I/usr/local/OpenCV/OpenCV4.5.3/include/opencv4 -c cvtest.cpp
```

编译通过，并生成了.o文件。

<div align=center> 

{% asset_img cgood.png 编译 %}

</div>

现在需要解决链接的问题，很明显链接的问题应该是和之前编译不过的问题一样，没有为编译器指定要链接的库。

但为g++指定链接参数不能仅指定所在路径，必须具体到库文件

<div align=center> 

{% asset_img lib.png cv库文件 %}

</div>

然而库文件有几十个，实在太多了，每次编译都为编译器指定几十个-l参数是不现实的，要解决这个问题，就使用到了先前配置过的pkgconfig。

使用

```shell
pkg-config -h
```

查看pkgconfig支持的命令

<div align=center> 

{% asset_img pkgch.png  %}

</div>

发现` --libs`命令可以为我们输出一个包的所有链接器标志，像这样：

<div align=center> 

{% asset_img lkflg.png  %}

</div>

于是我们就可以利用pkgconfig的libs功能做一个全局的替换，将这个命令的返回结果用作编译器的输入参数。

这里需要一点shell的知识，反引号````中可以执行一个shell命令

因此我们这样为g++指定编译参数：

```shell
g++ -std=c++11 -I/usr/local/OpenCV/OpenCV4.5.3/include/opencv4 cvtest.cpp `pkg-config --libs opencv4
```



编译链接均通过，生成了a.out

<div align=center> 

{% asset_img clgood.png 编译链接 %}

</div>

我们运行a.out

```shell
./a.out
```

<div align=center> 

{% asset_img success.png  %}

</div>

demo成功运行。

另外，参考指定链接参数的做法，注意到pkgconfig有`--cflags`选项输出编译器标志，那么也可以这样指定编译参数：

```shell
g++ -std=c++11 `pkg-config --cflags opencv4` cvtest.cpp `pkg-config --libs opencv4
```

也是一样可以成功的。