# 控制phy芯片的RESET引脚

这里我以DP83867IRPAP为例

## 1、在原理图中查看RESET引脚所对应的MIO

![](http://img.devenyu.top/img/image-20211029094950622.png)

如图所示，得到两个phy芯片所对应的RESET引脚分别为MIO27和MIO28

## 2、在Vitis中根据xsa创建应用程序

### a、打开vitis,创建应用程序

![](http://img.devenyu.top/img/image-20211027150507167.png)

### b、导入FPGA提供的xsa文件

![](http://img.devenyu.top/img/image-20211027150632470.png)

### c、选择Zynq MP FSBL完成创建

![](http://img.devenyu.top/img/image-20211027150825329.png)

## 3、在平台目录的zynqmp_fsbl中创建vimc_phy_config.c和vimc_phy_config.h

vimc_phy_config.c的内容如下：

```c
#include "vimc_phy_config.h"
#include "xparameters.h"
#include "xgpiops.h"
#include "xstatus.h"
#include "xplatform_info.h"
#include "xfsbl_error.h"
#include "xfsbl_debug.h"
#include <xil_printf.h>

#define  PHY_RESET_GPIO_DEVICE_ID		0  /*gpio 1 */
XGpioPs Gpio;
#define ETH1_RSTN_PIN   28
#define ETH2_RSTN_PIN   27
#define GPIO_OUTPUT_DIR 1
#define GPIO_INPUT_DIR  0


/*signal=0 is low ;signal =1 is high*/
int config_phy_gpio(int signal)
{
	int Status = 0;
	XGpioPs_Config *ConfigPtr;
	ConfigPtr = XGpioPs_LookupConfig(PHY_RESET_GPIO_DEVICE_ID);

	Status = XGpioPs_CfgInitialize(&Gpio, ConfigPtr,
					ConfigPtr->BaseAddr);

	if (Status != XST_SUCCESS)
	{
		XFsbl_Printf(DEBUG_GENERAL,"Config GPIO Failed\n\r");
		return XST_FAILURE;
	}
	else
	{
		XFsbl_Printf(DEBUG_GENERAL,"Config GPIO successfully\n\r");
	}

	XGpioPs_SetDirectionPin(&Gpio, ETH1_RSTN_PIN, GPIO_OUTPUT_DIR);
	XGpioPs_SetDirectionPin(&Gpio, ETH2_RSTN_PIN, GPIO_OUTPUT_DIR);
	/*GPIO0[15] GPIO0[16]*/
	XGpioPs_SetOutputEnablePin(&Gpio,ETH1_RSTN_PIN , 1);
	XGpioPs_SetOutputEnablePin(&Gpio,ETH2_RSTN_PIN , 1);

	/* Set the GPIO output to be low. */
	XGpioPs_WritePin(&Gpio, ETH1_RSTN_PIN, signal);
	XGpioPs_WritePin(&Gpio, ETH2_RSTN_PIN, signal);

	return Status;
}
```

vimc_phy_config.h的内容如下:

```c
#ifndef __VIMC_PHY_CONFIG_H
#define __VIMC_PHY_CONFIG_H

int config_phy_gpio(int signal);

#endif /*__VIMC_PHY_CONFIG_H*/
```

## 4、在phy芯片的datasheet中查看RESET的有效电平是多少

![](http://img.devenyu.top/img/image-20211029102038881.png)

如图可知：该芯片的RESET是低电平有效。

## 5、在xfsbl_main.c中添加config_phy_gpio函数的使用

在case XFSBL_STAGE1的末尾添加如下代码来使用config_phy_gpio函数：将RESET引脚设置为低电平有效。

```c
//config phy reset
XFsbl_Printf(DEBUG_INFO,"[debug] phy reset start!!!\n");
config_phy_gpio(0);
```

在case XFSBL_STAGE4的开头添加如下代码来使用config_phy_gpio函数：将RESET引脚拉高。

>如果不把RESET拉高则无法读出phy地址

```c
/*user add configure phy chip*/
XFsbl_Printf(DEBUG_INFO,"[debug] phy reset end!!!\n");
config_phy_gpio(1);
```

## 6、编译应用程序并烧录到Flash中或者SD中

