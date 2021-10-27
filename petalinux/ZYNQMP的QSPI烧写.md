# ZYNQMP的QSPI烧写

## 1、在petalinux中设定Flash的分区

```markdown
petalinux-config
	-->Subsystem AUTO Hardware Settings
		-->Flash Settings
```

分别设置boot、bootenv、kernel、bootscr四个分区

>（具体的大小需要根据具体工程打包所得到的BOOT.BIN、image.ub、boot.scr来进行修改）

![image-20211027143223016](http://img.devenyu.top/img/image-20211027143223016.png)

![image-20211027144420004](http://img.devenyu.top/img/image-20211027144420004.png)

## 2、设置u-boot

```markdown
petalinux-config -c u-boot
	-->ARM architecture
	设置Boot script offset的值为boot+bootenv+kernel大小的总和
```

![image-20211027144820996](http://img.devenyu.top/img/image-20211027144820996.png)

## 3、设置kernel的偏移地址和大小

进入工程目录下的/project-spec/meta-user/recipes-bsp/u-boot

修改u-boot-zynq-scr.bbappend文件

设置如下图中的三个值：

kernel的偏移地址=boot的大小+bootenv的大小

![image-20211027145213444](http://img.devenyu.top/img/image-20211027145213444.png)

## 4、petalinux-build -c u-boot

## 5、petalinux-build

## 6、打包

```markdown
petalinux-package --boot --fsbl --fpga --pmufw --u-boot --force
```

## 7、将打包生成的boot.src文件转换成BIN文件

**生成的boot.scr，要手动扩展空间到分区大小，且填入为“0”，否则u-boot会报crc校验错误**

```markdown
objcopy -I binary -O binary --pad-to=0x10000 --gap-fill=0x00 boot.scr bootscr-pad.bin
```

## 8、将BOOT.BIN、bootscr-pad.bin、image.ub三个文件烧录到Flash中

将image.ub重命名为image.bin，然后将三个文件分别烧录到Flash中

下面介绍烧录的步骤：

### a、打开vitis,创建应用程序

![image-20211027150507167](http://img.devenyu.top/img/image-20211027150507167.png)

### b、导入FPGA提供的xsa文件

![image-20211027150632470](http://img.devenyu.top/img/image-20211027150632470.png)

### c、选择Zynq MP FSBL完成创建

![image-20211027150825329](http://img.devenyu.top/img/image-20211027150825329.png)

### d、编译工程

![image-20211027151011673](http://img.devenyu.top/img/image-20211027151011673.png)

### e、烧录BOOT.BIN偏移地址为0x0

![image-20211027151746980](http://img.devenyu.top/img/image-20211027151746980.png)

### f、烧录image.bin偏移地址为BOOT.BIN大小+bootenv大小

![image-20211027153222422](http://img.devenyu.top/img/image-20211027153222422.png)

### g、烧录bootscr-pad.bin偏移地址为image.ub偏移地址+image.ub的大小

![image-20211027153339005](http://img.devenyu.top/img/image-20211027153339005.png)

### h、重新启动基板，成功启动！

![image-20211027153435786](http://img.devenyu.top/img/image-20211027153435786.png)