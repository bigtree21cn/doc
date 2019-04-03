
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
  ### namespace
  ### cgroup

# Docker Archicture

# Docker Image

# Docker Networking

# Docker Security

# Ochestration
