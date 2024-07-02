---
title: wine wechat in docker 实践
tags:
  - 折腾
  - docker
  - wine
categories:
  - - container
    - docker
  - - 折腾
    - wine
  - - 折腾
    - docker
date: 2024-07-02 13:16:42
---

> 本文参考：
>
>[DoChat](https://github.com/huan/docker-wechat)
>
>[Docker GUI 最佳实践](https://github.com/Document-Collection/Containerization-Automation/blob/982d54458b05ef75fe6436f4ea72bbb66c4cb931/docs/docker/gui/%5BDocker%5DGUI%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5.md)
>
>[Container sound: ALSA or Pulseaudio](https://github.com/mviereck/x11docker/wiki/Container-sound:-ALSA-or-Pulseaudio)
>
>[【已解决】关于安装wine微信的问题](https://bbs.archlinuxcn.org/viewtopic.php?id=14020)
>
>[wine+官方微信3.9.9 2024 年测试可用](https://www.cnblogs.com/hyaline-doc/p/18011541)
>
>[WeChat on Linux](https://leimao.github.io/blog/Docker-WeChat/)

无论哪种方面，从来都没有免费的便利，包括互联网。简中互联网尤为如此，使用微信更像是一种绑架，它绑架了笔者的生活、社交与更多的社会活动...但是这种一家独大场景的形成除了各方面因素外，用户本身也不是完全无辜的。起码很多人拥有选择权而不自知
# 前言
扯远了，本文主题是从dockerfile开始构建，实现在docker里安装wine，并通过套接字共享宿主的x11和pulseaudio服务以实现gui和声音，最后在其中安装windows微信并在宿主中使用。在对wine中运行微信有透明边问题和发行版的熟悉程度综合考虑后，我选择最新的`archlinux/archlinux:latest`作为基础镜像，因为`archlinuxcn`源中有相关工具是用以解决透明边问题的

在主流的发行版中打包了不少开箱即用的微信，也有很多是隔离环境/沙箱运行的。如果追求开箱即用，那么没必要折腾

<del>之所以叫toychat是因为真的没什么技术含量</del>

> 因为笔者能力和时间有限，构建过程中并不要求各种最佳实践和kiss原则，而是优先实现最佳效果

# 思路
archlinux是滚动发行的且版本更新很快，相对的，写好dockerfile构建出的镜像没有问题一般是不怎么更新的。所以日后如果需要重新构建可能就得调整包管理中的内容

由于构建过程中无法使用宿主套接字(或者说我没了解到)，所以从dockerfile开始构建起，还要分为多个步骤:
 - step-1 构建镜像`base`，该镜像使用包管理将wine及其所有需要用上的软件装上，在dockerfile中配置好wine初始化脚本，将ENTRYPOINT设置为该脚本，并将windows字体拷贝到基础镜像的字体目录
 - step-2 使用上一步`base`镜像创建一个容器，这个容器需要共享宿主的x11套接字及在宿主机中挂载`winetricks`所需要的资源，资源准备完毕才能初始化成功(有些资源无法直接通过winetricks下载)。初始化完成后将容器提交为新镜像`base_wine`
 - step-3 使用上一步`base_wine`镜像作为基础镜像编写微信的启动脚本并设置ENTRYPOINT为启动脚本
 - step-4 构建完成，将最终镜像命名为`toychat`。此时该镜像所创建的容器是能运行微信基础功能的，但是尚未安装微信。需要将微信安装包放置容器内特定目录以实现自动安装，具体细节由脚本实现
 
 也就是说实现这个需求需要有两个中间镜像，一个最终镜像。首个镜像`base`用于系统初始化，布置好wine所需要的可执行文件；使用`base`镜像并挂载winetricks所需资源后启动容器，初始化wine运行环境后提交为第二个镜像`base_wine`；最后使用`base_wine`作为基础镜像，将微信自动安装及启动脚本在dockerfile中布置好，构建完成即可得到最终镜像`toychat`
 
 ## 先决条件
  - 宿主机运行x11
  - 科学上网
 
 # 实践
 ## step-1 构建镜像`base`
 这一步很简单，首先需要准备的东西有
  - 一份`dockerfile`
  - 一个打包为tar的windows字体目录，重命名为模式`ms-fonts.tar`
  - 一份用以下载winetricks无法自动下载的库的`脚本`
  - 将以上工具放在一个空文件夹
 
 首先是写dockerfile,命名为`base.dockerfile`。基本是修改镜像原更新软件并安装软件：
 ```dockerfile
 FROM archlinux/archlinux:latest

#
# ASSIGN BUILD VAL IF WITHOUT THOSE VAR
#
# - PROXY_SERVER
# - 代理服务器设置
# ARG http_proxy=${http_proxy:-"IP:PORT"}
# ARG https_proxy=${https_proxy:-"IP:PORT"}
# ARG all_proxy=${all_proxy:-"IP:PORT"}
# - USER_ID
# - 宿主中运行容器内应用的用户UID
ARG UID=${UID:-"1000"}
ARG GID=${GID:-"1000"}

#
# WINE PATH
#
ENV HOME=/root
ENV WINEPREFIX=/root/wine64

#
# SYSTEM INITE
#
# - ADD ALIYUN MIRRORS AND MAKE IT PRIOR, ADD 'archlinux' ALIYUN REPOSITORY
# - 更换镜像原源为阿里并增加archlinux源
RUN sed -i '1i Server = http://mirrors.aliyun.com/archlinux/$repo/os/$arch\nServer = https://mirrors.aliyun.com/archlinux/$repo/os/$arch' /etc/pacman.d/mirrorlist && \
 sed -i '$a [multilib]\nInclude = /etc/pacman.d/mirrorlist' /etc/pacman.conf && \
 sed -i '$a [archlinuxcn]\nServer=https://mirrors.aliyun.com/archlinuxcn/$arch' /etc/pacman.conf &&\
 sed -i -e 's/^#MAKEFLAGS="-j2"/MAKEFLAGS="-j4"/g' /etc/makepkg.conf

#
# INSTALL BASE SOFTWARE
# THE STAGE THAT INSTALLS SOFTWARE COULD BE BETTER
#
RUN pacman-key --init && \
  pacman-key --lsign-key "farseerfc@archlinux.org" && \
	pacman -Sy --noconfirm && \
  pacman -S --noconfirm archlinuxcn-keyring && \
  pacman -Syu --needed --noconfirm && \
  pacman -S --noconfirm wget \
	curl \
	sudo \
	mesa \
	lib32-mesa \
	xorg-server 
#
# INSTALL WINE
# THE STAGE THAT INSTALLS SOFTWARE COULD BE BETTER
#
RUN  sed -i 's/^# %wheel ALL=(ALL:ALL) NOPASSWD: ALL/%wheel ALL=(ALL:ALL) NOPASSWD: ALL/' /etc/sudoers && \
  sed -i '$a wx ALL=(ALL:ALL) NOPASSWD:ALL' /etc/sudoers && \
  useradd -u ${UID} wx && \
  pacman -S --needed --noconfirm wine-for-wechat wine-mono wine_gecko giflib lib32-giflib libpng lib32-libpng libldap lib32-libldap gnutls lib32-gnutls \
pulseaudio-alsa pulseaudio mpg123 lib32-mpg123 openal lib32-openal v4l-utils lib32-v4l-utils libpulse lib32-libpulse libgpg-error \
lib32-libgpg-error alsa-plugins lib32-alsa-plugins alsa-lib lib32-alsa-lib libjpeg-turbo lib32-libjpeg-turbo \
sqlite lib32-sqlite libxcomposite lib32-libxcomposite libxinerama lib32-libgcrypt libgcrypt lib32-libxinerama \
ncurses lib32-ncurses opencl-icd-loader lib32-opencl-icd-loader libxslt lib32-libxslt libva lib32-libva gtk3 \
lib32-gtk3 gst-plugins-base-libs lib32-gst-plugins-base-libs vulkan-icd-loader lib32-vulkan-icd-loader winetricks && \
  pacman -Scc --noconfirm && \
	winetricks --self-update && \
	echo -e '#!/bin/bash\nwinetricks riched20 riched30 richtx32 msftedit gdiplus' >> /root/winetricks_init.sh && \
	chmod +x /root/winetricks_init.sh
#
# CONFIG WINE INITIAL SCRIPT FOR WINE
#

RUN sed -i '$a cd $HOME\nalias cd_doc="cd \\"$wx_doc\\""' /etc/bash.bashrc  && \
	printf "#!/bin/bash\nsource /etc/bash.bashrc\nsudo chown \"\$(id -u)\":\"\$(id -g)\" -R /root" >> /usr/bin/wx && \
	chmod +x /usr/bin/wx && \
	chown ${UID}:${GID} /usr/bin/wx && \
	chown ${UID}:${GID} -R /root 
	
	ENTRYPOINT /root/winetricks_init.sh

# - INSTALL MS-FONTS
# - 安装windows字体于容器字体目录
ADD ./ms-fonts.tar /usr/share/fonts/
 ``` 

将此dockerfile命名为`base.dockerfile`并放置于一个空目录，到达该目录并执行以下命令以完成构建:
```bash
docker build -f ./base.dockerfile -t=base .
```

检查镜像:
```bash
docker images | grep base
```

此步骤指定了宿主机中UID为1000的普通用户，并以此用户作为wine的拥有者，后续步骤也需要指定回此用户执行wine应用

另外，笔者直接将KVM中的` Win10 LTSC 2016 `中的字体目录打包为tar并解压到容器的字体目录中。这一步将带来不少不需要的字体。如果有一定的追求可以参考连接字体和修改注册表等等方法  <del>我都失败了</del>

## step-2 操作中间容器并提交中间镜像`base_wine`
回到刚才的目录中，假设该目录为`/home/baka/Documents/dockerfile/base_wine`。为`winetricks`提前准备好部分需要且无法通过winetricks直接下载的库文件(参考`base.dockerfile`第64行，`winetricks`
命令后的参数都是需要的库文件)，这里通过一个脚本`downloads.sh`实现下载这些文件:
```bash
#!/bin/bash

set -e

# 设置代理
# export https_proxy="IP:PORT"
# export http_proxy="IP:PORT"

mkdir -p './.cache/winetricks/win7sp1' 
mkdir -p './.cache/winetricks/msls31' 
mkdir -p './.cache/winetricks/riched30' 
mkdir -p './.cache/winetricks/win2ksp4' 
mkdir -p './.cache/winetricks/7zip' 
curl -LO --output-dir './.cache/winetricks/7zip' 'https://www.7-zip.org/a/7z1900.exe' 
curl -LO --output-dir './.cache/winetricks/win7sp1' 'http://download.windowsupdate.com/msdownload/update/software/svpk/2011/02/windows6.1-kb976932-x86_c3516bc5c9e69fee6d9ac4f981f5b95977a8a2fa.exe' 
curl -LO --output-dir './.cache/winetricks/win2ksp4' 'http://x3270.bgp.nu/download/specials/W2KSP4_EN.EXE' 
curl -LO --output-dir './.cache/winetricks/msls31' 'https://web.archive.org/web/20060701000000/http://download.microsoft.com/download/WindowsInstaller/Install/2.0/NT45/EN-US/InstMsiW.exe' 
curl -LO --output-dir './.cache/winetricks/riched30' 'https://web.archive.org/web/20060701000000/http://download.microsoft.com/download/WindowsInstaller/Install/2.0/W9XMe/EN-US/InstMsiA.exe' 
```

为该脚本添加可执行权限并执行:
```bash
chmod +x ./dowloads.sh
./downloads.sh
```

> 该脚本将下载到的内容存储到`.cache`隐藏目录

确保所有文件下载完成之后取消本机x认证，允许所有用户使用本地x服务器:
```bash
xhost +
```

创建一个以`base`为镜像的容器并挂载上述`.cache`到`/root/.cache`并共享x套接字:
```base
 docker run -it \
--name temp \
-u 1000 \
-e DISPLAY=$DISPLAY \
-v /tmp/.X11-unix:/tmp/.X11-unix:ro \
-v /home/baka/Documents/dockerfile/base_wine/.cache:/root/.cache:rw \ # 注意路径
--entrypoint /root/winetricks_init.sh \
base
```
之后稍等就会自动下载东西，等到东西下好会自动安装库。

中间会弹出两个安装窗口，只需点下一步即可(7z安装完需要手动close):

<img title="" src="https://telegraph.7cmb.com/file/61a06ebf68c76afb43cb6.png" alt="wine_init">

<img title="" src="https://telegraph.7cmb.com/file/d151ab98922f3263e183c.png" alt="wtricks_7z">

<img title="" src="https://telegraph.7cmb.com/file/f0238bf7da82506c27f1f.png" alt="wtricks_reg">

一切完成且并未出错后等到容器关闭，即可提交容器为`base_wine`镜像:
```bash
docker commit temp base_wine
```

> `.cache`及其他需要挂载的目录其实大可以放置于别的位置而不是构建目录，省去守护进程读取上下文的时间。此处方便演示故都放置于一个目录

## step-3 为上述镜像微信启动脚本层，并构建为`toychat`
这一步仅仅需要一个dockerfile，假设还在上述步骤的目录，命名为`toychat.dockerfile`:
```dockerfile
FROM base_wine

#
# ASSIGN BUILD VAL IF WITHOUT THOSE VAR
#
# - PROXY_SERVER
#ARG http_proxy=${http_proxy:-"IP:PORT"}
#ARG https_proxy=${https_proxy:-"IP:PORT"}
#ARG all_proxy=${all_proxy:-"IP:PORT"}
# - USER_ID
ARG UID=${UID:-"1000"}
ARG GID=${GID:-"1000"}

#
# WX_ENV
# WARNNING:DON'T MODIFY IT IF YOU DON'T KNOW WHAT YOU DO
#
# - ALL USER WILL USER '/root' AS THEIR HOME
ENV wx_doc="$WINEPREFIX/drive_c/users/wx/Documents/WeChat Files"

#
#	WX_WINE_LIGHT_LAYER
# CONFIG WECHAT AND GENERATE AN ENTRIPOINT
#
RUN sudo sed -i '$a cd $HOME\nalias cd_doc="cd \\"$wx_doc\\""' /etc/bash.bashrc  && \
	sudo printf "#!/bin/bash\nsource /etc/bash.bashrc\nsudo chown \"\$(id -u)\":\"\$(id -g)\" -R /root" >> /usr/bin/wx && \
	sudo chmod +x /usr/bin/wx && \
	sudo chown ${UID}:${GID} /usr/bin/wx && \
	sudo chown ${UID}:${GID} -R /root && \
	sudo printf "#%c/bin/bash\n. /etc/bash.bashrc\nsudo chown  -R \"\$(id -u)\":\"\$(id -g)\" /root\n#if [ -z \"\$(find /root -type f -regex '\\\/root\\\/wine64\\\/.*\\\.reg' 2> /dev/null )\" ];then\n#  . /root/winetricks_init.sh\nif [ -z \"\$(find /root -type f -regex '\\\/root\\\/wine64.*WeChat.exe' 2> /dev/null )\" ];then\n  wine \"\$wx_doc\WeChatSetup.exe\" 2>&1 >> /dev/null \n  [ \$? ] || (echo 'PLEASE PUT INSTALLER WeChatSetup.exe IN WECHAT DOCUMENT OF YOUR HOST' && false) || exit\n  export wx=\$(find \$WINEPREFIX -type f -regex \"\$WINEPREFIX\/.*\/WeChat.exe\" 2>/dev/null | head -n1)\n  sudo printf '\\\n  wine \"%%s\" &>> /dev/null &' \"\$wx\" >> /usr/bin/wx\nsleep 3\nfi\n#fi\nexport wx=\$(find \$WINEPREFIX -type f -regex \"\$WINEPREFIX\/.*\/WeChat.exe\" 2>/dev/null | head -n1)\n[ -n \"\$(pidof -x \"WeChat.exe\")\" ] || wx\nsleep 30\nwhile [ -n \"\$(pidof -x \"WeChat.exe\")\" ];do\n  sleep 60\ndone" ! > /root/init.sh && \
	sudo chmod +x /root/init.sh

ENTRYPOINT /root/init.sh

```

这一切只是为容器内添加`/root/init.sh`并设置为ENTRYPOINT，当然也能提前在宿主机写好并`ADD`进去...着一大坨printf着实看着眼花缭乱。`/root/init.sh`内容:
```bash
#!/bin/bash
. /etc/bash.bashrc
sudo chown  -R "$(id -u)":"$(id -g)" /root
if [ -z "$(find /root -type f -regex '\/root\/wine64.*WeChat.exe' 2> /dev/null
)" ];then
  wine "$wx_doc\WeChatSetup.exe" 2>&1 >> /dev/null
  [ $? ] || (echo 'PLEASE PUT INSTALLER WeChatSetup.exe IN WECHAT DOCUMENT OF YOUR HOST' && false) || exit
  export wx=$(find $WINEPREFIX -type f -regex "$WINEPREFIX\/.*\/WeChat.exe" 2>/dev/null | head -n1)
  sudo printf '\n  wine "%s" &>> /dev/null &' "$wx" >> /usr/bin/wx
sleep 3
fi
export wx=$(find $WINEPREFIX -type f -regex "$WINEPREFIX\/.*\/WeChat.exe" 2>/dev/null | head -n1)
[ -n "$(pidof -x "WeChat.exe")" ] || wx
```
构建命令:
```bash
docker build -f ./toychat.dockerfile -t=toychat .
```

构建完成后即可得到`toychat`镜像


## step-4 以`toychat`为镜像启动容器，在容器内安装任意版本微信



！记得将微信安装包放置于 宿主存放微信数据的目录 ，否则微信无法安装

！微信安装目录必须为wine中的c盘，初次点要求外路径随意



直接放脚本了，如果是`pulseaudio`为声音服务后端则:
```bash
#!/bin/bash
xhost +:local
pactl load-module module-native-protocol-unix socket=/tmp/pulseaudio.socket
sudo cat > '/tmp/pulseaudio.client.conf' <<EOF
default-server = unix:/tmp/pulseaudio.socket
# Prevent a server running in the container
autospawn = no
daemon-binary = /bin/true
# Prevent the use of shared memory
enable-shm = false
EOF
docker run \
    -d \
    --name toyChat-pulseaudio \
    -e PULSE_SERVER=unix:/tmp/pulseaudio.socket \
    -e PULSE_COOKIE=/tmp/pulseaudio.cookie \
    -v /tmp/pulseaudio.socket:/tmp/pulseaudio.socket \
    -v /tmp/pulseaudio.client.conf:/etc/pulse/client.conf \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix:ro \
    --ipc=host \
    -u 1000 \
    -e GTK_IM_MODULE=fcitx \
    -e QT_IM_MODULE=fcitx \
    -e XMODIFIERS=@im=fcitx \
    -e DefaultIMModule=fcitx \
	-e TZ="时区" \
    -v "宿主存放微信数据的目录":/root/wine64/drive_c/users/wx/Documents/WeChat\ Files:rw \
    toychat
```

不需要声音服务:
```bash
#!/bin/bash
xhost +:local
docker run \
    -d \
    --name toyChat \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix:ro \
    --ipc=host \
    -u 1000 \
    -e GTK_IM_MODULE=fcitx \
    -e QT_IM_MODULE=fcitx \
    -e XMODIFIERS=@im=fcitx \
    -e DefaultIMModule=fcitx \
	-e TZ="时区" \
    -v "宿主存放微信数据的目录":/root/wine64/drive_c/users/wx/Documents/WeChat\ Files:rw \
    toychat
```

微信安装路径可以为c盘任意目录，安装成功后就能直接使用微信了

图示:

<img title="" src="https://telegraph.7cmb.com/file/51529c8d285bae4f68d88.png" alt="wx_01">

<img title="" src="https://telegraph.7cmb.com/file/560fca1a217d03cb46576.png" alt="wx_02">


# 无需构建，一键脚本
无需构建，放弃思考脚本，啥也不用干，装了docker能科学上网就能用

作者已经将镜像上传至公共仓库，该脚本可以自动拉取，自动部署容器:
```bash
#!/bin/bash

set -eo pipefail
set -u

#
# VARIABLE SETTING
#
WX_DOC="$HOME/Documents/WeChatFiles"
IMG='7cmb/toychat:v0.01'
TZ="$(timedatectl | grep 'Time zone' | awk '{print $3}')"

#
# COLORFUL OUTPUT
#
GREEN='\e[42m\e[2m'
CYAN='\e[46m\e[2m'
BLINK='\e[5m'
NORMAL='\e[0m'
RED='\e[101m\e[2m'

# COPY FROM https://github.com/huan/docker-wechat/blob/main/dochat.sh#L39C4-L40C1
if [ "$EUID" -eq 0 ] && [ "${ALLOWROOT:-0}" -ne "1" ]
then
  echo -e "${RED}[WARNNING]${NORMAL}Please do not run this script as root."
  exit 1
fi

function hello () {
  cat <<'EOF'

   ______   ______     __  __     ______     __  __     ______     ______  
  /\__  _\ /\  __ \   /\ \_\ \   /\  ___\   /\ \_\ \   /\  __ \   /\__  _\ 
  \/_/\ \/ \ \ \/\ \  \ \____ \  \ \ \____  \ \  __ \  \ \  __ \  \/_/\ \/ 
     \ \_\  \ \_____\  \/\_____\  \ \_____\  \ \_\ \_\  \ \_\ \_\    \ \_\ 
      \/_/   \/_____/   \/_____/   \/_____/   \/_/\/_/   \/_/\/_/     \/_/ 
                                                                           
 
  NOTIFICATION:
    PLEASE PUT YOUR WECHAT INSTALLER INTO YOUR WECHAT DATA DIR 
		THAT IS IN "/home/$USER//Documents/WeChatFiles"BY DEFAULT.
		AND THE INSTALLER's NAME **MUST BE** "WeChatSetup.exe".

  提示：
    请提前将微信安装包放置于微信的文件目录 默认位置为
    "/home/$USER/Documents/WeChatFiles" 此外，微信安
    装包名称**必须**为"WeChatSetup.exe"
EOF
}

function pulseaudio_init () {
	if [ "$(pactl info | grep "Server Name" | awk '{print $NF}')"=="pulseaudio" ];then
		if [ -z "$(find /tmp -type s -regex '\/tmp\/pulseaudio.*' 2>/dev/null; find /tmp -type f -regex '\/tmp\/pulseaudio.*' 2>/dev/null)" ];then	
		  if [ -n "$(find /tmp -type d -regex '\/tmp\/pulseaudio.*' 2>/dev/null)" ];then
			  rm -rf '/tmp/pulseaudio.socket' '/tmp/pulseaudio.client.conf'
		  fi
		  pactl load-module module-native-protocol-unix socket=/tmp/pulseaudio.socket
		  echo -e "${CYAN}[PROCESSING]${NORMAL}THIS STEP IS TO WRITE A PULSEAUDIO CONFIG FOR CONTAINER,PLEASE ENTER YOUR PASSWORD"
		  sudo cat > '/tmp/pulseaudio.client.conf' <<EOF
default-server = unix:/tmp/pulseaudio.socket
# Prevent a server running in the container
autospawn = no
daemon-binary = /bin/true
# Prevent the use of shared memory
enable-shm = false
EOF
		fi
	fi
}

function install_pulseaudio_edition () {
	if [ "$(pactl info | grep "Server Name" | awk '{print $NF}')"=="pulseaudio" ];then
		if [ -z "$(find /tmp -type s -regex '\/tmp\/pulseaudio.*' 2>/dev/null; find /tmp -type f -regex '\/tmp\/pulseaudio.*' 2>/dev/null)" ];then	
		  if [ -n "$(find /tmp -type d -regex '\/tmp\/pulseaudio.*' 2>/dev/null)" ];then
			  rm -rf '/tmp/pulseaudio.socket' '/tmp/pulseaudio.client.conf'
		  fi
		  pactl load-module module-native-protocol-unix socket=/tmp/pulseaudio.socket
		  echo -e "${CYAN}[PROCESSING]${NORMAL}THIS STEP IS TO WRITE A PULSEAUDIO CONFIG FOR CONTAINER,PLEASE ENTER YOUR PASSWORD"
		  sudo cat > '/tmp/pulseaudio.client.conf' <<EOF
default-server = unix:/tmp/pulseaudio.socket
# Prevent a server running in the container
autospawn = no
daemon-binary = /bin/true
# Prevent the use of shared memory
enable-shm = false
EOF
		fi
  echo -e "${CYAN}[PROCESSING]${NORMAL}CREATING CONTAINER toyChat WITH PULSEAUDIO..." 
  docker run \
    -d \
    --name toyChat-pulseaudio \
    -e PULSE_SERVER=unix:/tmp/pulseaudio.socket \
    -e PULSE_COOKIE=/tmp/pulseaudio.cookie \
    -v /tmp/pulseaudio.socket:/tmp/pulseaudio.socket \
    -v /tmp/pulseaudio.client.conf:/etc/pulse/client.conf \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix:ro \
    --ipc=host \
    -u 1000 \
    -e GTK_IM_MODULE=fcitx \
    -e QT_IM_MODULE=fcitx \
    -e XMODIFIERS=@im=fcitx \
    -e DefaultIMModule=fcitx \
		-e TZ="$TZ" \
    -v "$WX_DOC":/root/wine64/drive_c/users/wx/Documents/WeChat\ Files:rw \
    "$IMG" &>> /dev/null
	 echo -e "${GREEN}[STATUS]${NORMAL}DONE"
	 fi
}

function install_base_edition () {
	echo -e "${CYAN}[PROCESSING]${NORMAL}CREATING CONTAINER toyChat..."
  docker run \
    -d \
    --name toyChat-base \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix:ro \
    --ipc=host \
    -u 1000 \
    -e GTK_IM_MODULE=fcitx \
    -e QT_IM_MODULE=fcitx \
    -e XMODIFIERS=@im=fcitx \
    -e DefaultIMModule=fcitx \
		-e TZ="$TZ" \
    -v "$WX_DOC":/root/wine64/drive_c/users/wx/Documents/WeChat\ Files:rw \
		"$IMG" &>> /dev/null
	echo -e "${GREEN}[STATUS]${NORMAL}DONE"
}

function detect_instance () {
  if [ -n "$(docker ps -a | grep "toyChat")" ];then
		echo -e "${CYAN}[PROCESSING]${NORMAL}INSTANCE FOUND .. CONTAINER WILL START AFTER A WHILE"
		if [ -n "$(docker ps -a | grep "toyChat-pulseaudio")" ];then
      pulseaudio_init
		  docker start toyChat-pulseaudio
			echo -e "${GREEN}[STATUS]${NORMAL}DONE"
			exit 0
		else
		 docker start toyChat
		 echo -e "${GREEN}[STATUS]${NORMAL}DONE"
		 exit 0
		fi
	fi
}

function main () {
	xhost +local:
  hello
	detect_instance
  docker pull "$IMG" &>> /dev/null
	install_pulseaudio_edition
	if [ -z "$(docker ps -a | grep toychat)" ];then
		install_base_edition
	fi
	echo 'ALL DONE, ENJOY'
	echo 'IT WILL AUTO START IF YOU INSTALL IT FIRST TIME, JUST WAIT'
	echo 'USE `docker ps -a | grep -i toychat` TO FIND YOUR CONTAINER '
	echo 'USE `docker start [CONTAINER NAME]` TO RUN IT'
}

main
```

复制粘贴到某个文件中运行，并放置好微信安装包到指定路径(默认`~/Documents/WeChatFiles`)即可

> 记得export代理，否则网络不通将导致运行失败

后续启动容器也可以直接运行脚本

脚本截图
<img title="" src="https://telegraph.7cmb.com/file/3b4ac85392092a6548f6b.png" alt="toychat">