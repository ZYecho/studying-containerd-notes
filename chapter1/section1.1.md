# 简介

Containerd 是一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性。Containerd 可以在宿主机中管理完整的容器生命周期：容器镜像的传输和存储、容器的执行和管理、存储和网络等。

![image.png-16.5kB][1]
从上图中我们可以看到containerd在整个docker engine架构中的作用。containerd作为抽象的容器运行时，是比较薄的一层，而containerd-shim作为底层执行容器的父进程，完成具体容器进程的监控和回收工作，使得容器可控。

下图中包含了containerd的整个组件，其中底层存储通过golang实现的boltdb来完成，容器以metadata的形式存在于containerd，创建销毁容器通过task的接口来完成，底层的Runtime通过runc/runv等符合OCI标准的运行时来完成。
![image.png-105.9kB][2]

在线版grpc接口文档:
https://dnephin.github.io/containerd/


  [1]: http://static.zybuluo.com/myecho/frkxdgjsk7f4xwjboadifk4n/image.png
  [2]: http://static.zybuluo.com/myecho/i7csubxibdandpywp1wl0jew/image.png
