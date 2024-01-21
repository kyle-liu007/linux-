# linux-
使用wsl2+vscode搭建一个可以实现根据编译配置进行函数跳转以及自动补全的linux驱动(模块)开发环境，可以In-tree，也可以Out-of-tree
本文介绍的开发环境可以实现内核开发、驱动（模块）out-of-tree开发，实现函数跳转以及自动补全功能
# 一、wsl2搭建linux虚拟机

wsl2搭建linux虚拟机参考网上各种文档，这里说一下LxRunOffline工具。LxRunOffline工具是对wsl2命令行功能的一个扩展，可以对虚拟机进行安装、卸载、迁移、复制、注册、注销等功能，使用注册和注销功能可以将虚拟机装在移动硬盘上，在其他PC上使用时只需要注册一下虚拟机即可。
	LxRunOffline仓库地址：<https://github.com/DDoSolitary/LxRunOffline>
	LxRunOffline使用方式如下：
-  解压后将.exe和.dll文件复制到C:\Windows\System32目录下
- 管理员启动power shell，输入`regsvr32 LxRunOfflineShellExt.dll`
	然后就可以使用LxRunOffline命令行了
	安装虚拟机的方式：
```
LxRunOffline i -n <name> -f <target_path> -d <insatll_path>
```
- name是你想要安装的虚拟机名字，不能与系统上的其他的虚拟机重名
- targe_path是你想要安装的linux发行版的镜像所在目录
- insatll_path是你想要将虚拟安装到的目录
	剩余指令可以直接运行LxRunOffline查看
	我使用的LxRunOffline有问题，不能直接将发行版安装到移动硬盘上，只能先安装到C盘，然后将虚拟机复制到移动硬盘上，直接安装会有这样的错误：
```
[ERROR] Couldn't set the case sensitive attribute of the directory
```

如果复制或移动虚拟机时也出现这样的问题，可以使用下面的版本：
[https://github.com/Andy1208-lee/LxRunOffline/raw/main/LxRunOffline-v3.5.0-11-gfdab71a-msvc.zip](https://github.com/Andy1208-lee/LxRunOffline/raw/main/LxRunOffline-v3.5.0-11-gfdab71a-msvc.zip)

将虚拟机复制到移动硬盘后就可以在不同主机上使用同一个开发环境进行开发了，担心硬盘出问题可以将虚拟机云备份一下，个人感觉这种方式比使用云服务器开发更好，成本是一个1T的成品移动硬盘600多（618买只要400多，自己买硬盘盒装更便宜），外加Microsoft365（1T云空间）40，总成本700多，移动硬盘3年保修，比云服务器划算。
	
# 二、vscode插件和虚拟机内核编译环境
vscode必须的插件：
```
WSL
C/C++
```
直接搜索安装就行

虚拟机内核编译环境搭建：
```
sudo apt update
sudo apt install build-essential flex bison dwarves libssl-dev libelf-dev
```
一般这样就能编译linux内核和模块了，如果不行的话，参考linux/Documentation/process/changes.rst文档，把文档里要求的包都装上。
还有一点，如果是用的微软商店的linux发行版，那么/lib/目录下是没有modules的，因为wsl2的虚拟机本身就不支持加载模块，没法使用这个目录来编译模块。如果只是想编译模块的话（后续开发环境搭建需要编译模块），有两种方法：
	1、可以通过下列方式安装编译需要的头文件：
```
sudo apt install linux-headers-generic
```
然后将下载的目录改成'uname -r'
```
mv /lib/modules/<你下载的目录> /lib/modules/`uname -r`
```
或者修改你的模块的Makefile的-C参数：
```
make -C /lib/modules/<你下载的目录> M=$(CURDIR) $@
```
	
2、还可以直接下载linux的源码来编译，在源码根目录执行：
```
make ARCH=<你想编译的架构> CROSS_COMPILE=<编译链工具> defconfig
```
然后把make -C后的目录设置为源码更目录即可

# 三、linux驱动（模块）开发环境搭建

重头戏来了，搭建linux驱动（模块）开发环境，可以在linux源码树中开发，也可以out-of-tree开发，实现各种内核函数的跳转。

首先，建立这样一个目录树：
![目录树](linux学习/images/驱动开发环境linux源码目录树.png)
建议放在~/目录下，可以直接使用我的脚本，解释一下这个目录：
-  build.sh是编译脚本，build_clean.sh是清理脚本
-  linux-source是用于存放内核源码的目录
-  linux-build是编译输出的目录
-  linux-vscode内存放不同版本以及不同目标架构的vscode配置文件和配置生成文件
-  tool-chain是编译工具链，不同架构有不同的工具链，下载地址：
		https://toolchains.bootlin.com/

接下来针对这个目录树进行分析：
### 1、linux-source

这个保存不同版本的内核源码，内核源码的根目录名称必须是这样的：
```
linux-<version>
```
建议使用4.0以上的内核，太老的内核编译不过，我用3.10的内核编译不过。
内核下载地址：<https://github.com/torvalds/linux>
我这里使用的是4.19.305和5.15.147，这两个版本都是内核长期维护的版本

### 2、linux-build

这个目录是linux编译输出目录，只需要创建一个这样的目录即可，剩下的完全自动化，可以看一下这个目录的目录树：
![linux-build目录树](linux学习/images/linux-build目录树.png)
首先是不同的内核版本，然后是不同的架构

### 3、linux-vscode

这个目录包含为vscode生产配置的文件以及配置文件，可以看一下目录结构
![](linux学习/images/linux-vscode目录树.png)
	
和linux-build非常类似，但是里面的内容需要正对不同的内核版本和架构进行修改，这个里面的主要内容来自下面的仓库：<https://github.com/amezin/vscode-linux-kernel>
对于不同内核版本和不同架构，只需要修改c_cpp_properties.json这个文件：
```
{

    "configurations": [

        {

            "name": "Linux",

            "cStandard": "c11",

            "intelliSenseMode": "gcc-x64",

            "compileCommands": "${workspaceFolder}/compile_commands.json",

            "includePath": ["~/linux/linux-source/linux-<linux_version>/include/**, ~/linux/linux-source/linux-<linux_version>/arch/<arch>/include/**, ~/linux/linux-build/linux-<linux_version>/<arch>/include/**, ~/linux/linux-build/linux-<linux_version>/<arch>/arch/<arch>/include/**"]

        }

    ],

    "version": 4

}
```
	加一行includePath，将<linux_version>改为你用来开发的内核版本，<arch>（注意一定是<>内的arch，没有<>的arch不要改！！）改为你用来开发的目标架构

### 4、tool-chain
tool-chain是用来交叉编译的工具链，下载地址：<https://toolchains.bootlin.com/>，下载好放到对应架构的目录即可

### 5、build.sh和build_clean.sh脚本

在给出这两个脚本之前，首先修改一下~/.bashrc文件，导出这两个脚本使用的环境变量
```
export LINUX_ROOT=~/linux

export LINUX_SOURCE=${LINUX_ROOT}/linux-source

export LINUX_BUILD=${LINUX_ROOT}/linux-build

export LINUX_TOOL_CHAIN=${LINUX_ROOT}/tool-chain

export LINUX_VSCODE=${LINUX_ROOT}/linux-vscode
```
如果你的linux不是放在~/下，那就修改一下LINUX_ROOT

build.sh：
```
#!/bin/bash

  

if [ $# -ne 2 ]; then

    echo "Usage: $1 <linux version> $2 <arch>"

    exit 1

fi

  

linux_version=$1

arch=$2

export PATH=$PATH:${LINUX_TOOL_CHAIN}/${arch}/bin/

cross_compile=`echo ${arch}-linux- | sed -e 's/x86/x86_64/'`

source=${LINUX_SOURCE}/linux-${linux_version}

output=${LINUX_BUILD}/linux-${linux_version}/${arch}

  

cd ${source}

  

if [ ! -e ${output} ]; then

    mkdir ${output}

fi

  

make mrproper O=${output}

  

make ARCH=$(arch) CROSS_COMPILE=${cross_compile} defconfig O=${output}

make ARCH=$(arch) CROSS_COMPILE=${cross_compile} -j$(nproc) O=${output}
```
build_clean.sh：
```
#!/bin/bash

  

if [ $# -ne 2 ]; then

    echo "Usage: $1 <linux version> $2 <arch>"

    exit 1

fi

  

linux_version=$1

arch=$2

export PATH=$PATH:${LINUX_TOOL_CHAIN}/${arch}/bin/

cross_compile=`echo ${arch}-linux- | sed -e 's/x86/x86_64/'`

source=${LINUX_SOURCE}/linux-${linux_version}

output=${LINUX_BUILD}/linux-${linux_version}/${arch}

  

#rm -r ${source}/include/generated/

#rm -r ${source}/include/config/

#rm -r ${source}/arch/${arch}/include/generated/

  

cd ${source}

  

make mrproper

make mrproper O=${output}
```


### 6、linux驱动（模块）开发启动脚本
有了上面的开发环境，就可以进行linux驱动（模块）的开发了，除了上面的工作，要实现out-of-tree驱动开发，并且能够随意的进行函数跳转，还有一些工作需要做，不过都被我集成到下面的脚本了：
	start_code.sh
```
#!/bin/bash

  

if [ $# -lt 2 ]; then

    if [ -e compile_commands.json ]; then

        code .

        exit 0

    else

        echo "Usage: $1 <linux version> $2 <arch> $3 <moudle output path, optional>"

        exit 1

    fi

fi

  

if [ $1 = 4 ]; then

    linux_version="4.19.305"

else

    inux_version="5.15.147"

fi

  

arch=$2

working_dir=`pwd`

if [ ! -z $3 ]; then

    module_output=$3

else

    module_output=${working_dir}

fi

linux_source=${LINUX_SOURCE}/linux-${linux_version}

linux_output=${LINUX_BUILD}/linux-${linux_version}/${arch}

  

if [ -e compile_commands.json ]; then

    code .

    exit 0

fi

  

# test wether C/C++ configuration for vscode exists

if [ ! -e ${LINUX_VSCODE}/linux-${linux_version}/${arch} ]; then

    echo "no vscode directory linux-${linux_version}/${arch}"

    exit 1

fi

  

ln -sf -T ${LINUX_VSCODE}/linux-${linux_version}/${arch} .vscode

  

# test if vmlinux exists, if not, build kernel

if [ ! -e ${linux_output}/vmlinux ]; then

    echo "******* start building linux kernel *******"

    cd ${LINUX_ROOT}

    ./build.sh ${linux_version} ${arch}

#    cp -rf ${linux_output}/include/ ${linux_source}/

#    cp -rf ${linux_output}/arch/${arch}/include/ ${linux_source}/arch/${arch}/

    cd ${working_dir}

fi

  

# generate compile_commands.json

# by default, $working_dir is where your module ouput path, or you should pass it to $3

python3 .vscode/generate_compdb.py -r ${linux_source} ${module_output} ${linux_output}

code .
```
这个脚本可以接收三个参数：
- $1：内核版本，可以直接输入4或5这样的大版本号，脚本内会将大版本号改为细分的版本号，也可以输入细分的版本号
- $2：目标架构，目前vscode的C/C++插件只支持x86(64)和arm(64)架构
- $3：你的模块输出路径，可选，如果为空的话，直接用的当前目录

将脚本放到你的工作目录下，然后使用如下命令：
```
./start_code 4 x86
```
之后脚本将会对内核进行编译，生成vscode进行符号分析需要的配置文件，然后开启一个新的vscode窗口，等待vscode解析完之后就可以愉快的进行开发了。第二次如果不需要更改内核版本和目标架构，那么直接输入./start_code即可
