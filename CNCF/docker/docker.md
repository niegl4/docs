[TOC]

# Docker项目

## 作用

Docker解决了在不同环境中应用难以迁移的问题。它是一种便于做持续集成并有助于整体发布的容器虚拟化技术。容器虚拟化与虚拟机不同，它不需要模拟一整套操作系统，而是对进程进行隔离和限制，将所需要的所有资源打包到一个隔离的容器中。系统因而变得高效轻量。

它发明的容器镜像，不仅打通了”开发-测试-部署“流程的每一个环节，更重要的是：容器镜像将会成为未来软件的主流发布方式。

## 基本组成

### 镜像

**应用程序**与**完整的操作系统文件和目录**，打包形成一个可交付的运行环境，即是镜像文件。镜像文件是容器的模板，同一个镜像文件可以运行为多个容器实例。

### 容器

容器是一个“单进程”模型。

### 镜像仓库

镜像仓库用来存放镜像。可以把镜像推送到仓库，也可以从仓库中拉取。



# 容器原理

**容器，其实是一个“单进程”模型，并不是说容器里只能运行一个进程，而是容器没有管理多个进程的能力。用户的应用进程实际上就是容器里的PID为1的进程，也是其他后续创建的所有进程的父进程。用户编写的应用，不能像操作系统里的systemd那样拥有进程管理的功能。**所以，在一个容器中，不能同时运行两个不同的应用，除非能事先找到一个公共的程序来充当两个不同应用的父进程。

容器，实际上是一个由Linux Namespace，Linux Cgroups和rootfs三种技术构建出来的进程的隔离环境。一个正在运行的容器，可以一分为二地看待。由Namespace+Cgroups构成隔离+限制的环境，被称为“容器运行时”，是容器的动态视图。rootfs，被称为“容器镜像”，是容器的静态视图。

## 限制

使用Cgroups技术制造约束。

Linux Cgroups，即Linux Control Groups。它主要的作用，就是限制一个进程能够使用的资源上限，包括CPU，内存，磁盘，网络带宽等等。

在Linux中，Cgroups给用户暴露出来的操作接口是文件系统，它以文件和目录的方式组织在操作系统/sys/fs/cgroup路径下。对于Docker等Linux容器项目来说，它们只需要在各种资源的子系统下面，为每个容器创建一个控制组（即创建一个新目录），然后在启动容器进程之后，把这个进程的PID填写到对应控制组的tasks文件中就可以了。

在**/sys/fs/cgroup**下，定义了许多子系统：

- cpu：为进程设定cpu使用的限制。
- cpuacct：可以统计cgroups中的进程的cpu使用报告。
- cpuset：为进程分配单独的CPU核和对应的内存节点。
- memory：为进程设定内存使用的限制。
- blkio：为块设备设定I/O限制，一般用于磁盘等设备。
- devices：控制进程能够访问某些设备。
- net_cls：可以标记cgroups中进程的网络数据包，然后可以用tc模块对数据包进行控制。
- fleezer：可以挂起或者恢复cgroups中的进程。

## 隔离

使用Namespace技术修改进程看待整个计算机的视图。

Linux的Namespace机制，用来对不同的进程上下文进行隔离操作。Docker实际在创建容器进程时，指定了这个进程所要启用的一组Namespace参数。

从容器视角，只能看到当前Namespace所限定的资源，文件，设备，状态，配置，而对宿主机以及其他不相关的进程，它就完全看不到了。

从宿主机视角，这些被不同Namespace隔离了的进程跟其他进程没有太大区别。

在**/proc/$pid/ns**下，可以看到指定进程的ns：

- PID：进程都会认为自己是1号进程，它们既看不到宿主机里面真正的进程空间，也看不到其他PID Namespace里的进程号。
- Mount：被隔离进程只看到当前Namespace里的挂载点信息。它与其他ns不同的是：它对容器进程视图的改变，一定伴随着挂载操作（mount）才能生效。
- UTS：被隔离进程可以配置不同的hostname。
- IPC
- Network：被隔离进程只看到当前Namespace里的网络设备和配置。
- User：被隔离进程可以配置不同的用户和组。

## 文件系统

使用UnionFS实现rootfs。

为了能够让容器的根目录看起来更加真实，一般会在容器的根目录下挂载一个完整操作系统的文件系统。

**挂载在容器根目录上，用来为容器进程提供隔离后执行环境的文件系统，就是容器镜像，也即rootfs，根文件系统。**

rootfs只是一个操作系统所包含的文件和目录，并不包含操作系统内核。在Linux中，这两部分是分开存放的。

正是由于rootfs的存在，容器才有了一致性这个重要特性。由于rootfs里打包的不只是应用，而是整个操作系统的文件和目录，这就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起。对于一个应用来说，操作系统本身才是它运行所需要的最完整的“依赖库”。这种深入到操作系统级别的运行环境一致性，打通了应用的本地开发环境和远端运行环境。

系统调用：**chroot，用来改变进程的根目录到你指定的位置**。

## volume卷

允许将宿主机上指定的目录或文件，挂载到容器里面进行读取和修改操作。

当容器进程被创建之后，尽管开启了Mount Namespace，但在它执行chroot之前，容器进程一直可以看到宿主机上的整个文件系统。需要在rootfs准备好之后，在执行chroot之前，把volume指定的宿主机目录，挂载在指定的容器目录在宿主机上对应的目录上，整个volume的挂载工作就完成了。

**这其实是一个替换dentry指向的inode的过程。**

由于执行这个挂载操作时，容器进程已经创建，也就意味着此时Mount Namespace已经开启。所以，volume的挂载事件只在这个容器内可见，宿主机并不知道这个绑定挂载的存在，这正是Mount Namespace的隔离作用。

举例，把宿主机/home目录，挂载到容器的/test目录下。在容器里，在/test目录里新建文件。在宿主机上的/home目录里，就会发现新建的文件，因为这两个目录的dentry指向同一个inode。但是，在宿主机里，查看容器的联合文件系统，会找不到该文件，因为这是从宿主机的角度而不是容器的mount ns角度来查看该挂载点。

## 与虚拟机对比

### 虚拟机优缺点

优点：

- 虚拟机拥有硬件虚拟化技术和独立的OS，宿主机和客户机可以跨os，跨版本。比如，微软的云平台Azure运行在Windows服务器集群，但可以方便创建各种Linux虚拟机。
- 虚拟机里面自由度就比较大。

缺点：

- 虚拟机需要运行一个完整的OS才能执行用户的应用进程。对于宿主机，这就不可避免地带来了额外的资源消耗和占用。一个运行着CentOS的KVM虚拟机启动后，虚拟机自己就需要占用100～200MB内存。
- 用户的应用运行在虚拟机里面，它对宿主机的系统调用就不可避免地要经过虚拟化软件的拦截和处理，尤其是计算资源，网络和磁盘IO的损耗比较大。

### 容器优缺点

优点：**敏捷**和**高性能**是容器相较于虚拟机最大的优势。

- 容器不需要单独的OS，这就使得容器额外的资源占用几乎可以忽略不计。
- 容器化后的应用，还是一个宿主机上的普通进程，由于虚拟化而带来的性能损耗都是不存在的。

缺点：**容器隔离得不彻底**是容器相较于虚拟机的不足之处。

-  多个容器使用的还是同一个宿主机的操作系统内核。意味着，在Windows宿主机上运行Linux容器，或者低版本Linux宿主机上运行高版本Linux容器，都行不通。
- 很多资源和对象是不能namespace化的。比如，时间。容器中的应用程序，使用系统调用修改了时间，整个宿主机的时间都会被随之改变。



# 镜像原理

Docker在镜像的设计中，引入了层（layer）的概念。用户制作镜像的每一步操作，都会生成一个层，也就是一个增量rootfs。这种增量的设计，用到了联合文件系统（Union File System）的能力。

挂载在容器根目录上，用来为容器进程提供隔离后执行环境的文件系统，就是容器镜像，也即rootfs，根文件系统。

## UnionFS

UnionFS，最主要的功能是将多个不同位置的目录联合挂载（union mount）到同一个目录下。它是docker镜像的基础。

## docker镜像分层

### 特点

- 镜像可以通过分层来进行继承，基于基础镜像，可以制作各种具体配置的应用镜像。
- 共享只读层和可读写层，只提交可读写层，init层用于存放/etc/hosts等信息，它只对当前容器有效，commit不会提交init层。

### 优点

- 共享资源

  多个镜像都是从相同的base镜像构建而来，一份base镜像就可以为所有容器服务。然后，各个可读写层也是可以共享的。

- 敏捷高效

  共享层的存在，可以使得所有这些容器镜像需要的总空间，比每个镜像的总和要小。

### aufs

在Ubuntu中，UnionFS默认使用aufs。

<img src="https://github.com/NieGuanglin/docs/blob/main/pics/CNCF/docker/%E9%95%9C%E5%83%8F%E5%88%86%E5%B1%82.png">

<img src="/Users/nieguanglin/docs/pics/CNCF/docker/镜像分层.png" alt="镜像分层.png" style="zoom:100%;" />

- 只读层，ro+wh，readonly+whiteout：

  这些层，都以增量的方式，分别包含了操作系统的一部分。

- init层：

  专门用来存放/etc/hosts，/etc/resolv.conf等信息。这些文件本来属于只读层，但是用户往往需要写入一些指定的值。这些改动往往只对当前容器有效，所以不希望把它们连同可读写层一起提交。所以docker以一个单独的层挂载了出来。

- 可读写层，rw：

  一旦在容器里做了写操作，修改产生的内容就会以增量的方式出现在这个层中。当要删除只读层的内容时，会在可读写层生成一个特殊的文件，用来把只读层的文件遮挡起来，这就是**whiteout**的含义。当要修改只读层的内容时，会先复制到可读写层，然后再修改，就是**copy-on-write**的含义。

  当docker commit和docker push时，原先的只读层里的内容不会改变，保存的是这个被修改过的可读写层。

### overlay2

在Cent0S7中，UnionFS默认使用overlay2。

```bash
$ docker inspect $containerID
...
"GraphDriver": {
	"Data": {
		"LowerDir": "$DockerRootDir/overlay2/xxxa-init/diff:$DockerRootDir/overlay2/xxxb/diff:\
$DockerRootDir/overlay2/xxxc/diff:$DockerRootDir/overlay2/xxxd/diff",
		"MergedDir": "/$DockerRootDir/overlay2/xxxa/merged",
		"UpperDir": "/$DockerRootDir/overlay2/xxxa/diff"，
		"WorkDir": "/$DockerRootDir/overlay2/xxxa/work"
	},
	"Name": "overlay2"
}
...
$ ls /$DockerRootDir/overlay2/xxxa/merged
app	bin	dev	etc	go home lib media mnt opt proc root run sbin srv sys tmp usr var
```

## Dockerfile

Dockerfile使用一些标准的原语，描述所要构建的Docker镜像，并且这些原语都是按顺序处理的。

```dockerfile
# Building stage：使用官方提供的开发镜像作为“编译”基础镜像
FROM registry/golang:1.11-alpine3.9 as builder

# 设置并切换容器镜像内的工作目录，如果该目录不存在则新建
WORKDIR /go/src/github.com/project/ops

# Source code, building tools and dependences
COPY . /go/src/github.com/project/ops

ENV CGO_ENABLED 0
ENV GOOS linux
ENV GO111MODULE=on
ENV TIMEZONE "Asia/Shanghai"

RUN go build -mod=vendor -o ops cmd/main.go
RUN mv ops /go/bin

# Production stage：使用官方提供的简易操作系统镜像作为“运行”基础镜像
FROM registry/alpine:3.9

WORKDIR /go/bin

# copy the go binaries from the building stage
COPY --from=builder /go/bin /go/bin

# copy the config files from the current working dir
# COPY etc/ops.yaml /etc/ops/ops.yaml

# 允许外界访问容器的端口。
# 注意：
# 1.如果不开启network namespace，则此设置不生效，app实际监听的端口生效。
# 2.如果开启network namespace，则此设置必须与app实际监听的端口一致，否则访问不了。  
EXPOSE 9096

# 完整的执行格式是：“ENTRYPOINT CMD”，也即CMD就是ENTRYPOINT的参数。
ENTRYPOINT ["./ops"]
```

```bash
$ docker build -t ops:v0.3.0 -f docker/Dockerfile .
```

这个过程，实际上可以等同于Docker使用基础镜像启动了一个容器，然后在容器中依次执行Dockerfile中的原语。每个原语执行后，都会生成一个对应的镜像层。

















