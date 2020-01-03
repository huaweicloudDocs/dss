# Linux磁盘扩容后处理（parted）<a name="ZH-CN_TOPIC_0170873723"></a>

## 操作场景<a name="sf157dc33debd4bf092e82bda15eb9735"></a>

扩容成功后，对于linux操作系统而言，需要将扩容部分的容量划分至原有分区内，或者为扩容部分的磁盘分配新的分区。

本文以“EulerOS 2.0 64位”操作系统为例，采用parted分区工具为扩容后的磁盘分配分区。

不同操作系统的操作可能不同，本文仅供参考，具体操作步骤和差异请参考对应操作系统的产品文档。

为扩容后的磁盘分配分区，您可以根据业务需要以及实际的磁盘情况选择以下两种扩容方式，具体如下：

-   不中断业务，新增分区

    为扩容后的磁盘增加新的分区，不需要卸载原有分区，相比替换原有分区的方法，对业务影响较小。推荐系统盘或者需要保证业务不中断的磁盘扩容场景使用。

    如果当前磁盘使用的是MBR分区形式，则此时要求扩容后的数据盘最大容量为2 TB，并且磁盘的分区数量还未达到上限。

-   中断业务，替换原有分区

    如果当前磁盘使用的是MBR分区形式，并且磁盘的分区数量已经达到上限，则此时需要替换原有分区，替换原有分区不会删除原有分区的数据，但是需要先卸载原有分区，会影响线上业务运行。

    如果当前磁盘使用的是MBR分区形式，并且扩容后磁盘容量已经超过2 TB，则超过2 TB的部分容量无法使用。此时若需要使用超过2 TB的部分容量，则必须将MBR分区形式换为GPT，更换磁盘分区形式时会清除磁盘的原有数据，请先对数据进行备份。


## 前提条件<a name="s84e1b667f6fe4194b86bfa3437edbd94"></a>

-   已登录云服务器。
    -   裸金属服务器请参见《裸金属服务器用户指南》。
    -   弹性云服务器请参见[登录弹性云服务器](https://support.huaweicloud.com/qs-ecs/zh-cn_topic_0092494193.html)。
    -   裸金属服务器请参见[登录裸金属服务器](https://support.huaweicloud.com/qs-bms/zh-cn_topic_0140737435.html)。

-   已挂载磁盘至云服务器，且该磁盘的扩容部分未分配分区。

## 查看分区形式<a name="se39f7f7cdf9440f2b702ee46b67d44ef"></a>

分区前，需要查看当前磁盘的分区形式，当为MBR时可以选择fdisk或者parted工具，当为GPT时需要使用parted工具。

1.  执行以下命令，查看磁盘情况。

    **lsblk**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# lsblk
    NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    xvda    202:0    0   40G  0 disk 
    └─xvda1 202:1    0   40G  0 part /
    xvdb    202:16   0  150G  0 disk 
    ├─xvdb1 202:17   0  100G  0 part /mnt/sdc
    └─xvdb2 202:18   0   50G  0 part /mnt/opt
    xvdc    202:32   0   40G  0 disk 
    ├─xvdc1 202:33   0    8G  0 part 
    └─xvdc2 202:34   0   32G  0 part 
    ```

2.  <a name="lb5bc8e5ebbdf4ced91ec501814c135cf"></a>执行以下命令，然后输入“p”，查看当前数据盘的分区形式。

    **parted** _磁盘_

    以查看“/dev/xvdb”的分区形式为例：

    **parted /dev/xvdb**

    回显类似如下信息：

    ```
    root@ecs-1120 linux]# parted /dev/xvdb
    GNU Parted 3.1
    Using /dev/xvdb
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) p                                                                
    Model: Xen Virtual Block Device (xvd)
    Disk /dev/xvdb: 161GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags: 
    
    Number  Start   End    Size    File system  Name  Flags
     1      17.4kB  107GB  107GB   ext4         opt
     2      107GB   161GB  53.7GB  ext4         opt1
    ```

    “Partition Table”表示当前磁盘的分区形式，msdos表示磁盘分区形式为MBR，gpt表示磁盘分区形式为GPT。

3.  <a name="lcf9289cd94da4c1f8d28a33b482a34a6"></a>查看完成后，输入“q”，退出parted模式。
4.  参考[2](#lb5bc8e5ebbdf4ced91ec501814c135cf)\~[3](#lcf9289cd94da4c1f8d28a33b482a34a6)，查看其它磁盘的分区形式。

## 新增分区<a name="s2764a24543c04c6fa92c88a5f02dc793"></a>

本操作以该场景为例，为系统盘扩容后的空间分配一个新的分区，并挂载到“/opt”目录下，此时可以不中断业务。

1.  执行以下命令，查看磁盘的分区信息。

    **lsblk**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# lsblk
    NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    xvda    202:0    0   80G  0 disk 
    └─xvda1 202:1    0   40G  0 part /
    xvdb    202:16   0  250G  0 disk 
    ├─xvdb1 202:17   0  100G  0 part 
    └─xvdb2 202:18   0   50G  0 part 
    xvdc    202:32   0   40G  0 disk 
    ├─xvdc1 202:33   0    8G  0 part 
    └─xvdc2 202:34   0   32G  0 part  
    ```

    表示当前系统盘“dev/xvda”容量为80 GB，当前正在使用的分区“dev/xvda1”为40 GB，新扩容的40 GB还未分配分区。

2.  执行以下命令，进入parted分区工具，开始对系统盘的新扩容空间分配分区。

    **parted** _系统盘_

    以“/dev/xvda”为例：

    **parted /dev/xvda**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# parted /dev/xvda
    GNU Parted 3.1
    Using /dev/xvda
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    ```

3.  输入“unit s”，按“Enter”，设置磁盘的计量单位为磁柱。
4.  输入“p”，按“Enter”，查看当前磁盘分区形式。

    回显类似如下信息：

    ```
    (parted) unit s                                                   
    (parted) p                                                        
    Model: Xen Virtual Block Device (xvd)
    Disk /dev/xvda: 167772160s
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags: 
    
    Number  Start  End        Size       Type     File system  Flags
     1      2048s  83886079s  83884032s  primary  ext4
    ```

5.  新增分区，输入“mkpart”，按“Enter”。
6.  以新增一个主分区为例，输入“p”，按“Enter”。

    回显类似如下信息：

    ```
    (parted) mkpart
    Partition type?  primary/extended? p
    File system type?  [ext2]? ext4                                   
    Start? 83886080
    End? 1677722159         
    ```

7.  设置文件系统格式以及新增分区的容量大小。

    磁柱“_83886080_”表示新增分区“dev/xvda2”磁柱初始值，“_167772159_”表示截止磁柱值，此处仅供参考，您可以根据业务需要自行规划磁盘分区数量及容量。

    回显类似如下信息：

    ```
    (parted) mkpart
    Partition type?  primary/extended? p
    File system type?  [ext2]? ext4                                   
    Start? 83886080
    End? 1677722159         
    ```

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >此处为新建分区设置文件系统格式的操作可能无效，请在分区创建完成后参考[10](#l35e830033fca4082bcd22552a4331fb4)重新设置文件系统格式。  
    >获取最大截止磁柱值的方法如下：  
    >-   通过**fdisk -l**命令查询磁盘的最大截止磁柱值。  
    >-   可以输入**-1s**，即默认为磁盘的最大截止磁柱值。  

8.  输入“p”，按“Enter”，查看新建分区。

    回显类似如下信息：

    ```
    (parted) p                                                        
    Model: Xen Virtual Block Device (xvd)
    Disk /dev/xvda: 167772160s
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Disk Flags: 
    
    Number  Start      End         Size       Type     File system  Flags
     1      2048s      83886079s   83884032s  primary  ext4
     2      83886080s  167772159s  83886080s  primary
    ```

    新增分区“dev/xvda2”创建完成。

9.  输入“q”，按“Enter”，退出parted分区工具。
10. <a name="l35e830033fca4082bcd22552a4331fb4"></a>执行以下命令，设置新建分区文件系统格式。

    以“ext4” 文件格式为例：

    **mkfs -t ext4 /dev/xvda2**

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >设置xfs文件系统的操作与ext3或ext4一样，命令为：**mkfs -t xfs  **/dev/xvda2****  

    回显类似如下信息：

    ```
    [[root@ecs-1120 linux]# mkfs -t ext4 /dev/xvda2
    mke2fs 1.42.9 (28-Dec-2013)
    Filesystem label=
    OS type: Linux
    Block size=4096 (log=2)
    Fragment size=4096 (log=2)
    Stride=0 blocks, Stripe width=0 blocks
    2621440 inodes, 10485760 blocks
    524288 blocks (5.00%) reserved for the super user
    First data block=0
    Maximum filesystem blocks=2157969408
    320 block groups
    32768 blocks per group, 32768 fragments per group
    8192 inodes per group
    Superblock backups stored on blocks: 
    ?32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
    ?4096000, 7962624
    
    Allocating group tables: done                            
    Writing inode tables: done                            
    Creating journal (32768 blocks): done
    Writing superblocks and filesystem accounting information: done   
    
    ```

    格式化需要等待一段时间，请观察系统运行状态，若回显中进程提示为done，则表示格式化完成。

11. 执行以下命令，将新建分区挂载到需要增加空间的目录下，以“/opt”为例。

    **mount /dev/xvda6 /opt**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# mount /dev/xvda2 /opt
    [root@ecs-1120 linux]# 
    ```

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >新增加的分区挂载到不为空的目录时，该目录下原本的子目录和文件会被隐藏，所以，新增的分区最好挂载到空目录或者新建目录。如确实要挂载到不为空的目录，可将该目录下的子目录和文件临时移动到其他目录下，新分区挂载成功后，再将子目录和文件移动回来。  

12. 执行以下命令，查看挂载结果。

    **df -TH**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# df -TH
    Filesystem     Type      Size  Used Avail Use% Mounted on
    /dev/xvda1     ext4       43G  8.3G   33G  21% /
    devtmpfs       devtmpfs  885M     0  885M   0% /dev
    tmpfs          tmpfs     894M     0  894M   0% /dev/shm
    tmpfs          tmpfs     894M   18M  877M   2% /run
    tmpfs          tmpfs     894M     0  894M   0% /sys/fs/cgroup
    tmpfs          tmpfs     179M     0  179M   0% /run/user/2000
    tmpfs          tmpfs     179M     0  179M   0% /run/user/0
    tmpfs          tmpfs     179M     0  179M   0% /run/user/1001
    /dev/xvda2     ext4       43G   51M   40G   1% /opt
    ```


## 替换原有分区<a name="s5b1a124212e24ae296b3a6954b5ebdd1"></a>

本操作以该场景为例，云服务器上已挂载两块磁盘，磁盘“/dev/xvdc”有1个分区，其中分区“/dev/xvdc1”已挂载至“/mnt/sdc”目录下，现在需要替换原有分区“/dev/xvdc1”，由于只有一个分区，因此该分区也算作末尾分区。将新增容量加到该分区内，此时需要中断业务。

>![](public_sys-resources/icon-notice.gif) **须知：**   
>扩容后的新增空间是添加在磁盘末尾的，对具有多个分区的的磁盘扩容时，只支持替换排在末尾的分区。  

1.  执行以下命令，查看磁盘的分区信息。

    **lsblk**

    回显类似如下信息：

    ```
    [root@ecs-1120 sdc]# lsblk
    NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    xvda    202:0    0   80G  0 disk 
    ├─xvda1 202:1    0   40G  0 part /
    └─xvda2 202:2    0   40G  0 part /opt
    xvdb    202:16   0  350G  0 disk 
    ├─xvdb1 202:17   0  100G  0 part 
    └─xvdb2 202:18   0  200G  0 part
    xvdc    202:32   0   60G  0 disk 
    └─xvdc1 202:33   0    10G  0 part /mnt/sdc
    ```

    表示当前数据盘“/dev/xvdc“总容量为60 GB，已分配分区的容量为10 GB，其中末尾分区为“/dev/xvdc1”，为已挂载至“/mnt/sdc”目录下。

    查看回显中磁盘“/dev/xvdc“的容量，扩容的容量是否已经包含在容量总和中。

    -   若扩容的容量未在磁盘容量总和中，请参考[Linux SCSI数据盘扩容后处理（fdisk）](Linux-SCSI数据盘扩容后处理（fdisk）.md)章节刷新系统内容量。
    -   若扩容的容量已在磁盘容量总和中，请执行[2](#l71f4c979d39e42c48983a5396c5b0a49)。

2.  <a name="l71f4c979d39e42c48983a5396c5b0a49"></a>执行以下命令，卸载磁盘分区。

    **umount /mnt/sdc**

3.  执行以下命令，查看“/dev/xvdc”分区的卸载结果。

    **lsblk**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# umount /mnt/sdc
    [root@ecs-1120 linux]# lsblk
    NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    xvda    202:0    0   80G  0 disk 
    ├─xvda1 202:1    0   40G  0 part /
    └─xvda2 202:2    0   40G  0 part /opt
    xvdb    202:16   0  350G  0 disk 
    ├─xvdb1 202:17   0  100G  0 part 
    └─xvdb2 202:18   0  200G  0 part
    xvdc    202:32   0   60G  0 disk 
    └─xvdc1 202:33   0    10G  0 part
    ```

4.  执行以下命令，进入parted分区工具，开始对数据盘的新扩容空间分配分区。

    **parted** _数据盘_

    以“/dev/xvdc”为例：

    **parted /dev/xvdc**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# parted /dev/xvdc
    GNU Parted 3.1
    Using /dev/xvdc
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    ```

5.  输入“unit s”，按“Enter”，设置磁盘的计量单位为磁柱。
6.  <a name="lab2ce9cab68d4d2bba4c64c9082f49f4"></a>以新增一个主分区为例，输入“p”，按“Enter”。

    回显类似如下信息：

    ```
    (parted) mkpart
    Partition type?  primary/extended? p
    File system type?  [ext2]? ext4                                   
    Start? 83886080
    End? 1677722159         
    ```

7.  删除待替换的末尾分区“/dev/xvdc1”，分区编号为“1”，输入“rm 1”，按“Enter”。
8.  输入“p”，按“Enter”，查看当前“/dev/xvdc1”分区是否删除成功。

    回显类似如下信息：

    ```
    (parted) rm 1
    (parted) p                                                                
    Model: Xen Virtual Block Device (xvd)
    Disk /dev/xvdc: 125829120s
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags: 
    
    Number  Start  End         Size        File system  Name  Flags
    ```

9.  重新划分分区，输入“mkpart opt  _2048s_ _125829119_”，按“Enter”。

    “_2048_”为[6](#lab2ce9cab68d4d2bba4c64c9082f49f4)中记录的初始磁柱值，“125829119”表示截止磁柱值，应该大于等于[6](#lab2ce9cab68d4d2bba4c64c9082f49f4)中记录的截止磁柱值。

    回显类似如下信息：

    ```
    (parted) mkpart opt 2048s 125829119s
    Warning: You requested a partition from 2048s to 125829199s (sectors 2048..125829199).
    The closest location we can manage is 2048s to 125829036s (sectors 2048..125829036).
    Is this still acceptable to you?
    Yes/No? Yes
    ```

    根据系统提示输入“Yes”，设置截止磁柱值。

    若出现以下性能优化提示，请输入“Ignore”，忽视即可。如果已经是最优性能，则不会出现该提示，本操作中性能最优的初始磁柱值即为2048s，因此系统没有该提示。

    ```
    Warning: The resulting partition is not properly aligned for best performance.
    Ignore/Cancel? Ignore  
    ```

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >以下操作会导致数据丢失：  
    >-   选择的初始磁柱值与原分区的不一致。  
    >-   选择的截止磁柱值小于原分区的值。  

10. 输入“p”，按“Enter”，查看当前“/dev/xvdc1”分区是否替换成功。

    回显类似如下信息：

    ```
    (parted) p                                                                
    Model: Xen Virtual Block Device (xvd)
    Disk /dev/xvdb: 125829120s
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags: 
    
    Number  Start       End         Size        File system  Name  Flags
     1      2048s       125829086s  125827039s  ext4         opt
    ```

    表示“/dev/xvdc1”分区替换成功。

11. 输入“q”，按“Enter”，退出parted分区工具。
12. 根据磁盘的文件系统，分别执行以下操作。
    -   若磁盘文件系统为ext3或ext4，请执行以下步骤。
        1.  执行以下命令，检查“/dev/xvdc1”文件系统的正确性。

            **e2fsck -f /dev/xvdc1**

            回显类似如下信息：

            ```
            [root@ecs-1120 linux]# e2fsck -f /dev/xvdb2
            e2fsck 1.42.9 (28-Dec-2013)
            Pass 1: Checking inodes, blocks, and sizes
            Pass 2: Checking directory structure
            Pass 3: Checking directory connectivity
            Pass 4: Checking reference counts
            Pass 5: Checking group summary information
            /dev/xvdc1: 11/655360 files (0.0% non-contiguous), 83137/2620928 blocks
            ```

        2.  执行以下命令，扩展“/dev/xvdc1”文件系统的大小。

            **resize2fs /dev/xvdc1**

            回显类似如下信息：

            ```
            [root@ecs-1120 linux]# resize2fs /dev/xvdc1
            resize2fs 1.42.9 (28-Dec-2013)
            Resizing the filesystem on /dev/xvdc1 to 15728379 (4k) blocks.
            The filesystem on /dev/xvdc1 is now 15728379 blocks long.
            ```

        3.  执行以下命令，查看替换分区后数据盘的情况。

            **lsblk**

            回显类似如下信息：

            ```
            [root@ecs-1120 linux]# lsblk
            NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
            NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
            xvda    202:0    0   80G  0 disk 
            ├─xvda1 202:1    0   40G  0 part /
            └─xvda2 202:2    0   40G  0 part /opt
            xvdb    202:16   0  350G  0 disk 
            ├─xvdb1 202:17   0  100G  0 part 
            └─xvdb2 202:18   0  200G  0 part
            xvdc    202:32   0   60G  0 disk 
            └─xvdc1 202:33   0    60G  0 part
            ```

            表示当前“/dev/xvdc“总容量为60 GB，新增的50GB已经划分在“/dev/xvdc1”分区内。

        4.  执行以下命令，将新建分区挂载到“/mnt/sdc”目录下。

            **mount /dev/xvdc1 /mnt/sdc**

    -   若磁盘文件系统为xfs，请执行以下步骤。
        1.  执行以下命令，将新建分区挂载到“/mnt/sdc”目录下。

            **mount /dev/xvdc1 /mnt/sdc**

        2.  执行以下命令，扩展“/dev/xvdc1”文件系统的大小。

            **sudo** **xfs\_growfs** **/dev/xvdc1**

        3.  执行以下命令，查看替换分区后数据盘的情况。

            **lsblk**

            回显类似如下信息：

            ```
            [root@ecs-1120 linux]# lsblk
            NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
            NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
            xvda    202:0    0   80G  0 disk 
            ├─xvda1 202:1    0   40G  0 part /
            └─xvda2 202:2    0   40G  0 part /opt
            xvdb    202:16   0  350G  0 disk 
            ├─xvdb1 202:17   0  100G  0 part 
            └─xvdb2 202:18   0  200G  0 part
            xvdc    202:32   0   60G  0 disk 
            └─xvdc1 202:33   0    60G  0 part
            ```

            表示当前“/dev/xvdc“总容量为60 GB，新增的50GB已经划分在“/dev/xvdc1”分区内。


13. 行以下命令，查看“/dev/xvdc1”分区挂载结果。

    **df -TH**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# mount /dev/xvdc1 /mnt/sdc
    [root@ecs-1120 linux]# df -TH
    Filesystem     Type      Size  Used Avail Use% Mounted on
    /dev/xvda1     ext4       43G  8.3G   33G  21% /
    devtmpfs       devtmpfs  885M     0  885M   0% /dev
    tmpfs          tmpfs     894M     0  894M   0% /dev/shm
    tmpfs          tmpfs     894M   18M  877M   2% /run
    tmpfs          tmpfs     894M     0  894M   0% /sys/fs/cgroup
    tmpfs          tmpfs     179M     0  179M   0% /run/user/2000
    tmpfs          tmpfs     179M     0  179M   0% /run/user/0
    tmpfs          tmpfs     179M     0  179M   0% /run/user/1001
    /dev/xvda2     ext4       43G   51M   40G   1% /opt
    /dev/xvdc1     ext4       64G   55M   60G   1% /mnt/sdc
    ```

    表示“/dev/xvdc1”已挂载至“/mnt/sdc”目录下。


## 设置开机自动挂载磁盘<a name="s86e1a6cb2ed44c4896a18f152abba2ca"></a>

如果您需要在云服务器系统启动时自动挂载磁盘，不能采用在 /etc/fstab直接指定 /dev/xvdb1的方法，因为云中设备的顺序编码在关闭或者开启云服务器过程中可能发生改变，例如/dev/xvdb1可能会变成/dev/xvdb2。推荐使用UUID来配置自动挂载数据盘。

>![](public_sys-resources/icon-note.gif) **说明：**   
>磁盘的UUID（universally unique identifier）是Linux系统为磁盘分区提供的唯一的标识字符串。  

1.  <a name="l86d9aafae3f94c9cbc94e935cdc779c0"></a>执行如下命令，查询磁盘分区的UUID。

    **blkid** _磁盘分区_

    以查询磁盘分区“/dev/xvdb1“的UUID为例：

    **blkid /dev/xvdb1**

    回显类似如下信息：

    ```
    [root@ecs-b656 test]# blkid /dev/xvdb1
    /dev/xvdb1: UUID="1851e23f-1c57-40ab-86bb-5fc5fc606ffa" TYPE="ext4"
    ```

    表示“/dev/xvdb1“的UUID。

2.  执行以下命令，使用VI编辑器打开“fstab”文件。

    **vi /etc/fstab**

3.  按“i”，进入编辑模式。
4.  将光标移至文件末尾，按“Enter”，添加如下内容。

    ```
    UUID=1851e23f-1c57-40ab-86bb-5fc5fc606ffa /mnt/sdc      ext3 defaults     0   2
    ```

    ```
    UUID=1851e23f-1c57-40ab-86bb-5fc5fc606ffa /mnt/sdc      ext4 defaults     0   2
    ```

    以内容上仅为示例，具体请以实际情况为准，参数说明如下：

    -   第一列为UUID，此处填写[1](#l86d9aafae3f94c9cbc94e935cdc779c0)中查询到的磁盘分区的UUID。
    -   第二列为磁盘分区的挂载目录，可以通过**df -TH**命令查询。
    -   第三列为磁盘分区的文件系统格式， 可以通过**df -TH**命令查询。
    -   第四列为磁盘分区的挂载选项，此处通常设置为defaults即可。
    -   第五列为Linux dump备份选项。
        -   0表示不使用Linux dump备份。现在通常不使用dump备份，此处设置为0即可。
        -   1表示使用Linux dump备份。

    -   第六列为fsck选项，即开机时是否使用fsck检查磁盘。
        -   0表示不检验。
        -   挂载点为（/）根目录的分区，此处必须填写1。

            根分区设置为1，其他分区只能从2开始，系统会按照数字从小到大依次检查下去。


5.  按“ESC”后，输入“:wq”，按“Enter”。

    保存设置并退出编辑器。


