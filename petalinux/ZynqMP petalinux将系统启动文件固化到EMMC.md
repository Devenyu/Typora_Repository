# ZynqMP petalinux将系统启动文件固化到EMMC

## 1、编译petalinux：执行petalinux-config

(1) 选择Subsystem AUTO Hardware Setting

​	->Advanced bootable images storage settings

​	->boot image settings

​	选择primary sd，这样将BOOT.bin设置为从emmc启动。

![](http://img.devenyu.top/img/image-20211115103721634.png)

(2) 选择Subsystem AUTO Hardware Setting

​	->Advanced bootable images storage settings

​	->kernel image settings

​	选择primary sd，这样将image.ub设置为从emmc启动。

![](http://img.devenyu.top/img/image-20211115103805247.png)

 (3) 执行编译：

```markdown
petalinux-package --boot --fsbl --fpga --pmufw --u-boot --force
```

## 2、把EMMC分区

(1) 执行命令：

```markdown
fdisk /dev/mmcblk0
```

(2) 使用n命令，添加一个新的分区：

>Command(m for help): n
>
>Command action
>
>e  extended
>
>p  primary partition (1-4)

选择p，添加主分区

(3) 选择分区号，选择1

>Partition number (1-4): 1                    // 选择分区号
>
>First cylinder (1-238592, default 1): Using default value 1               // 选择分区的第一个柱面，选择1
>
>Last cylinder or +size or +sizeM or +sizeK (1-238592, default 238592): Using default value 238592  // 选择最后一个柱面
>
>注意：1-238592，first要选第一个数，last要选择的比238592小，其中1024就是表示1M

(4) 使用t命令，设置分区格式

>Command (m for help): t
>
> Selected partition 1
>
>1. Hex code (type L to list codes): b
>2. Changed system type of partition 1 to b (Win95 FAT32)

(5) 使用w命令，保存配置，必须保存配置
>Command (m for help): w
>
>The partition table has been altered.
>
>Calling ioctl() to re-read partition table

(6) 使用对应文件系统工具对分析进行格式化

```markdown
mkfs.fat /dev/mmcblk1p1 设置为fat32格式
```

## ３、重启基板，拷贝启动文件到固化分区中

重新启动后，刚刚划分的EMMC分区会自动挂载到/media中。

然后将petalinux中生成的BOOT.bin, image.ub以及boot.scr拷贝到EMMC分区中。

## 4、EMMC启动成功
