translating by ezio

Part 1 - LXD 2.0: LXD 入门
======================================

这是 [LXD 2.0 系列介绍文章][1]的第一篇。

![](https://linuxcontainers.org/static/img/containers.png)


### 关于 LXD 几个常见问题

#### 什么是 LXD ?

At its simplest, LXD is a daemon which provides a REST API to drive LXC containers.

Its main goal is to provide a user experience that’s similar to that of virtual machines but using Linux containers rather than hardware virtualization.

简单来说 LXD 就是一个提供了 REST API 的 LXC 容器管理器。

LXD 最主要的目标就是使用 Linux 容器而不是硬件虚拟化向用户提供一种接近虚拟机的使用体验。

#### LXD 和 Docker/Rkt 又有什么关系呢 ?

这是一个最常被问起的问题，现在就让我们直接指出其中的不同吧。

LXD 聚焦于系统容器，通常也被称为架构容器。这就是说 LXD 容器实际上运行了一个完整的 Linux 操作系统，就像在裸机或虚拟机上运行的一样。

这些容器一般都是基于一个干净的发布镜像，会长时间运行。传统的配置管理工具和发布工具可以和 LXD 一起使用，就像你在虚拟机、云或者物理机器上使用一样。

相对的， Docker 关注于短视的、无状态的最小容器，这些容器通常并不会升级或者重新配置，它们一般都是作为一个整体被替换掉。这就使得 Docker 和类似的项目更像是一种软件发布机制，而不是一个机器管理工具。

这两种模型并不是完全互斥的。你完全可以使用 LXD 为你的用户提供一个完整的 Linux 系统，而他们可以在 LXD 上面安装 Docker 来运行他们想要的软件。


#### 为什么要用 LXD?

我们已经在 LXC 上工作了好几个年头了。 LXC 成功的实现了它的目标，它提供了一系列很棒的底层工具和库来创建、管理容器。

然而这些底层工具的使用界面对用户并不是很友好。他们需要用户有很多的基础知识来理解他们要干什么和怎么去干。同时向后兼容旧的容器和部署策略已经阻止 LXC 默认使用一些安全特性，这导致用户需要进行更多人工操作来实现本可以自动完成的工作。

我们把 LXD 作为解决这些缺陷的一个很好的机会。作为一个长时间运行的守护进程可以让我们定位到很多 LXC 的限制，比如动态资源限制，容器迁移和有效的进行在线迁移，它同时也给了我们机会来提供一个新的默认体验：默认开启安全特性，对用户更加友好。


### LXD 的主要组件

There are a number of main components that make LXD, those are typically visible in the LXD directory structure, in its command line client and in the API structure itself.

#### 容器

Containers in LXD are made of:

- A filesystem (rootfs)
- A list of configuration options, including resource limits, environment, security options and more
- A bunch of devices like disks, character/block unix devices and network interfaces
- A set of profiles the container inherits configuration from (see below)
- Some properties (container architecture, ephemeral or persistent and the name)
- Some runtime state (when using CRIU for checkpoint/restore)

#### 快照

Container snapshots are identical to containers except for the fact that they are immutable, they can be renamed, destroyed or restored but cannot be modified in any way.

It is worth noting that because we allow storing the container runtime state, this effectively gives us the concept of “stateful” snapshots. That is, the ability to rollback the container including its cpu and memory state at the time of the snapshot.

#### 镜像

LXD is image based, all LXD containers come from an image. Images are typically clean Linux distribution images similar to what you would use for a virtual machine or cloud instance.

It is possible to “publish” a container, making an image from it which can then be used by the local or remote LXD hosts.

Images are uniquely identified by their sha256 hash and can be referenced by using their full or partial hash. Because typing long hashes isn’t particularly user friendly, images can also have any number of properties applied to them, allowing for an easy search through the image store. Aliases can also be set as a one to one mapping between a unique user friendly string and an image hash.

LXD comes pre-configured with three remote image servers (see remotes below):

- “ubuntu:” provides stable Ubuntu images
- “ubunt-daily:” provides daily builds of Ubuntu
- “images:” is a community run image server providing images for a number of other Linux distributions using the upstream LXC templates
Remote images are automatically cached by the LXD daemon and kept for a number of days (10 by default) since they were last used before getting expired.

Additionally LXD also automatically updates remote images (unless told otherwise) so that the freshest version of the image is always available locally.

#### 配置

Profiles are a way to define container configuration and container devices in one place and then have it apply to any number of containers.

A container can have multiple profiles applied to it. When building the final container configuration (known as expanded configuration), the profiles will be applied in the order they were defined in, overriding each other when the same configuration key or device is found. Then the local container configuration is applied on top of that, overriding anything that came from a profile.

LXD ships with two pre-configured profiles:

- “default” is automatically applied to all containers unless an alternative list of profiles is provided by the user. This profile currently does just one thing, define a “eth0” network device for the container.
- “docker” is a profile you can apply to a container which you want to allow to run Docker containers. It requests LXD load some required kernel modules, turns on container nesting and sets up a few device entries.

#### 远程

As I mentioned earlier, LXD is a networked daemon. The command line client that comes with it can therefore talk to multiple remote LXD servers as well as image servers.

By default, our command line client comes with the following remotes defined

local: (default remote, talks to the local LXD daemon over a unix socket)
ubuntu: (Ubuntu image server providing stable builds)
ubuntu-daily: (Ubuntu image server providing daily builds)
images: (images.linuxcontainers.org image server)
Any combination of those remotes can be used with the command line client.

You can also add any number of remote LXD hosts that were configured to listen to the network. Either anonymously if they are a public image server or after going through authentication when managing remote containers.

It’s that remote mechanism that makes it possible to interact with remote image servers as well as copy or move containers between hosts.

### 安全性

One aspect that was core to our design of LXD was to make it as safe as possible while allowing modern Linux distributions to run inside it unmodified.

The main security features used by LXD through its use of the LXC library are:

- Kernel namespaces. Especially the user namespace as a way to keep everything the container does separate from the rest of the system. LXD uses the user namespace by default (contrary to LXC) and allows for the user to turn it off on a per-container basis (marking the container “privileged”) when absolutely needed.
- Seccomp. To filter some potentially dangerous system calls.
- AppArmor: To provide additional restrictions on mounts, socket, ptrace and file access. Specifically restricting cross-container communication.
- Capabilities. To prevent the container from loading kernel modules, altering the host system time, …
- CGroups. To restrict resource usage and prevent DoS attacks against the host.
Rather than exposing those features directly to the user as LXC would, we’ve built a new configuration language which abstracts most of those into something that’s more user friendly. For example, one can tell LXD to pass any host device into the container without having to also lookup its major/minor numbers to manually update the cgroup policy.

Communications with LXD itself are secured using TLS 1.2 with a very limited set of allowed ciphers. When dealing with hosts outside of the system certificate authority, LXD will prompt the user to validate the remote fingerprint (SSH style), then cache the certificate for future use.

### REST 接口

Everything that LXD does is done over its REST API. There is no other communication channel between the client and the daemon.

The REST API can be access over a local unix socket, only requiring group membership for authentication or over a HTTPs socket using a client certificate for authentication.

The structure of the REST API matches the different components described above and is meant to be very simple and intuitive to use.

When a more complex communication mechanism is required, LXD will negotiate websockets and use those for the rest of the communication. This is used for interactive console session, container migration and for event notification.

With LXD 2.0, comes the /1.0 stable API. We will not break backward compatibility within the /1.0 API endpoint however we may add extra features to it, which we’ll signal by declaring additional API extensions that the client can look for.

### 容器规模化

While LXD provides a good command line client, that client isn’t meant to manage thousands of containers on multiple hosts. For that kind of use cases, we have nova-lxd which is an OpenStack plugin that makes OpenStack treat LXD containers in the exact same way it would treat VMs.

This allows for very large deployments of LXDs on a large number of hosts, using the OpenStack APIs to manage network, storage and load-balancing.

### 额外信息

The main LXD website is at: <https://linuxcontainers.org/lxd>
Development happens on Github at: <https://github.com/lxc/lxd>
Mailing-list support happens on: <https://lists.linuxcontainers.org>
IRC support happens in: #lxcontainers on irc.freenode.net

And if you can’t wait until the next few posts to try LXD, you can [take our guided tour online][2] and try it for free right from your web browser!

--------------------------------------------------------------------------------

via: https://www.stgraber.org/2016/03/11/lxd-2-0-introduction-to-lxd-112/

作者：[Stéphane Graber][a]
译者：[译者ID](https://github.com/译者ID)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创翻译，[Linux中国](https://linux.cn/) 荣誉推出

[a]: https://www.stgraber.org/author/stgraber/
[1]: https://www.stgraber.org/2016/03/11/lxd-2-0-blog-post-series-012/
[2]: https://linuxcontainers.org/lxd/try-it