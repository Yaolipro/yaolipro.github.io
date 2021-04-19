---
layout: post
title: Mac轻松几步搭建Python源码阅读环境 | Python基础
category: ARCHIVES
description: 描述
tags: Python
---

> 以下都是基于 Mac 环境的搭建细节，其他环境请做简单调整

## Anaconda介绍

### 什么是 Anaconda？
`Anaconda`是一个打包的集合，里面预装好了`conda`、某个版本的`Python`、众多的`packages`包和科学计算工具等等，所以也称为`Python`的一种发行版。

### 什么是 conda ？
`conda`可以理解为一个工具，也是一个可执行命令，其核心功能是包管理与环境管理。包管理与`pip`的使用类似，环境管理则允许用户方便地安装不同版本的`Python`，并可以快速切换。

## Anaconda安装

### 安装教程
我们选用图形界面的安装方式，具体参考官网 [0] 教程 [https://docs.anaconda.com/anaconda/install/mac-os/#](https://docs.anaconda.com/anaconda/install/mac-os/#)

```shell
1. Download the graphical macOS installer for your version of Python.
2. RECOMMENDED: Verify data integrity with SHA-256. For more information on hashes, see What about cryptographic hash verification?
3. Double-click the downloaded file and click continue to start the installation.
4. Answer the prompts on the Introduction, Read Me, and License screens.
5. Click the Install button to install Anaconda in your ~/opt directory (recommended):
```

### 基本操作命令

1) 查看安装了哪些包

```shell
conda list
```

2) 查看当前存在哪些虚拟环境

```shell
conda env list
```

3) Python 创建虚拟环境

```shell
conda create -n your_env_name python=x.x
# anaconda命令创建python版本为x.x，名字为your_env_name的虚拟环境
```

4) 激活或者切换虚拟环境

```shell
conda activate base
# 打开命令行，输入python --version检查当前 python 版本。
```

5) 对虚拟环境中安装额外的包

```shell
conda install -n your_env_name [package]
```

6) 关闭虚拟环境

```shell
conda deactivate
```

7) 删除虚拟环境

```shell
conda remove -n your_env_name --all
```

8) 删除环境中的某个包

```shell
conda remove --name $your_env_name $package_name 
```

9) 设置国内镜像

```shell
# 添加Anaconda的TUNA镜像
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
# 设置搜索时显示通道地址
conda config --set show_channel_urls yes
```

10) 恢复默认镜像

```shell
conda config --remove-key channels
```

## VS Code 编辑器

`VSCode`全称`Visual Studio Code`，是微软出的一款轻量级代码编辑器，免费、开源而且功能强大。它支持几乎所有主流的程序语言的语法高亮、智能代码补全、自定义热键、括号匹配、代码片段、代码对比`Diff`、`GIT`等特性，支持插件扩展，并针对网页开发和云端应用开发做了优化。软件跨平台支持`Win`、`Mac`以及`Linux`。

### 安装教程

我们也选用图形界面的安装方式，具体参考官网 [1] 教程 [https://code.visualstudio.com/](https://code.visualstudio.com/)

### 配置快捷启动

如果项目中每次打开`VS Code`很麻烦，那我们可以这样配置类似于`Sublime Text`的一键启动方式，具体操作如下：
* 1) 在`VS Code`窗口中，按下`Command + Shift + P`打开 `Command Palette`
* 2) 输入`shell command`，选择下面的`Install 'code' command in PATH`安装
* 3) 完成后在终端中即可以使用`code .`即可快速启动`VS Code`啦

![image-20210414204332779](../../assets/images/image-20210414204332779.png)

## Python 源码

### 源码下载

```shell
git clone https://github.com/python/cpython.git
```

### 依赖安装

在`Mac`环境下编译`Python`主要分为两个步骤：
* Python 环境准备
* 编译、安装

首先安装编译依赖的工具库：
* gcc 		  // 编译工具
* zlib 		  // 压缩、解压相关库
* libffi 		 // Python 所以来的用于支持 C 扩展的库
* openssl 	// 安全套接字层密码库，Linux 中通常已具备

对于 `MacOS`环境，我们执行以下安装命令：

```shell
xcode-select --install
```

### 源码编译
进入`Python`源码根目录，执行命令：

```shell
./configure
make
make install
```

`Python`将被编译并安装到默认目录中。如果希望将`Python`安装在特定目录，则需要在`make`前修改`configure`命令，具体如下：

```shell
./configure --prefix=<Python待安装目录【绝对路径】>
```

编译后的文件:
* bin &emsp;存放的是可执行文件
* include &emsp;存放的是 Python 源码的头文件
* lib &emsp;存放的是 Python 标准库
* lib/python3.9/config-3.9m-{platform} &emsp;存放的是libpython3.9m.a，该静态库用于使用C语言进行扩展Python。{platform}代表平台，比如在Mac OS上为"darwin"
* share &emsp;存放的是帮助等文件

### 源码介绍

源码根目录中跟`Python`语言直接相关的目录及其功能解释如下:
* Doc &emsp;主要是官方文档的说明。
* Include &emsp;主要包括了Python的运行的头文件。
* Lib &emsp;主要包括了用Python实现的标准库。
* Modules &emsp;主要包含了所有用C语言编写的模块，比如random、cStringIO等。Modules中的模块是那些对速度要求非常严格的模块，而有一些对速度没有太严格要求的模块，比如os，就是用Python编写，并且放在Lib目录下的
* Objects &emsp;主要包含了所有Python的内建对象，包括整数、list、dict等。同时，该目录还包括了Python在运行时需要的所有的内部使用对象的实现。
* Parser &emsp;主要包含了Python解释器中的Scanner和Parser部分，即对Python源码进行词法分析和语法分析的部分。除了这些，Parser目录下还包含了一些有用的工具，这些工具能够根据Python语言的语法自动生成Python语言的词法和语法分析器，将python文件编译生成语法树等相关工作。
* Programs &emsp;主要包括了python的入口函数。
* Python &emsp;主要包括了Python动态运行时执行的代码，里面包括编译、字节码解释器等工作。

### 尝试修改源码

下面编译验证`Python`的`Python C API`打印对象接口 [2]，源文件在`Objects/object.c`

```
int
PyObject_Print(PyObject *op, FILE *fp, int flags)
```

假如我们希望在解释器交互界面中打印整数值的时候输出一段字符串，则可以修改如下函数，源文件在`Objects/longobject.c`

```
static PyObject *
long_to_decimal_string(PyObject *aa)
{
    PyObject *str = PyUnicode_FromString("Hello！This is my new code！");
    PyObject_Print(str, stdout, 0);
    printf("\n");

    PyObject *v;
    if (long_to_decimal_string_internal(aa, &v, NULL, NULL, NULL) == -1)
        return NULL;
    return v;
}
```

函数实现中的前 3 行为我们加入的代码，代码具体介绍如下：
* `PyUnicode_FromString`用于把`C`中的原生字符数组转换为`Python`中的字符串`Unicode`对象
* `PyObject_Print`则将转换好的字符串对象打印至我们指定的标准输出`stdout`

然后我们重新编译`Python`环境

```
make && make bininstall
# 假如不符合预期也先执行
make clean
```

运行编译后的`Python`，输入`print`语句即可验证效果：

```
>>> print(1)
'Hello！This is my new code！'
1
```

## 参考文献
[0] https://docs.anaconda.com/anaconda/install/mac-os/#<br>
[1] https://code.visualstudio.com/<br>
[2] https://flaggo.github.io/python3-source-code-analysis/preface/unix-linux-build/<br>
