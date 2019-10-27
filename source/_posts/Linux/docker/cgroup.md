---
title: 聊一聊linux中的cgroup
---

最近由于工作原因接触了docker，也就研究一下它的原理，主要看了阿里出的<<自己动手写docker>>，感觉蛮有意思，姑且记录一下。docker可以算是当前非常火热的技术了。我们知道docker基于Namespce和Cgroups，其中

- Namespace主要用于隔离资源
- Cgroups用来提供对一组进程以及将来子进程的资源限制

## Cgrougps的三个组件
cgrpups包含三个组件

- **控制组** 一个cgroups包含一组进程，并可以在这个cgroups上增加Linux subsystem的各种参数配置，将一组进程和一组subsystem关联起来。

- **subsystem** 子系统 是一组资源控制模块，可以通过`lssubsys -a`命令查看当前内核支持哪些subsystem。
```
cpuset
cpu,cpuacct
blkio
memory
devices
freezer
net_cls,net_prio
perf_event
hugetlb
pids
rdma
```
subsystem作用于hierarchy的cgroup节点，并控制节点中进程的资源占用。

- **hierarchy 层级树** 主要功能是把cgroups串成一个树型结构，使cgruops可以做到继承。也就是说将cgroup通过树状结构串起来，通过虚拟文件系统的方式暴露给用户。

### 三个组件之前的关系
cgroup中的组件是相互关联的

- 系统创建新的hierarchy之后，系统中所有的进程都会加入这个hierarchy的cgroup的根节点，这个cgroup根节点是hierarchy默认创建的，在这个hierarchy中创建的所有cgroup都是这个cgroup根节点的子节点。

- 一个subsystem只能附加到一个hierarchy上

- 一个hierarchy可以附加多个subsystem

- 一个进程可以作为多个cgroup的成员，但是这些cgroup必须在不同的hierarchy下

- 一个进程fork出子进程时，子进程和父进程是在同一个cgroup中的，根据需要也可以移动到其他的cgroup中

## 使用Cgroup

### 创建挂载点
我们知道kernel是通过一个虚拟树状文件系统来配置Cgroups的。我们首先需要创建并挂载一个hierarchy(cgroup树)。即先mkdir后mount。
```shell
readlnh@readlnh-Inspiron-3542:~$ sudo mount -t cgroup -o none,name=cgroup-test cgroup-test cgroup-test/
readlnh@readlnh-Inspiron-3542:~$ ls ./c
cgroup-test/  clone_test.c
readlnh@readlnh-Inspiron-3542:~$ ls ./cgroup-test/
cgroup.clone_children  cgroup.sane_behavior  release_agent
cgroup.procs           notify_on_release     tasks
```
可以看到挂载后系统在目录下生成了一系列文件，这些文件其实就是根节点文件的配置项。

### 创建子cgroup 
创建cgroup-1，我们可以看到在cgroup文件夹下再创建文件夹，系统会将这个文件夹也标记为cgroup，并且它是上一个cgroup的子cgroup，它会继承父cgroup的属性
```
readlnh@readlnh-Inspiron-3542:~/cgroup-test$ sudo mkdir cgroup-1
readlnh@readlnh-Inspiron-3542:~/cgroup-test$ tree
.
├── cgroup-1
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── notify_on_release
│   └── tasks
├── cgroup.clone_children
├── cgroup.procs
├── cgroup.sane_behavior
├── notify_on_release
├── release_agent
└── tasks

1 directory, 10 files
```

### 移动进程
将终端进程移动到cgroup-1(只要将进程ID写到相应的cgroup的tasks文件)
```sh
readlnh@readlnh-Inspiron-3542:~/cgroup-test/cgroup-1$ echo $$
10644
readlnh@readlnh-Inspiron-3542:~/cgroup-test/cgroup-1$ cat /proc/10644/cgroup 
13:name=cgroup-test:/
12:perf_event:/
11:memory:/user.slice
10:hugetlb:/
9:cpuset:/
8:pids:/user.slice/user-1000.slice/user@1000.service
7:rdma:/
6:devices:/user.slice
5:freezer:/
4:cpu,cpuacct:/user.slice
3:net_cls,net_prio:/
2:blkio:/user.slice
1:name=systemd:/user.slice/user-1000.slice/user@1000.service/gnome-terminal-server.service
0::/user.slice/user-1000.slice/user@1000.service/gnome-terminal-server.service
readlnh@readlnh-Inspiron-3542:~/cgroup-test/cgroup-1$ sudo sh -c "echo $$ >> tasks"
readlnh@readlnh-Inspiron-3542:~/cgroup-test/cgroup-1$ cat /proc/10644/cgroup 
13:name=cgroup-test:/cgroup-1
12:perf_event:/
11:memory:/user.slice
10:hugetlb:/
9:cpuset:/
8:pids:/user.slice/user-1000.slice/user@1000.service
7:rdma:/
6:devices:/user.slice
5:freezer:/
4:cpu,cpuacct:/user.slice
3:net_cls,net_prio:/
2:blkio:/user.slice
1:name=systemd:/user.slice/user-1000.slice/user@1000.service/gnome-terminal-server.service
0::/user.slice/user-1000.slice/user@1000.service/gnome-terminal-server.service
```
可以看到现在终端进程已经在cgroup-1了

### 通过subsystem限制cgroups中的进程资源
之前我们创建的cgroup是没有和任何subsystem相关联的，所以没法通过hierarchy中的cgroup节点限制资源。实际系统中已经默认为每个subsystem创建了一个默认的hierarchy，我们这里就直接在这个hierarchy下创建cgroup
```sh
readlnh@readlnh-Inspiron-3542:~/cgroup-test/test-limit-memory$ mount | grep memory
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
readlnh@readlnh-Inspiron-3542:~/cgroup-test/test-limit-memory$ cd /sys/fs/cgroup/memory/
readlnh@readlnh-Inspiron-3542:/sys/fs/cgroup/memory$ sudo mkdir test-limit-memory
readlnh@readlnh-Inspiron-3542:/sys/fs/cgroup/memory$ cd test-limit-memory/
readlnh@readlnh-Inspiron-3542:/sys/fs/cgroup/memory/test-limit-memory$ ls
cgroup.clone_children               memory.limit_in_bytes
cgroup.event_control                memory.max_usage_in_bytes
cgroup.procs                        memory.move_charge_at_immigrate
memory.failcnt                      memory.numa_stat
memory.force_empty                  memory.oom_control
memory.kmem.failcnt                 memory.pressure_level
memory.kmem.limit_in_bytes          memory.soft_limit_in_bytes
memory.kmem.max_usage_in_bytes      memory.stat
memory.kmem.slabinfo                memory.swappiness
memory.kmem.tcp.failcnt             memory.usage_in_bytes
memory.kmem.tcp.limit_in_bytes      memory.use_hierarchy
memory.kmem.tcp.max_usage_in_bytes  notify_on_release
memory.kmem.tcp.usage_in_bytes      tasks
memory.kmem.usage_in_bytes
readlnh@readlnh-Inspiron-3542:/sys/fs/cgroup/memory/test-limit-memory$ stress --vm-bytes 200m --vm-keep -m 1
stress: info: [15357] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
```
用top查看，发现内存为100m，实际上这里限制内存就是一个向memory_limit_in_bytes文件中写入100m这个操作，可以说相简单了
```sh
15358 readlnh   20   0  213044 100916    212 D  39.5  1.3   7:54.10 stress
```

从这里我们就可以猜测，docker实际上就是通过go语言挂载，创建cgroups再向相应文件写入相应条件来限制容器资源的。cgroup确实是一个很有意思也很方便的功能。