---
title: Build a Mini Docker
tags:
  - docker
  - go
  - linux
description: In this post, we are going to build a mini docker with minimum codes but full functions.
cover: 'https://pic.imgdb.cn/item/639debd8b1fccdcd36fe1b3d.jpg'
abbrlink: 6cc923ae
date: 2022-12-15 21:14:09
categories:
---

This post mainly refers to a "GOTO; Conferences" YouTube video "[Learn how to code a container using Go code • Liz Rice](https://www.youtube.com/watch?app=desktop&v=8fi7uSYlOdc&feature=youtu.be)". 

She build a mini docker step by step with less than 100 lines codes, using the `Namespace` and `CGroups` provided by OS Kernel. It is not difficult to understand the main concept of the Container, as if you don't dig into the details and have some basic knowledge to the Linux OS. And this video explains what the Docker really is one the other side, which confused me since I first saw the word Docker.

Talk is cheap, show me the code. Let's start.

# Background

We'll build a 'mini-docker' that receive and execute a raw shell command for example.

> hint:  You must run this code in a Linux-Like OS

```bash
# a docker command and a mini-docker command that we defined

docker		run	image	<cmd>	<args>
go run main.go	run		<cmd>	<args>
```

It's easy to write the `main.go` as bellow:

```go
// go run main.go run <cmd> <args>
func main() {
	switch os.Args[1] {
	case "run":
		run()
	default:
		fmt.Printf("do nothing, exit!!!")
	}
}

func run() {
	fmt.Printf("Running %v\n", os.Args[2:])
	cmd := exec.Command(os.Args[2], os.Args[3:]...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	cmd.Run()
}
```

And the result is:

```bash
bytedance@C02D83XAML85 [17:23:06] [~/myhub/containers-from-scratch] [master *]
-> % ls
LICENSE   README.md main.go

bytedance@C02D83XAML85 [17:23:06] [~/myhub/containers-from-scratch] [master *]
-> % go run main.go run ls -l                    
Running [ls -l]
total 24
-rw-r--r--  1 bytedance  staff  1065 12 16 16:23 LICENSE
-rw-r--r--  1 bytedance  staff   568 12 16 16:23 README.md
-rw-r--r--  1 bytedance  staff   380 12 16 17:20 main.go
```

It's simplely execute a cmd on the host machine and the current process, what we really want is that the cmd **execute in a seperate environment from the host** current process. In order to achieve that, we need two feature provided by Operating System. In the Linux case, they are Namespaces and CGroups.



# Namespace: Visibility

Firstly, let's get some basic prior knowledge of the [Namespace](https://man7.org/linux/man-pages/man7/namespaces.7.html) in the Linux OS. 

Namepace provided by the Linux kernel can isolate a process  into a seperate environment. Changes to the global resource in a namespace are visible to other processes that are members of the namespace, but are invisible to other processes. 

There are some types of namespace that seperate different resources.

| Namespace                         | Flag           | Kernel Version | Isolates                             |
| --------------------------------- | -------------- | -------------- | ------------------------------------ |
| UTS (Unix Time-sharing System)    | CLONE_NEWUTS   | Linux 2.4.19   | Hostname and NIS domain name         |
| IPC (Inter-Process Communication) | CLONE_NEWIPC   | Linux 2.6.19   | System V IPC, POSIX message queues   |
| PID (Process ID)                  | CLONE_NEWPID   | Linux 2.6.19   | Process IDs                          |
| Network                           | CLONE_NEWNET   | Linux 2.6.24   | Network devices, stacks, ports, etc. |
| Mount                             | CLONE_NEWNS    | Linux 2.6.29   | Mount points                         |
| User                              | CLONE_NEWUSER  | Linux 3.8      | User and group IDs                   |
| CGroup                            | CLONE_NEWGROUP | Linux 4.6      | Cgroup root directory                |

And the namespaces API includes the following system calls:

- `clone(...)`: Creates a new process. If the flags argument of the call specifies one or more of the `CLONE_NEW* ` flags listed above, then new namespaces are created for each flag, and the child process is made a member of those namespaces. 
- `setns(...)`: Allows the calling process to an existing namespace. 
- `unshare(...)`: Allows the calling process to a new namespace.

Each process has a `/proc/[pid]/ns/` subdirectory containing one entry for each namespace. All these files are linux link files with the format of  `namespace_type:[inode_number]`.

```bash
yanhua.y@n37-144-139:~$ ls -l /proc/$$/ns
total 0
lrwxrwxrwx 1 yanhua.y yanhua.y 0 Dec 16 21:13 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 yanhua.y yanhua.y 0 Dec 16 21:13 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 yanhua.y yanhua.y 0 Dec 16 21:13 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 yanhua.y yanhua.y 0 Dec 16 21:13 net -> 'net:[4026531992]'
lrwxrwxrwx 1 yanhua.y yanhua.y 0 Dec 16 21:13 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 yanhua.y yanhua.y 0 Dec 16 21:13 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 yanhua.y yanhua.y 0 Dec 16 21:13 user -> 'user:[4026531837]'
lrwxrwxrwx 1 yanhua.y yanhua.y 0 Dec 16 21:13 uts -> 'uts:[4026531838]'
```



OK, that is enough, let's back to the code.

If we want to **isolate the hostname** and specify a new hostname for the container, we can easily think of that just create a process by the `clone(...)` and pass the `CLONE_NEWUTS` flag to the function, like this:

```go
// go run main.go run <cmd> <args>
func main() {
	switch os.Args[1] {
	case "run":
		run()
	case "child":
		child()
	default:
		panic("help")
	}
}

func run() {
	fmt.Printf("Running run() %v \n", os.Args[2:])

	cmd := exec.Command("/proc/self/exe", append([]string{"child"}, os.Args[2:]...)...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS,
	}

	must(cmd.Run())
}

func child() {
	fmt.Printf("Running child() %v \n", os.Args[2:])

	cmd := exec.Command(os.Args[2], os.Args[3:]...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	must(syscall.Sethostname([]byte("container")))

	must(cmd.Run())
}

func must(err error) {
	if err != nil {
		panic(err)
	}
}
```

There is a point we need to notice is that the `line 16` execute the `/proc/self/exe` which means the current process. By doing this, wen can `fork` the main process and pass a new argument `child` to the `main()`. In other language like `C/C++`, it may be something like `fork()` and `execv()`.

```bash
# some tricky problem still not resolved 
yanhua.y@n37-144-139:~/containers-from-scratch$ go run main.go run /bin/bash
Running run() [/bin/bash] 
panic: fork/exec /proc/self/exe: operation not permitted
...

# I can only do this
yanhua.y@n37-144-139:~/containers-from-scratch$ go build main.go 
yanhua.y@n37-144-139:~/containers-from-scratch$ ls
LICENSE  main  main.go	README.md
yanhua.y@n37-144-139:~/containers-from-scratch$ sudo ./main run /bin/bash
Running run() [/bin/bash] 
Running child() [/bin/bash] 
root@container:/data00/home/yanhua.y/containers-from-scratch$ hostname 
container
root@container:/data00/home/yanhua.y/containers-from-scratch$ exit
exit
yanhua.y@n37-144-139:~/containers-from-scratch$ hostname 
n37-144-139
```



Next is **PID Isolation**. 

```go
func main() {
	//...
}

func run() {
	//...
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID,
	}
	//...
}

func child() {
	fmt.Printf("running child() %v as pid: %d\n", os.Args[2:], os.Getpid())
	//...
}
```

 Similarly, we can pass the `CLONE_NEWPID` and see what happened. 

```bash
yanhua.y@n37-144-139:~/containers-from-scratch$ sudo ./main run /bin/bash
Running run() [/bin/bash] 
running child() [/bin/bash] as pid: 1
root@container:/data00/home/yanhua.y/containers-from-scratch# ps
    PID TTY          TIME CMD
 469953 pts/0    00:00:00 sudo
 469954 pts/0    00:00:00 main
 469960 pts/0    00:00:00 exe
 469965 pts/0    00:00:00 bash
 470040 pts/0    00:00:00 ps
root@container:/data00/home/yanhua.y/containers-from-scratch# echo $$
6
```

Oops, something looks like went wrong!  The pid of the child container echo the `pid 1` and the current shell with `pid 6` , but when we use `ps`, **it echos the process IDs of the host**. 

And the reason for that is because the `ps` command look up the `/proc` dir (actually it's a mount pseudo-filesystem communicate with the kernel) for process information and this directory is inherited from the host, but not the same for the `os.GetPid()`. 

Anyway the  `PID` is isolated now, what not isolated is the **file system**. 

So we need to: 

1. create a new Linux file system directory, or you can download from [here](https://cdimage.ubuntu.com/ubuntu-base/releases/16.04.1/release/)
2. change the container‘s root `/` to the new file system's root path, and switch to it
2. and finally mount the container's `proc`

 ```go
 func run() {
 	fmt.Printf("Running run() %v \n", os.Args[2:])
 
 	cmd := exec.Command("/proc/self/exe", append([]string{"child"}, os.Args[2:]...)...)
 	cmd.Stdin = os.Stdin
 	cmd.Stdout = os.Stdout
 	cmd.Stderr = os.Stderr
 	cmd.SysProcAttr = &syscall.SysProcAttr{
 		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID,
 	}
 
 	must(cmd.Run())
 }
 
 func child() {
 	fmt.Printf("running child() %v as pid: %d\n", os.Args[2:], os.Getpid())
 
 	cmd := exec.Command(os.Args[2], os.Args[3:]...)
 	cmd.Stdin = os.Stdin
 	cmd.Stdout = os.Stdout
 	cmd.Stderr = os.Stderr
 
 	must(syscall.Sethostname([]byte("container")))
 	must(syscall.Chroot("/var/lib/ubuntu-fs"))  //this is the path of your contaner file system to use
 	must(os.Chdir("/"))
 	must(syscall.Mount("proc", "proc", "proc", 0, "")) 
 
 	must(cmd.Run())
 
 //	must(syscall.Unmount("proc", 0))
 }
 ```

At this time, it works! The `ps` echo the right content! 

```bash
yanhua.y@n37-144-139:~/containers-from-scratch$ sudo ./main run /bin/bash
Running run() [/bin/bash] 
running child() [/bin/bash] as pid: 1
root@container:/# ps
    PID TTY          TIME CMD
      1 ?        00:00:00 exe
      6 ?        00:00:00 bash
     11 ?        00:00:00 ps
root@container:/# ls proc
1              bus       diskstats        fs          key-users    locks    pagetypeinfo  softirqs       timer_list
13             cgroups   dma              interrupts  keys         meminfo  partitions    stat           tty
6              cmdline   driver           iomem       kmsg         misc     pressure      swaps          uptime
acpi           consoles  elkeid-endpoint  ioports     kpagecgroup  modules  sched_debug   sys            version
asound         cpuinfo   execdomains      irq         kpagecount   mounts   schedstat     sysrq-trigger  vmallocinfo
bpf_unsafe_gh  crypto    fb               kallsyms    kpageflags   mtrr     self          sysvipc        vmstat
buddyinfo      devices   filesystems      kcore       loadavg      net      slabinfo      thread-self    zoneinfo


root@container:/# mount
proc on /proc type proc (rw,relatime)
root@container:/# exit
exit
yanhua.y@n37-144-139:~/containers-from-scratch$ mount
#...
proc on /var/lib/ubuntu-fs/proc type proc (rw,relatime)

```



Next is **mount isolation**. 

As you can notice the last several lines, the **mount info of the container can be seen in the host** which we would not want.

Naturally we put the `CLONE_NEWNS` to the `clone(...)`  and see what happen.

```go
// codes...
cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS, 
}
// codes..
```

```bash
yanhua.y@n37-144-139:~/containers-from-scratch$ sudo ./main run /bin/bash
Running run() [/bin/bash] 
running child() [/bin/bash] as pid: 1
root@container:/# mount
proc on /proc type proc (rw,relatime)
root@container:/# exit
exit
yanhua.y@n37-144-139:~/containers-from-scratch$ mount
#...
proc on /var/lib/ubuntu-fs/proc type proc (rw,relatime)
```

**Nothing changed !** This is because the mount isolation have a special setting of `propagation type` , the default is `MS_SHARED` which means the mount action propagate between host and container. If we want break this rule, we can set the `propagation type` to `MS_SLAVE`. In Go, we can add the `CLONE_NEWNS`  to the `SysProcAttr.Unshareflags`.  And finnally it works

```go
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags:   syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
		Unshareflags: syscall.CLONE_NEWNS,
	}
```

```bash
yanhua.y@n37-144-139:~/containers-from-scratch$ sudo ./main run /bin/bash
Running run() [/bin/bash] 
running child() [/bin/bash] as pid: 1
root@container:/# mount
proc on /proc type proc (rw,relatime)
root@container:/# exit
exit

yanhua.y@n37-144-139:~/containers-from-scratch$ mount |grep proc
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
# no proc mount on /var/lib/ubuntu-fs
```

The rest of namespace isolation is similar and won't be covered here.



# CGroups: Availability

The namespaces controls the visibility of the container. Now we'll solve how to control the **availability of resource**.

As before, let's get some basic prior knowledge of the [CGroups](https://man7.org/linux/man-pages/man7/cgroups.7.html) in the Linux OS.

CGroups are a Linux kernel feature which allow processes to be organized into hierarchical groups whose usage of various types of resources can then be **limited and monitored**.  The kernel's cgroup interface is provided through a pseudo-filesystem called `cgroupfs` (same as `/proc`). All of the cgroups files are maintained in the path `/sys/fs/cgroup` .

```bash
yanhua.y@n37-144-139:/sys/fs/cgroup$ ls
blkio  cpuacct	    cpuset   freezer  net_cls		net_prio    pids     unified
cpu    cpu,cpuacct  devices  memory   net_cls,net_prio	perf_event  systemd
yanhua.y@n37-144-139:/sys/fs/cgroup$ cd memory/

yanhua.y@n37-144-139:/sys/fs/cgroup/memory$ ls
cgroup.clone_children	    memory.kmem.max_usage_in_bytes	memory.memsw.limit_in_bytes	 memory.usage_in_bytes
cgroup.event_control	    memory.kmem.slabinfo		memory.memsw.max_usage_in_bytes  memory.use_hierarchy
cgroup.procs		    memory.kmem.tcp.failcnt		memory.memsw.usage_in_bytes	 memory.watermark
cgroup.sane_behavior	    memory.kmem.tcp.limit_in_bytes	memory.move_charge_at_immigrate  memory.watermark_scale_factor
etrace_cpu		    memory.kmem.tcp.max_usage_in_bytes	memory.numa_stat		 notify_on_release
etrace_mem		    memory.kmem.tcp.usage_in_bytes	memory.oom_control		 release_agent
memory.failcnt		    memory.kmem.usage_in_bytes		memory.pressure_level		 system.slice
memory.force_empty	    memory.limit_in_bytes		memory.soft_limit_in_bytes	 tao_tasks
memory.kmem.failcnt	    memory.max_usage_in_bytes		memory.stat			 tasks
memory.kmem.limit_in_bytes  memory.memsw.failcnt		memory.swappiness		 user.slice
```

Here is the Go code. We call `cg()` **before** the `Chroot()` function. The `cg()` function first create a cgroup dir  for the container, and then  limit the max process num to `20`, in the end add the child process to the cgroup by add its pid into the `cgroup.proc` file.

```go
func child() {
	fmt.Printf("Running %v \n", os.Args[2:])

	cg()
  
	cmd := exec.Command(os.Args[2], os.Args[3:]...)
	....
  
	must(cmd.Run())
	must(syscall.Unmount("proc", 0))
}

func cg() {
  // Create a cgroup dir for the container
	cgroups := "/sys/fs/cgroup/"
	pids := filepath.Join(cgroups, "pids")
	os.Mkdir(filepath.Join(pids, "liz"), 0755)
  
	must(ioutil.WriteFile(filepath.Join(pids, "liz/pids.max"), []byte("20"), 0700))
	// Removes the new cgroup in place after the container exits
	must(ioutil.WriteFile(filepath.Join(pids, "liz/notify_on_release"), []byte("1"), 0700))
  // Write the pid into the cgroup.procs means add this process to the cgroup
	must(ioutil.WriteFile(filepath.Join(pids, "liz/cgroup.procs"), []byte(strconv.Itoa(os.Getpid())), 0700))
}

```

Below is a tricky way to check the container process is in the cgroup.  In the container process call `sleep`, find the pid in the host, check the pid in the new cgroups dir's `cgroup.procs`.

```bash
# container bash ...
yanhua.y@n37-144-139:~/containers-from-scratch$ sudo ./main run /bin/bash
running child() [/bin/bash] as pid: 643812
running child() [/bin/bash] as pid: 1
root@container:/# sleep 100

# host bash ...
yanhua.y@n37-144-139:/sys/fs/cgroup/pids/liz$ cat pids.max
20
yanhua.y@n37-144-139:/sys/fs/cgroup/pids/liz$ cat cgroup.procs 
643817
643822
yanhua.y@n37-144-139:/sys/fs/cgroup/pids/liz$ ps -C sleep
    PID TTY          TIME CMD
 644805 ?        00:00:00 sleep
 644817 pts/0    00:00:00 sleep
yanhua.y@n37-144-139:/sys/fs/cgroup/pids/liz$ cat cgroup.procs 
643817
643822
644817
```

At last, we can check the limit of pid amount works. We run the `:() { : | : & } ; :`  which keeps fork child without break. 

Everything goes well, the host isn't crash and the total number of process is limited to `20`.

Similarly, other resource listed in the `/sys/fs/cgroup` can also be controlled by this way. 

```bash
# container bash
root@container:/# :() { : | : & } ; :
[1] 15
root@container:/# bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: No child processes
bash: fork: retry: No child processes
bash: fork: retry: No child processes
bash: fork: retry: No child processes
bash: fork: retry: No child processes
bash: fork: retry: No child processes
bash: fork: retry: No child processes
bash: fork: retry: No child processes
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: No child processes
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: No child processes
bash: fork: retry: No child processes
bash: fork: retry: No child processes

# host bash
yanhua.y@n37-144-139:/sys/fs/cgroup/pids/liz$ cat pids.current 
20
yanhua.y@n37-144-139:/sys/fs/cgroup/pids/liz$ cat cgroup.procs 
643817
643822
yanhua.y@n37-144-139:/sys/fs/cgroup/pids/liz$ ps fa
    PID TTY      STAT   TIME COMMAND
 641238 pts/1    Ss     0:00 -bash
 645972 pts/1    R+     0:00  \_ ps fa
 637779 pts/0    Ss     0:00 -bash
 643811 pts/0    S      0:00  \_ sudo ./main run /bin/bash
 643812 pts/0    Sl     0:00      \_ ./main run /bin/bash
 643817 pts/0    Sl     0:00          \_ /proc/self/exe child /bin/bash
 643822 pts/0    S+     0:00              \_ /bin/bash
 645470 pts/0    Z      0:00              \_ [bash] <defunct>
 645471 pts/0    Z      0:00              \_ [bash] <defunct>
 645472 pts/0    Z      0:00              \_ [bash] <defunct>
 645473 pts/0    Z      0:00              \_ [bash] <defunct>
 645474 pts/0    Z      0:00              \_ [bash] <defunct>
 645475 pts/0    Z      0:00              \_ [bash] <defunct>
 645476 pts/0    Z      0:00              \_ [bash] <defunct>
 645477 pts/0    Z      0:00              \_ [bash] <defunct>
 645478 pts/0    Z      0:00              \_ [bash] <defunct>
 645479 pts/0    Z      0:00              \_ [bash] <defunct>
 645480 pts/0    Z      0:00              \_ [bash] <defunct>
 645481 pts/0    Z      0:00              \_ [bash] <defunct>
 645482 pts/0    Z      0:00              \_ [bash] <defunct>
 645483 pts/0    Z      0:00              \_ [bash] <defunct>
   1174 ttyS0    Ss+    0:00 /sbin/agetty -o -p -- \u --keep-baud 115200,38400,9600 ttyS0 vt220
   1172 tty1     Ss+    0:00 /sbin/agetty -o -p -- \u --noclear tty1 linux

```



# Conclusion

I believe that through the above introduction, you must get deeper into Docker, as least for me. 

Let's back to a classic question: What is Docker or Container?

From this post, we can feel that a container is a capsulate of a process's environment that isolate from the host machine and other containers. This definition is not accurately but can help you to understand what this post is doing. 

Last is the comparison of Docker and Virtual Machine. As we know, an operating system is is divided into two parts: Kernel and Application(Run Time Library). The most difference between the Docker and the Virtual Machine is the isolation level. The Virtual Machine Virtualized a totally new Guest Operating System without using the host's kernel, but the docker just virtualized a new application level engine and all containers shared the same host kernel.

<img src="https://pic.imgdb.cn/item/639df1d7b1fccdcd3608a069.jpg" alt="docker-vs-container" style="zoom: 50%;" />





