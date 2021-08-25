SPI的基本介绍
SPI的简介
SPI，是英语Serial Peripheral interface的缩写，顾名思义就是串行外围设备接口，是Motorola首先在其MC68HCXX系列处理器上定义的。

SPI接口主要应用在EEPROM、FLASH、实时时钟、AD转换器，还有数字信号处理器和数字信号解码器之间。SPI是一种高速的，全双工，同步的通信总线，并且在芯片的管脚上只占用四根线，节约了芯片的管脚，同时为PCB的布局上节省空间，提供方便，正是出于这种简单易用的特性，现在越来越多的芯片集成了这种通信协议，比如AT91RM9200。

SPI分为主、从两种模式，一个SPI通讯系统需要包含一个（且只能是一个）主设备，一个或多个从设备。SPI接口的读写操作，都是由主设备发起。当存在多个从设备时，通过各自的片选信号进行管理。

优点：支持全双工通信、通信简单、数据传输速率快；
缺点：没有指定的流控制，没有应答机制确认是否接收到数据，所以跟IIC总线协议比较在数据的可靠性上有一定的缺陷。
STM32中SPI接口的特点

3线全双工同步传输；
8或16位传输帧格式选择；
主或从操作，支持多主模式；
主模式和从模式下均可以由软件或硬件进行NSS管理：主/从操作模式的动态改变；
可编程的时钟极性和相位；
可编程的数据顺序，MSB在前或LSB在前；
可触发中断的专用发送和接收标志；
SPI总线忙状态标志；
支持可靠通信的硬件CRC；
可触发中断的主模式故障、过载以及CRC错误标志；
支持DMA功能的1字节发送和接收缓冲器：产生发送和接受请求。


SPI协议
SPI引脚说明
SPI的通信原理很简单，它以主从方式工作，这种模式通常有一个主设备和一个或多个从设备，需要至少4根线，事实上3根也可以（单向传输时）。这四根线分别是MISO、MOSI、SCLK、CS，具体的描述见下表：

SPI各根线的描述
名称	描述
MISO	主设备数据输出，从设备数据输入
MOSI	主设备数据输出，从设备数据输入
SCLK	时钟信号，主设备产生
CS	片选信号，主设备控制
CS：控制芯片是否被选中的，也就是说只有片选信号为预先规定的使能信号时（一般默认为低电位），对此芯片的操作才有效，这就允许在同一总线上连接多个SPI设备成为可能。

也就是说：当有多个从设备的时候，因为每个从设备上都有一个片选引脚接入到主设备机中，当我们的主设备和某个从设备通信时将需要将从设备对应的片选引脚电平拉低。



MISO/MOSI/SCLK：通讯是通过数据交换完成的，这里先要知道SPI是串行通讯协议，也就是说数据是一位一位的传输的。这就是SCLK时钟线存在的原因，由SCLK提供时钟脉冲，MISO，MOSI则基于此脉冲完成数据传输。数据输出通过MOSI线，数据在时钟上升沿或下降沿时采样，同时也会有返回数据用于接受。完成一位数据传输，输入也使用同样原理。这样，在至少8次时钟信号的改变（上沿和下沿为一次），就可以完成8位数据的传输。

要注意的是：

SCLK信号线只由主设备控制，从设备不能控制信号线。同样，在一个基于SPI的设备中，至少有一个主控设备；
在点对点的通信中，SPI接口不需要进行寻址操作，且为全双工通信，显得简单高效。在多个从设备的系统中，每个从设备需要独立的使能信号，硬件上比I2C系统要稍微复杂一些。
SPI通讯模式
SPI通信有4种不同的模式，不同的从设备可能在出厂是就是配置为某种模式，这是不能改变的；但我们的通信双方必须是工作在同一模式下，所以我们可以对我们的主设备的SPI模式进行配置，通过CPOL（时钟极性）和CPHA（时钟相位）来控制我们主设备的通信模式，具体如下：

SPI通讯模式
模式	CPOL（时钟极性）	CPHA（时钟相位）
MODE0	0	0
MODE1	0	1
MODE2	1	0
MODE3	1	1
时钟极性CPOL是用来配置SCLK的电平出于哪种状态时是空闲态或者有效态，时钟相位CPHA是用来配置数据采样是在第几个边沿：

CPOL=0，表示当SCLK=0时处于空闲态，所以有效状态就是SCLK处于高电平时；
CPOL=1，表示当SCLK=1时处于空闲态，所以有效状态就是SCLK处于低电平时；
CPHA=0，表示数据采样是在第1个边沿，数据发送在第2个边沿；
CPHA=1，表示数据采样是在第2个边沿，数据发送在第1个边沿。
具体四种模式的时序图如下：



对于SPI的四种通讯模式，总结起来，就是：

CPOL=0，CPHA=0：此时空闲态时，SCLK处于低电平，数据采样是在第1个边沿，也就是SCLK由低电平到高电平的跳变，所以数据采样是在上升沿；
CPOL=0，CPHA=1：此时空闲态时，SCLK处于低电平，数据发送是在第1个边沿，也就是SCLK由低电平到高电平的跳变，所以数据采样是在下降沿；
CPOL=1，CPHA=0：此时空闲态时，SCLK处于高电平，数据采集是在第1个边沿，也就是SCLK由高电平到低电平的跳变，所以数据采集是在下降沿；
CPOL=1，CPHA=1：此时空闲态时，SCLK处于高电平，数据发送是在第1个边沿，也就是SCLK由高电平到低电平的跳变，所以数据采集是在上升沿。
SPI内部工作机制
下面对照一个SPI单主机与单从机连接图，理解其内部工作机制：



硬件上为4根线；
主机和从机都有一个串行移位寄存器，主机通过向它的SPI串行寄存器写入一个字节来发起一次传输；
串行移位寄存器通过MOSI信号线将字节传送给从机，同时从机也将自己的串行移位寄存器中的内容通过MISO信号线返回给主机。这样，两个移位寄存器中的内容就被交换；
外设的写操作和读操作是同步完成的。如果只进行写操作，主机只需忽略接收到的字节；反之，若主机要读取从机的一个字节，就必须发送一个空字节来引发从机的传输。
也就是说：SPI是一个环形总线结构，由CS、SCLK、MISO、MOSI构成，其时序其实很简单，主要是在SCLK的控制下，数据按照从高位到低位的方式依次移出主机寄存器和从机寄存器，并且依次移入从机寄存器和主机寄存器。当寄存器中的内容全部移出时，相当于完成了两个寄存器内容的交换。

假设主机的8位寄存器装的是待发送的数据10101010，上升沿发送、下降沿接收、高位先发送。那么第一个上升沿来的时候，主机将会通过MOSI信号线传输给从机最高位1，自身寄存器变成0101010x。同时，MISO信号线会从从机处返回一个数据给主机，那么这时寄存器为0101010MISO，这样在 8个时钟脉冲以后，两个寄存器的内容互相交换一次。这样就完成里一个SPI时序。 

这个时候就会有一个疑问，或者说产生一个必然了：

为什么主机发送一个数据给从机，从机就同时通过MISO返回的一个数据给主机呢？

解释：主机和从机的发送数据是同时完成的，两者的接收数据也是同时完成的。也就是说，当上升沿主机发送数据的时候，从机也发送了数据。

所以为了保证主从机正确通信，应使得它们的SPI具有相同的时钟极性和时钟相位。

 

STM32的SPI接口
SPI可分为主、从两种模式，并且支持全双工模式，所以这也就导致STM32的SPI接口比较复杂。比如：配置SPI为主模式、配置SPI为从模式、配置SPI为单工通信、配置SPI为双工通信等等。这里的内容就非常庞大，涉及到的寄存器的位也比较多，所以就不介绍太多，想要了解更多可以去查看STM32F1xx官方资料的第23章节。

SPI接口的框图


SPI引脚
STM32的SPI接口通过4个引脚与外部器件相连，与标准的SPI协议是一致的：

MISO：主设备输入/从设备输出引脚。该引脚在从模式下发送数据，在主模式下接收数据；
MOSI：主设备输出/从设备输入引脚。该引脚在主模式下发送数据，在从模式下接收数据；
SCK：串口时钟，作为主设备的输入，从设备的输入；
NSS：从设备选择。这是一个可选的引脚，用来选择主/从设备。它的功能是用来作为“片选引脚”，让主设备可以单独地与特定从设备通讯，避免数据线上的冲突。
从选择（NSS）脚管理

有2种NSS模式：

软件NSS模式：可以通过设置SPI_CR1寄存器的SSM位来使能这种模式。在这种模式下NSS引脚可以用作它用，而内部NSS信号电平可以通过写SPI_CR1的SSI位来驱动；
硬件NSS模式，分两种情况：
NSS输出被使能：当STM32F10xxx工作为主SPI，并且NSS输出已经通过SPI_CR2寄存器的SSOE位使能，这时NSS引脚被拉低，所有NSS引脚与这个主SPI的NSS引脚相连并配置为硬件NSS的SPI设备，将自动变成从SPI设备。 当一个SPI设备需要发送广播数据，它必须拉低NSS信号，以通知所有其它的设备它是主设备；如果它不能拉低NSS，这意味着总线上有另外一个主设备在通信，这时将产生一个硬件失败错误；
NSS输出被关闭：允许操作于多主环境。
数据帧格式
根据SPI_CR1寄存器中的LSBFIRST位，输出数据位时可以左对齐（MSB对齐标准）也可以右对齐（LSB对齐标准）。
根据SPI_CR1寄存器的DFF位，每个数据帧可以是8位或是16位。所选择的数据帧格式对发送和/或接收都有效。
状态标志
应用程序通过3个状态标志可以完全监控SPI总线的状态：

发送缓冲器空闲标志（TXE）
此标志为1时表明发送缓冲器为空，可以写下一个待发送的数据进入缓冲器中。当写入SPI_DR时，TXE标志被清除。

接收缓冲器非空（RXNE）
此标志为1时表明在接收缓冲器中包含有效的接收数据。读SPI数据寄存器可以清除此标志。

忙（Busy）标志
BSY标志由硬件设置与清除（写入此位无效果），此标志表明SPI通信层的状态。

当它被设置为1时，表明SPI正忙于通信，但有一个例外：在主模式的双向接收模式下（MSTR=1、BDM=1并且BDOE=0），在接收期间BSY标志保持为低。

在软件要关闭SPI模块并进入停机模式(或关闭设备时钟)之前，可以使用BSY标志检测传输是否结束，这样可以避免破坏最后一次传输，因此需要严格按照下述过程执行。

SPI中断




STM32的SPI引脚
SPI引脚位置


外设的GPIO配置




SPI相关配置库函数
1个初始化函数
void SPI_Init(SPI_TypeDef* SPIx, SPI_InitTypeDef* SPI_InitStruct);
作用：初始化SPI的相关参数，比如方向（全双工）、主从模式、数据大小、CPOL、CPHA、片选软件模式、预分频系数等。

3个使能函数
void SPI_Cmd(SPI_TypeDef* SPIx, FunctionalState NewState);
void SPI_I2S_ITConfig(SPI_TypeDef* SPIx, uint8_t SPI_I2S_IT, FunctionalState NewState);
void SPI_I2S_DMACmd(SPI_TypeDef* SPIx, uint16_t SPI_I2S_DMAReq, FunctionalState NewState);
作用：使能SPI接口；使能SPI中断；使能SPI的DMA功能。

2个数据传输函数
void SPI_I2S_SendData(SPI_TypeDef* SPIx, uint16_t Data);
uint16_t SPI_I2S_ReceiveData(SPI_TypeDef* SPIx);
作用：分别用于SPI传输数据、接收数据。

4个状态位函数
FlagStatus SPI_I2S_GetFlagStatus(SPI_TypeDef* SPIx, uint16_t SPI_I2S_FLAG);
void SPI_I2S_ClearFlag(SPI_TypeDef* SPIx, uint16_t SPI_I2S_FLAG);
ITStatus SPI_I2S_GetITStatus(SPI_TypeDef* SPIx, uint8_t SPI_I2S_IT);
void SPI_I2S_ClearITPendingBit(SPI_TypeDef* SPIx, uint8_t SPI_I2S_IT);
作用：前两者用于获得和清除SPI的各种状态位；后两者则针对SPI的中断标志位。

 

SPI一般步骤
实验目标：利用SPI2进行初始化等操作。

配置相关引脚的复用功能，使能SPIx时钟。调用函数：void GPIO_Init()；
初始化SPIx，设置SPIx工作模式。调用函数：void SPI_Init()；
使能SPIx。调用函数：void SPI_Cmd()；
SPI传输数据。调用函数：void SPI_I2S_SendData()；uint16_t SPI_I2S_ReceiveData()；
查看SPI传输状态。调用函数：SPI_I2S_GetFlagStatus(SPI2, SPI_I2S_FLAG_RXNE)。
下面按照这个一般步骤来进行一个简单的SPI程序：

void SPI2_Init(void)
{
 	GPIO_InitTypeDef GPIO_InitStructure;
  SPI_InitTypeDef  SPI_InitStructure;

	RCC_APB2PeriphClockCmd(	RCC_APB2Periph_GPIOB, ENABLE );//PORTB时钟使能 
	RCC_APB1PeriphClockCmd(	RCC_APB1Periph_SPI2,  ENABLE );//SPI2时钟使能 	
	 
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13 | GPIO_Pin_14 | GPIO_Pin_15;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;  //PB13/14/15复用推挽输出 
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOB, &GPIO_InitStructure);//初始化GPIOB
	 
	GPIO_SetBits(GPIOB,GPIO_Pin_13|GPIO_Pin_14|GPIO_Pin_15);  //PB13/14/15上拉
	 
	SPI_InitStructure.SPI_Direction = SPI_Direction_2Lines_FullDuplex;  //设置SPI单向或者双向的数据模式:SPI设置为双线双向全双工
	SPI_InitStructure.SPI_Mode = SPI_Mode_Master;		//设置SPI工作模式:设置为主SPI
	SPI_InitStructure.SPI_DataSize = SPI_DataSize_8b;		//设置SPI的数据大小:SPI发送接收8位帧结构
	SPI_InitStructure.SPI_CPOL = SPI_CPOL_High;		//串行同步时钟的空闲状态为高电平
	SPI_InitStructure.SPI_CPHA = SPI_CPHA_2Edge;	//串行同步时钟的第二个跳变沿（上升或下降）数据被采样
	SPI_InitStructure.SPI_NSS = SPI_NSS_Soft;		//NSS信号由硬件（NSS管脚）还是软件（使用SSI位）管理:内部NSS信号有SSI位控制
	SPI_InitStructure.SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_256;		//定义波特率预分频的值:波特率预分频值为256
	SPI_InitStructure.SPI_FirstBit = SPI_FirstBit_MSB;	//指定数据传输从MSB位还是LSB位开始:数据传输从MSB位开始
	SPI_InitStructure.SPI_CRCPolynomial = 7;	//CRC值计算的多项式
	SPI_Init(SPI2, &SPI_InitStructure);  //根据SPI_InitStruct中指定的参数初始化外设SPIx寄存器
	 
	SPI_Cmd(SPI2, ENABLE); //使能SPI外设
	
	SPI2_ReadWriteByte(0xff);//启动传输		 

 


```
}   
//SPI 速度设置函数
//SpeedSet:
//SPI_BaudRatePrescaler_2   2分频   
//SPI_BaudRatePrescaler_8   8分频   
//SPI_BaudRatePrescaler_16  16分频  
//SPI_BaudRatePrescaler_256 256分频 

void SPI2_SetSpeed(u8 SPI_BaudRatePrescaler)
{
  assert_param(IS_SPI_BAUDRATE_PRESCALER(SPI_BaudRatePrescaler));
	SPI2->CR1&=0XFFC7;
	SPI2->CR1|=SPI_BaudRatePrescaler;	//设置SPI2速度 
	SPI_Cmd(SPI2,ENABLE); 

} 

//SPIx 读写一个字节
//TxData:要写入的字节
//返回值:读取到的字节
u8 SPI2_ReadWriteByte(u8 TxData)
{		
	u8 retry=0;				 	
	while (SPI_I2S_GetFlagStatus(SPI2, SPI_I2S_FLAG_TXE) == RESET) //检查指定的SPI标志位设置与否:发送缓存空标志位
		{
		retry++;
		if(retry>200)return 0;
		}			  
	SPI_I2S_SendData(SPI2, TxData); //通过外设SPIx发送一个数据
	retry=0;
```



	while (SPI_I2S_GetFlagStatus(SPI2, SPI_I2S_FLAG_RXNE) == RESET) //检查指定的SPI标志位设置与否:接受缓存非空标志位
		{
		retry++;
		if(retry>200)return 0;
		}	  						    
	return SPI_I2S_ReceiveData(SPI2); //返回通过SPIx最近接收的数据					    
```
}
```
