# Linux磁盘扩容后处理（fdisk）<a name="dss_01_2310"></a>

## 操作场景<a name="s616be0d54fbf455db8cd0520ea4cc5e8"></a>

扩容成功后，对于linux操作系统而言，需要将扩容部分的容量划分至原有分区内，或者为扩容部分的磁盘分配新的分区。

本文以“CentOS 7.0 64位”操作系统为例，采用fdisk分区工具为扩容后的磁盘分配分区。

不同操作系统的操作可能不同，本文仅供参考，具体操作步骤和差异请参考对应操作系统的产品文档。

为扩容后的磁盘分配分区，您可以根据业务需要以及实际的磁盘情况选择以下两种扩容方式，具体如下：

-   不中断业务，新增分区

    为扩容后的磁盘增加新的分区，不需要卸载原有分区，相比替换原有分区的方法，对业务影响较小。推荐系统盘或者需要保证业务不中断的磁盘扩容场景使用。

    如果当前磁盘使用的是MBR分区形式，则此时要求扩容后的数据盘最大容量为2 TB，并且磁盘的分区数量还未达到上限。

-   中断业务，替换原有分区

    如果当前磁盘使用的是MBR分区形式，并且磁盘的分区数量已经达到上限，则此时需要替换原有分区，替换原有分区不会删除原有分区的数据，但是需要先卸载原有分区，会影响线上业务运行。

    如果当前磁盘使用的是MBR分区形式，并且扩容后磁盘容量已经超过2 TB，则超过2 TB的部分容量无法使用。此时若需要使用超过2 TB的部分容量，则必须将MBR分区形式换为GPT，更换磁盘分区形式时会清除磁盘的原有数据，请先对数据进行备份。


## 前提条件<a name="s63c21ba264174d969b0f54ad0b1c79db"></a>

-   已登录云服务器。
    -   裸金属服务器请参见《裸金属服务器用户指南》。
    -   弹性云服务器请参见[登录弹性云服务器](https://support.huaweicloud.com/qs-ecs/zh-cn_topic_0092494193.html)。
    -   裸金属服务器请参见[登录裸金属服务器](https://support.huaweicloud.com/qs-bms/bms_qs_0004.html)。

-   已挂载磁盘至云服务器，且该磁盘的扩容部分未分配分区。

## 检查待扩容磁盘的文件系统<a name="s9d00b9b1417e4d5fb4177e4fb9ce8481"></a>

扩容前，需要检查待扩容磁盘的文件系统是否可正常挂载。

1.  （可选）如果待扩容磁盘分区未挂载，请执行以下命令，挂载磁盘分区至指定目录。

    **mount** _磁盘分区_ _挂载目录_

    命令示例：

    **mount /dev/xvdb1 /mnt/sdc**

    若系统提示挂载异常，请检查待扩容磁盘的文件系统是否有误。例如，某个用户最初格式化磁盘“/dev/xvdb”时操作有误，为“/dev/xvdb”创建了文件系统，而实际并没有为磁盘下的分区“/dev/xvdb1”创建文件系统，并且此前使用时系统之前实际挂载的应该为磁盘“/dev/xvdb”，而不是磁盘分区“/dev/xvdb1”。

2.  执行以下命令，查看磁盘的挂载情况。

    **df -TH**

    回显类似如下信息：

    ```
    [root@ecs-b656 test]# df -TH
    Filesystem     Type      Size  Used Avail Use% Mounted on
    /dev/xvda2     xfs        11G  7.4G  3.2G  71% /
    devtmpfs       devtmpfs  4.1G     0  4.1G   0% /dev
    tmpfs          tmpfs     4.1G   82k  4.1G   1% /dev/shm
    tmpfs          tmpfs     4.1G  9.2M  4.1G   1% /run
    tmpfs          tmpfs     4.1G     0  4.1G   0% /sys/fs/cgroup
    /dev/xvda3     xfs       1.1G   39M  1.1G   4% /home
    /dev/xvda1     xfs       1.1G  131M  915M  13% /boot
    /dev/xvdb1     ext4       11G   38M  9.9G   1% /mnt/sdc
    ```

    此时可以看到，“/dev/xvdb1“的文件系统为“ext4”，并且已挂载至“/mnt/sdc“。

3.  执行以下命令，进入挂载目录查看磁盘上的文件。

    **ll** _挂载目录_

    命令示例：

    **ll** **/mnt/sdc**

    若可以查看到磁盘上的文件，则证明待扩容的磁盘情况正常。


## 查看分区形式<a name="s8e370c87ff2e4d54b9347769ab3b71e2"></a>

分区前，需要查看当前磁盘的分区形式，当为MBR时可以选择fdisk或者parted工具，当为GPT时需要使用parted工具。

1.  执行以下命令，查看当前磁盘的分区形式。

    **fdisk -l**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# fdisk -l
    
    Disk /dev/xvda: 42.9 GB, 42949672960 bytes, 83886080 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0x000c5712
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/xvda1            2048    83886079    41942016   83  Linux
    WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.
    
    Disk /dev/xvdb: 161.1 GB, 161061273600 bytes, 314572800 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: gpt
    
    
    #         Start          End    Size  Type            Name
     1           34    209715166    100G  Microsoft basic opt
     2    209715167    314572766     50G  Microsoft basic opt1
    WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.
    
    Disk /dev/xvdc: 42.9 GB, 42949672960 bytes, 83886080 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: gpt
    
    
    #         Start          End    Size  Type            Name
     1           34     16777215      8G  Microsoft basic opt
     2     16777216     83884031     32G  Microsoft basic opt
    ```

    “Disk label type”表示当前磁盘的分区形式，dos表示磁盘分区形式为MBR，gpt表示磁盘分区形式为GPT。


## 新增分区<a name="s3a63696e05034a2db04094ab12d767eb"></a>

本操作以该场景为例，为系统盘扩容后的空间分配一个新的分区，并挂载到“/opt”下，此时可以不中断业务。

1.  执行以下命令，查看磁盘的分区信息。

    **fdisk -l**

    回显类似如下信息，“/dev/xvda“表示系统盘。

    ```
    [root@ecs-bab9 test]# fdisk -l
    
    Disk /dev/xvda: 64.4 GB, 64424509440 bytes, 125829120 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0x000cc4ad
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/xvda1   *        2048     2050047     1024000   83  Linux
    /dev/xvda2         2050048    22530047    10240000   83  Linux
    /dev/xvda3        22530048    24578047     1024000   83  Linux
    /dev/xvda4        24578048    83886079    29654016    5  Extended
    /dev/xvda5        24580096    26628095     1024000   82  Linux swap / Solaris
    ```

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
    All primary partitions are in use
    Adding logical partition 6
    First sector (26630144-83886079, default 26630144): 
    ```

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >磁盘使用MBR分区形式，最多可以创建4个主分区，或者3个主分区加1个扩展分区，扩展分区不可以直接使用，需要划分成若干个逻辑分区才可以使用。  
    >此示例中系统盘主分区已满，且原来已经有5个分区（3个主分区加2个逻辑分区），所以系统自动在扩展分区中新增逻辑分区，编号为6。  
    >若需要查看系统盘主分区未满的操作示例，请参考[Linux系统盘扩容后处理（fdisk）](Linux系统盘扩容后处理（fdisk）.md)。  

4.  输入新分区的起始磁柱编号，如设置默认值，按“Enter”。

    起始磁柱编号必须大于原有分区的结束磁柱编号。

    回显类似如下信息：

    ```
    First sector (26630144-83886079, default 26630144):
    Using default value 26630144
    Last sector, +sectors or +size{K,M,G} (26630144-83886079, default 83886079):
    ```

5.  输入新分区的截止磁柱编号，按“Enter”。

    本步骤中使用默认截止磁柱编号为例。

    回显类似如下信息：

    ```
    Last sector, +sectors or +size{K,M,G} (26630144-83886079, default 83886079):
    Using default value 83886079
    Partition 6 of type Linux and of size 27.3 GiB is set
    
    Command (m for help): 
    ```

6.  输入“p”，按“Enter”，查看新建分区。

    回显类似如下信息：

    ```
    Disk /dev/xvda: 64.4 GB, 64424509440 bytes, 125829120 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0x000cc4ad
    
    Device Boot      Start         End      Blocks   Id  System
    /dev/xvda1   *        2048     2050047     1024000   83  Linux
    /dev/xvda2         2050048    22530047    10240000   83  Linux
    /dev/xvda3        22530048    24578047     1024000   83  Linux
    /dev/xvda4        24578048    83886079    29654016    5  Extended
    /dev/xvda5        24580096    26628095     1024000   82  Linux swap / Solaris
    /dev/xvda6        26630144    83886079    28627968   83  Linux
    
    Command (m for help): 
    ```

7.  输入“w”，按“Enter”，将分区结果写入分区表中。

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

8.  执行以下命令，将新的分区表变更同步至操作系统。

    **partprobe**

9.  执行以下命令，设置新建分区文件系统格式。

    以“ext4” 文件格式为例：

    **mkfs -t ext4 /dev/xvda6**

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >设置xfs文件系统的操作与ext3或者ext4一样，命令为：**mkfs -t xfs /dev/xvda6**  

    回显类似如下信息：

    ```
    [root@ecs-bab9 test]# mkfs -t ext4 /dev/xvda6
    mke2fs 1.42.9 (28-Dec-2013)
    Filesystem label=
    OS type: Linux
    Block size=4096 (log=2)
    Fragment size=4096 (log=2)
    Stride=0 blocks, Stripe width=0 blocks
    1790544 inodes, 7156992 blocks
    357849 blocks (5.00%) reserved for the super user
    First data block=0
    Maximum filesystem blocks=2155872256
    219 block groups
    32768 blocks per group, 32768 fragments per group
    8176 inodes per group
    Superblock backups stored on blocks:
            32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
            4096000
    
    Allocating group tables: done
    Writing inode tables: done
    Creating journal (32768 blocks): done
    Writing superblocks and filesystem accounting information: done
    ```

    格式化需要等待一段时间，请观察系统运行状态，若回显中进程提示为done，则表示格式化完成。

10. 执行以下命令，将新建分区挂载到需要增加空间的目录下，以“/opt”为例。

    **mount /dev/xvda6 /opt**

    回显类似如下信息：

    ```
    [root@ecs-bab9 test]# mount /dev/xvda6 /opt
    [root@ecs-bab9 test]#
    ```

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >新增加的分区挂载到不为空的目录时，该目录下原本的子目录和文件会被隐藏，所以，新增的分区最好挂载到空目录或者新建目录。如确实要挂载到不为空的目录，可将该目录下的子目录和文件临时移动到其他目录下，新分区挂载成功后，再将子目录和文件移动回来。  

11. 执行以下命令，查看挂载结果。

    **df -TH**

    回显类似如下信息：

    ```
    [root@ecs-bab9 test]# df -TH
    Filesystem     Type      Size  Used Avail Use% Mounted on
    /dev/xvda2     xfs        11G  7.4G  3.2G  71% /
    devtmpfs       devtmpfs  4.1G     0  4.1G   0% /dev
    tmpfs          tmpfs     4.1G   82k  4.1G   1% /dev/shm
    tmpfs          tmpfs     4.1G  9.2M  4.1G   1% /run
    tmpfs          tmpfs     4.1G     0  4.1G   0% /sys/fs/cgroup
    /dev/xvda3     xfs       1.1G   39M  1.1G   4% /home
    /dev/xvda1     xfs       1.1G  131M  915M  13% /boot
    /dev/xvda6     ext4       29G   47M   28G   1% /opt
    ```


## 替换原有分区<a name="sc18be5e0483541c189f45ade80aad2c6"></a>

本操作以该场景为例，云服务器上已挂载一块磁盘，分区“/dev/xvdb1”，挂载目录“/mnt/sdc”，需要替换原有分区“/dev/xvdb1”，将新增容量加到该分区内，此时需要中断业务。

>![](public_sys-resources/icon-notice.gif) **须知：**   
>扩容后的新增空间是添加在磁盘末尾的，对具有多个分区的的磁盘扩容时，只支持替换排在末尾的分区。  

1.  <a name="lc34bc8b8cf6a4849a78e4237fef77c57"></a>执行以下命令，查看磁盘的分区信息。

    **fdisk -l**

    回显类似如下信息：

    ```
    [root@ecs-b656 test]# fdisk -l
    
    Disk /dev/xvda: 42.9 GB, 42949672960 bytes, 83886080 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0x000cc4ad
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/xvda1   *        2048     2050047     1024000   83  Linux
    /dev/xvda2         2050048    22530047    10240000   83  Linux
    /dev/xvda3        22530048    24578047     1024000   83  Linux
    /dev/xvda4        24578048    83886079    29654016    5  Extended
    /dev/xvda5        24580096    26628095     1024000   82  Linux swap / Solaris
    
    Disk /dev/xvdb: 21.5 GB, 21474836480 bytes, 41943040 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0xb00005bd
    
    Device Boot      Start         End      Blocks   Id  System
    /dev/xvdb1            2048    20971519    10484736   83  Linux
    ```

    表示当前数据盘“/dev/xvdb“总容量为21.5 GB，数据盘当前只有一个分区“dev/xvdb1”，该分区的初始磁柱值为2048，截止磁柱值为20971519。

    查看回显中数据盘“/dev/xvdb“的容量，扩容的容量是否已经包含在容量总和中。

    -   若扩容的容量未在数据盘容量总和中，请参考[Linux SCSI数据盘扩容后处理（fdisk）](Linux-SCSI数据盘扩容后处理（fdisk）.md)章节刷新系统内容量。
    -   若扩容的容量已在数据盘容量总和中，请记录待替换分区“dev/xvdb1”的初始和截止磁柱值，这些值在后续重新创建分区时需要使用，记录完成后执行[2](#l71f4c979d39e42c48983a5396c5b0a49)。

2.  <a name="l71f4c979d39e42c48983a5396c5b0a49"></a>执行以下命令，卸载磁盘分区。

    **umount /mnt/sdc**

3.  执行以下命令之后，进入fdisk分区工具，并输入“d”，删除原来的分区“/dev/xvdb1“。

    **fdisk /dev/xvdb**

    屏幕回显如下：

    ```
    [root@ecs-b656 test]# fdisk /dev/xvdb
    Welcome to fdisk (util-linux 2.23.2).
    
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.
    
    Command (m for help): d
    Selected partition 1
    Partition 1 is deleted
    
    Command (m for help):
    ```

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >删除分区后，请参考以下操作步骤替换原有分区，则不会导致数据盘内数据的丢失。  

4.  输入“n”，按“Enter”，开始新建分区。

    输入“n“表示新增一个分区。

    回显类似如下信息：

    ```
    Command (m for help): n
    Partition type:
       p   primary (0 primary, 0 extended, 4 free)
       e   extended
    ```

    表示磁盘有两种分区类型：

    -   “p“表示主要分区。
    -   “e“表示延伸分区。

5.  此处分区类型需要与原分区保持一致，以原分区类型是主要分区为例，输入“p”，按“Enter”，开始重新创建一个主分区。

    回显类似如下信息

    ```
    Select (default p): p
    Partition number (1-4, default 1):
    ```

    “Partition number“表示主分区编号。

6.  此处分区编号需要与原分区保持一致，以原分区编号是“1“为例，输入分区编号“1“，按“Enter”。

    回显类似如下信息：

    ```
    Partition number (1-4, default 1): 1
    First sector (2048-41943039, default 2048):
    ```

    “First sector“表示初始磁柱值。

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >以下操作会导致数据丢失：  
    >-   选择的初始磁柱值与原分区的不一致。  
    >-   选择的截止磁柱值小于原分区的值。  

7.  此处必须与原分区保持一致，以[1](#lc34bc8b8cf6a4849a78e4237fef77c57)中记录的初始磁柱值2048为例，按“Enter”。

    回显类似如下信息：

    ```
    First sector (2048-41943039, default 2048):
    Using default value 2048
    Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039):
    ```

    “Last sector“表示截止磁柱值。

8.  此处截止磁柱值应大于等于[1](#lc34bc8b8cf6a4849a78e4237fef77c57)中记录的截止磁柱值20971519，以选择默认截止磁柱值41943039为例，按“Enter”。

    回显类似如下信息：

    ```
    Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039):
    Using default value 41943039
    Partition 1 of type Linux and of size 20 GiB is set
    Command (m for help):
    ```

    表示分区完成。

9.  输入“p”，按“Enter”，查看新建分区的详细信息。

    回显类似如下信息：

    ```
    Command (m for help): p
    
    Disk /dev/xvdb: 21.5 GB, 21474836480 bytes, 41943040 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0xb00005bd
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/xvdb1            2048    41943039    20970496   83  Linux
    
    Command (m for help): 
    ```

    表示新建分区“/dev/xvdb1“的详细信息。

10. 输入“w”，按“Enter”，将分区结果写入分区表中。

    回显类似如下信息：

    ```
    Command (m for help): w
    The partition table has been altered!
    
    Calling ioctl() to re-read partition table.
    Syncing disks.
    ```

    表示分区创建完成。

    >![](public_sys-resources/icon-note.gif) **说明：**   
    >如果之前分区操作有误，请输入“q”，则会退出fdisk分区工具，之前的分区结果将不会被保留。  

11. 根据磁盘的文件系统，分别执行以下操作。
    -   若磁盘文件系统为ext3或ext4，请执行以下步骤。
        1.  执行以下命令，检查“/dev/xvdb1“文件系统的正确性。

            **e2fsck -f /dev/xvdb1**

            回显类似如下信息：

            ```
            [root@ecs-b656 test]# e2fsck -f /dev/xvdb1
            e2fsck 1.42.9 (28-Dec-2013)
            Pass 1: Checking inodes, blocks, and sizes
            Pass 2: Checking directory structure
            Pass 3: Checking directory connectivity
            Pass 4: Checking reference counts
            Pass 5: Checking group summary information
            /dev/xvdb1: 11/655360 files (0.0% non-contiguous), 83137/2621184 blocks
            ```

        2.  执行以下命令，扩展“/dev/xvdb1“文件系统的大小。

            **resize2fs /dev/xvdb1**

            回显类似如下信息：

            ```
            [root@ecs-b656 test]# resize2fs /dev/xvdb1
            resize2fs 1.42.9 (28-Dec-2013)
            Resizing the filesystem on /dev/xvdb1 to 5242624 (4k) blocks.
            The filesystem on /dev/xvdb1 is now 5242624 blocks long.
            ```

        3.  执行以下命令，将新建分区挂载到“/mnt/sdc”目录下。

            **mount /dev/xvdb1 /mnt/sdc**

    -   若磁盘文件系统为xfs，请执行以下步骤。
        1.  执行以下命令，将新建分区挂载到“/mnt/sdc”目录下。

            **mount /dev/xvdb1 /mnt/sdc**

        2.  执行以下命令，扩展“/dev/xvdb1“文件系统的大小。

            **sudo** **xfs\_growfs** **/dev/xvdb1**


12. 执行以下命令，查看“/dev/xvdb1”分区挂载结果。

    **df -TH**


## 设置开机自动挂载磁盘<a name="s85cdb9fbcfe24ccbb0f07e43d3cef576"></a>

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


