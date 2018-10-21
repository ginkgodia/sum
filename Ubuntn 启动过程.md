**Ubuntu启动过程**

操作系统的启动：

Linux 操作系统有三种方式： 'System V init , upstart , systemd'

System V init 启动方式：

Debian System V 启动过程：

 内核文件加载后，就开始运行第一个程序是/sbin/init ，他的作用是初始化系统环境

由于init 是第一个运行的程序， 他的进程编号pid 就是1， 其他所有进程都从他衍生，都是他的子进程。

许多程序都需要开机启动，他们在windows叫服务(service), 在Linux就叫守护进程(daemon)， init 进程的一大任务就是运行这些开机启动的程序。

Debian init 进程首先读取文件/etc/inittab ， 他是运行级别的设置文件，如果你打开它， 可以看到

```
id:2:initdefault:

对于CentOS系统来说： 

init程序类型：  
1. SysV: init, CentOS 5之前  
配置文件：/etc/inittab  
2. Upstart: init,CentOS6  
配置文件：/etc/inittab, /etc/init/*.conf  
3. Systemd：systemd, CentOS 7  
配置文件：/usr/lib/systemd/system；/etc/systemd/system  
```

initdefault 的值是2 ， 表明系统运行时的运行级别是2， 如果需要改动