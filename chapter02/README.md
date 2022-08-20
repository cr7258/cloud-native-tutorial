## 资源隔离：Namespace 

Namespace 是 Linux 内核的一个特性，该特性可以实现在同一主机系统中，对进程 ID、主机名、用户 ID、文件名、网络和进程间通信等资源的隔离。Docker 利用 Linux 内核的 Namespace 特性，实现了每个容器的资源相互隔离，从而保证容器内部只能访问到自己 Namespace 的资源。



目前 Linux 内核中提供了 8 种类型的 Namespace：
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210321194422.png)



### 隔离挂载点：Mount Namespace

首先我们使用以下命令创建一个 bash 进程并且新建一个 Mount Namespace：

```sh
unshare --mount --fork /bin/bash
```
执行完上述命令后，这时我们已经在主机上创建了一个新的 Mount Namespace，并且当前命令行窗口加入了新创建的 Mount Namespace。下面我通过一个例子来验证下，在独立的 Mount Namespace 内创建挂载目录是不影响主机的挂载目录的。

然后在 /tmp 目录下创建一个目录，创建好目录后使用 mount 命令挂载一个 tmpfs 类型的目录：
```sh
mkdir /tmp/tmpfs
mount -t tmpfs -o size=20m tmpfs /tmp/tmpfs
```
然后使用 `df -h` 命令查看一下已挂载的目录信息：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820212254.png)

可以看到 /tmp/tmpfs 目录已经被正确挂载。为了验证主机上并没有挂载此目录，我们新打开一个命令行窗口，同样执行 `df -h` 命令查看主机的挂载信息：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820212436.png)

通过上面输出可以看到主机上并没有挂载 /tmp/tmpfs，可见我们独立的 Mount Namespace 中执行 mount 操作并不会影响主机。

为了进一步验证我们的想法，我们继续在当前命令行窗口查看一下当前进程的 Namespace 信息，命令如下：

```sh
ls -l /proc/self/ns/
```
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820212542.png)

然后新打开一个命令行窗口，使用相同的命令查看一下主机上的 Namespace 信息：

```bash
ls -l /proc/self/ns/
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820212637.png)

通过对比两次命令的输出结果，我们可以看到，除了 Mount Namespace 的 ID 值不一样外，其他Namespace 的 ID 值均一致。

通过以上结果我们可以得出结论，使用 unshare 命令可以新建 Mount Namespace，并且在新建的 Mount Namespace 内 mount 是和外部完全隔离的。

### 隔离进程：Pid Namespace

PID Namespace 的作用是用来隔离进程。在不同的 PID Namespace 中，进程可以拥有相同的 PID 号，利用 PID Namespace 可以实现每个容器的主进程为 1 号进程，而容器内的进程在主机上却拥有不同的 PID。例如一个进程在主机上 PID 为 122，使用 PID Namespace 可以实现该进程在容器内看到的 PID 为 1。

下面我们通过一个实例来演示下 PID Namespace 的作用。首先我们使用以下命令创建一个 bash 进程，并且新建一个 PID Namespace：

```sh
unshare --pid --fork --mount-proc /bin/bash
```
执行完上述命令后，我们在主机上创建了一个新的 PID Namespace，并且当前命令行窗口加入了新创建的 PID Namespace。在当前的命令行窗口使用 `ps aux` 命令查看一下进程信息：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820213003.png)

通过上述命令输出结果可以看到当前 Namespace 下 bash 为 1 号进程，而且我们也看不到主机上的其他进程信息。 该进程在主机上实际的进程 ID 是 277404。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820213133.png)



### 隔离网络：Net Namespace

Net Namespace 是用来隔离网络设备、IP 地址和端口等信息的。Net Namespace 可以让每个进程拥有自己独立的 IP 地址，端口和网卡信息。



同样用实例验证，我们首先使用 `ip addr` 命令查看一下主机上的网络信息：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820214449.png)

然后我们使用以下命令创建一个 Net Namespace：

```sh
unshare --net --fork /bin/bash
```
同样的我们使用 `ip addr` 命令查看一下网络信息：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820214541.png)

可以看到，宿主机上有 lo、ens192 等网络设备，而我们新建的 Net Namespace 内只有 lo 环回口。我们可以在 Net Namespace 中新增网卡并配置 IP。

```sh
ip link add eth0 type dummy
ip addr add 100.100.100.1/24 dev eth0
```

查看新建的 Net Namespace 的网络。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820215955.png)



容器里怎么抓包？

```bash
# 查看容器网络命名空间，获取容器 pid
root@ebpf-demo:/tmp/overlay# lsns -t net
        NS TYPE NPROCS   PID USER    NETNSID NSFS                           COMMAND
4026531992 net     222     1 root unassigned                                /sbin/init auto automatic-ubiquity noprompt
4026532645 net       1  4063 root          0 /run/docker/netns/3655e4dc9159 /bin/bash
4026532714 net       1  4865 root          1 /run/docker/netns/060e886e440c sleep 3600

# 通过 nsenter 命令进入容器命名空间
root@ebpf-demo:/tmp/overlay# nsenter -n -t 4063 tcpdump -i any port 8080 -nnA
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked v1), capture size 262144 bytes
```



### 隔离主机名：UTS Namespace

UTS Namespace 主要是用来隔离主机名的，它允许每个 UTS Namespace 拥有一个独立的主机名。例如我们的主机名称为 docker，使用 UTS Namespace 可以实现在容器内的主机名称为 lagoudocker 或者其他任意自定义主机名。

同样我们通过一个实例来验证下 UTS Namespace 的作用，首先我们使用 unshare 命令来创建一个 UTS Namespace：

```sh
unshare --uts --fork /bin/bash
```
创建好 UTS Namespace 后，当前命令行窗口已经处于一个独立的 UTS Namespace 中，下面我们使用 hostname 命令设置一下主机名：

```sh
hostname -b lagoudocker
```

执行 `hostname` 命令查看更改后的主机名

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820213952.png)

通过上面命令的输出，我们可以看到当前UTS Namespace 内的主机名已经被修改为 lagoudocker。然后我们新打开一个命令行窗口，使用相同的命令查看一下主机的 hostname：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820214037.png)

可以看到主机的名称仍然为 ebpf-demo，并没有被修改。由此，可以验证 UTS Namespace 可以用来隔离主机名。

### 隔离用户：User Namespace

User Namespace 主要是用来隔离用户和用户组的。一个比较典型的应用场景就是在主机上以非 root 用户运行的进程可以在一个单独的 User Namespace 中映射成 root 用户。使用 User Namespace 可以实现进程在容器内拥有 root 权限，而在主机上却只是普通用户。User Namesapce 的创建是可以不使用 root 权限的。下面我们以普通用户的身份创建一个 User Namespace，命令如下：
```sh
useradd -s /bin/bash -m test123
su - test123
unshare --user -r /bin/bash
```
然后执行 `id` 命令查看一下当前的用户信息：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820221103.png)

通过上面的输出可以看到我们在新的 User Namespace 内已经是 root 用户了。下面我们使用只有主机 root 用户才可以执行的 reboot 命令来验证一下，在当前命令行窗口执行 `reboot` 命令，提示没有权限执行该命令。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820221122.png)执行 `echo $$` 命令查看当前 User Namespace 所在的进程 ID：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820221313.png)

在宿主机上查看该进程对应的用户，可以看到实际上在宿主机上是 chengzw 用户：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820221223.png)

可以看到，我们在新创建的 User Namespace 内虽然是 root 用户，但是并没有权限执行 reboot 命令。这说明在隔离的 User Namespace 中，并不能获取到主机的 root 权限，也就是说 User Namespace 实现了用户和用户组的隔离。



## 资源限制：Cgroups

cgroups（全称：control croups）是 Linux 内核的一个功能，它可以实现限制进程或者进程组的资源（如 CPU、内存、磁盘 IO 等）。

cgroups 主要提供了如下功能：

* 资源限制： 限制资源的使用量，例如我们可以通过限制某个业务的内存上限，从而保护主机其他业务的安全运行。
* 优先级控制：不同的组可以有不同的资源（ CPU 、磁盘 IO 等）使用优先级。
* 审计：计算控制组的资源使用情况。
* 控制：控制进程的挂起或恢复。

**cgroups功能的实现依赖于三个核心概念：子系统、控制组、层级树。**
* 子系统（subsystem）：是一个内核的组件，一个子系统代表一类资源调度控制器。例如内存子系统可以限制内存的使用量，CPU 子系统可以限制 CPU 的使用时间。
* 控制组（cgroup）：表示一组进程和一组带有参数的子系统的关联关系。例如，一个进程使用了 CPU 子系统来限制 CPU 的使用时间，则这个进程和 CPU 子系统的关联关系称为控制组。
* 层级树（hierarchy）：是由一系列的控制组按照树状结构排列组成的。这种排列方式可以使得控制组拥有父子关系，子控制组默认拥有父控制组的属性，也就是子控制组会继承于父控制组。比如，系统中定义了一个控制组 c1，限制了 CPU 可以使用 1 核，然后另外一个控制组 c2 想实现既限制 CPU 使用 1 核，同时限制内存使用 2G，那么 c2 就可以直接继承 c1，无须重复定义 CPU 限制。

### Cgroups 子系统实例

下面我通过一个实例演示一下在 Linux 上默认都启动了哪些子系统。我们先通过 `mount -t cgroup` 命令查看一下当前系统已经挂载的 cgroups 信息：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820222020.png)

通过输出，可以看到当前系统已经挂载了我们常用的 cgroups 子系统，例如 cpu、memory、pids 等我们常用的 cgroups 子系统。这些子系统中，cpu 和 memory 子系统是容器环境中使用最多的子系统，接下来对这两个子系统做详细介绍。

#### Cpu 子系统

首先我们在没有限制 CPU 使用率的情况下在后台执行这样一条脚本。

```sh
 while : ; do : ; done &
```
显然，它执行了一个死循环，可以把计算机的 CPU 吃到 100%，根据它的输出，我们可以看到这个脚本在后台运行的进程号（PID）是 2937。

我们可以用 top 指令来确认一下 CPU 有没有被打满：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820222611.png)



在输出里可以看到，CPU 的使用率已经接近 100% 了。



现在我们以 cpu 子系统为例，演示一下 cgroups 如何限制进程的 cpu 使用时间。由于 cgroups 的操作很多需要用到 root 权限，我们在执行命令前要确保已经切换到了 root 用户，以下命令的执行默认都是使用 root 用户。

**第一步：在 cpu 子系统下创建 cgroup**
cgroups 的创建很简单，只需要在相应的子系统下创建目录即可。下面我们到 cpu 子系统下创建测试文件夹：

```sh
mkdir /sys/fs/cgroup/cpu/mydocker
```
这个目录就称为一个“控制组”。你会发现，操作系统会在你新创建的目录下，自动生成该子系统对应的资源限制文件：
```sh
ls -l /sys/fs/cgroup/cpu/mydocker
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820222319.png)



你会在它的输出里注意到 cfs_period 和 cfs_quota 这样的关键词。这两个参数需要组合使用，可以用来限制进程在长度为 cfs_period 的一段时间内，只能被分配到总量为 cfs_quota 的 CPU 时间（**cfs_quota/cfs_period**）。
如果用这个值去除以调度周期（也就是 cpu.cfs_period_us），50ms/100ms = 0.5，这样这个控制组被允许使用的 CPU 最大配额就是 0.5 个 CPU。从这里能够看出，cpu.cfs_quota_us 是一个绝对值。如果这个值是 200000，也就是 200ms，那么它除以 period，也就是 200ms/100ms=2，结果超过了 1 个 CPU，这就意味着这时控制组需要 2 个 CPU 的资源配额。


而此时，我们可以通过查看 mydocker 目录下的文件，看到 mydocker 控制组里的 CPU quota 还没有任何限制（即：-1），**CPU period 则是默认的 100 ms（100000 us）**：

```sh
cat /sys/fs/cgroup/cpu/mydocker/cpu.cfs_quota_us
# 返回结果
-1
cat /sys/fs/cgroup/cpu/mydocker/cpu.cfs_period_us
# 返回结果
100000
```

**第二步：设置限制参数**

接下来，我们可以通过修改这些文件的内容来设置限制。向 mydocker 组里的 cfs_quota 文件写入 20 ms（20000 us）：

```sh
echo 20000 > /sys/fs/cgroup/cpu/mydocker/cpu.cfs_quota_us
```
它意味着在每 100 ms 的时间里，被该控制组限制的进程只能使用 20 ms 的 CPU 时间，也就是说这个进程只能使用到 20% 的 CPU 带宽。（如果 CPU 是多核，那么 20% 指的是这个进程在所有 CPU 中能使用的带宽总和，例如 CPU1 是 10%，CPU2 是 10%）


**第三步：将进程加入cgroup控制组**

接下来，我们把被限制的进程的 PID 写入 mydocker 组里的 tasks 文件，上面的设置就会对该进程生效了：

```sh
echo 2937 > /sys/fs/cgroup/cpu/mydocker/tasks
```
此时我们用 top 命令查看一下：
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820222802.png)

可以看到，计算机的 CPU 使用率立刻降到了 20%。

#### Memroy 子系统

**第一步：在 memory 子系统下创建 cgroup**

```sh
mkdir /sys/fs/cgroup/memory/mydocker
```
同样，我们查看一下新创建的目录下自动创建的文件：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820222900.png)

其中 memory.limit_in_bytes 文件代表内存使用总量，单位为 byte。

**第二步：设置限制参数**
例如，这里我希望对内存使用限制为 1G，则向 memory.limit_in_bytes 文件写入 1073741824，命令如下：

```sh
echo 1073741824 > /sys/fs/cgroup/memory/mydocker/memory.limit_in_bytes
```

**第三步：将进程加入 cgroup 控制组**
把当前 shell 进程 ID 写入 tasks 文件内：

```sh
echo $$ > /sys/fs/cgroup/memory/mydocker/tasks 
```

这里我们需要借助一下工具 memtester，执行以下命令安装 memtester（Ubuntu 系统）

```bash
apt install -y memtester
```

安装好 memtester 后，我们执行以下命令：

```sh
memtester 1500M 1
```
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820223059.png)



该命令会申请 1500 M 内存，并且做内存测试。由于上面我们对当前 shell 进程内存限制为 1 G，当 memtester 使用的内存达到 1G 时，cgroup 便将 memtester 杀死。

上面最后一行的输出结果表示 memtester 想要 1500 M 内存，但是由于 cgroup 限制，达到了内存使用上限，被杀死了，与我们的预期一致。

我们可以使用以下命令，降低一下内存申请，将内存申请调整为 500M：

```sh
memtester 500M 1
```
这里可以看到，此时 memtester 已经成功申请到 500M 内存并且正常完成了内存测试。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820223344.png)

#### 删除 Cgroups

上面创建的 cgroups 如果不想使用了，直接删除创建的文件夹即可。
例如我想删除内存下的 mydocker 目录，使用以下命令即可：

```sh
rmdir /sys/fs/cgroup/memory/mydocker/
```

### Docker 使用 Cgroups
#### 限制容器 Cpu

限制容器只能使用到 20% 的 CPU 带宽。
```sh
docker run -itd --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
```
在启动这个容器后，我们可以通过查看 Cgroups 文件系统下，CPU 子系统中，“docker” 这个控制组里的资源限制文件的内容来确认：

```sh
cat /sys/fs/cgroup/cpu/docker/99cdb60e28191871c456f736470c9a1d344b95c6bcdb39036926428378db55c4/cpu.cfs_period_us

# 返回结果
100000

cat /sys/fs/cgroup/cpu/docker/99cdb60e28191871c456f736470c9a1d344b95c6bcdb39036926428378db55c4/cpu.cfs_quota_us

# 返回结果
20000
```
很长的数字+字母为容器的 id，可以使用 `docker inspect <容器名>` 命令来查看容器的 id。

#### 限制容器 Memory

```sh
docker run -itd -m=1g ubuntu /bin/bash
```
上述命令创建并启动了一个 ubuntu 容器，并且限制内存为 1G。然后我们查看该容器对应的 cgroup 控制组 memory.limit_in_bytes 文件的内容，可以看到内存限制值正好为 1G：

```sh
cat /sys/fs/cgroup/memory/docker/2e5f7db4b9a08b08471b3fcf9f71cb14396fb081db3cdc25140714ae037f2136/memory.limit_in_bytes

# 返回结果
1073741824
```

## Union FS 联合文件系统

联合文件系统（Union File System，Unionfs）是一种分层的轻量级文件系统，它可以把多个目录内容联合挂载到同一目录下，从而形成一个单一的文件系统，这种特性可以让使用者像是使用一个目录一样使用联合文件系统。



overlay2 是目前 Docker 默认使用的联合文件系统。overlay2 把目录的下一层叫作 lowerdir，上一层叫作 upperdir，联合挂载后的结果叫作 merged。

### Docker overlay2 文件系统

下面我们通过拉取一个 Ubuntu 操作系统的镜像来看下 overlay2 是如何存放镜像文件的。

首先，我们通过以下命令拉取 Ubuntu 镜像：

```bash
docker pull ubuntu:16.04
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820224848.png)

可以看到镜像一共被分为四层拉取，拉取完镜像后我们查看一下 overlay2 的目录：

```bash
 ls -altr /var/lib/docker/overlay2/
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820225037.png)

可以看到 overlay2 目录下出现了四个镜像层目录和一个l目录，我们首先来查看一下 l 目录的内容：

```bash
ls -altr /var/lib/docker/overlay2/l
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820225204.png)

可以看到l目录是一堆软连接，把一些较短的随机串软连到镜像层的 diff 文件夹下，这样做是为了避免达到 mount 命令参数的长度限制。
下面我们查看任意一个镜像层下的文件内容：

```bash
ls -l /var/lib/docker/overlay2/
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820225400.png)

镜像层的 link 文件内容为该镜像层的短 ID，diff 文件夹为该镜像层的改动内容，lower 文件为该层的所有父层镜像的短 ID。

我们可以通过 `docker image inspect` 命令来查看某个镜像的层级关系，例如我想查看刚刚下载的 Ubuntu 镜像之间的层级关系，可以使用以下命令：

```bash
docker image inspect ubuntu:16.04
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820225738.png)

其中 MergedDir 代表当前镜像层在 overlay2 存储下的目录，LowerDir 代表当前镜像的父层关系，使用冒号分隔，冒号最后代表该镜像的最底层。



下面我们将镜像运行起来成为容器：

```bash
 docker run --name=ubuntu -d ubuntu:16.04 sleep 3600
```

我们使用 `docker inspect ubuntu` 命令来查看一下容器的工作目录：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820230008.png)

MergedDir 后面的内容即为容器层的工作目录，LowerDir 为容器所依赖的镜像层目录。 然后我们查看下 overlay2 目录下的内容：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820230139.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820230250.png)

link 和 lower 文件与镜像层的功能一致，link 文件内容为该容器层的短 ID，lower 文件为该层的所有父层镜像的短 ID 。**diff 目录为容器的读写层，容器内修改的文件都会在 diff 中出现，merged 目录为分层文件联合挂载后的结果，也是容器内的工作目录。**



目前还未对容器进行修改，diff 目录为空。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820230557.png)

尝试往容器中添加一个文件。

```bash
 docker exec -it ubuntu touch /tmp/testfile
```

再次查看 diff 目录，可以看到出现了新添加的 testfile 文件。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820230655.png)

### overlay2 读写过程
overlay2 的工作过程中对文件的操作分为读取文件和修改文件。

#### 读取文件

容器内进程读取文件分为以下三种情况。

- 文件在容器层中存在：当文件存在于容器层并且不存在于镜像层时，直接从容器层读取文件；

- 当文件在容器层中不存在：当容器中的进程需要读取某个文件时，如果容器层中不存在该文件，则从镜像层查找该文件，然后读取文件内容；

- 文件既存在于镜像层，又存在于容器层：当我们读取的文件既存在于镜像层，又存在于容器层时，将会从容器层读取该文件。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820231743.png)

#### 修改文件或目录

overlay2 对文件的修改采用的是写时复制的工作机制，这种工作机制可以最大程度节省存储空间。具体的文件操作机制如下：

- 第一次修改文件：当我们第一次在容器中修改某个文件时，overlay2 会触发写时复制操作，overlay2 首先从镜像层复制文件到容器层，然后在容器层执行对应的文件修改操作。
- overlay2 写时复制的操作将会复制整个文件，如果文件过大，将会大大降低文件系统的性能，因此当我们有大量文件需要被修改时，overlay2 可能会出现明显的延迟。好在，写时复制操作只在第一次修改文件时触发，对日常使用没有太大影响。

- 删除文件或目录：当文件或目录被删除时，overlay2 并不会真正从镜像中删除它，因为镜像层是只读的，overlay2 会创建一个特殊的文件或目录，这种特殊的文件或目录会阻止容器的访问。



### 模拟 Docker Overlayfs 

创建相关目录和文件。

```bash
mkdir /tmp/overlay
cd /tmp/overlay

mkdir lower1
mkdir lower2
mkdir upper
mkdir worker
mkdir merged
touch lower1/lower1.txt
touch lower2/lower2.txt
touch upper/upper.txt
```

- lower1、lower2、upper 三个文件夹，这个三个文件夹是用来合并的。
- 一个空的 worker 文件夹，这文件夹不能有任何内容。
- 最后需要一个 merged 文件夹，用来作为给用户呈现的最终文件夹。
- worker 文件夹是一个空文件夹，它是作为中间临时文件夹而存在，需要和 upperdir 是同一个文件系统。

它最终展示的效果应该是这样:

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820232330.png)

设置 overlay 文件系统。

```bash
mount -t overlay overlay -olowerdir=./lower1:./lower2,upperdir=./upper,workdir=./worker ./merged
```

通过 ls 命令看一下 merge 目录，可以看到 merge 目录中的文件是 lower 和 upper 层合并的结果。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820232438.png)



接下来我们可以尝试一下增加删除修改。

**增加文件**

```shell
# touch ./merged/merged.txt
# ls ./merged/
lower1.txt  lower2.txt  merged.txt  upper.txt
# ls ./upper/
merged.txt  upper.txt
```

可以看到，增加文件是直接添加到了 upper 文件夹中。



**删除文件**

```shell
# rm -f ./merged/lower1.txt
# ls ./merged/
lower2.txt  merged.txt  upper.txt
# ls ./lower1/
lower1.txt
```

可以看到，删除文件之后，下面的 lower 文件夹里的内容并没有被真正的删除，但是上面的 merged 文件夹表现是确实已经被删除了。

我们可以直接去查看  upper 文件夹下面的内容：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20220820232734.png)

多了一个 c 类型的文件，名字刚好为 lower1.txt，它表示此文件已经被删除了，会将下层的该文件直接忽略。



**修改文件**

```shell
# echo 'hello' > ./merged/lower2.txt
# cat ./merged/lower2.txt
hello
# ls -al ./upper/
总用量 12
drwxr-xr-x 2 root root 4096 9月  26 19:13 .
drwxr-xr-x 7 root root 4096 9月  26 19:16 ..
c--------- 1 root root 0, 0 9月  26 19:13 lower1.txt
-rw-r--r-- 1 root root    6 9月  26 19:17 lower2.txt
-rw-r--r-- 1 root root    0 9月  26 19:10 merged.txt
-rw-r--r-- 1 root root    0 9月  26 18:56 upper.txt
# cat ./lower2/lower2.txt
```

当文件修改后，我们查看上层的 upper 文件夹，会发现，多了一个 lower2.txt。而底层的 lower2.txt 仍然没有内容变化。依此可以推断出，其实修改文件是将下层的文件复制到上层来之后再进行修改的。



## 参考资料

- [overlayfs 文件系统](https://www.zido.site/blog/2021-09-26-overlayfs-filesystem/)
- [由浅入深吃透 Docker](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=455#/detail/pc?id=4587)