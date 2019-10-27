---
title: 如何在docker中运行ubuntukylin桌面系统
---

## 背景
由于要和开源社合办一个活动，要求线上线下同时进行，需要使用对方的云平台，本来以为只需要提供iso镜像就行了，没想到对方只支持docker镜像。虽然之前用过docker，但是完全没想过docker里跑桌面。

## 思路调研和已有开源项目
说实话，由于没怎么接触过docker，所以花了一上午时间看了下docker实践教程，但是对如何运行桌面还是没什么头绪。但是我之前在win10刚出wsl的时候好奇去尝试过，当时有一种使用ximage映射使wsl运行图形界面的方案，我猜测docker也可以通过这种类似远程桌面的方式来跑桌面。

同时我又寻找了一些开源项目，这里不得不吐槽下，大家似乎对在docker里启桌面都没什么兴趣，相关资料是真的少...

首先是kde neno，kde neno有docker镜像的试用，看了下发现采用的是xserver-xwphyr这个方案，但是对于docker镜像的细节并看不到，遂放弃。

然后我想到了deepin，似乎曾经听说过他们有相应的docker镜像，我抱着试试看的心态去找了找，发现确实有一个在docker里运行桌面的方案，然而是使用xdocker，这个显然不符合我的预期，只能放弃。


最后终于在github上找到了这个[docker-ubuntu-vnc-desktop](https://github.com/fcwu/docker-ubuntu-vnc-desktop) 这个项目是在docker里运行lxde桌面的ubuntu，并通过浏览器来访问。效果如下

![](https://camo.githubusercontent.com/0a7d00e1480bc9c15cd83833698c89292e934b2d/68747470733a2f2f7261772e6769746875622e636f6d2f666377752f646f636b65722d7562756e74752d766e632d6465736b746f702f6d61737465722f73637265656e73686f74732f6c7864652e706e673f7631) 

效果相当不错，赶紧看看人家的dockerfile是如何构建的

```
# Built with arch: amd64 flavor: lxde image: ubuntu:18.04 localbuild: 1
#
################################################################################
# base system
################################################################################

FROM ubuntu:18.04 as system



RUN sed -i 's#http://archive.ubuntu.com/#http://tw.archive.ubuntu.com/#' /etc/apt/sources.list; 


# built-in packages
ENV DEBIAN_FRONTEND noninteractive
RUN apt update \
    && apt install -y --no-install-recommends software-properties-common curl apache2-utils \
    && apt update \
    && apt install -y --no-install-recommends --allow-unauthenticated \
        supervisor nginx sudo net-tools zenity xz-utils \
        dbus-x11 x11-utils alsa-utils \
        mesa-utils libgl1-mesa-dri \
    && apt autoclean -y \
    && apt autoremove -y \
    && rm -rf /var/lib/apt/lists/*
# install debs error if combine together
RUN add-apt-repository -y ppa:fcwu-tw/apps \
    && apt update \
    && apt install -y --no-install-recommends --allow-unauthenticated \
        xvfb x11vnc=0.9.16-1 \
        vim-tiny firefox chromium-browser ttf-ubuntu-font-family ttf-wqy-zenhei  \
    && add-apt-repository -r ppa:fcwu-tw/apps \
    && apt autoclean -y \
    && apt autoremove -y \
    && rm -rf /var/lib/apt/lists/*

RUN apt update \
    && apt install -y --no-install-recommends --allow-unauthenticated \
        lxde gtk2-engines-murrine gnome-themes-standard gtk2-engines-pixbuf gtk2-engines-murrine arc-theme \
    && apt autoclean -y \
    && apt autoremove -y \
    && rm -rf /var/lib/apt/lists/*
 
 
# Additional packages require ~600MB
# libreoffice  pinta language-pack-zh-hant language-pack-gnome-zh-hant firefox-locale-zh-hant libreoffice-l10n-zh-tw

# tini for subreap
ARG TINI_VERSION=v0.18.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /bin/tini
RUN chmod +x /bin/tini

# ffmpeg
RUN mkdir -p /usr/local/ffmpeg \
    && curl -sSL https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz | tar xJvf - -C /usr/local/ffmpeg/ --strip 1

# python library
COPY image/usr/local/lib/web/backend/requirements.txt /tmp/
RUN apt-get update \
    && dpkg-query -W -f='${Package}\n' > /tmp/a.txt \
    && apt-get install -y python-pip python-dev build-essential \
	&& pip install setuptools wheel && pip install -r /tmp/requirements.txt \
    && dpkg-query -W -f='${Package}\n' > /tmp/b.txt \
    && apt-get remove -y `diff --changed-group-format='%>' --unchanged-group-format='' /tmp/a.txt /tmp/b.txt | xargs` \
    && apt-get autoclean -y \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /var/cache/apt/* /tmp/a.txt /tmp/b.txt


################################################################################
# builder
################################################################################
FROM ubuntu:18.04 as builder


RUN sed -i 's#http://archive.ubuntu.com/#http://tw.archive.ubuntu.com/#' /etc/apt/sources.list; 


RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates gnupg patch

# nodejs
RUN curl -sL https://deb.nodesource.com/setup_8.x | bash - \
    && apt-get install -y nodejs

# yarn
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
    && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
    && apt-get update \
    && apt-get install -y yarn

# build frontend
COPY web /src/web
RUN cd /src/web \
    && yarn \
    && npm run build



################################################################################
# merge
################################################################################
FROM system
LABEL maintainer="fcwu.tw@gmail.com"

COPY --from=builder /src/web/dist/ /usr/local/lib/web/frontend/
COPY image /

EXPOSE 80
WORKDIR /root
ENV HOME=/home/ubuntu \
    SHELL=/bin/bash
HEALTHCHECK --interval=30s --timeout=5s CMD curl --fail http://127.0.0.1:6079/api/health
ENTRYPOINT ["/startup.sh"]
```

发现这个镜像是从最基础的ubuntu18.04开始构建的，然后安装桌面对应的包。然后很关键的，安装了xvfb和x11vnc这两个包。xvfb是虚拟显卡，x11vnc则是用来提供x11服务的，以便主机通过x11客户端来访问桌面。

继续往下看，略过那些库，还有两个重要的操作，一个是把上下文中的web后端部署到docker中，但是我并不清楚开源社会采用什么方案，我理解里应该只需要提供相应的vnc端口即可，所以这一步没什么必要。另一个则是`ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /bin/tini`我去到github项目主页下发现这是个用来初始化并启动系统的项目，但是似乎新版的docker已经自带这个功能？于是乎思路已经基本定了，在docker的系统中安装xvfb和x11vnc,用xvfb虚拟显卡，通过x11vnc提供显示服务。

## 制作过程
等等，一般的docker镜像制作都是从基础镜像开始构建的，但是我并不清楚ubuntukylin需要哪些包...这下子尴尬了。好在docker可以通过导入tar包来制作镜像。我找了一台已经安卓ubuntukylin19.04增强版的机器，把根目录下除了启动时自动生成的那些目录打包到一个tar包里，最后导入到docker制作成镜像。
```shell
tar -cvpf /tmp/system.tar --directory=/ --exclude=proc --exclude=sys --exclude=dev --exclude=run --exclude=boot .
```
其中/proc、/sys、/run、/dev这几个目录都是系统启动时自动生成的，虽然也属于文件系统一部分，但是他们每次开机都会有变化，所以打包的时候就应该忽略它们。
然后执行
```shell
cat system.tar | docker import - ubuntukylin:19.04
```
这样，镜像就制作完毕了。

接下来运行docker，恩？什么都没有发生？？？进入容器一看，发现ligtdm根本没有启动，这就很尴尬了...找了半天问题，没什么头绪，大概猜测和systemd有关，一不做二不休，手动启动ukui-session！恩？什么情况，我的桌面怎么没了，被docker内的ukui占用了，开始运行开机动画？我进到容器里查看，发现/dev目录下挂载了宿主机的设备，去看了下runc的源码，最后得出的结论是systemd的改动，使得某些目录变成shared by default，所以主机显卡被docker占用了导致主机桌面挂掉了...吸取教训，先把/dev的挂载点删了再启动...结果卡在登录进不去？？？然后同事说ukui-greeter出了bug，但是作者上周刚刚才离职了...行吧，不要登录锁屏了，直接把这个包卸载了。ok，终于成功的进了桌面，等等，窗口管理器怎么不见了，鼠标变成x了，想了想发现是自己头脑混乱了，在管理员权限下启动了桌面，用su lm(用户名),切换到普通用户lm，终于正常了。

但是我不能每次都手动去启动吧，必须在启动时自动执行脚本才行，于是在/etc新建了一个rc.local脚本，每次先删除设备节点，然后通过xvfb创建虚拟显卡，设置显示区域，切换到普通用户并执行runsession脚本(chmod + x)

```shell
#!/bin/bash

rm /dev/fb0
rm -rf /dev/dri

export DISPLAY=:1
Xvfb :1 -screen 0 1024x768x16 > /opt/xvfb.log 2>&1 &

su lm -c /home/lm/runsession

sleep 10

su lm -c /home/lm/runukwm
```
runsession脚本
```shell
#!/bin/sh

export DISPLAY=:1
DISPLAY=:1 x11vnc -display :1 -forever -bg -nopw -xkb
#sleep 30


export QT4_IM_MODULE=fcitx 
export QT_IM_MODULE=fcitx 
export XMODIFIERS=@im=fcitx 
export GTK_IM_MODULE=fcitx 

/usr/bin/fcitx


QT4_IM_MODULE=fcitx QT_IM_MODULE=fcitx XMODIFIERS=@im=fcitx GTK_IM_MODULE=fcitx DISPLAY=:1 ukui-session > /home/lm/ukui-session.log 2>&1 &
```

```
#!/bin/bash

export DISPLAY=:1
XDG_SESSION_TYPE=x11 DISPALY=:1 UKWM_VERBOSE=1 ukwm -r > /home/lm/ukwm.log 2>&1 &

```
脚本里主要就是启动了x11vnc设置了一些默认参数，最后启动ukui-session。这里要注意的是，rc.local执行脚本的时候会缺失很多环境变量，必须在脚本里指定，否则会导致一些异常行为。

禁用*plymouth*相关服务
```shell
find -name *plymouth*.service
```
全删了即可

至此，docker镜像完成了。

## 运行结果
执行
```bash
docker run --name test1(容器名) --cap-add ALL --privileged=true -td readlnh/ubuntukylin-vnc-docker:19.04 /sbin/init
```
创建并启动容器
使用如下命令查看ip地址
```bash
docker inspect test1(容器名)
```
最后通过vncview来连接
```bash
vncviewer 172.17.0.2(容器ip):5900(默认端口)
```
效果如下
![](https://raw.githubusercontent.com/readlnh/picture/master/ukui-docker.png) 