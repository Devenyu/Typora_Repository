# Linux中配置RNDIS Gadget（u_ether)

## 1、内核配置和编译

需要开启configfs RNDIS Gadget功能需要在menuconfig中配置如下选项：

```markdown
Device Drivers  --->
        [*] USB support  --->
        <M> Inventra Highspeed Dual Role Controller (TI, ADI, AW, ...)
                MUSB Mode Selection (Dual Role mode)  --->
                *** Platform Glue Layer ***
            <M> Allwinner (sunxi)
                *** MUSB DMA mode ***
            [*] Disable DMA (always use PIO)
        USB Physical Layer drivers  --->
            <M> NOP USB Transceiver Driver
        <M>   USB Gadget Support  --->
            <M>   USB Gadget functions configurable through configfs
            [*]     RNDIS
```

编译好后把内核放到boot分区，然后把对应内核的模块目录放到/lib/modules下。

## 2、加载内核模块

配置gadget前要加载这些模块：

```markdown
modprobe sunxi
modprobe configfs
modprobe libcomposite
modprobe u_ether
modprobe usb_f_rndis
```

懒得每次开机都手动加载可以把模块的名字写到/etc/modules里。

## 3、configfs里的配置

```markdown
cd /sys/kernel/config/usb_gadget
mkdir g1
cd g1
echo "0x0502" > idVendor
echo "0x3235" > idProduct
mkdir functions/rndis.rn0
mkdir configs/c1.1
ln -s functions/rndis.rn0 configs/c1.1/
```

## 4、配置UDC

```markdown
ls /sys/class/udc
```

## 5、配置网络

把以下内容加入到/etc/network/interfaces

```markdown
iface usb0 inet static
        address 192.168.137.2
        netmask 255.255.255.0
        network 192.168.137.0
        broadcast 192.168.137.255
        gateway 192.168.137.1
```

选择192.168.137.x网段是因为Windows网卡间共享网络就是在这个网段，你可以根据自己需求修改。然后

```markdown
ifconfig usb0 up
```

这个时候再ifconfig一下就可以看到usb0这个interface了。