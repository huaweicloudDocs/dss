# Linux SCSI数据盘扩容后处理（fdisk）<a name="dss_01_2312"></a>

## 操作场景<a name="s7be40ed5212144fe85c517e0bb62634d"></a>

扩容成功后，对于linux操作系统而言，需要将扩容部分的容量划分至原有分区内，或者为扩容部分的磁盘分配新的分区。

本文以“SUSE Linux Enterprise Server 11 SP4 64bit”操作系统、并挂载SCSI数据盘为例，采用fdisk分区工具为扩容后的磁盘分配分区。

当前云服务器上已挂载一块磁盘，原容量为10GB，已经在管理控制台成功扩容了10GB，当前总容量应为20GB，但是登录至云服务器中看不到新增容量，本操作指导用户查看新增容量并替换原有分区。

不同操作系统的操作可能不同，本文仅供参考，具体操作步骤和差异请参考对应操作系统的产品文档。

为扩容后的磁盘分配分区，您可以根据业务需要以及实际的磁盘情况选择以下两种扩容方式，具体如下：

-   不中断业务，新增分区

    为扩容后的磁盘增加新的分区，不需要卸载原有分区，相比替换原有分区的方法，对业务影响较小。推荐系统盘或者需要保证业务不中断的磁盘扩容场景使用。

    如果当前磁盘使用的是MBR分区形式，则此时要求扩容后的数据盘最大容量为2 TB，并且磁盘的分区数量还未达到上限。

-   中断业务，替换原有分区

    如果当前磁盘使用的是MBR分区形式，并且磁盘的分区数量已经达到上限，则此时需要替换原有分区，替换原有分区不会删除原有分区的数据，但是需要先卸载原有分区，会影响线上业务运行。

    如果当前磁盘使用的是MBR分区形式，并且扩容后磁盘容量已经超过2 TB，则超过2 TB的部分容量无法使用。此时若需要使用超过2 TB的部分容量，则必须将MBR分区形式换为GPT，更换磁盘分区形式时会清除磁盘的原有数据，请先对数据进行备份。

    >![](public_sys-resources/icon-notice.gif) **须知：** 
    >扩容时请谨慎操作，误操作可能会导致数据丢失或者异常，建议扩容前对数据进行备份，可以使用CBR功能，请参见[管理备份磁盘](管理备份磁盘.md)。


## 前提条件<a name="s3cc6769cab8c453c8802c28796ecf003"></a>

-   已登录云服务器。
    -   裸金属服务器请参见《裸金属服务器用户指南》。
    -   弹性云服务器请参见[登录弹性云服务器](https://support.huaweicloud.com/qs-ecs/zh-cn_topic_0092494193.html)。
    -   裸金属服务器请参见[登录裸金属服务器](https://support.huaweicloud.com/qs-bms/bms_qs_0004.html)。

-   已挂载磁盘至云服务器，且该磁盘的扩容部分未分配分区。

## 替换原有分区<a name="scf72b15c0256452987149ee0b1309998"></a>

本操作以该场景为例，当前云服务器上已挂载一块磁盘，分区“/dev/sda1”已挂载至“/mnt/sdc”目录下，现在需要替换原有分区“/dev/sda1”，将新增容量加到该分区内，此时需要中断业务。

1.  执行以下命令，查看磁盘的分区信息。

    **fdisk -l**

    回显类似如下信息：

    ```
    ecs-xen-02:/home/linux # fdisk -l
    
    Disk /dev/xvda: 107.4 GB, 107374182400 bytes
    255 heads, 63 sectors/track, 13054 cylinders, total 209715200 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00065c40
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/xvda1            2048    41945087    20971520   82  Linux swap / Solaris
    /dev/xvda2   *    41945088    83892223    20973568   83  Linux
    /dev/xvda3        83892224   209715199    62911488   83  Linux
    
    Disk /dev/sda: 10.7 GB, 10737418240 bytes
    64 heads, 32 sectors/track, 10240 cylinders, total 20971520 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x2f1c057a
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1            2048    20971519    10484736   83  Linux
    ```

    当前在管理控制台上已经将数据盘“/dev/sda”由10GB扩容为20GB，但是扩容的容量未包含在容量总和中。此类情况需要执行命令刷新云服务器内数据盘的容量。

2.  执行以下命令，刷新云服务器内数据盘的容量。

    **echo 1 \> /sys/class/scsi\_device/**_**%d:%d:%d:%d**_**/device/rescan &**

    其中“%d:%d:%d:%d”为“/sys/class/scsi\_device/”路径下的文件夹，执行**ll /sys/class/scsi\_device/**命令获取。

    回显类似如下信息，“2:0:0:0”即为待获取的文件夹。

    ```
    cs-xen-02:/sys/class/scsi_device # ll /sys/class/scsi_device/
    total 0
    lrwxrwxrwx 1 root root 0 Sep 26 11:37 2:0:0:0 -> ../../devices/xen/vscsi-2064/host2/target2:0:0/2:0:0:0/scsi_device/2:0:0:0
    ```

    命令示例：

    **echo 1 \> /sys/class/scsi\_device/2:0:0:0/device/rescan &**

3.  <a name="l4e31601beaec4a7aa80e5edf0f77638e"></a>刷新完成后，执行以下命令，再次查看磁盘分区信息。

    **fdisk -l**

    回显类似如下信息：

    ```
    ecs-xen-02:/sys/class/scsi_device # fdisk -l
    
    Disk /dev/xvda: 107.4 GB, 107374182400 bytes
    255 heads, 63 sectors/track, 13054 cylinders, total 209715200 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00065c40
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/xvda1            2048    41945087    20971520   82  Linux swap / Solaris
    /dev/xvda2   *    41945088    83892223    20973568   83  Linux
    /dev/xvda3        83892224   209715199    62911488   83  Linux
    
    Disk /dev/sda: 21.5 GB, 21474836480 bytes
    64 heads, 32 sectors/track, 20480 cylinders, total 41943040 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x2f1c057a
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1            2048    20971519    10484736   83  Linux
    ```

    数据盘“/dev/sda”容量已经增加，刷新成功。当前数据盘“/dev/sda”有一个分区“/dev/sda1”待替换，请记录“/dev/sda1”的初始和截止磁柱值，这些值在后续重新创建分区时需要使用，初始磁柱值为2048，截止磁柱值为20971519。

4.  执行如下命令，卸载磁盘分区。

    **umount /mnt/sdc**

5.  执行如下命令之后，进入fdisk分区工具，并输入**d**，删除原来的分区“/dev/sda1“。

    **fdisk /dev/sda1**

    回显类似如下信息：

    ```
    [ecs-xen-02:/sys/class/scsi_device # fdisk /dev/sda
    
    Command (m for help): d
    Selected partition 1
    
    Command (m for help): 
    ```

6.  输入“n”，按“Enter”，开始新建分区。

    输入“n“表示新增一个分区。

    回显类似如下信息：

    ```
    Command (m for help): n
    Command action
       e   extended
       p   primary partition (1-4)
    ```

    表示磁盘有两种分区类型：

    -   “p“表示主要分区。
    -   “e“表示延伸分区。

7.  此处分区类型需要与原分区保持一致，以原分区类型是主要分区为例，输入“p”，按“Enter”，开始创建一个主分区。

    回显类似如下信息：

    ```
    p
    Partition number (1-4, default 1):
    ```

    “Partition number“表示主分区编号。

8.  此处分区编号需要与原分区保持一致，以原分区编号是“1”为例，输入分区编号“1“，按“Enter”。

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

9.  此处必须与原分区保持一致，以[3](#l4e31601beaec4a7aa80e5edf0f77638e)中记录的初始磁柱值2048为例，按“Enter”。

    回显类似如下信息：

    ```
    First sector (2048-41943039, default 2048): 
    Using default value 2048
    Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039): 
    ```

    “Last sector“表示截止磁柱值。

10. 此处截止磁柱值应大于或者等于[3](#l4e31601beaec4a7aa80e5edf0f77638e)中记录的值20971519，以选择默认截止磁柱编号41943039为例，按“Enter”。

    回显类似如下信息：

    ```
    Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039): 
    Using default value 41943039
    
    Command (m for help):
    ```

    表示分区完成。

11. 输入“p”，按“Enter”，查看新建分区的详细信息。

    回显类似如下信息，表示新建分区“/dev/sda1“的详细信息。

    ```
    CCommand (m for help): p
    
    Disk /dev/sda: 21.5 GB, 21474836480 bytes
    64 heads, 32 sectors/track, 20480 cylinders, total 41943040 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x2f1c057a
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1            2048    41943039    20970496   83  Linux
    Command (m for help): 
    
    ```

12. 输入“w”，按“Enter”，将分区结果写入分区表中。

    回显类似如下信息，表示分区创建完成。

    ```
    Command (m for help): w
    The partition table has been altered!
    
    Calling ioctl() to re-read partition table.
    Syncing disks.
    ```

    >![](public_sys-resources/icon-note.gif) **说明：** 
    >如果之前分区操作有误，请输入“q”，则会退出fdisk分区工具，之前的分区结果将不会被保留。

13. 根据磁盘的文件系统，分别执行以下操作。
    -   若磁盘文件系统为ext3或ext4，请执行以下步骤。
        1.  执行以下命令，检查分区“/dev/sda1“文件系统的正确性。

            **e2fsck -f /dev/sda1**

            回显类似如下信息：

            ```
            ecs-xen-02:/sys/class/scsi_device # e2fsck -f /dev/sda1
            e2fsck 1.41.9 (22-Aug-2009)
            Pass 1: Checking inodes, blocks, and sizes
            Pass 2: Checking directory structure
            Pass 3: Checking directory connectivity
            Pass 4: Checking reference counts
            Pass 5: Checking group summary information
            /dev/sda1: 11/655360 files (0.0% non-contiguous), 79663/2621184 blocks
            ```

        2.  执行以下命令，扩展“/dev/sda1“文件系统的大小。

            **resize2fs /dev/sda1**

            回显类似如下信息：

            ```
            ecs-xen-02:/sys/class/scsi_device # resize2fs /dev/sda1
            resize2fs 1.41.9 (22-Aug-2009)
            Resizing the filesystem on /dev/sda1 to 5242624 (4k) blocks.
            The filesystem on /dev/sda1 is now 5242624 blocks long.
            ```

        3.  执行以下命令，将新建分区挂载到“/mnt/sdc”目录下。

            **mount /dev/sda1 /mnt/sdc**

    -   若磁盘文件系统为xfs，请执行以下步骤。
        1.  执行以下命令，将新建分区挂载到“/mnt/sdc”目录下。

            **mount /dev/sda1 /mnt/sdc**

        2.  执行以下命令，扩展“/dev/sda1“文件系统的大小。

            **sudo** **xfs\_growfs** **/dev/sda1**


14. 执行以下命令，查看“/dev/sda1”分区挂载结果。

    **df -TH**


## 设置开机自动挂载磁盘<a name="s2fa3e1e62f4f48d89fbe13b75df1b7ae"></a>

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


