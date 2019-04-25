**本文图片和部分代码来源互联网**

# Introduction
  ### container vs virtual machine
  ![](https://github.com/bigtree21cn/doc/blob/master/docker/docker.vm.vs.container.png)
  虚拟机运行在虚拟化层之上，虚拟化层对硬件资源进行抽象和共享。 每个虚拟机有自己独立的虚拟设备，在此之上可以创建和运行自己的操作系统。所有虚拟机是在整个操作系统层面进行隔离。  容器是利用Linux内核的隔离机制来创建和运行容器的。同一个主机上的容器实例是共享同一个操作系统内核，通过内核的namspace来创建运行环境隔离，通过内核cgroup来进行资源的隔离。
  
  虚拟机有更好的隔离性，但因为他虚拟了整个操作系统环境，所有对资源占用高，启动的速度慢。并且因为不同的硬件抽象实现不同，导致不同虚拟化系统之间的兼容性较差。 而容器因为是共享同一个操作系统内核，所有相对虚拟机隔离性要弱，但占用系统资源少，启动速度快。可以说容器是非常轻量化的虚拟技术。容器适合在对应用程序级别的应用。一个容器实例包含一个单独的应用程序。
  
  ### lxc vs rkt vs docker
  **LXC**是一种更像虚拟机的容器。一般一个LXC容器进程可以加载一个完整的操作系统，可以是一个fedreo, ubuntu或centos实例。 每一个操作系统独有自己的虚拟内核，主机系统更像是虚拟化层。但相对虚拟机，因为不需要格外的硬件指令转化，LXC实例有更好的性能。 
  
  **Docker**是一个linux容器的生命周期管理工具，包括容器的创建、运行、销毁。同时也有自己的镜像系统。现在docker也组建包含了更多容器生命周期之外的工具集合，比如容器的集群管理，编排管理等等。
  ![](https://github.com/bigtree21cn/doc/blob/master/docker/docker.lxc.vs.docker.png)
  **rkt** linux容器管理的另一个实现，它是遵守appc标准。相对docker更加轻量化。有coreos维护。
  ![](https://github.com/bigtree21cn/doc/blob/master/docker/docker.rkt.vs.docker.png)
  
# Container Isolation
Linux容器技术是建立的操作系统内核namespace和cgroup技术之上。通过这两种技术，实现了容器进程的隔离。 namespace主要提供了对系统运行时环境的隔离，比如进程属、ipc、 网络和文件系统等资源的隔离。 cgroup提供了物理资源，如内存、cpu、io的资源分配和限制。在进程创建(clone)的时候，通过制定其namespace信息，实现了资源的隔离，这就是容器隔离的本质。
  ### namespace
  在最先的Linux操作系统版本中，所有的资源是进行全局管理的。每个进程度拥有的资源，也可以在其他进进程中看到。就像一个学校，每个同学可能知道其他同学的情况，可以影响其他同学。 这样的实现对资源的隔离性，安全性都不能很好的保证。一个危险的进程，有可能破坏系统中的其他进程。namespace是一种轻量级的虚拟机制，是的每个进程，每种资源可以在不同的namespace中。 资源至对相同namespace中的进程可见，这样提供了一种安全的隔离机制。就像一个学校每个班级一样，班级同学之间可以互相看到和沟通，但不能影响其他的班级同学。 
  
  Linux中namespace一共对以下六种资源提供了namespace的支持，基本上涵盖了一个小型操作系统的运行要素，包括主机名、用户权限、文件系统、网络、进程号、进程间通信。
  ![](https://github.com/bigtree21cn/doc/blob/master/docker/docker.namespace.png)
  
  以下代码通过在系统调用`clone`中，传入资源类型，进程创建的时候可以分配不同的namespace.
  ```c
#define STACK_SIZE (1024 * 1024)

static char container_stack[STACK_SIZE];
char* const container_args[] = {
   "/bin/bash",
   NULL
};

// 容器进程运行的程序主函数
int container_main(void *args)
{
   printf("在容器进程中！\n");
   sethostname("container", 9);
   execv(container_args[0], container_args);   // 执行/bin/bash   return 1;
}

int main(int args, char *argv[])
{
   printf("程序开始\n");
   // clone 容器进程
   // int container_pid = clone(container_main, container_stack + STACK_SIZE, SIGCHLD, NULL);
   int container_pid = clone(container_main, container_stack + STACK_SIZE, SIGCHLD | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID, NULL);
   // 等待容器进程结束
   waitpid(container_pid, NULL, 0);
   return 0;
}
  ```
  
下面是如何查看进程使用了那些namespace
![](https://github.com/bigtree21cn/doc/blob/master/docker/docker.namespace.list.png)

有上面的命令可以看出，一个进程的每种资源都有各自的namespace, 而多个进程可以共享一个namespace。这提供了容器创建的灵活性，比如多个容器实例共用一个network namespace, 这样这些容器共享网络协议中, 有相同的ip，路由规则，端口空间，内部互相访问而不需要特别的网络配置。 在容器之中，也可以调用clone并且传入不同的namespace, 暨在上面`container_main`函数内再执行系统调用`clone`。这样就实现了容器的嵌套，运行在容器中的容器。
![](https://github.com/bigtree21cn/doc/blob/master/docker/docker.namespace.tree.png)

附内核task_struct结构体关于nsproxy以及nsproxy的定义
```c
struct task_struct {
    ......
    /* namespaces */
    struct nsproxy *nsproxy;
    ......
};

struct nsproxy {
         atomic_t count;
         struct uts_namespace *uts_ns;
         struct ipc_namespace *ipc_ns;
         struct mnt_namespace *mnt_ns;
         struct pid_namespace *pid_ns_for_children;
         struct net             *net_ns;
};
```


  ### cgroup
  namespace创建了一个隔离的运行环境，对操作系统资源进行了隔离。而cgroup是对物理的计算计算资源进行隔离，用户的可以限定某个进程或进程组占用的cpu, mem和磁盘io的资源使用。以下列出了cgroup常用的一些场景：
  - 隔离进程组，并限制它们使用的cpu资源。比如限制使用cpu的核心
  - 为进程组分配内存资源
  - 为进程组分配网络带宽和磁盘使用量
  - 控制进程组对设备的访问，比如通过黑白名单
  - 上面的组合
  
  cgroup由不同的子系统来控制各种资源的使用，每种资源有一个控制器负责具体的控制任务。cgroup提供了一个cgroup虚拟文件系统来来提供对各个子系统的访问，要使用cgroup就必须挂载cgroup虚拟文件系统。 cgoup包括以下子系统:
  - blkio — 这个子系统为块设备设定输入/输出限制，比如物理设备（磁盘，固态硬盘，USB 等等）。
  - cpu — 这个子系统使用调度程序提供对 CPU 的 cgroup 任务访问。
  - cpuacct — 这个子系统自动生成 cgroup 中任务所使用的 CPU 报告。
  - cpuset — 这个子系统为 cgroup 中的任务分配独立 CPU（在多核系统）和内存节点。
  - devices — 这个子系统可允许或者拒绝 cgroup 中的任务访问设备。
  - freezer — 这个子系统挂起或者恢复 cgroup 中的任务。
  - memory — 这个子系统设定 cgroup 中任务使用的内存限制，并自动生成??内存资源使用报告。
  - net_cls — 这个子系统使用等级识别符（classid）标记网络数据包，可允许 Linux 流量控制程序（tc）识别从具体 cgroup 中生成的数据包。
  - net_prio — 这个子系统用来设计网络流量的优先级
  - hugetlb — 这个子系统主要针对于HugeTLB系统进行限制，这是一个大页文件系统。
  下面代码演示如何通过systemd创建一个服务，并且限制改服务的cpu和内存使用。
  ```bash
#服务脚本: myservice.sh------------------------
#! /bin/bash                                                                       
cpu_loop()                                                                         
{                                                                                  
        while true                                                                 
        do                                                                         
                i=i+100;                                                           
                i=100                                                              
        done                                                                       
}                                                                                  
mem_loop()                                                                         
{                                                                                  
        rm -rf /tmp/memory                                                         
        mkdir /tmp/memory                                                          
        mount -t tmpfs -o size=512M tmpfs /tmp/memory                              
        dd if=/dev/zero of=/tmp/memory/block                                       
        sleep 10                                                                   
        rm /tmp/memory/block                                                       
        umount /tmp/memory                                                         
        rmdir /tmp/memory                                                          
}                                                                                  
cpu_loop &                                                                         
pid1=$!                                                                            
mem_loop &                                                                         
pid2=$!   
wait $pid1
wait $pid2

#systemd  /etc/systemd/system/myservice.service unit文件--------------
[Unit]                                               
Description=cgroup test demo service                 
After=network.target                                 
                                                     
[Service]                                            
Type=simple                                          
Restart=always                                       
RestartSec=1                                         
User=root                                            
ExecStart=/bin/bash /home/jefli/myservice.sh      
CPUQuota=10%                                         
MemoryLimit=500M                                     
                                                     
[Install]                                            
WantedBy=multi-user.target                           

  ```

# Docker Archicture

# Docker Image

# Docker Networking
**Linux network namespace**
前面我们讲到容器是通过namespace来进行隔离的。 每个linux network namespace可以有自己独立的网络接口，协议栈，ip，iptables, socket等网络资源。而容器首先是先创建进程，然后把进程分配给不同的network namespace。 因此每个容器都可以有自己独立的ip地址，接口，路由等。 如果两个容器创建的时候被分配在同一个网络空间中，则这两个容器共享同一个网络协议栈，它们有享同的ip。容器之间的网络访问没有障碍。同样的道理，如果容器和主机被分配在同一个网络空间中，则容器和主机共享ip。 理解了network namespace，就能自然的理解每个容器的网络是怎么工作的。  

下面的代码演示创建三个不同的network namespace，每个network namespace被赋予了不同的网络接口和ip地址。通过Linux veth pair技术和bridge技术，来创建容器之间网络的通信。
```bash
#创建三个不同的network namespace
ip netns add net1
ip netns add net2
ip netns add net3

#创建三个veth pair，并且把其中的一个接口分配给三个不同的network ns; 
ip link add type veth
ip link set dev  veth0 netns net1
ip link add type veth
ip link set dev veth0 netns net2
ip link add type veth
ip link set dev veth0 netns net3

#给每个接口分配ip地址，主要每个"容器"就有了自己的接口和ip
ip netns exec net1 ip link set veth0 up 
ip netns exec net1 ip addr add 10.0.1.1/24 veth0
ip netns exec net1 ip addr add 10.0.1.1/24 dev veth0
ip netns exec net2 ip link set veth0 up 
ip netns exec net2 ip addr add 10.0.1.2/24 dev veth0
ip netns exec net3 ip link set veth0 up 
ip netns exec net3 ip addr add 10.0.1.3/24 dev veth0

#创建Linux bridge, 把之前创建的veth pair的另一个接口都连接到该bridge上
#这样三个"容器” 之间就可以通过bridge进行二层交换
ip link add br0 type bridge
ip link set br0 up
ip addr add 10.0.1.254/24 dev br0
ip link set dev veth1 master br0
ip link set dev veth2 master br0
ip link set dev veth3 master br0
ip link set dev veth1 up
ip link set dev veth2 up
ip link set dev veth3 up
bridge link
ip netns exec net1 ping 10.0.1.1
ip netns exec net1 ping 10.0.1.2
ip netns exec net1 ping 10.0.1.3
ip netns exec net1 ping 10.0.1.0
ip netns exec net1 ping 10.0.1.1
ip netns exec net1 ip link set lo up
ip netns exec net1 ping 10.0.1.1

# 访问外网 (net1 -> host); 加上网关，并且打开了ip forward
ip netns exec net1 ip route add default via 10.0.1.254 dev veth0
```

**Docker网络**

上一节我们通过network namespace模拟了docker的网络原理。那实际上就是docker在host单机网络的一种实现。Docker引擎通过内部的网络驱动来创建不同的网络实现，docker引擎在初始化容器的时候，根据默认的配置或者用户传入的网络类型，来创建容器的网络模型。 目前，docker包括内建的网络驱动模型以及第三方的网络模型。  

**Docker内建网络驱动**

| Driver | 描述 |
| - | - |
| host | 没有网络隔离，容器和主机共享所有网络资源。容器可以直接访问主机的所有接口，容器的网络就好像主机上的一个进程一样，不能和主机其他的进程有端口冲突 |
| bridge | 容器有自己的ip地址，容器之间通过bridge技术进行通信。 但容器要和bridge之外进行通信，需要通过ip forward以及ip table规则实现 |
| overlay | 通过覆盖网络支持多主机上的容器之间进行通信。 综合使用了本地的bridge网络以及vxlan的覆盖网络，把多个在不同主机上的容器子网连接起来，实现跨主机跨网段的通信 |
|MACVLAN| xxxx |
| None | 容器有自己独立的网络空间，但没有创建任何的网络接口。容器完全和外界隔离|

**Docker第三方网络驱动**

Docker也支持第三方网络插件，来构造第三方网络模型。社区上已有的第三方网络插件有: contiv, weave, calico, kuryr, infoblox. 可以参考网上的资料来了解这些网络的特性，根据业务的场景选择合适的网络模型。

**bridge Network**

在默认的情况下，docker engine会在主机上创建docker0 网桥。 docker0为默认的容器之间的通信提供二层交换。当一个容器被创建的时候，一个veth pair会被创建， 其中的一个接口被attach到容器的网络空间中，并且在容器内部重命名为eth0。 而veth pair对应的接口被连接到docker0 网桥上。veth pair技术就像一个管道，在其中的一个接口发送数据，数据会完全复制到另一个接口上， 就像一个传声筒一样。 这样，所有在同一个host上创建的容器之前的通信都可以通过docker0来通信。
![](https://github.com/bigtree21cn/doc/blob/master/docker/bridge-network-bridge.png)

那容器是怎样访问外部世界呢？  当一个数据包发往要给外部ip: port的时候， 数据首先被发到容器内部eth0上，通过veth paire, 数据被发往docker0。因为ip_foward，docker0把数据转发给主机的eth0网卡。 主机发现目的ip地址不是自己网段，就把数据做了SNAT。 把容器的ip地址替换成主机eth0的ip地址发送出去。当目标主机响应时候，主机eth0收到该响应转发给docker0，根据iptable规则
``` bash
iptables -I FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```
该iptable规则的意思是，这条规则的意思是，在宿主机上转发给docker0网桥的网络数据报文，如果是该数据报文所处的连接已经建立的话，则无条件接受，并由Linux内核将其发送到原来的连接上，即回到Docker Container内部。

上面说的是容器主动访问外部世界以及外部响应数据包的流程。 那么，外部是怎么样访问容器内部的服务呢？ 在创建容器的时候，可以通过端口绑定，把容器要暴露给外部的服务端口绑定到容器内部端口上。 由于外部只认识主机的ip地址，当外部数据包到达主机ip上该端口时， 通过DNAT， 主机把外部的ip和端口映射成容器内部的ip和端口。由于host知道容器的ip，所有数据包可以通过veth pair结束发送到容器内部的eth0接口。从而使得外部能访问容纳的服务。

**overlay network - flannel **

覆盖网络是一种应用层网络，通过在已有的三层/四层网络之上，通过应用层协议，把不同子网连接起来形成更大更有弹性的网络。 比如p2p协议，下层的网络就是互联网，通过把互联网上不同的主机上运行同一个应用层协议p2p协议，把各个主机的网络资源共享。

flannel是一种覆盖网络，把tcp包封装在另一种协议中进行传输。 目前已知的支持的下层协议有UDP, TCP, VXLan, GCE等。
下图图示了通过UDP作为转发协议的流程图。
![](https://github.com/bigtree21cn/doc/blob/master/docker/docker.network.flannel.png)

数据包从容器出来以后到达docker0， 然后再经过flannel0. flannel0是一个p2p虚拟网卡，在网卡的另一端，数据包被重新封装从UDP包，发送到目的主机。 目的主机的flannel 服务收到UDP数据包之后，把数据解包，再转发给目的主机的docker0。 所以源容器和目的容器是不知道这个封包和解包的过程。 那么flannel数据包是如何实现跨主机之间的路由呢？ 答案是flannel通过etcd来管理各个节点和主机docker网段的映射。当flannel启动时候，它会向etcd申请一个网段(如172.1.0.0/16),并且把这个网段和主机IP的映射关系写入etcd。同时，docker daemon的启动参数被修改，通过--bip制定通过该daemon进程启动的所有容器ip都在这个网段中。 通过修改路由表，建立flannel0和docker0以及对应网段之间的路由关系。  这样，就实现了跨节点容器之间的网络通信。

这种覆盖网络的好处是非常灵活，很方便的实现网络的扩展。加入新的节点和新的网段都非常方便。但它也有一点性能上的损失。 节点之间的通信默认是不会加密的，通过配置节点之间的证书，可以实现节点之间的安全通信。

# Docker Security

# Ochestration
