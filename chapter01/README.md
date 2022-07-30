## 物理机、虚拟机、容器的关系

容器提供操作系统级别的进程隔离，而虚拟机提供硬件虚拟化的隔离。



![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730110503.png)



用个类比来极简说明一下。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730110657.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730110635.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730110706.png)

## Docker 核心概念



![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730101150.png)



**镜像**：一个只读的文件和文件夹组合。

（强烈推荐）gcr.io、k8s.gcr.io、quay.io、ghcr.io 等国外镜像加速下载服务：https://github.com/togettoyou/hub-mirror

**容器**：容器是镜像的运行实体。



![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730104728.png)

**仓库**：Docker 的镜像仓库类似于代码仓库，用来存储和分发 Docker 镜像。镜像仓库分为公共镜像仓库和私有镜像仓库。[Docker Hub](https://hub.docker.com/) 是 Docker 官方的公开镜像仓库。企业内私有仓库通常使用 [Harbor](https://mp.weixin.qq.com/s/Dg4AID3v9evMKtBAJrRq4g) 来搭建。



## Docker 架构

**Docker 客户端**：用户与 Docker 服务端交互的方式，包括 docker 命令，REST API，各种语言的 SDK。

**Docker 服务端**：负责响应和处理来自 Docker 客户端的请求，然后将客户端的请求转化为 Docker 的具体操作。例如镜像、容器、网络和挂载卷等具体对象的操作和管理。

**runc（低级容器运行时）**：runc 是 Docker 官方按照 OCI 容器运行时标准的一个实现。通俗地讲，runC 是一个用来运行容器的轻量级工具，是真正用来运行容器的。

**containerd（高级容器运行时）**：containerd是 Docker 服务端的一个核心组件，它是从dockerd中剥离出来的 ，它的诞生完全遵循 OCI 标准，是容器标准化后的产物。containerd通过 containerd-shim 启动并管理 runC，可以说containerd真正管理了容器的生命周期。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730095939.png)



## Kubernetes 和 Docker 的关系

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730100925.png)

## Docker 安装

参见 [Install Docker Engine](https://docs.docker.com/engine/install/)

## Docker 命令

参见 [Command-line reference](https://docs.docker.com/reference/)

### 容器命令

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730103213.png)

**创建并启动容器**

- `-d` 是`--detach`的简写，它的作用是**在后台运行容器，并且打印容器 id**
- `-t`是`--tty`的简写，它的作用是**分配一个伪 TTY**
- `-i`是`--interactive`的简写，它的作用是**即使没有 attached，也要保持 STDIN 打开状态**

在交互模式下，`-i`与`-t`选项必须结合使用，也就是`-it`。

```bash
docker run -itd --name mynginx nginx
```

**列出容器**

```
docker ps 
```

**查看容器日志**

```bash
docker logs mynginx
```

**查看容器中的进程**

```bash
docker top mynginx
```

查看容器信息（IP 地址，进程号）

```
docker inspect mynginx
```

**进入容器**

```bash
docker exec -it mynginx bash
```


**删除容器**

```bash
# 先停止容器
docker stop mynginx
# 然后删除容器
docker rm mynginx


# 也可以使用 -f 参数强制删除
docker rm -f mynginx
```

**临时调试**

--rm 退出容器以后，这个容器就被删除了，方便在临时测试使用。

```bash
docker run --rm -it --name test busybox:1.28
```





### 镜像命令



![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730103224.png)



**拉取镜像**：使用 `docker pull` 命令拉取远程仓库的镜像到本地。

```bash
# 语法：docker pull [Registry]/[Repository]/[Image]:[Tag]
docker pull busybox:1.28
```

**重命名镜像**：使用 `docker tag` 命令“重命名”镜像。

```bash
# 语法： docker tag [SOURCE_IMAGE][:TAG] [TARGET_IMAGE][:TAG]
docker tag busybox:1.28 mybusybox:1.28
```

**查看镜像**：使用 `docker image ls` 或 `docker images` 命令查看本地已经存在的镜像 。

**删除镜像**：使用 `docker rmi` 命令删除无用镜像。

**构建镜像**：构建镜像有两种方式。第一种方式是使用 `docker build` 命令基于 Dockerfile 构建镜像；第二种方式是使用 `docker commit` 命令基于已经运行的容器提交为镜像。



使用  `docker commit` 命令构建镜像。

```bash
# 先创建一个容器
docker run --rm --name=busybox -it busybox sh
# 进入容器
docker exec -it busybox sh
# 往容器中写入一个文件
touch hello.txt && echo "I love Docker. " > hello.txt
# 退出容器后，使用 docker commit 使用容器构建镜像
docker commit busybox busybox:hello
```



使用 `docker build` 命令基于Dockerfile 构建镜像。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730104019.png)



推荐阅读：[构建 Go 应用 docker 镜像的几种姿势](https://mp.weixin.qq.com/s/LJ5mECUh-jRcBlVZ98q61A)

**多阶段构建**

```yaml
# 编译 golang 代码
FROM golang:alpine AS builder

WORKDIR /build

ADD go.mod .
COPY . .
RUN go build -o hello hello.go


# 将编译好的二进制文件拷贝到较小的基础镜像中运行
FROM alpine

WORKDIR /build
COPY --from=builder /build/hello /build/hello

CMD ["./hello"]
```



## 容器网络

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730111211.png)

### 容器网络模式

#### Bridge 网络模式

默认情况下，新创建的容器在不指定网络的情况下会采用 bridge 模式，并自动桥到 docker0 网桥。容器出访时，会 SNAT 成宿主机的 IP 地址。

```bash
# 启动容器
docker run --name centos centos

# 查看网桥
# brctl 命令安装：yum install -y bridge-utils
brctl show
docker0         8000.0242c7fd143b       no              vethbf7ce62
```

查看容器和宿主机 veth pair 的对应关系。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730112245.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730111430.png)



#### Host 网络模式

host 网络模式需要在创建容器时通过参数 `--net host` 或者 `--network host` 指定。采用 host 网络模式的 容器，可以直接使用宿主机的 IP 地址与外界进行通信，同时容器内服务的端口也可以使用宿主机的端口，无需额外进行 NAT 转换。

```bash
# 启动容器
docker run -idt --name centos --net host centos

# 进入容器
docker exec -it centos bash

# 查看网络，和宿主机的网络一致
ip addr 
```



#### None 网络模式

none 网络模式是指禁用网络功能，只保留 localhost 本地环回接口。在创建容器时通过参数 `--net none` 或者 `--network none` 指定。

```bash
# 启动容器
docker run -idt --name centos --net none centos

# 进入容器
docker exec -it centos bash

# 查看网络，只有本地环回接口
ip addr 
```



#### Container 网络模式

- Container 网络模式是 Docker 中一种较为特别的网络的模式。在创建容器时通过参数 `--net container:已运行的容器名称|ID` 或者 `--network container:已运行的容器名称|ID` 指定；
- 处于这个模式下的 Docker 容器会共享一个网络栈，这样两个容器之间可以使用 localhost 高效快速通信。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730114435.png)

```bash
# 创建第一个容器
docker run -itd --name pause busybox:1.28
# 创建第二个容器连接第一个容器
docker run -itd --name myapp --net container:pause busybox:1.28
# 查看两个容器的网络，网络一致
docker exec -it pause ip addr
docker exec -it myapp ip addr
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730114752.png)



### 容器间通过名称访问

#### Docker link（废弃）

Docker link 是一个遗留的特性，在新版本的 Docker 中，一般不推荐使用。简单来说 Docker link 就是把两个容器连接起来，容器可以使用容器名进行通信，而不需要依赖 ip 地址（**其实就是在容器的 /etc/hosts 文件添加了 host 记录**，原本容器之间的 IP 就是通的，只是我们增加了 host 记录）



创建容器 centos-1：

```sh
[root@host1 ~]# docker run -itd --name centos-1  registry.cn-shanghai.aliyuncs.com/public-namespace/cr7-centos7-tool:v2
```
创建容器 centos-2，使用 --link name:alias，name 就是要访问的目标机器，alias 就是自定义的别名。

```sh
[root@host1 ~]# docker run -itd --name centos-2  --link centos-1:centos-1-alias  registry.cn-shanghai.aliyuncs.com/public-namespace/cr7-centos7-tool:v2
```
查看容器 centos-2 的 /etc/hosts 文件：
```sh
[root@host1 ~]# docker exec centos-2 cat /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.18.0.2      centos-1-alias 9dde6339057a centos-1  #容器 centos-1 的 host 记录
172.18.0.3      f1a7e5fa3d96  #容器 centos-2 自身的 host 记录
```
意味着 centos-2 可以用 centos-1-alias，9dde6339057a，centos-1 来访问原先创建的容器。centos-1 是不可以通过 hostname 访问 centos-2 的。

```vb
[root@host1 ~]# docker exec centos-2 ping centos-1-alias
PING centos-1-alias (172.18.0.2) 56(84) bytes of data.
64 bytes from centos-1-alias (172.18.0.2): icmp_seq=1 ttl=64 time=0.174 ms
^C
[root@host1 ~]# docker exec centos-2 ping centos-1
PING centos-1-alias (172.18.0.2) 56(84) bytes of data.
64 bytes from centos-1-alias (172.18.0.2): icmp_seq=1 ttl=64 time=1.37 ms
64 bytes from centos-1-alias (172.18.0.2): icmp_seq=2 ttl=64 time=0.523 ms
^C
[root@host1 ~]# docker exec centos-2 ping 9dde6339057a
PING centos-1-alias (172.18.0.2) 56(84) bytes of data.
64 bytes from centos-1-alias (172.18.0.2): icmp_seq=1 ttl=64 time=2.59 ms
64 bytes from centos-1-alias (172.18.0.2): icmp_seq=2 ttl=64 time=3.75 ms
```

#### Embedded DNS 

从 Docker 1.10 开始，Docker 提供了一个内置的 DNS 服务器,当创建的容器属于自定义网络时，容器的 /etc/resolv.conf 会使用内置的 DNS 服务器（地址永远是 127.0.0.11）来解析相同自定义网络内的其他容器。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210311230820.png)

为了向后兼容，default bridge 网络的 DNS 配置没有改变，默认的 docker 网络使用的是宿主机的 /etc/resolv.conf 的配置。

创建一个自定义网络：
```sh
[root@host1 ~]# docker network create my-network

# bridge,host,none 为 docker 默认创建的网络
[root@host1 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
2115f17cd9d0        bridge              bridge              local
19accfa096cf        host                host                local
a23c8b371c7f        my-network          bridge              local
0a33edc20fae        none                null                local
```
分别创建两个容器属于自定义网络 my-network 中：

```sh
[root@host1 ~]# docker run -itd --name centos-3 --net my-network  registry.cn-shanghai.aliyuncs.com/public-namespace/cr7-centos7-tool:v2 
[root@host1 ~]# docker run -itd --name centos-4 --net my-network  registry.cn-shanghai.aliyuncs.com/public-namespace/cr7-centos7-tool:v2  
```

查看容器 centos-4 的 /etc/resolv.conf，可以看到 nameserver 添加的 IP 为 127.0.0.11 的 Embedded DNS：

```sh
[root@host1 ~]# docker exec centos-4 cat /etc/resolv.conf
nameserver 127.0.0.11
options ndots:0
```
此时 centos-3 和 centos-4 可以互相解析：

```vb
[root@host1 ~]# docker exec centos-4 ping centos-3
PING centos-3 (172.19.0.2) 56(84) bytes of data.
64 bytes from centos-3.my-network (172.19.0.2): icmp_seq=1 ttl=64 time=0.128 ms
64 bytes from centos-3.my-network (172.19.0.2): icmp_seq=2 ttl=64 time=0.078 ms
64 bytes from centos-3.my-network (172.19.0.2): icmp_seq=3 ttl=64 time=0.103 ms
^C

[root@host1 ~]# docker exec centos-3 ping centos-4
PING centos-4 (172.19.0.3) 56(84) bytes of data.
64 bytes from centos-4.my-network (172.19.0.3): icmp_seq=1 ttl=64 time=0.087 ms
64 bytes from centos-4.my-network (172.19.0.3): icmp_seq=2 ttl=64 time=0.101 ms
64 bytes from centos-4.my-network (172.19.0.3): icmp_seq=3 ttl=64 time=0.076 ms
^C
```



#### 容器端口映射

默认情况下，外部无法直接访问到容器。假如一个容器想对外提供服务的话，需要进行端口映射。端口映射将容器的某个端口映射到 Docker 主机端口上。那么任何发送到该端口的流量，都会被转发到容器中。



![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730115952.png)



假设我们运行了一个新的 web 服务容器，并且将容器 80 端口映射到 Dokcer 主机的 5000 端口。

```bash
docker run -d --name web -p 5000:80 nginx
```

## 容器存储

- **bind mounts**：是将宿主机的文件或目录挂载到容器中，**注意宿主机的路径要写绝对路径**。
- **volumes** 的数据存放在 /var/lib/docker 目录中，而 **volumes** 完全由 Docker 管理。
- **tmpfs**：相对于 **bind mount** 和 **volume**，**tmpfs** 是临时的，并且只会存储在主机的内存中，一旦停止容器，**tmpfs** 的挂载将会被移除，并且文件不会持久化。 **tmpfs** 不能在容器间共享。对于想临时存储不想保留在主机或容器可写层中的敏感文件很有用，因为可写层数据存放在内存中，无法在硬盘上找到。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220730120706.png)



### Bind Mounts

将宿主机当前的目录挂载到容器的 /app目录。
**方式一：-v**

```vim
docker run -d \
  -it \
  --name devtest \
  -v "$(pwd)":/app \
  nginx:latest
```
**方式二：--mount**

```vim
docker run -d \
  -it \
  --name devtest \
  --mount type=bind,source="$(pwd)",target=/app \
  nginx:latest
```

### Volumes

**先创建一个 volume**

```vim
docker volume create myvol2
```

**方式一：-v**

```vim
docker run -d \
  --name devtest \
  -v myvol2:/app \
  nginx:latest
```

**方式二：--mount**

```vim
docker run -d \
  --name devtest \
  --mount source=myvol2,target=/app \
  nginx:latest
```



### Tmpfs

**方式一：--tmpfs**  

```vim
docker run -d \
  -it \
  --name tmptest \
  --tmpfs /app \
  nginx:latest
```

**方式二：--mount**    

```vim
docker run -d \
  -it \
  --name tmptest \
  --mount type=tmpfs,destination=/app \
  nginx:latest
```
## Docker Compose 容器编排

Compose 是一个用于定义和运行多容器的工具。

推荐阅读：[Get started with Docker Compose](https://docs.docker.com/compose/gettingstarted/#web-service)

```yaml
version: "3.9"
services:
  web:
    build: .
    ports:
      - "8000:5000"
    volumes:
      - .:/code
    environment:
      FLASK_ENV: development
  redis:
    image: "redis:alpine"
```



## 参考资料

- [拉勾-由浅入深吃透 Docker](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=455#/detail/pc?id=4572)
- [Containerd深度剖析-runtime篇](https://www.modb.pro/db/413691)
- [构建 Go 应用 docker 镜像的几种姿势](https://mp.weixin.qq.com/s/LJ5mECUh-jRcBlVZ98q61A)
- [Docker 网络模式详解及容器间网络通信](https://www.cnblogs.com/mrhelloworld/p/docker11.html)
- [花了三天时间终于搞懂 Docker 网络了](https://cloud.tencent.com/developer/article/1747307)
- [Get started with Docker Compose](https://docs.docker.com/compose/gettingstarted/#web-service)

