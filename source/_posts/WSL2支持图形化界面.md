---
title: WSL2支持图形化界面
date: 2022-09-22 22:28:22
tags: WSL2
categories: WSL2
top_img: https://s2.loli.net/2022/09/22/3pCi6eEP4ktNJoQ.jpg
cover: https://s2.loli.net/2022/09/22/3pCi6eEP4ktNJoQ.jpg
---

# WSL2支持图形化界面

WSL2是为开发人员准备的[命令行工具](https://cloud.tencent.com/product/cli?from=10680)，但是桌面环境可以在WSL2内部运行，并且可以使用XServer（例如Xming或VcXSrv）来侦听Linux中的X11（图形）程序。Xfce4是一个轻量级桌面环境，开发人员可以同时使用WSL和WSL2。

## 1.安装Xfce4 Xming

使用apt安装xfce4

```shell
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install xfce4
```

在安装xfce4的过程中出现报错

```shell
Err:1 http://mirrors.aliyun.com/ubuntu focal-updates/main amd64 libtiff5 amd64 4.1.0+git191117-2ubuntu0.20.04.3
  404  Not Found [IP: 125.72.129.219 80]
Err:2 http://mirrors.aliyun.com/ubuntu focal-updates/main amd64 libgdk-pixbuf2.0-common all 2.40.0+dfsg-3ubuntu0.3
  404  Not Found [IP: 125.72.129.219 80]
Err:3 http://mirrors.aliyun.com/ubuntu focal-updates/main amd64 libgdk-pixbuf2.0-0 amd64 2.40.0+dfsg-3ubuntu0.3
  404  Not Found [IP: 125.72.129.219 80]
Err:4 http://mirrors.aliyun.com/ubuntu focal-updates/main amd64 libjavascriptcoregtk-4.0-18 amd64 2.36.6-0ubuntu0.20.04.1
  404  Not Found [IP: 125.72.129.219 80]
Err:5 http://mirrors.aliyun.com/ubuntu focal-updates/main amd64 libwebkit2gtk-4.0-37 amd64 2.36.6-0ubuntu0.20.04.1
  404  Not Found [IP: 125.72.129.219 80]
Err:6 http://mirrors.aliyun.com/ubuntu focal-updates/main amd64 gir1.2-gdkpixbuf-2.0 amd64 2.40.0+dfsg-3ubuntu0.3
  404  Not Found [IP: 125.72.129.219 80]
Err:7 http://mirrors.aliyun.com/ubuntu focal-updates/main amd64 libgdk-pixbuf2.0-bin amd64 2.40.0+dfsg-3ubuntu0.3
  404  Not Found [IP: 125.72.129.219 80]
```

通过换源来解决，[换源教程参考这里](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/)

在下载xfce4的过程中可以安装Xming，Xfce4是Linux图形程序，而Xming 是用来连接并展示其图像界面

下载并安装[Xming](https://sourceforge.net/projects/vcxsrv/ )

## 2.配置vcxsrv xfce4

![image-20220922224254750](https://s2.loli.net/2022/09/22/UsAnLFaNdxmoSMK.png)

![image-20220922224320619](https://s2.loli.net/2022/09/22/Hag9keTAzmoqtV6.png)

![image-20220922224333657](https://s2.loli.net/2022/09/22/R3GNyiIAgz7ZwDE.png)

注意，这里第三步一定要选上`disable access control `



然后在linux中配置.bashrc

```shell
vi ~/.bashrc
```

在最后一行加上

```shell
export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}'):0
```

保存退出，再执行

```shell
source ~/.bashrc
```



到此，如果无误的话，我们已经可以使用GUI了，执行`startxfce4`就可以看到你的ubuntu桌面了
