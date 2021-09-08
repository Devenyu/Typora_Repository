# 在opt目录中安装petalinux

```markdown
sudo mkdir -p /opt/pkg/petalinux/2020.2
sudo chown -R $(whoami):$(whoami) /opt/pkg/petalinux/2020.2
./petalinux-v2020.2-final-installer.run -d /opt/pkg/petalinux/2020.2/
```

# 设置bash为默认sh

```markdown
sudo dpkg-reconfigure dash
```



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

### 当我们使用ssh root@ip登录Linux服务器时，服务器报错：

```markdown
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:Ms+BRn93GbOO1fwP6g1O+UwSRFv9KIUMGeoHDt70OfQ.
Please contact your system administrator.
Add correct host key in /Users/aliyunbaike/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /Users/aliyunbaike/.ssh/known_hosts:6
ECDSA host key for 47.74.190.156 has changed and you have requested strict checking.
```

这是由于，ssh连接服务器时，如果之前连接过，ssh会默认保存该ip的连接协议信息，当我们再次访问此ip服务器时，ssh会自动匹配之前ssh保存的信息，由于我们的服务器做了更改，例如重装系统等操作，会导致本地保存的ssh信息失效，于是再次连接时就会出现上述错误。

另外，远程服务器的ssh服务被卸载重装或ssh相关数据（协议信息）被删除也会导致这个错误。

解决方案：

>删除本地known_hosts里面的缓存信息即可。命令：ssh-keygen -R "你的远程服务器ip地址"
 >		注意：R是大写！



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

