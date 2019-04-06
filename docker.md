
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
  在最先的Linux操作系统中，所有的资源是进行全局管理。每个进程度拥有的资源，也可以在其他进每个班的同学有一个程中看到。就像一个公司，每个同事都知道其他同事在干什么。 这样资源的隔离性，安全性都不能很好的保重。一个危险的进程，有可能破坏系统中的其他进程。namespace是一种轻量级的虚拟机制，是的每个进程，每种资源可以在不同的namespace中。 资源至对相同namespace中的进程可见，这样提供了一种安全的隔离机制。就像一个学校每个班级一样，班级同学之间可以互相看到和沟通，但不能影响其他的班级同学。 
  
  ### cgroup

# Docker Archicture

# Docker Image

# Docker Networking

# Docker Security

# Ochestration
