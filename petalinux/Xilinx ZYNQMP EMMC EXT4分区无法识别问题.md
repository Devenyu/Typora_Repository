# Xilinx ZYNQMP EMMC EXT4分区无法识别问题

## Kernel MMC configuration:

```markdown
CONFIG_MMC=y
CONFIG_MMC_DEBUG=y
CONFIG_MMC_BLOCK=y
CONFIG_MMC_BLOCK_MINORS=8
CONFIG_MMC_BLOCK_BOUNCE=y
CONFIG_MMC_SDHCI=y
CONFIG_MMC_SDHCI_PLTFM=y
CONFIG_MMC_SDHCI_OF_ARASAN=y
```

## device tree:

```c
&sdhci1 {
	status = "okay";
	broken-adma2;
	bus-width = <4>;
	xlnx,has-cd = <0x1>;
	xlnx,has-power = <0x0>;
	xlnx,has-wp = <0x0>;
	non-removable;
};
```

## 使用Xilinx SDK生成的文件替换：

替换路径为： ~/components/yocto/layers/meta-xilinx/meta-xilinx-bsp/recipes-bsp/platform-init/platform-init/picozed-zynq7/ps7_init_gpl.c

​						 ~/components/yocto/layers/meta-xilinx/meta-xilinx-bsp/recipes-bsp/platform-init/platform-init/picozed-zynq7/ps7_init_gpl.h

替换内容可以自己在(https://github.com/Xilinx/u-boot-xlnx/tree/xilinx-v2017.4/board/xilinx/zynq/zynq-microzed）中获取或者我的仓库里面[Typora_Repository/petalinux at master · Devenyu/Typora_Repository (github.com)](https://github.com/Devenyu/Typora_Repository/tree/master/petalinux)
