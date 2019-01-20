---
categories: docker
layout: post
---

- Table
{:toc}

# 架构
Docker的核心组件有

- Docker客户端和服务器
- Docker镜像
- Registry
- Docker容器

## 客户端和服务器

Docker是一个C/S架构的应用。Docker客户端只需向Docker服务器（守护进程）发起请求，而服务器将完成所有工作并返回结果作为响应。Docker服务器也称为Docker引擎。

Docker提供了命令行工具docker以及一套RESTful API来与服务器交互。用户的客户端和服务器可以运行在相同的宿主机上，也可以运行在不同的宿主机上。

用户可以使用docker daemon命令控制Docker守护进程。

Docker以root权限运行服务器进程和客户端程序，来完成普通用户无权完成的操作（比如挂载文件系统）。

Docker守护进程会监听/var/run/docker.sock这个Unix套接字文件，来获取客户端的请求。如果系统中存在名为docker的用户组的话，Docker会将套接字文件的所有者设置为该用户组，这样docker用户组的所有用户都可以直接运行Docker，而无需使用sudo命令了。

## 镜像

镜像是Docker世界的基石，用户基于镜像来运行自己的容器。可以把镜像当做容器的源代码，有着体积小的优点。

## Registry

Docker利用Registry来保存用户构建的镜像。Registry分为公共和私有两种，Docker公司运营的Docker Hub是公共Registry。

## 容器

容器是基于镜像启动起来的应用，其中可以运行一个或多个进程。

Docker借鉴了集装箱的概念，集装箱将货物运往各地，集装箱不关心内部的货物。在Docker中， 容器就是集装箱，Docker就是运输集装箱的船只，而镜像就是集装箱中的货物，而你的笔记本、服务器等就是集装箱的目的地。

# 快速上手

## 安装

安装之前，需要先了解自己所在系统是否兼容。Docker有以下要求：

- 64位CPU
- Linux3.8或更高版本内核
- 内核支持一种合适的存储驱动
- 内核必须开启cgroup和namespace

## 运行docker守护进程

```sh
service docker start
```

启动docker守护进程

```sh
service docker status
```

查看docker守护进程状态

## 查看docker信息

```sh
docker info
```

会显示docker的各项信息

## 第一个容器，运行ubuntu镜像

```shj
docker run -i -t ubuntu /bin/bash
```

-i和-t是交互式shell的必要参数，前者保证容器的STDIN是开启的，而后者则告诉Docker为创建的容器分配一个伪tty终端。这样新建的容器才能提供一个交互式的shell。

ubuntu指定了使用的是ubuntu镜像，而ubuntu镜像是一个常备镜像，由docker公司提供，保存在Docker Hub上。可以以操作系统镜像为基础，在其上构建自己的镜像。

Docker首先会检查本地是否存在ubuntu镜像缓存，如果不存在，会连接Docker Hub，搜索它上面是否有该镜像副本，一旦找到，就会下载到本地。

随后，Docker在文件系统内部用这个镜像创建一个新容器，该容器拥有自己的网络，IP地址，以及用来与宿主机通信的桥接网络接口。

/bin/bash要求docker在容器创建完毕后执行容器中的/bin/bash命令。这时就可以看到容器内的shell了。

之后我们在容器中安装vim软件。

```sh
apt-get update && apt-get intall vim
```

用户可以在容器中做任何自己想做的事情，当工作完成后，输入exit返回到宿主机。exit会退出/bin/bash命令，而bin/bash命令是容器的唯一命令，一旦/bin/bash退出，容器也会相应地退出。

## 查看所有容器信息

但是即使容器退出了，容器依旧是存在的，可以用docker ps -a命令查看当前系统中所有容器。默认情况下docker ps只会查看正在运行的容器，加了-a可以查看所有的容器，包括运行的和停止的。

```sh
docker ps #查看运行中容器
docker ps -a #查看所有的容器
docker ps -l #查看最后一个执行的容器
```

## 容器标志符

有三种方式可以唯一指定容器，短UUID，长UUID以及容器名。短UUID是长UUID的一个前缀。

如果你没有指定容器名称，那么docker会为我们创建的容器自动生成一个随机名称，如果要自己指定的话，--name标志可以指定自定义名称。

```sh
docker run --name bob_the_contaienr -i -t ubuntu /bin/bash
exit
docker ps -l #显示最后一个容器
docker ps -n 10 #显示最后10个容器
```

一个合法的容器名必须满足正则表达式"[a-zA-Z0-9_.-]+"。

使用容器的名称有助于分辨容器，理解容器之间的连接关系，推荐使用容器名称管理容器。

容器的名称必须是唯一的，如果试图创建同名容器，则命令会失败。可以通过docker rm命令删除同名容器后再创建。

## 重启容器

之前的bob_the_container容器已经停止了，我们可以重新启动它。

```sh
docker start bob_the_container
```

继续使用ps命令可以看到bob_the_container处于UP状态。

除了用start命令外，也可以用restart重启一个容器，只需要将start替换为restart即可。

## 附着到容器

容器重启后，会沿用docker run命令时指定的参数来运行，因此我们容器重启后会运行一个交互式会话shell。此外，可以用docker attach命令重新附着到该容器的会话上。

```sh
docker attach bob_the_container
```

## 创建守护式容器

除了交互式容器外容器外，还可以创建守护式容器，守护式容器没有交互式会话，非常适合运行应用程序和服务。在运行时增加-d参数可以声明容器为守护式容器。

```sh
docker run --name deamon_dave -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

利用docker ps命令可以看到创建了一个正在运行的容器。

## 查看容器日志

由于守护容器在后台运行，要探究该容器内部都干了什么，可以用docker logs命令来获取容器的日志。

```sh
docker logs daemon_dave
```

要退出容器跟踪，输入ctrl+c。docker仅会返回最后的几条日志。使用-f参数可以持续监控日志。

```sh
docker logs -f daemon_dave
```

为了便于调试，使用-t标志位每条日志项追加上时间戳。

## docker日志驱动

自docker 1.6起，我们可以控制docker守护进程和容器所用的日志驱动，者可以通过--log-driver选项。

默认的日志驱动是json-file，可用的选项还有syslog，该选项会禁用所有docker logs命令，并将所有容器的日志输出都重定向到syslog上。可以在启动Docker守护进程时候指定该选项，这样所有容器的日志都会输出到syslog，或者通过docker run对个别容器进行日志冲定向输出。

```sh
docker run --log-driver="syslog" --name deamon_dwayne -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
tail -f /var/log/*
```

还有一个可用选项是none，这个选项会禁用所有容器中的日志。

## 查看容器内进程

我们不仅能看到容器内的日志，还能看到容器内部运行的进程。

```sh
docker top daemon_dave
```

利用top命令，可以看到容器内的所有进程。

## 查看容器统计信息

利用docker stats命令，可以看到一个或多个容器的统计信息。

```sh
docker stats --no-stream
```

## 在容器内部运行进程

在docker中，可以通过docker exec命令在容器内额外启动新的进程，可以在容器内运行的进程有两类：后台任务和交互式任务。后台任务在容器内运行且没有交互需求，而交互式任务则保持在前台运行。

```sh
docker exec -d daemon_dave touch /etc/new/config/file
```

这里-d标志表明运行的是一个后台进程。而不加-d则可以开启一个交互任务，比如开启一个新的shell。

```sh
docker exec -t -i daemon_dave /bin/bash
```

## 停止守护式容器

要停止守护式容器，只需要执行docker stop命令。

```sh
docker stop daemon_dave
```

docker stop命令会向Docker容器进程发送SIGTERM信号，如果想要快速停止某个容器，可以使用docker kill命令向容器进程发送SIGKILL命令。

```sh
docker kill daemon_dave
```

## 自动重启容器

可以增加--restart标记，让docker容器因错误而停止运行时自动重启。docker会检查容器的退出代码，并依据此决定是否要重启容器。默认情况下docker不会重启容器。

```sh
docker run --restart always --name daemon_restart -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

在上面这个例子中，--restart被设置为always，无论容器的退出代码实是什么，docker都会自动重启容器。除了always外，可选的值还有on-failure，只有当容器的退出代码非0时才会自动重启。同时on-failure还支持最大重启次数。

# 配置

## 守护进程

### 修改监听地址

```sh
docker daemon -H tcp://0.0.0.0:2375
```

上面的命令是一次性的。

```sh
export DOCKER_HOST="tcp://0.0.0.0:2375"
```

上面的改变会持久到重启。

```sh
docker deamon -H unix://home/docker/docker.sock
```

可以使用Unix套接字地址，这样就可以仅本地访问服务器。

```sh
docker daemon -H tcp://0.0.0.0:2375 -H unix://home/docker/docker.sock
```

也可以一次性监听多个地址。

### 调试模式

```sh
docker deamon -D
```

已调试模式启动守护进程，这样会输出额外的冗余信息。

可以修改/usr/lib/systemd/system/docker.service或/etc/sysconfig/docker文件修改ExecStart项来永久变更是否默认开启调试模式。




