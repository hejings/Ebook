# 管理逻辑卷管理（LVM）存储



# 实施LVM存储

## 一、创建逻辑卷

### 1.准备物理设备

* 使用`fdisk`、`gdisk`或`parted`创建新分区，以便于`LVM`结合使用。在`LVM`分区上始终将分区类型设置成`Linux LVM`；对于MBR式分区，使用0x8e。如有必要，使用partprobe向内核注册新分区。

* 也可以使用完整磁盘、`RAID`阵列或`SAN`磁盘

* 只有当没有已准备好的物理设备并且需要新物理设备来创建或扩展组件时，才需要准备物理设备。

```shell
[root@centos7 ~]# fdisk /dev/sdb
[root@centos7 ~]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0   30G  0 disk 
├─sda1            8:1    0    1G  0 part /boot
└─sda2            8:2    0   29G  0 part 
  ├─centos-root 253:0    0   27G  0 lvm  /
  └─centos-swap 253:1    0    2G  0 lvm  [SWAP]
sdb               8:16   0  100G  0 disk 
├─sdb1            8:17   0   10G  0 part 
└─sdb2            8:18   0   10G  0 part 
sr0              11:0    1 1024M  0 rom  
```

### 2.创建物理卷

* 使用`pvcreate`为分区（或其他物理设备）添加标签，使其作为物理卷与`LVM`结合使用。会将用于存储`LVM`配置数据的一个标头直接写入到`PV`。`PV`分为多个固定大小的物理范围（PE）；例如4MiB的快。使用以空格分隔的设备名称作为`pvcreate`的参数，同时标记多个设备。

```shell
[root@centos7 ~]# pvcreate /dev/sdb1 /dev/sdb2
  Physical volume "/dev/sdb1" successfully created.
  Physical volume "/dev/sdb2" successfully created.
[root@centos7 ~]# pvs
  PV         VG     Fmt  Attr PSize   PFree 
  /dev/sda2  centos lvm2 a--  <29.00g  4.00m
  /dev/sdb1         lvm2 ---   10.00g 10.00g
  /dev/sdb2         lvm2 ---   10.00g 10.00g

```

* 此命令会将设备`/dev/sdb1`和`/dev/sdb2`标记为`PV`，准备好分配到卷组中
* 仅当没有空闲的`PV`可以创建或扩展`VG`时，才需要创建`PV`

### 3.创建卷组

* `vgcreate`用于创建包含一个多个物理卷的池，称为卷组。`VG`的大小由池中物理范围的总数决定。`VG`负责通过向`LV`分配空闲`PE`来托管一个或多个逻辑卷；因此，在创建`LV`时，`VG`必须具有足够的空闲`PE`可用。
* 以`vgcreate`的参数形式，定义`VG`名称并列出一个或多个要分配给`VG`的`PV` 

```shell
[root@centos7 ~]# vgcreate vg-alpha /dev/sdb1 /dev/sdb2
  Volume group "vg-alpha" successfully created
[root@centos7 ~]# vgs
  VG       #PV #LV #SN Attr   VSize   VFree 
  centos     1   2   0 wz--n- <29.00g  4.00m
  vg-alpha   2   0   0 wz--n-  19.99g 19.99g

```

* 此命令将创建名为`vg-alpha`的`VG`,他的大小是`/dev/sdb1`和`/dev/sdb2`这两个`PV`的总和（以PE单位计）
* 仅当没有`VG`时才需要创建`VG`,可能会出于管理原因创建额外的`VG`，用于管理`PV`和`LV`的使用。否则，可在需要时扩展现有的`VG`以容纳新的`LV`

### 4.创建逻辑卷

* `lvcreate`根据卷组中可用的物理范围创建新的逻辑卷。至少为`lvcreate`使用以下参数：

  使用`-n`选项设置`LV`名称，使用`-L`选项设置`LV`大小（以字节为单位），并确定要在其中创建`LV`的`VG`名称

```shell
[root@centos7 ~]# lvcreate -n hercules -L 2G vg-alpha 
  Logical volume "hercules" created.
[root@centos7 ~]# lvs
  LV       VG       Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root     centos   -wi-ao---- 26.99g                                                    
  swap     centos   -wi-ao----  2.00g                                                    
  hercules vg-alpha -wi-a-----  2.00g  
```

* 此命令将在`VG` `vg-alpha`中创建一个名为`hercules`的`LV`，其大小时2GiB。必须有足够的空闲物理范围来分配2GiB，如有必要，会将其取整为`PE`单元大小的倍数

* 有很多种方式可以指定大小：

  -L 要求以字节或更大指定值为单位的大小，例如，兆字节（二进制兆字节，1048576字节）和千兆字节（二进制千兆字节）。

  -l 选项要求以物理范围数进行衡量的大小

* 实例：

  `lvcreate -L 128M`  #将逻辑卷的大小确定为正好128MiB

  `lvcreate -l 128`   #将逻辑卷的大小确定为正好128个范围的大小。字节总数取决于基础物理卷上物理范围块的大小

* 重要：

  不同的工具将使用传统名称`/dev/vgname/lvname`或内核设备映射程序名`/dev/mapper/vgname-lvname`

### 5.添加文件系统

使用`mkfs`在新逻辑卷上创建`xfs`文件系统。或者,选用其他文件系统`ext2/3/4`

```shell
[root@centos7 ~]# mkfs -t xfs /dev/vg-alpha/hercules
meta-data=/dev/vg-alpha/hercules isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

```

要使文件系统在重新启动后依然可以使用：

* 使用`mkdir`创建挂载点目录

```shell
[root@centos7 ~]# mkdir /mnt/hercules
```

* 向`/etc/fstab`文件添加条目

```shell
/dev/vg-alpha/hercules /mnt/hercules xfs defaults 0 0
```

* 运行`mount -a`检测配置正确性

```shell
[root@centos7 ~]# mount -a
```

## 二、删除逻辑卷

### 删除所有逻辑卷组件需要四个步骤

### 1.准备文件系统

* 将必须保留的所有数据移动到另一个文件系统，然后使用`umount`卸载该文件系统。记得删除`/etc/fstab`中相应的条目

```shell
[root@centos7 ~]# umount /mnt/hercules/
```

* `警告：`

​			删除逻辑卷将会破坏该逻辑卷上存储的是所有数据。删除之前要备份或移动走！！！

### 2.删除逻辑卷

* 使用`lvremove`删除不再需要的逻辑卷。使用设备名称作为参数

```shell
[root@centos7 ~]# lvs
  LV       VG       Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root     centos   -wi-ao---- 26.99g                                                    
  swap     centos   -wi-ao----  2.00g                                                    
  hercules vg-alpha -wi-a-----  2.00g                                                    

[root@centos7 ~]# lvremove /dev/vg-alpha/hercules 
Do you really want to remove active logical volume vg-alpha/hercules? [y/n]: y
  Logical volume "hercules" successfully removed

```

* 运行此命令前，必须卸载`LV`文件系统。删除`LV`之前，将请求确认
* `LV`的物理范围将会被释放，并可用于分配给卷组中现有额`LV`或新`LV`

### 3.删除卷组

* 使用`vgremove`删除不再需要的卷组，使用`VG`名称作为参数

```shell
[root@centos7 ~]# vgs
  VG       #PV #LV #SN Attr   VSize   VFree 
  centos     1   2   0 wz--n- <29.00g  4.00m
  vg-alpha   2   0   0 wz--n-  19.99g 19.99g
[root@centos7 ~]# vgremove vg-alpha 
  Volume group "vg-alpha" successfully removed

```

* `VG`的物理卷将会被释放，并可用于分配给系统中现有的`VG`或新`VG`

### 4.删除物理卷

* 使用`pvremove`删除不再需要的物理卷、使用空格分隔的PV设备列表同时删除多个PV。PV元数据将会从分区（或磁盘）清楚。分区现已空闲，可重新分配或重新格式化

```shell
[root@centos7 ~]# pvs
  PV         VG     Fmt  Attr PSize   PFree 
  /dev/sda2  centos lvm2 a--  <29.00g  4.00m
  /dev/sdb1         lvm2 ---   10.00g 10.00g
  /dev/sdb2         lvm2 ---   10.00g 10.00g
[root@centos7 ~]# pvremove /dev/sdb{1,2}
  Labels on physical volume "/dev/sdb1" successfully wiped.
  Labels on physical volume "/dev/sdb2" successfully wiped.

```

## 三、查看`LVM`状态信息

### 1.物理卷

* 使用`pvdisplay` 显示有关物理卷的信息。如果未随命令指定任何参数，则它将列出有关系统上所有的`PV`信息，如果参数未特定设备名称，则将仅显示特定`PV`的信息

```shell
[root@centos7 ~]# pvdisplay /dev/sdb1
  --- Physical volume ---
  PV Name               /dev/sdb1  #PV name映射到设备名称
  VG Name               alpha  # 显示将PV分配到的卷组
  PV Size               10.00 GiB / not usable 4.00 MiB #显示PV的物理大小，包括任何不可用的空间
  Allocatable           yes 
  PE Size               4.00 MiB  #见下说明
  Total PE              2559
  Free PE               2047 #见下说明
  Allocated PE          512
  PV UUID               hvQN7r-XJcv-pGwz-78qQ-zLJE-g0Ie-PjbTjE
   
```

* `PE size ` 说明：

  `PE size`是物理范围大小，它是逻辑卷中可分配的最小大小

  它也是计算以`PE`单位报告的任何值（如Free PE）的大小时的倍数；例如：26个`PE` x 4MiB(PE Size)可提供104MiB可用空间。逻辑卷大小将取整为`PE`单位的倍数

  `LVM`会自动设置`PE`大小，默认是4MiB，但也可以指定其大小

* `Free PE`说明

  显示有多少`PE`单位可用于分配给新逻辑卷

### 2.卷组

* 使用`vgdisplay`显示有关卷组的信息，如果没有为命令提示任何变量，则将显示机器上所有的`VG`信息。使用`VG`名称作为变量将仅显示特定`VG`信息

```shell
[root@centos7 ~]# vgdisplay alpha 
  --- Volume group ---
  VG Name               alpha  #此卷组的名称
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  4
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               19.99 GiB  #存储池，可用于逻辑卷分配的总大小
  PE Size               4.00 MiB 
  Total PE              5118 #以PE单位表示的总大小
  Alloc PE / Size       512 / 2.00 GiB #已经分配了多大给LV，512*4=2048MiB
  Free  PE / Size       4606 / 17.99 GiB #显示VG中有多少空闲空间可用于分配给新LV或扩展现有LV
  VG UUID               VXWaiK-Bz30-jeoO-aNEA-ki5h-ppWF-pJObUz

```

3.逻辑卷

* 使用`pvdisplay`显示有关逻辑卷的信息。同样，不加参数将显示机器所有逻辑卷信息。而使用`LV`设备名称作为参数将显示特定信息

```shell
[root@centos7 ~]# lvdisplay /dev/alpha/hercules 
  --- Logical volume ---
  LV Path                /dev/alpha/hercules #显示此逻辑卷的设备名称，有些工具可能显示/dev/mapper/vgname-lvname,表示的是同一LV
  LV Name                hercules
  VG Name                alpha  #显示从那个卷组分配的LV
  LV UUID                wwKnsm-bBUA-vx2n-5tCy-ZCPa-cVlk-KkLYFm
  LV Write Access        read/write
  LV Creation host, time centos7, 2021-04-10 17:44:37 +0800
  LV Status              available
  # open                 0
  LV Size                2.00 GiB #显示LV总的大小。使用文件系统工具检查可用空间的数据存储的已用空间
  Current LE             512 #显示此LV使用的逻辑范围数。LE通常映射到VG中的物理范围，并因此映射到物理卷。（可理解为512个PE的大小）
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2

```



















