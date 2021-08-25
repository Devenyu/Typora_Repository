```c
/include/ "system-conf.dtsi"
/ {
    model = "Zynq ALINX Development Board";//设备制造商的描述，如果有两款板子配置基本一致，它们的compatible是一样的那么就通过model来分辨这两款板子。
    compatible = "alinx,zynq", "xlnx,zynq-7000";//定义一系列的字符串，用来指定内核中哪个machine_desc可以支持本设备，即这个板子兼容哪些平台。一般“供应商，产品”。
    aliases {//定义的一些别称。
        ethernet0 = "&gem0";
        serial0 = "&uart1";
	};
    usb_phy0: usb_phy@0 {
        compatible = "ulpi-phy";
        \#phy-cells = <0>;
        reg = <0xe0002000 0x1000>;//描述设备资源在其父总线定义的地址空间中的地址。通常这意味着内存映射IO寄存器块的偏移量和长度，但在某些总线类型上可能有不同的含义。根节点定义的地址空间中的地址是CPU实际地址。格式是”<address,length>"，作为平台内存资源；reg = <0x30000000 0x4000000>; 表示寄存器(在嵌入式系统中寄存器和内存是同等对待的)的首地址是0x30000000，大小为0x4000000；
        //#address-cells 在它的子节点的reg属性中，用来描述使用了多少个u32整数来描述地址。
        //#address-cells = <1>;//表示子节点的地址宽度是32位
        //#size-cells 在它的子节点的reg属性中，用来描述使用了多少个u32整数来描述地址长度。
        //#size-cells = <1>;//表示子节点的位宽是32位
        view-port = <0x0170>;
        drv-vbus;
    };
};
&i2c0 {
	clock-frequency = <100000>;//设置时钟的频率
};
&usb0 {
    dr_mode = "host";
    usb-phy = <&usb_phy0>;
};
&sdhci0 {
	u-boot,dm-pre-reloc;
};
&uart1 {//串口1
	u-boot,dm-pre-reloc;
};
&flash0 {
	compatible = "micron,m25p80", "w25q256", "spi-flash";
};
&gem0 {//网口
    phy-handle = <&ethernet_phy>;
    ethernet_phy: ethernet-phy@1 {
        reg = <1>;
        device_type = "ethernet-phy";
	};
};
&amba_pl {
    hdmi_encoder_0:hdmi_encoder {
        compatible = "digilent,drm-encoder";
        digilent,edid-i2c = <&i2c0>;
    };
    xilinx_drm {
            compatible = "xlnx,drm";
            xlnx,vtc = <&v_tc_0>;
            xlnx,connector-type = "HDMIA";
            xlnx,encoder-slave = <&hdmi_encoder_0>;
            clocks = <&axi_dynclk_0>;
            dglnt,edid-i2c = <&i2c0>;
            planes {
                xlnx,pixel-format = "rgb888";
            plane0 {
                dmas = <&axi_vdma_0 0>;
                dma-names = "dma";
            };
        };
    };
};
&axi_dynclk_0 {
    compatible = "digilent,axi-dynclk";
    #clock-cells = <0>;
    clocks = <&clkc 15>;
};
&v_tc_0 {
	compatible = "xlnx,v-tc-5.01.a";
};
```

