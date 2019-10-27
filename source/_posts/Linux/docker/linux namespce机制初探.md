---
title: docker技术分析 - linux namespce机制
---

docker技术是近年来无比火热的技术，不管是哪个领域大家都在研究和使用docker。
我们在使用docker的时候经常听人说，docker是基于linux内核的Namespace和Cgroup技术实现的。那么究竟什么是Namespace技术呢？

## linux namespce
linux的namespce机制提供了一种资源隔离方案，它是Linux平台实现容器这种轻量级的虚拟化的基础。namespce使得PID，IPC，NetWork等系统资源不再是全局性，而是属于特定的namespace的，只需要在clone的时候指定相应的flag，就可以创建相应的namespce。大部分容器就是利用了这一技术实现了资源的隔离，不同容器内的进程属于不同的namespace，彼此透明互不干扰。

目前linux实现了以下6种Namespace

- Mount namespaces(2.4.19)
- UTS namespaces(2.6.19)
- IPC namespaces(2.6.19)
- PID namespaces(2.6.24)
- Network namespaces(2.6.29)
- User namespaces(3.8)

### Mount namespaces
Mount namespaces为一组进程隔离了一系列的文件系统挂载点。在不同Mount namespaces的进程有不同的文件系统层次视图。通过使用Mount namespaces，进程调用mount()和umount()函数时不再操作对所有进程可见的全局挂载点，而是只操作调用进程所关联的mount namespace。

使用Mount namespaces可以象chroot方式一样创建环境。但是，与使用chroot调用对比，Mount namespaces方式更加安全和灵活。

Mount namespaces是Linux中实现的第一种namespace(2002年)。

### UTS namespaces
UTS namespaces实现uname()系统调用返回的对两个系统描述符nodename和domainname的隔离（这两个值可以由sethostname()和setdomainname()系统调用来设置）。

在容器的上下文中，UTS namespaces允许每个容器有自己的hostname和NIS domain name。这对于基于这两个名字的初始化和配置脚本很有用。

###  PID namespaces
PID namespaces实现进程间ID空间隔离。在不同PID namespaces中的进程可以有相同的PID。PID namespaces的一个主要好处是容器可以在不同的主机上移植，保持容器内部的进程ID不变。

PID namespaces还允许每个容器有自己的init进程，及PID为1的进程，这个进程是容器中所有进程的祖先。

### NET namespaces
NET namespaces提供对与网络相关的系统资源的隔离。每个Network namespace有自己的网络设备、IP地址、IP路由表、/proc/net目录，端口号等等。

也就是说同一台主机上的多个容器中的web服务器都可以绑定80端口。

### User namespaces
User namespaces提供对用户id和组id号码空间的资源隔离。在一个user namespaces内部和外部，一个进程有不同的userid和groupid。

User namespace可以让一个进程在User namespace内有root权限，而在User namespace外则只有普通权限。

### 用go语言来操作namespace
由于docker是用go语言实现的，故我这里也使用go语言来作为范例，创建一个utsnamespace。代码如下
```go
package main

import (
	"os"
	"os/exec"
	"syscall"
	"log"
)

func main()  {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```
这里go语言帮我们封装了对clone函数的调用，我们在这里指定了系统调用的参数`Cloneflags: syscall.CLONE_NEWUTS`，系统会根据该参数去创建一个UTC Namespace并进入到一个sh环境中。
我们执行go build，再运行该程序，注意，这里我们需要以管理员权限来执行该程序，不然会提示如下错误。
![](https://raw.githubusercontent.com/readlnh/picture/master/utc-namespace-test1.png) 
我们运行该程序后似乎什么都没有发生，实际上我们已经进入新的namespace了
此时在终端输入hostname命令，发现和主机的hostname是一样的
![](https://raw.githubusercontent.com/readlnh/picture/master/utc-namespace-test2.png) 
我们在新建的namespace里修改hostname，此时我们在宿主机上再启动一个终端，执行hostname命令，发现宿主机的hostname并没有改变，由此可见UTC Namespce可以拥有自己的hostname。
![](https://raw.githubusercontent.com/readlnh/picture/master/utc-namespce-test3.png) 
这里我们只指定了一个参数，实际上我们可以在clone时同时指定多重命名空间，由此来达到进程的隔离，各位读者可以自行尝试。如果想使用c语言来进行尝试，只需要在调用clone()函数时指定相应参数即可，这里就不过多展开了。