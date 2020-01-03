# Linux系统盘扩容后处理（fdisk）<a name="ZH-CN_TOPIC_0170873683"></a>

## 操作场景<a name="sa50a0c14fa19473d99f4308d55058e1c"></a>

扩容成功后，对于linux操作系统而言，需要将扩容部分的容量划分至原有分区内，或者为扩容部分的磁盘分配新的分区。

本文以“CentOS 7.4 64bit”操作系统为例，采用fdisk分区工具为扩容后的系统盘分配分区。

本手册还提供了其他系统盘新增分区的操作指导，还可以参考以下章节：

-   “CentOS 7.0 64bit”操作系统，使用fdisk工具，[新增分区](Linux磁盘扩容后处理（fdisk）.md#s3a63696e05034a2db04094ab12d767eb)。
-   “CentOS 7.0 64bit”操作系统，使用parted工具，[新增分区](Linux磁盘扩容后处理（parted）.md#s2764a24543c04c6fa92c88a5f02dc793)。

当前云服务器上挂载的系统盘原容量为40GB，已经在管理控制台成功扩容了40GB，当前总容量应为80GB，本操作指导用户为系统盘新增分区。

不同操作系统的操作可能不同，本文仅供参考，具体操作步骤和差异请参考对应操作系统的产品文档。

为扩容后的磁盘分配分区，您可以根据业务需要以及实际的磁盘情况选择以下两种扩容方式，具体如下：

-   不中断业务，新增分区

    为扩容后的磁盘增加新的分区，不需要卸载原有分区，相比替换原有分区的方法，对业务影响较小。推荐系统盘或者需要保证业务不中断的磁盘扩容场景使用。

    如果当前磁盘使用的是MBR分区形式，则此时要求扩容后的数据盘最大容量为2 TB，并且磁盘的分区数量还未达到上限。

-   中断业务，替换原有分区

    如果当前磁盘使用的是MBR分区形式，并且磁盘的分区数量已经达到上限，则此时需要替换原有分区，替换原有分区不会删除原有分区的数据，但是需要先卸载原有分区，会影响线上业务运行。

    如果当前磁盘使用的是MBR分区形式，并且扩容后磁盘容量已经超过2 TB，则超过2 TB的部分容量无法使用。此时若需要使用超过2 TB的部分容量，则必须将MBR分区形式换为GPT，更换磁盘分区形式时会清除磁盘的原有数据，请先对数据进行备份。


## 前提条件<a name="sd431db1125ba4ef4a261d696c2105215"></a>

-   已登录云服务器。
    -   裸金属服务器请参见《裸金属服务器用户指南》。
    -   弹性云服务器请参见[登录弹性云服务器](https://support.huaweicloud.com/qs-ecs/zh-cn_topic_0092494193.html)。
    -   裸金属服务器请参见[登录裸金属服务器](https://support.huaweicloud.com/qs-bms/zh-cn_topic_0140737435.html)。

-   已挂载磁盘至云服务器，且该磁盘的扩容部分未分配分区。

## 新增分区<a name="s639e31b25ce04764b7f7df0fc8d9a89f"></a>

本操作以该场景为例，当前云服务器上挂载的系统盘已有分区“/dev/vda1”，并挂载至“/”目录下，现在为系统盘扩容后的空间分配一个新的分区“/dev/vda2”，并挂载到“/opt”目录下，此时可以不中断业务。

1.  执行以下命令，查看磁盘的分区信息。

    **fdisk -l**

    回显类似如下信息：

    ```
    [root@ecs-2220 ~]# fdisk -l
    
    Disk /dev/vda: 85.9 GB, 85899345920 bytes, 167772160 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0x0008d18f
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/vda1   *        2048    83886079    41942016   83  Linux
    ```

    表示当前系统盘“dev/vda”容量为80 GB，当前正在使用的分区“dev/vda1”为40 GB，新扩容的40 GB还未分配分区。

2.  执行如下命令之后，进入fdisk分区工具。

    **fdisk /dev/vda**

    回显类似如下信息：

    ```
    [root@ecs-2220 ~]# fdisk /dev/vda
    Welcome to fdisk (util-linux 2.23.2).
    
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.
    
    
    Command (m for help): 
    ```

3.  输入“n”，按“Enter”，开始新建分区。

    回显类似如下信息：

    ```
    Command (m for help): n
    Partition type:
       p   primary (1 primary, 0 extended, 3 free)
       e   extended
    ```

    表示磁盘有两种分区类型：

    -   “p”表示主要分区。
    -   “e”表示延伸分区。

4.  以创建一个主要分区为例，输入“p”，按“Enter”，开始创建一个主分区。

    回显类似如下信息：

    ```
    Select (default p): p
    Partition number (2-4, default 2): 
    ```

5.  以分区编号选择“2”为例，输入主分区编号“2”，按“Enter”。

    回显类似如下信息：

    ```
    Partition number (2-4, default 2): 2
    First sector (83886080-167772159, default 83886080):
    ```

6.  输入新分区的起始磁柱编号，如设置默认值，按“Enter”。

    本步骤中使用默认起始磁柱编号为例。

    回显类似如下信息：

    ```
    First sector (83886080-167772159, default 83886080):
    Using default value 83886080
    Last sector, +sectors or +size{K,M,G} (83886080-167772159,default 167772159):
    ```

7.  输入新分区的截止磁柱编号，按“Enter”。

    本步骤中使用默认截止磁柱编号为例。

    回显类似如下信息：

    ```
    Last sector, +sectors or +size{K,M,G} (83886080-167772159,
    default 167772159):
    Using default value 167772159
    Partition 2 of type Linux and of size 40 GiB is set
    
    Command (m for help): 
    ```

8.  输入“p”，按“Enter”，查看新建分区。

    回显类似如下信息：

    ```
    Command (m for help): p
    
    Disk /dev/vda: 85.9 GB, 85899345920 bytes, 167772160 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0x0008d18f
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/vda1   *        2048    83886079    41942016   83  Linux
    /dev/vda2        83886080   167772159    41943040   83  Linux
    Command (m for help): 
    ```

9.  输入“w”，按“Enter”，将分区结果写入分区表中。

    回显类似如下信息：

    ```
    Command (m for help): w
    The partition table has been altered!
    
    Calling ioctl() to re-read partition table.
    
    WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
    The kernel still uses the old table. The new table will be used at
    the next reboot or after you run partprobe(8) or kpartx(8)
    Syncing disks.
    ```

    表示分区创建完成。

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >如果之前分区操作有误，请输入“q”，则会退出fdisk分区工具，之前的分区结果将不会被保留。  

10. 执行以下命令，将新的分区表变更同步至操作系统。

    **partprobe**

11. 执行以下命令，设置新建分区文件系统格式。

    以“ext4” 文件格式为例：

    **mkfs -t ext4 /dev/vda2**

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >设置xfs文件系统的操作与ext3或者ext4一样，命令为：**mkfs -t xfs /dev/vda2**  

    回显类似如下信息：

    ```
    [root@ecs-2220 ~]# mkfs -t ext4 /dev/vda2
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
            32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
            4096000, 7962624
    
    Allocating group tables: done
    Writing inode tables: done
    Creating journal (32768 blocks): done
    Writing superblocks and filesystem accounting information: done
    ```

    格式化需要等待一段时间，请观察系统运行状态，若回显中进程提示为done，则表示格式化完成。

12. 执行以下命令，将新建分区挂载到需要增加空间的目录下，以“/opt”为例。

    **mount /dev/vda2 /opt**

    回显类似如下信息：

    ```
    [root@ecs-bab9 test]# mount /dev/vda2 /opt
    [root@ecs-bab9 test]#
    ```

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >新增加的分区挂载到不为空的目录时，该目录下原本的子目录和文件会被隐藏，所以，新增的分区最好挂载到空目录或者新建目录。如确实要挂载到不为空的目录，可将该目录下的子目录和文件临时移动到其他目录下，新分区挂载成功后，再将子目录和文件移动回来。  

13. 执行以下命令，查看挂载结果。

    **df -TH**

    回显类似如下信息：

    ```
    [root@ecs-2220 ~]# df -TH
    Filesystem     Type      Size  Used Avail Use% Mounted on
    /dev/vda1      ext4       43G  2.0G   39G   5% /
    devtmpfs       devtmpfs  509M     0  509M   0% /dev
    tmpfs          tmpfs     520M     0  520M   0% /dev/shm
    tmpfs          tmpfs     520M  7.2M  513M   2% /run
    tmpfs          tmpfs     520M     0  520M   0% /sys/fs/cgroup
    tmpfs          tmpfs     104M     0  104M   0% /run/user/0
    /dev/vda2      ext4       43G   51M   40G   1% /opt
    ```


## 设置开机自动挂载磁盘<a name="s772020854094454b8a4a3da0d167b3aa"></a>

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


