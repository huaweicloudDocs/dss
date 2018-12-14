# 初始化Linux数据盘（parted）<a name="ZH-CN_TOPIC_0085498503"></a>

## 操作场景<a name="section5832280492132"></a>

本文以云服务器的操作系统为“EulerOS x.x.x 64位”为例，采用Parted分区工具为数据盘设置分区。

不同云服务器的操作系统的格式化操作可能不同，本文仅供参考，具体操作步骤和差异请参考对应的云服务器操作系统的产品文档。

## 前提条件<a name="section2975385892226"></a>

-   已以“root”账户登录云服务器。

    弹性云服务器请参见《弹性云服务器用户指南》中章节“登录弹性云服务器”。

-   已挂载数据盘至云服务器，且该数据盘未初始化。

## 划分分区并挂载磁盘<a name="section6037932792658"></a>

本操作以该场景为例，当云服务器挂载了一块新的数据盘时，采用parted分区工具为数据盘设置分区，分区方式设置为GPT，文件系统设为ext4格式，挂载在“/mnt/sdc”下，并设置开机启动自动挂载。

1.  执行以下命令，查看新增数据盘。

    **lsblk**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# lsblk
    NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    xvda    202:0    0   40G  0 disk 
    └─xvda1 202:1    0   40G  0 part /
    xvdb    202:16   0  100G  0 disk
    ```

    表示当前的云服务器有两块磁盘，“/dev/xvda“是系统盘，“/dev/xvdb“是新增数据盘。

2.  执行以下命令，开始对新增数据盘执行分区操作。

    **parted **_新增数据盘_

    以新挂载的数据盘“/dev/xvdb”为例：

    **parted /dev/xvdb**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# parted /dev/xvdb
    GNU Parted 3.1
    Using /dev/xvdb
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    ```

3.  输入“p”，按“Enter”，查看当前磁盘分区方式。

    回显类似如下信息：

    ```
    (parted) p                                                                
    Error: /dev/xvdb: unrecognised disk label
    Model: Xen Virtual Block Device (xvd)                                     
    Disk /dev/xvdb: 107GB
    Sector size (logical/physical): 512B/512B
    Partition Table: unknown
    Disk Flags:     
    ```

    “Partition Table”为“unknown”表示磁盘分区方式未知。

4.  输入以下命令，设置磁盘分区方式。

    **mklabel** _磁盘分区方式_

    磁盘分区方式有MBR和GPT两种，以GPT为例：

    **mklabel gpt**

5.  输入“p”，按“Enter”，设置分区方式后查看磁盘分区方式。

    回显类似如下信息：

    ```
    (parted) mklabel gpt
    (parted) p                                                                
    Model: Xen Virtual Block Device (xvd)
    Disk /dev/xvdb: 107GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags: 
    
    Number  Start  End  Size  File system  Name  Flags
    ```

6.  以为整个磁盘创建一个分区为例，输入“mkpart opt  _0_ _100%_”，按“Enter”。

    “0”表示磁盘起始容量，“100%”表示磁盘截止容量，此处仅供参考，您可以根据业务需要自行规划磁盘分区数量及容量。

    回显类似如下信息：

    ```
    (parted) mkpart opt 0 100%                                                
    Warning: The resulting partition is not properly aligned for best performance.
    Ignore/Cancel? Ignore     
    ```

    可以选择1%到100%。

7.  输入“p”，按“Enter”，查看新建分区的详细信息。

    回显类似如下信息：

    ```
    (parted) p                                                                
    Model: Xen Virtual Block Device (xvd)
    Disk /dev/xvdb: 107GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags: 
    
    Number  Start   End    Size   File system  Name  Flags
     1      17.4kB  107GB  107GB               opt
    ```

    表示新建分区“/dev/xvdb1“的详细信息。

8.  输入“q”，按“Enter”，退出parted分区模式。
9.  执行以下命令，查看磁盘分区信息。

    **lsblk**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# lsblk                                              
    NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    xvda    202:0    0   40G  0 disk 
    └─xvda1 202:1    0   40G  0 part /
    xvdb    202:16   0  100G  0 disk 
    └─xvdb1 202:17   0  100G  0 part 
    ```

    此时可以查看到新建分区“/dev/xvdb1”

10. 执行以下命令，将新建分区文件系统设为系统所需格式。

    **mkfs -t **_文件系统格式_** /dev/xvdb1**

    以设置文件系统为“ext4“为例：

    **mkfs -t ext4 /dev/xvdb1**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# mkfs -t ext4 /dev/xvdb1
    mke2fs 1.42.9 (28-Dec-2013)
    Filesystem label=
    OS type: Linux
    Block size=4096 (log=2)
    Fragment size=4096 (log=2)
    Stride=0 blocks, Stripe width=0 blocks
    6553600 inodes, 26214391 blocks
    1310719 blocks (5.00%) reserved for the super user
    First data block=0
    Maximum filesystem blocks=2174746624
    800 block groups
    32768 blocks per group, 32768 fragments per group
    8192 inodes per group
    Superblock backups stored on blocks: 
    ?32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
    ?4096000, 7962624, 11239424, 20480000, 23887872
    
    Allocating group tables: done                            
    Writing inode tables: done                            
    Creating journal (32768 blocks): done
    Writing superblocks and filesystem accounting information: done 
    ```

    格式化需要等待一段时间，请观察系统运行状态，不要退出。

    >![](public_sys-resources/icon-notice.gif) **注意：**   
    >不同文件系统支持的分区个数和分区大小不同，请根据您的业务需求选择合适的文件系统。  

11. <a name="li379984593442"></a>执行以下命令，新建挂载点。

    **mkdir **_挂载点_

    以新建挂载点“/mnt/sdc“为例：

    **mkdir /mnt/sdc**

12. 执行以下命令，将新建分区挂载到[11](#li379984593442)中新建的挂载点下。

    **mount /dev/xvdb1 **_挂载点_

    以挂载新建分区至“/mnt/sdc“为例：

    **mount /dev/xvdb1 /mnt/sdc**

13. 执行以下命令，查看挂载结果。

    **df -TH**

    回显类似如下信息：

    ```
    [root@ecs-1120 linux]# df -TH
    Filesystem     Type      Size  Used Avail Use% Mounted on
    /dev/xvda1     ext4       43G  8.3G   33G  21% /
    devtmpfs       devtmpfs  4.1G     0  4.1G   0% /dev
    tmpfs          tmpfs     4.1G     0  4.1G   0% /dev/shm
    tmpfs          tmpfs     4.1G   18M  4.1G   1% /run
    tmpfs          tmpfs     4.1G     0  4.1G   0% /sys/fs/cgroup
    tmpfs          tmpfs     807M     0  807M   0% /run/user/2000
    tmpfs          tmpfs     807M     0  807M   0% /run/user/0
    tmpfs          tmpfs     807M     0  807M   0% /run/user/1001
    /dev/xvdb1     ext4      106G   63M  101G   1% /mnt/sdc
    
    ```

    表示新建分区“/dev/xvdb1“已挂载至“/mnt/sdc“。

14. 执行以下步骤，验证新分区读写功能是否可以正常使用。
    1.  执行以下命令，使用vi编辑器新建测试文件，命名为“test01”。

        **vi test01**

    2.  按“i”进入编辑模式。
    3.  在测试文件“test01”中增加一行“hello test”。
    4.  按“Esc”，输入“:wq”，并按“Enter”。

        保存设置并退出vi编辑器。

    5.  执行以下命令，查看测试文件“test01”中的数据。

        **cat test01**



