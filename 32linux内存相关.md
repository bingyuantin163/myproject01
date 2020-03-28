# linux内存详解

### 1.centos7内存说明

##### 1.1查看内存命令

```sh
free -m
              total        used        free      shared  buff/cache   available
Mem:           1819         918          87          86         813         631
Swap:          2047         138        1909
#因此在CentOS7之后，用户不需要去计算buffer/cache，直接看available 即可以看到还有多少内存可用，更加简单直观
#used：当前已使用的内存总量
#free：空闲的或可以使用的内存总量
#shared：共享内存大小，主要用于进程间通信
#buff(buffers):主要用于块设备数据缓冲，例如记录文件系统的metadata（目录、权限等等信息）
#cache:主要用于文件内容缓冲
#available:可以使用的内存总量

#Linux下内存buff/cache占用过多问题解决
echo 1 > /proc/sys/vm/drop_caches
echo 2 > /proc/sys/vm/drop_caches
echo 3 > /proc/sys/vm/drop_caches
```

##### 1.2buffer和cached被合成一组，加入了一个available，关于此available，英文你文档说明如下

```sh
	1.MemAvailable: An estimate of how much memory is available for starting new applications,       without swapping.
	2.即系统可用内存，之前说过由于buffer和cache可以在需要时被释放回收，系统可用内存即 free + buffer + cache，在CentOS7之后这种说法并不准确，因为并不是所有的buffer/cache空间都可以被回收。
	3.即available = free + buffer/cache - 不可被回收内存(共享内存段、tmpfs、ramfs等)。
	4.因此在CentOS7之后，用户不需要去计算buffer/cache，直接看available 即可以看到还有多少内存可用，更加简单直观
	5.
```

##### 1.3推荐命令

```sh
free -mlth
              total        used        free      shared  buff/cache   available
Mem:           1.8G        916M         91M         86M        810M        632M
Low:           1.8G        1.7G         91M
High:            0B          0B          0B
Swap:          2.0G        138M        1.9G
Total:         3.8G        1.0G        2.0G
```

### 2.Swap交换分区概念

##### 2.1什么是Linux swap space，英文介绍

```sh
Linux  divides its physical RAM (random access memory) into chucks of memory  called pages. Swapping is the process whereby a page of memory is copied  to the preconfigured space on the hard disk, called swap space, to free  up that page of memory. The combined sizes of the physical memory and  the swap space is the amount of virtual memory available.
Swap  space in Linux is used when the amount of physical memory (RAM) is  full. If the system needs more memory resources and the RAM is full,  inactive pages in memory are moved to the swap space. While swap space  can help machines with a small amount of RAM, it should not be  considered a replacement for more RAM. Swap space is located on hard  drives, which have a slower access time than physical memory.Swap space  can be a dedicated swap partition (recommended), a swap file, or a  combination of swap partitions and swap files.

Linux内核为了提高读写效率与速度，会将文件在内存中进行缓存，这部分内存就是Cache  Memory(缓存内存)。即使你的程序运行结束后，Cache  Memory也不会自动释放。这就会导致你在Linux系统中程序频繁读写文件后，你会发现可用物理内存变少。当系统的物理内存不够用的时候，就需要将物理内存中的一部分空间释放出来，以供当前运行的程序使用。那些被释放的空间可能来自一些很长时间没有什么操作的程序，这些被释放的空间被临时保存到Swap空间中，等到那些程序要运行时，再从Swap分区中恢复保存的数据到内存中。这样，系统总是在物理内存不够时，才进行Swap交换。
```

##### 2.2查看swap分区大小

```sh
	1.free -m
		              total        used        free      shared  buff/cache   available
	Mem:           1819         911         106          86         801         638
	Swap:          2047         139        1908
	2.swapon -s
	Filename				Type		Size	Used	Priority
	/dev/dm-1                              	partition	2097148	142592	-2
	3.cat /proc/swaps
	Filename				Type		Size	Used	Priority
	/dev/dm-1                               partition	2097148	142592	-2
```

##### 2.3swap分区大小设置

```sh
	1.按照oracle的官方推荐设置如下
```

![image-20200110140308618](C:\Users\Owner\AppData\Roaming\Typora\typora-user-images\image-20200110140308618.png)

```sh
	2.其他的一些设置标准，仅供参考
		4G以内的物理内存，SWAP 设置为内存的2倍。
		4-8G的物理内存，SWAP 等于内存大小。	
		8-64G 的物理内存，SWAP 设置为8G。	
		64-256G物理内存，SWAP 设置为16G。
```

##### 2.4释放Swap分区空间

```sh
	1.查看swap分区的名称
	swapon
	NAME      TYPE      SIZE  USED PRIO
	/dev/dm-1 partition   2G 12.5M   -2
	2.关闭交换分区
	swapoff	/dev/dm-1
	3.启用交换分区
	swapon	/dev/dm-1
```

##### 2.5swap分区空间什么时候使用

```sh
	1.Linux通过一个参数swappiness来控制的
	2.当然还涉及到复杂的算法
	3.这个参数值可为 0-100，控制系统 swap  的使用程度。高数值可优先系统性能，在进程不活跃时主动将其转换出物理内存。低数值可优先互动性并尽量避免将进程转换处物理内存，并降低反应延迟。默认值为   60。注意：这个只是一个权值，不是一个百分比值，涉及到系统内核复杂的算法
	4.关于swappiness的相关英文资料
	The  Linux 2.6 kernel added a new kernel parameter called swappiness to let  administrators tweak the way Linux swaps. It is a number from 0 to 100.  In essence, higher values lead to more pages being swapped, and lower  values lead to more applications being kept in memory, even if they are  idle. Kernel maintainer Andrew Morton has said that he runs his desktop  machines with a swappiness of 100, stating that "My point is that  decreasing the tendency of the kernel to swap stuff out is wrong. You  really don't want hundreds of megabytes of BloatyApp's untouched memory  floating about in the machine. Get it out on the disk, use the memory  for something useful."

Swappiness is a property of the  Linux kernel that changes the balance between swapping out runtime  memory, as opposed to dropping pages from the system page cache.  Swappiness can be set to values between 0 and 100 inclusive. A low value  means the kernel will try to avoid swapping as much as possible where a  higher value instead will make the kernel aggressively try to use swap  space. The default value is 60, and for most desktop systems, setting it  to 100 may affect the overall performance, whereas setting it lower  (even 0) may improve interactivity (by decreasing response latency.
	5.swappiness值得设置参考
	  #60改为10,这可以大大降低系统对于swap的写入，建议内存为512m或更多的朋友采用此方法
	  #你发现你对于swap的使用极少，可 以将值设为0
	  #1G内存，将此值设为5是最合适
	6.临时修改swappiness参数方法，系统重启后失效
         1.[root@vm1 ~]# more /proc/sys/vm/swappiness
					30
           [root@vm1 ~]# echo 10 > /proc/sys/vm/swappiness
         2.[root@vm1 ~]# sysctl vm.swappiness=10
	7.永久修改swappiness的值
		在配置文件/etc/sysctl.conf里面修改vm.swappiness的值，然后重启系统
		echo 'vm.swappinsess=10' >> /etc/sysctl.conf  
```

##### 2.6调整Swap分区的大小

```sh
1.关闭交换分区
	swapoff /dev/mapper/centos-swap 
2.查看
	swapon -s
3.减小swap分区的逻辑卷到1.0G
  [root@vm1 ~]#	lvreduce -L 1.0G /dev/mapper/centos-swap
  	WARNING: Reducing active logical volume to 1.00 GiB.
    THIS MAY DESTROY YOUR DATA (filesystem etc.)
  Do you really want to reduce centos/swap? [y/n]: y
    Size of logical volume centos/swap changed from 2.00 GiB (512 extents) to 1.00 GiB (256 extents).
    Logical volume centos/swap successfully resized.
  #如果要扩增，先查对应的卷组内存够不够扩增，不够的话先扩增卷组，之后再扩增逻辑卷
  	lvextend -L 1.8G /dev/mapper/centos-swap
 4.格式化
 	[root@vm1 ~]# mkswap /dev/mapper/centos-swap
     mkswap: /dev/mapper/centos-swap: warning: wiping old swap signature.
     Setting up swapspace version 1, size = 1048572 KiB
     no label, UUID=9aa45639-484f-45f4-a05e-4ab90af5ce0c
 5.启动swap分区并增加到/etc/fstab自动挂载
 	[root@vm1 ~]# swapon /dev/mapper/centos-swap
 	[root@vm1 ~]# swapon -s
 	Filename				Type		Size	Used	Priority
 	/dev/dm-1                 partition	  1048572	0	   -2
```











