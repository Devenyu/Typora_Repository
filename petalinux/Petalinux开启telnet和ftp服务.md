# Petalinux开启telnet和ftp服务

## 1、在Kernel Configuration中勾选功能

```markdown
petalinux-config -c kernel
CONFIG_ETHERNET
CONFIG_NET_VENDOR_XILINX
CONFIG_XILINX_EMACLITE
```

## 2、在Busybox Configuration中勾选功能

```markdown
 petalinux-config -c busybox 
 -> Networking utilities 
 -> ftpd 
 -> [*] Enable -w option
```

## 3、设备树中配置phy

axi-ethernet的配置：

```markdown
axi_ethernetlite_1: ethernet@40e00000 {
                       compatible = "xlnx,axi-ethernetlite-3.0", "xlnx,xps-ethernetlite-1.00.a";
                       device_type = "network";
                       interrupt-parent = <&axi_intc_1>;
                       interrupts = <1 0>;
                       local-mac-address = [00 0a 35 00 00 00];
                       phy-handle = <&phy0>;
                       reg = <0x40e00000 0x10000>;
                       xlnx,duplex = <0x1>;
                       xlnx,include-global-buffers = <0x1>;
                       xlnx,include-internal-loopback = <0x0>;
                       xlnx,include-mdio = <0x1>;
                       xlnx,instance = "axi_ethernetlite_inst";
                       xlnx,rx-ping-pong = <0x1>;
                       xlnx,s-axi-id-width = <0x1>;
                       xlnx,tx-ping-pong = <0x1>;
                       xlnx,use-internal = <0x0>;
                       mdio {
                               #address-cells = <1>;
                               #size-cells = <0>;
                               phy0: phy@7 {
                                       compatible = "marvell,88e1111";
                                       device_type = "ethernet-phy";
                                       reg = <7>;
                               } ;
                       } ;
               } ;
```

ps-ethernet的配置：

```markdown
&gem1 {
        status = "okay";
        compatible = "cdns,zynqmp-gem";
        local-mac-address = [00 0a 35 00 22 02];
        phy-mode = "rgmii-id";  
        phy-handle = <&phy0>;  
        phy0: phy@0x01 {
                device_type = "ethernet-phy";
                reg = <0x01>;  
		ti,rx-internal-delay = <0x8>;
		ti,tx-internal-delay = <0xa>;
		ti,fifo-depth = <0x1>;
		ti,dp83867-rxctrl-strap-quirk;
        }; 
};

&gem3 {
        status = "okay";
        compatible = "cdns,zynqmp-gem";
        local-mac-address = [00 0a 35 00 22 01];
        phy-mode = "rgmii-id";
        phy-handle = <&phy1>;     
        phy1: phy@0x11 {
                device_type = "ethernet-phy";
                reg = <0x11>;
		ti,rx-internal-delay = <0x8>;
		ti,tx-internal-delay = <0xa>;
		ti,fifo-depth = <0x1>;
		ti,dp83867-rxctrl-strap-quirk;
        };             
};
```

## 4、配置基板中的/etc/inetd.conf文件

将telnet和ftp服务的注释取消

![](https://img.devenyu.top/img/image-20211230101446845.png)

>telnet服务也可以使用busybox telnetd命令来开启