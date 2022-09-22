---
title: 树莓派pico linux C语言开发环境搭建
date: 2022-09-22 23:07:50
tags: raspberrypi
categories: raspberrypi
top_img:
cover:
---

# 树莓派pico搭建 Linux C语言开发环境

或者直接通过wget获得树莓派的启动脚本

```shell
wget https://raw.githubusercontent.com/raspberrypi/pico-setup/master/pico_setup.sh
chmod +x pico_setup.sh
./pico_setup.sh
```

这个脚本可以帮你

• 创建一个名为pico的目录

•安装所需的依赖项

•下载pico-sdk, pico-examples, pico-extras和pico-playground仓库

•在~/.bashrc中定义PICO_SDK_PATH, PICO_EXAMPLES_PATH, PICO_EXTRAS_PATH和PICO_PLAYGROUND_PATH

•在pico-examples/ Build /blink和pico-examples/ Build /hello_world中构建blink和hello_world示例

•下载并构建picotool(参见附录B)，并将其复制到/usr/local/bin。

•下载并构建picoprobe(参见附录A)。

•下载并编译OpenOCD(用于调试支持)

•下载并安装Visual Studio Code

•安装所需的Visual Studio Code扩展(详见第7章)

•配置树莓派UART与树莓派Pico一起使用



也可以直接去网上拷贝脚本下来自己创建脚本文件执行

```shell
#!/bin/bash

# Exit on error
set -e

if grep -q Raspberry /proc/cpuinfo; then
    echo "Running on a Raspberry Pi"
else
    echo "Not running on a Raspberry Pi. Use at your own risk!"
fi

# Number of cores when running make
JNUM=4

# Where will the output go?
OUTDIR="$(pwd)/pico"

# Install dependencies
GIT_DEPS="git"
SDK_DEPS="cmake gcc-arm-none-eabi gcc g++"
OPENOCD_DEPS="gdb-multiarch automake autoconf build-essential texinfo libtool libftdi-dev libusb-1.0-0-dev"
VSCODE_DEPS="code"
UART_DEPS="minicom"

# Build full list of dependencies
DEPS="$GIT_DEPS $SDK_DEPS"

if [[ "$SKIP_OPENOCD" == 1 ]]; then
    echo "Skipping OpenOCD (debug support)"
else
    DEPS="$DEPS $OPENOCD_DEPS"
fi

echo "Installing Dependencies"
sudo apt update
sudo apt install -y $DEPS

echo "Creating $OUTDIR"
# Create pico directory to put everything in
mkdir -p $OUTDIR
cd $OUTDIR

# Clone sw repos
GITHUB_PREFIX="https://github.com/raspberrypi/"
GITHUB_SUFFIX=".git"
SDK_BRANCH="master"

for REPO in sdk examples extras playground
do
    DEST="$OUTDIR/pico-$REPO"

    if [ -d $DEST ]; then
        echo "$DEST already exists so skipping"
    else
        REPO_URL="${GITHUB_PREFIX}pico-${REPO}${GITHUB_SUFFIX}"
        echo "Cloning $REPO_URL"
        git clone -b $SDK_BRANCH $REPO_URL

        # Any submodules
        cd $DEST
        git submodule update --init
        cd $OUTDIR

        # Define PICO_SDK_PATH in ~/.bashrc
        VARNAME="PICO_${REPO^^}_PATH"
        echo "Adding $VARNAME to ~/.bashrc"
        echo "export $VARNAME=$DEST" >> ~/.bashrc
        export ${VARNAME}=$DEST
    fi
done

cd $OUTDIR

# Pick up new variables we just defined
source ~/.bashrc

# Build a couple of examples
cd "$OUTDIR/pico-examples"
mkdir build
cd build
cmake ../ -DCMAKE_BUILD_TYPE=Debug

for e in blink hello_world
do
    echo "Building $e"
    cd $e
    make -j$JNUM
    cd ..
done

cd $OUTDIR

# Picoprobe and picotool
for REPO in picoprobe picotool
do
    DEST="$OUTDIR/$REPO"
    REPO_URL="${GITHUB_PREFIX}${REPO}${GITHUB_SUFFIX}"
    git clone $REPO_URL

    # Build both
    cd $DEST
    mkdir build
    cd build
    cmake ../
    make -j$JNUM

    if [[ "$REPO" == "picotool" ]]; then
        echo "Installing picotool to /usr/local/bin/picotool"
        sudo cp picotool /usr/local/bin/
    fi

    cd $OUTDIR
done

if [ -d openocd ]; then
    echo "openocd already exists so skipping"
    SKIP_OPENOCD=1
fi

if [[ "$SKIP_OPENOCD" == 1 ]]; then
    echo "Won't build OpenOCD"
else
    # Build OpenOCD
    echo "Building OpenOCD"
    cd $OUTDIR
    # Should we include picoprobe support (which is a Pico acting as a debugger for another Pico)
    INCLUDE_PICOPROBE=1
    OPENOCD_BRANCH="rp2040"
    OPENOCD_CONFIGURE_ARGS="--enable-ftdi --enable-sysfsgpio --enable-bcm2835gpio"
    if [[ "$INCLUDE_PICOPROBE" == 1 ]]; then
        OPENOCD_CONFIGURE_ARGS="$OPENOCD_CONFIGURE_ARGS --enable-picoprobe"
    fi

    git clone "${GITHUB_PREFIX}openocd${GITHUB_SUFFIX}" -b $OPENOCD_BRANCH --depth=1
    cd openocd
    ./bootstrap
    ./configure $OPENOCD_CONFIGURE_ARGS
    make -j$JNUM
    sudo make install
fi

cd $OUTDIR

if [[ "$SKIP_VSCODE" == 1 ]]; then
    echo "Skipping VSCODE"
else
    echo "Installing VSCODE"
    sudo apt install -y $VSCODE_DEPS

    # Get extensions
    code --install-extension marus25.cortex-debug
    code --install-extension ms-vscode.cmake-tools
    code --install-extension ms-vscode.cpptools
fi

# Enable UART
if [[ "$SKIP_UART" == 1 ]]; then
    echo "Skipping uart configuration"
else
    sudo apt install -y $UART_DEPS
    echo "Disabling Linux serial console (UART) so we can use it for pico"
    sudo raspi-config nonint do_serial 2
    echo "You must run sudo reboot to finish UART setup"
fi
```





## 1.获得源码

首先创建一个文件夹

```shell
cd ~/
mkdir pico
cd pico
```

然后克隆树莓派的仓库

```shell
git clone -b master https://github.com/raspberrypi/pico-sdk.git
cd pico-sdk
git submodule update --init
cd ..
git clone -b master https://github.com/raspberrypi/pico-examples.git
git clone https://github.com/raspberrypi/pico-project-generator.git
cd pico-project-generator/
export PICO_SDK_PATH="/home/yuh/code/other/pico/pico-sdk"
```

这里我已经配置过wsl的gui了所以我就不用执行`export DISPLAY=127.0.0.1:0`

这里执行`git submodule update --init`失败的话，可以在`/lib`目录下对应的分别执行`git clone `对应的子模块

## 2.安装编译工具





```shell
sudo apt update
sudo apt install cmake gcc-arm-none-eabi libnewlib-arm-none-eabi build-essential tk python3-tk
```



## 3.创建工程

```shell
cd pico-project-generator/
./pico_project.py --gui
```

执行完成就会出一个这个界面

![image-20220923003214291](https://s2.loli.net/2022/09/23/2ItVWz5ZKg1ldCJ.png)

点击ok就可以生成一个树莓派的项目

![image-20220923003358586](https://s2.loli.net/2022/09/23/tw3TMbhOziA7kB8.png)

然后我们就可以得到一个非常简单的blink的树莓派项目

![image-20220923003527127](https://s2.loli.net/2022/09/23/HyfeL34SwKk6rDb.png)
