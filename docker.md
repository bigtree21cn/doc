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



# Docker Archicture

# Do

# Docker Networking

# Docker Security

# Ochestration
