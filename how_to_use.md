### 使用说明
> 说明介绍： http://www.voidcn.com/article/p-prehfrox-nz.html

> http://tinylab.org/based-on-ssh-build-docker-xpra-desktop/

> http://mobile.www.cnblogs.com/SkyYChen/articles/4890581.html
* 修改为固定密码(此方法暂时有问题，不使用，使用随机密码)
```
# 原始startup.sh脚本中采用pwgen生成密码，每次启动新容器时密码都会变化，为了测试使用，可将startup.sh中DOCKER_PASSWD变量设置为固定值。可采用两种方法进行修改，法1重新build镜像，法2进入容器内，修改startup.sh脚本，然后到commit得到新的镜像

# 修改结果如下
# DOCKER_PASSWORD=`pwgen -c -n -1 12`
DOCKER_PASSWORD=`docker`
echo User: docker Password: $DOCKER_PASSWORD
```
* 构建镜像
```
git clone https://github.com/linrong1994/docker-desktop.git
cd docker-desktop
docker build -t linrong/docker-desktop:1.0 .
```
* 运行
```
# 后台运行
# -p 2222:22 把容器内的 Ssh 端口地址 22 映射到主机的 2222 端口
CONTAINER_ID=$(docker run -d -p 2222:22 linrong/docker-desktop:1.0)
# 打印运行的log获取运行时获得的密码
echo $(docker logs $CONTAINER_ID | sed -n 1p)
显示：
User: docker Password: weas0viXaDuv
```
* 连接
```
查看端口映射
docker port $CONTAINER_ID 22

# 查看本机地址
$ ifconfig | grep "inet addr:" 
inet addr:192.168.56.102  Bcast:192.168.56.255  Mask:255.255.255.0 # This is the LAN's IP for this machine

# 通过 Ssh 启动一个 Xpra 会话
# -p 2222 连上 docker 那边的 ssh 服务
# -s 800x600 设置桌面的分辨率
# -d 10 设置显示服务会话编号
$ ssh docker@172.16.50.4 -p 2222 "sh -c './docker-desktop -s 800x600 -d 10 > /dev/null 2>&1 &'" # Here is where we use the external port
docker@172.16.50.4's password: xxxxxxxxxxxx 

# xpra连接（这个需要使用xpra的cmd或者软件）
xpra --ssh="ssh -p 49153" attach ssh:docker@192.168.56.102:10 # user@ip_address:session_number
docker@192.168.56.102's password: xxxxxxxxxxxx 
# 使用软件时的填充
docker 172.16.50.4 2222 10
密码不用填
点击连接，弹窗输入密码
```

### 其他的资料说明
> 基于ssh+Xpra+Xephyr构建Docker桌面系统
* Xpra
```
Xpra是一个允许你本地显示一个窗口的工具，该窗口的显示的是在远端主机上运行的X Clients。它不同于X转发，它能在不打断远端进程的同时断开或者重连远端主机。它也不同于VNC，通过xpra, 远端的应用能被本地的窗口管理其当作一个普通的桌面应用进行管理，而不像VNC只能在vncviewer或类似软件（如上文中的浏览器）中显示。
```
* Xephyr
```
Xephyr 是一个 Xserver，但是它执行在一个存在的 X server 里面，这个可以用来做很多事情，比如需要通过 XDMCP 连接到另外一台主机，那么不需要另外打开一个新的 X server；又比如正在写一个 window manager，那么在一个 X server 里面打开的 X server 里面调试，将会比直接在现有的 X server 里面替换现有的 window manager 方便很多。

在本方案中，Xephyr是xpra运行的一个应用，之后通过Xephyr运行完整的一套桌面环境。
```
* 使用脚本运行（暂无测试，目前使用上面的）
```
#!/bin/bash
xhost+

# docker image to use
DOCKER_IMAGE_NAME="desktop_ssh_passwd"
 
# local name for the container
DOCKER_CONTAINER_NAME="lc_test_desktop"
 
# check if container already present
TMP=$(docker ps -a | grep ${DOCKER_CONTAINER_NAME})
CONTAINER_FOUND=$?
 
TMP=$(docker ps | grep ${DOCKER_CONTAINER_NAME})
CONTAINER_RUNNING=$?
 
if [ $CONTAINER_FOUND -eq 0 ]; then
 
        echo -n "container '${DOCKER_CONTAINER_NAME}' found, "
 
        if [ $CONTAINER_RUNNING -eq 0 ]; then
               echo "already running"
        else
               echo -n "not running, starting..."
               TMP=$(docker start ${DOCKER_CONTAINER_NAME})
               echo "done"
        fi
 
else
        echo -n "container '${DOCKER_CONTAINER_NAME}' not found, creating..."
        TMP=$(docker run -d -P --name ${DOCKER_CONTAINER_NAME} ${DOCKER_IMAGE_NAME})
        echo "done"
fi
 
#wait for container to come up
sleep 2
 
# find ssh port
SSH_URL=$(docker port ${DOCKER_CONTAINER_NAME} 22)
SSH_URL_REGEX="(.*):(.*)"
 
SSH_INTERFACE=$(echo $SSH_URL | awk -F  ":" '/1/ {print $1}')
SSH_PORT=$(echo $SSH_URL | awk -F  ":" '/1/ {print $2}')
 
echo "ssh running at ${SSH_INTERFACE}:${SSH_PORT}"
 
ssh docker@${SSH_INTERFACE} -p ${SSH_PORT} "sh -c './docker-desktop -s 1440x900 -d 10 > /dev/null 2>&1 &'"
 
sleep 15
 
xpra --ssh="ssh -p ${SSH_PORT}" attach ssh:docker@${SSH_INTERFACE}:10
#echo "ssh docker@${SSH_INTERFACE} -p ${SSH_PORT} \"sh -c './docker-desktop -s 1440x900 -d 10 > /dev/null 2>&1 &'\""
#echo xpra --ssh=\"ssh -p ${SSH_PORT}\" attach ssh:docker@${SSH_INTERFACE}:10
```
* Dockerfile介绍
```
FROM ubuntu:14.04
MAINTAINER Roberto G.Hashioka "roberto_hashioka@hotmail.com"
RUN apt-get update -y
RUN apt-get upgrade -y

# Set the env variableDEBIAN_FRONTEND to noninteractive
ENV DEBIAN_FRONTENDnoninteractive

# Installing theenvironment required: xserver, xdm, flux box, roc-filer and ssh
RUN apt-get install -yxpra rox-filer openssh-server pwgen xserver-xephyr xdm fluxbox xvfb sudo

# Configuring xdm toallow connections from any IP address and ssh to allow X11 Forwarding.
RUN sed -i's/DisplayManager.requestPort/!DisplayManager.requestPort/g'/etc/X11/xdm/xdm-config
RUN sed -i '/#anyhost/c\*' /etc/X11/xdm/Xaccess
RUN ln -s/usr/bin/Xorg /usr/bin/X
RUN echo X11Forwardingyes >> /etc/ssh/ssh_config #配置X转发

# Fix PAM login issuewith sshd
RUN sed -i's/session    required     pam_loginuid.so/#session    required    pam_loginuid.so/g' /etc/pam.d/sshd

# Upstart and DBushave issues inside docker. We work around in order to install firefox.
RUN dpkg-divert--local --rename --add /sbin/initctl && ln -sf /bin/true /sbin/initctl

# Installing fusepackage (libreoffice-java dependency) and it's going to try to create
# a fuse devicewithout success, due the container permissions. || : help us to ignore it.
# Then we are going todelete the postinst fuse file and try to install it again!
# Thanks Jerome forhelping me with this workaround solution! :)
# Now we are able toinstall the libreoffice-java package 
RUN apt-get -y installfuse  || :
RUN rm -rf/var/lib/dpkg/info/fuse.postinst
RUN apt-get -y installfuse

# Installing the apps:Firefox, flash player plugin, LibreOffice and xterm
# libreoffice-baseinstalls libreoffice-java mentioned before
RUN apt-get install -ylibreoffice-base firefox libreoffice-gtk libreoffice-calc xterm

# Set locale (fix thelocale warnings)
RUN localedef -v -c -ien_US -f UTF-8 en_US.UTF-8 || :

# Copy the files intothe container
ADD . /src
EXPOSE 22

# Start xdm and sshservices.
CMD["/bin/bash", "/src/startup.sh"] 
#启动容器时运行脚本startup.sh，该脚本主要负责创建用户并设置用户密码以供ssh连接时使用
```