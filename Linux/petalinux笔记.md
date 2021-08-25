# 使用Petalinux定制Linux系统

1、进入工作目录运行 source指令设置petalinux的环境变量和Vivado的环境变量

2、使用指令创建一个工程

```
petalinux-create --type project --template zynq --name ax_peta
```

3、进入工作目录	

```
cd ax_peta
```

4、配置Petalinux工程的硬件信息

```
petalinux-config --get-hw-description ../***.sdk
```

5、配置Linux内核

```
petalinux-config -c kernel
```

6、配置根文件系统

```c
petalinux-config -c rootfs
```

7、编译

```
petalinux-build
```

8、生成BOOT文件

```
petalinux-package --boot --fsbl ./images/linux/zynq.fsbl.elf --u-boot
```



# 通过NFS共享运行

```C
mount -t nfs -o nolock 192.168.1.77:/home/alinx/work /mnt
```



# 增加文件运行权限

```c
chmod +x gpio_test.sh

chmod 777  *******
```



# 给板子定义一个IP地址

```
ifconfig eth0 192.22.113.197 //设置IP地址
```



# 在虚拟机中将文件通过网络传输到板子上

```
scp ***.** root@172.22.113.**:/目的文件地址
```



# uboot重置环境变量

```c
env default -a
```



# uboot保存环境变量

```c
saveenv
```



# petalinux创建新的模块

```
petalinux-create -t modules --name hello --enable
```



# 将虚拟机挂载到板子上通过QT进行测试

```
mount -t nfs -o nolock 192.168.1.107:/home/alinx/work /mnt
cd /mnt
mkdir /tmp/qt
mount qt_lib.img /tmp/qt
cd /tmp/qt
source ./qt_env_set.sh
cd /mnt
Insmod ax-led-drv.ko
cat /proc/devices
mknod /dev/alinx-led c 200 0
./build-axleddev_test-ZYNQ-Debug/axleddev_test /dev/alinx-led on
./build-axleddev_test-ZYNQ-Debug/axleddev_test /dev/alinx-led off
rmmod ax_led_drv
```



# 在VMware中就显示lo回环IP：127.0.0.1的解决办法

```
1.sudo lshw -numeric -class network

2.sudo route -nv

3.sudo dhclient -v
```

