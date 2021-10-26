# Linux I2C驱动实验

## I2C简介

I2C是很常见的一种总线协议，I2C是NXP公司设计的，I2C使用两条线在主控制器和从机之间进行数据通信。一条是SCL（串行时钟线），另外一条是SDA（串行数据线），这两条数据线需要接上拉电阻，总线空闲的时候SCL和SDA处于高电平。I2C总线标准模式下速度可以达到100Kb/S，快速模式下可以达到400Kb/S。I2C总线工作是按照一定的协议来运行的，接下来就看一下I2C协议。

I2C是支持多从机的，也就是一个I2C控制器下可以挂多个I2C从设备，这些不同的I2C从设备有不同的器件地址，这样I2C主控制器就可以通过I2C设备的器件地址访问指定的I2C设备了，一个I2C总线连接多个I2C设备如下图所示：

![image-20211018093916514](http://img.devenyu.top/img/image-20211018093916514.png)

图中SDA和SCL这两根线必须要接一个上拉电阻，一般是4.7K。其余的I2C从器件都挂接到SDA和SCL这两根线上，这样就可以通过SDA和SCL这两根线来访问多个I2C设备。

接下来看一下I2C协议有关的术语：

**1、起始位**

顾名思义，也就是I2C通信起始标志，通过这个起始位就可以告诉I2C从机，”我“要开始进行I2C通信了。在SCL为高电平的时候，SDA出现下降沿就表示为起始位，如下图所示：

​												![image-20211018105222416](http://img.devenyu.top/img/image-20211018105222416.png)

**2、停止位**

停止位就是停止I2C通信的标志位，和起始位的功能相反。在SCL位高电平的时候，SDA出现上升沿就表示为停止位，如下图所示：

![image-20211018105500295](http://img.devenyu.top/img/image-20211018105500295.png)

**3、数据传输**

I2C总线在数据传输的时候要保证在SCL高电平期间，SDA上的数据稳定，因此SDA上的数据变化只能在SCL低电平期间发生，如下图所示：

![image-20211018105644212](http://img.devenyu.top/img/image-20211018105644212.png)

**4、应答信号**

当I2C主机发送完8位数据以后会将SDA设置为输入状态，等待I2C从机应答，也就是等待I2C从机告诉主机它接受到数据了。应答信号是由从机发出的，主机需要提供应答信号所需的时钟，主机发送完8位数据以后紧跟着的一个时钟信号就是给应答信号使用的。从机通过将SDA拉低来表示发出应答信号，表示通信成功，否则表达通信失败。

**5、I2C写时序**

主机通过I2C总线与从机之间进行通信不外乎两个操作：写和读，I2C总线单字节写时序如图所示：

![image-20211018110621133](http://img.devenyu.top/img/image-20211018110621133.png)

写时序的具体步骤：

1）、开始信号。

2）、发送I2C设备地址，每个I2C器件都有一个设备地址，通过发送具体的设备地址来决定访问哪个I2C器件。这是一个8位的数据，其中高7位是设备地址，最后一位是读写位，为1的话表示这是一个读操作，为0的话表示这是一个写操作。

3）、I2C器件地址后面跟着一个读写位，为0表示写操作，为1表示读操作。

4）、从机发送的ACK应答信号。

5）、重新发送开始信号。

6）、发送要写入数据的寄存器地址。

7）、从机发送的ACK应答信号。

8）、发送要写入寄存器的数据。

9）、从机发送的ACK应答信号。

10）、停止信号。

6、I2C读时序

I2C总线单字节读时序如下图所示：

![image-20211018111747338](http://img.devenyu.top/img/image-20211018111747338.png)

I2C单字节读时序比写时序要复杂一点，读时序分为4大步，第一步是发送设备地址，第二步是发送要读取的寄存器地址，第三步重新发送设备地址，最后一步就是I2C从器件输出要读取的寄存器值，具体步骤如下：

1）、主机发送起始信号。

2）、主机发送要读取的I2C从设备地址。

3）、读写控制位，因为是向I2C从设备发送数据，因此是写信号。

4）、从机发送的ACK应答信号。

5）、重新发送START信号。

6）、主机发送要读取的寄存器地址。

7）、从机发送的ACK应答信号。

8）、重新发送START信号。

9）、重新发送要读取I2C从设备地址。

10）、读写控制位，这里是读信号，表示接下来是从I2C从设备里面读取数据。

11）、从机发送的ACK应答信号。

12）、从I2C器件里面读取到的数据。

13）、主机发出NO ACK信号，表示读取完成，不需要从机再发送ACK信号了。

14）、主机发送STOP信号，停止I2C通信。

**7、I2C多字节读写时序**

有时候我们需要读写多个字节，多字节读写时序和单字节的基本一致，只是在读写数据的时候可以连续发送多个自己的数据，其他的控制时序都是和单字节一样的。

## Linux I2C总线框架简介

使用裸机的方式编写一个I2C器件的驱动程序，我们一般需要实现两部分：

①、I2C主机驱动。

②、I2C设备驱动。

I2C主机驱动也就是SoC的I2C控制器对应的驱动程序，I2C设备驱动其实就是挂在I2C总线下的具体设备对应的驱动程序，例如eeprom、触摸屏IC、传感器IC等；对于主机驱动来说，一旦编写完成就不需要再做修改，其他的I2C设备直接调用主机驱动提供的API函数完成读写操作即可。这个正好符合Linux的驱动分离与分层的思想，因此Linux内核也将I2C驱动分为两部分。

Linux内核开发者为了让驱动开发工程师在内核中方便的添加自己的I2C设备驱动程序，方便大家更容易的在Linux下驱动自己的I2C接口硬件，进而引入了I2C总线框架，我们一般也叫作I2C子系统，Linux下I2C子系统总体框架如下所示：

![image-20211018132132297](http://img.devenyu.top/img/image-20211018132132297.png)

由图可知，I2C子系统分为三大组成部分：

**1、I2C核心（I2C-core)**

​		I2C核心提供了I2C总线驱动（适配器）和设备驱动的注册、注销方法，I2C通信方法与具体硬件无关的代码，以及探测设备地址的上层代码等；

**2、I2C总线驱动（I2C adapter）**

​		I2C总线驱动是I2C适配器的软件实现，提供I2C适配器与从设备间完成数据通信的能力。I2C总线驱动由i2c_adapter和i2c_algorithm来描述。I2C适配器是SoC中内置i2c控制器的软件抽象，可以理解为他所代表的是一个I2C主机；

**3、I2C设备驱动（I2C client driver)**

包括两部分：设备的注册和驱动的注册。

I2C子系统帮助内核统一管理I2C设备，让驱动开发工程师在内核中可以更加容易地添加自己的I2C设备驱动程序。

## I2C总线驱动

I2C总线驱动重点是I2C适配器（也就是SoC的I2C接口控制器）驱动，这里要用到两个重要的数据结构：i2c_adapter和i2c_algorithm,I2C子系统将SoC的I2C适配器（控制器）抽象成一个i2c_adapter结构体，i2c_adapter结构体定义在include/linux/i2c.h文件中，结构体内容如下：

```c
685 struct i2c_adapter { 
686     struct module *owner; 
    687     unsigned int class;        
688     const struct i2c_algorithm *algo;  
689     void *algo_data; 
690 
691     /* data fields that are valid for all devices   */ 
692     const struct i2c_lock_operations *lock_ops; 
693     struct rt_mutex bus_lock; 
694     struct rt_mutex mux_lock; 
695 
696     int timeout;                /* in jiffies */ 
697     int retries; 
698     struct device dev;          /* the adapter device */ 
699     unsigned long locked_flags; /* owned by the I2C core */ 
700 #define I2C_ALF_IS_SUSPENDED          0 
701 #define I2C_ALF_SUSPEND_REPORTED      1 
702 
703     int nr; 
704     char name[48]; 
705     struct completion dev_released; 
706 
707     struct mutex userspace_clients_lock; 
708     struct list_head userspace_clients; 
709 
710     struct i2c_bus_recovery_info *bus_recovery_info; 
711     const struct i2c_adapter_quirks *quirks; 
712 
713     struct irq_domain *host_notify_domain; 
714 }; 
```

第  688  行，i2c_algorithm  类型的指针变量  algo，对于一个  I2C  适配器，肯定要对外提供
读写  API  函数，设备驱动程序可以使用这些  API  函数来完成读写操作。i2c_algorithm  就是 
I2C  适配器与  IIC  设备进行通信的方法。 
	i2c_algorithm  结构体定义在  include/linux/i2c.h  文件中，内容如下： 

```c
526 struct i2c_algorithm { 
527     /* 
528      * If an adapter algorithm can't do I2C-level access, set  
529      * master_xfer to NULL. If an adapter algorithm can do SMBus  
530      * access, set smbus_xfer. If set to NULL, the SMBus protocol is  
531      * simulated using common I2C messages. 
532      * 
533      * master_xfer should return the number of messages successfully 
534      * processed, or a negative value on error 
535      */ 
536     int (*master_xfer)(struct i2c_adapter *adap,  
struct i2c_msg *msgs, 
537                int num); 
538     int (*master_xfer_atomic)(struct i2c_adapter *adap, 
539                    struct i2c_msg *msgs, int num); 
540     int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr, 
541               unsigned short flags, char read_write, 
542               u8 command, int size, union i2c_smbus_data *data); 
543     int (*smbus_xfer_atomic)(struct i2c_adapter *adap, u16 addr, 
544                  unsigned short flags, char read_write, 
545                  u8 command, int size, union i2c_smbus_data *data); 
546 
547     /* To determine what the adapter supports */ 
548     u32 (*functionality)(struct i2c_adapter *adap); 
549 
550 #if IS_ENABLED(CONFIG_I2C_SLAVE) 
551     int (*reg_slave)(struct i2c_client *client); 
552     int (*unreg_slave)(struct i2c_client *client); 
553 #endif 
554 }; 
```

第 536 行，master_xfer 就是 I2C 适配器的传输函数，可以通过此函数来完成与 IIC 设备之
间的通信。 
		第 540 行，smbus_xfer 就是 SMBUS 总线的传输函数。smbus 协议是从 I2C 协议的基础上
发展而来的，他们之间有很大的相似度，SMBus 与 I2C 总线之间在时序特性上存在一些差别，
应用于移动 PC 和桌面 PC 系统中的低速率通讯。 
		综上所述，I2C 总线驱动，或者说 I2C 适配器驱动的主要工作就是初始化 i2c_adapter 结构
体变量，然后设置 i2c_algorithm 中的 master_xfer 函数。完成以后通过 i2c_add_numbered_adapter或 i2c_add_adapter 这两个函数向 I2C 子系统注册设置好的 i2c_adapter，这两个函数的原型如下：

```c
int i2c_add_adapter(struct i2c_adapter *adapter) 
int i2c_add_numbered_adapter(struct i2c_adapter *adap) 
```

这 两 个 函 数 的 区 别 在 于 i2c_add_adapter 会 动 态 分 配 一 个 总 线 编 号 ， 而
i2c_add_numbered_adapter 函数则指定一个静态的总线编号。函数参数和返回值含义如下： 
		adapter 或 adap：要添加到 Linux 内核中的 i2c_adapter，也就是 I2C 适配器。 
		返回值：0，成功；负值，失败。 
		如果要删除 I2C 适配器的话使用 i2c_del_adapter 函数即可，函数原型如下： 

```c
void i2c_del_adapter(struct i2c_adapter * adap) 
```

函数参数和返回值含义如下： 
		adap：要删除的 I2C 适配器。 
		返回值：无。 
		关于 I2C 的总线(控制器或适配器)驱动就讲解到这里，一般 SoC 的 I2C 总线驱动都是由半
导体厂商编写的，比如 STM32MP1 的 I2C 适配器驱动 ST 官方已经编写好了，这个不需要用户
去编写。因此 I2C 总线驱动对我们这些 SoC 使用者来说是被屏蔽掉的，我们只要专注于 I2C 设
备驱动即可，除非你是在半导体公司上班，工作内容就是写 I2C 适配器驱动。

## I2C总线设备

I2C 设备驱动重点关注两个数据结构：i2c_client 和 i2c_driver，根据总线、设备和驱动模型，
I2C 总线上一小节已经讲了。还剩下设备和驱动，i2c_client 用于描述 I2C 总线下的设备，
i2c_driver 则用于描述 I2C 总线下的设备驱动，类似于 platform 总线下的 platform_device 和
platform_driver。 

**1、i2c_client 结构体** 
		i2c_client 结构体定义在 include/linux/i2c.h 文件中，内容如下： 

```c
313     struct i2c_client { 
314         unsigned short flags;          /* div., see below        */ 
...... 
328         struct i2c_adapter *adapter;  /* the adapter we sit on */ 
329         struct device dev;            /* the device structure  */ 
330         int init_irq;            /* irq set at initialization    */ 
331         int irq;                /* irq issued by device        */ 
332         struct list_head detected; 
333     #if IS_ENABLED(CONFIG_I2C_SLAVE) 
334         i2c_slave_cb_t slave_cb;    /* callback for slave mode   */ 
335     #endif 
336     }; 
```

一个 I2C 设备对应一个 i2c_client 结构体变量，系统每检测到一个 I2C 从设备就会给这个
设备分配一个 i2c_client。 

**2、i2c_driver 结构体** 
		i2c_driver 类似 platform_driver，是我们编写 I2C 设备驱动重点要处理的内容，i2c_driver 结
构体定义在 include/linux/i2c.h 文件中，内容如下： 

2、i2c_driver 结构体 
i2c_driver 类似 platform_driver，是我们编写 I2C 设备驱动重点要处理的内容，i2c_driver 结
构体定义在 include/linux/i2c.h 文件中，内容如下： 

```c
253     struct i2c_driver { 
254         unsigned int class; 
255  
256         /* Standard driver model interfaces */ 
257         int (*probe)(struct i2c_client *client,  
const struct i2c_device_id *id); 
258         int (*remove)(struct i2c_client *client); 
259  
260         /* New driver model interface to aid the seamless removal of  
261          * the current probe()'s, more commonly unused than used  
262          second parameter.*/ 
263         int (*probe_new)(struct i2c_client *client); 
264  
265       /* driver model interfaces that don't relate to enumeration  */ 
266         void (*shutdown)(struct i2c_client *client); 
267  
268         /* Alert callback, for example for the SMBus alert protocol. 
269          * The format and meaning of the data value depends on the  
270          * protocol. For the SMBus alert protocol, there is a single  
271          * bit of data passed as the alert response's low bit ("event  
272          * flag"). For the SMBus Host Notify protocol, the data  
273          * corresponds to the 16-bit payload data reported by the  
274          slave device acting as master.*/ 
275         void (*alert)(struct i2c_client *client,  
enum i2c_alert_protocol protocol, 
276                          unsigned int data); 
277  
278         /* a ioctl like command that can be used to perform specific  
279          * functions with the device. 
280          */ 
281         int (*command)(struct i2c_client *client, unsigned int cmd,  
void *arg); 
282  
283         struct device_driver driver; 
284         const struct i2c_device_id *id_table; 
285  
286         /* Device detection callback for automatic device creation */ 
287         int (*detect)(struct i2c_client *client,  
struct i2c_board_info *info); 
288         const unsigned short *address_list; 
289         struct list_head clients; 
290  
291         bool disable_i2c_core_irq_mapping; 
292     }; 
```

第 257 行，当 I2C 设备和驱动匹配成功以后 probe 函数就会执行，和 platform 驱动一样。 
		第 283 行，device_driver 驱动结构体，如果使用设备树的话，需要设置 device_driver 的
of_match_table 成员变量，也就是驱动的兼容(compatible)属性。 
		第 284 行，id_table 是传统的、未使用设备树的设备匹配 ID 表。 
		对于我们 I2C 设备驱动编写人来说，重点工作就是构建 i2c_driver，构建完成以后需要向
I2C 子系统注册这个 i2c_driver。i2c_driver 注册函数为 int i2c_register_driver，此函数原型如下：

```c
int i2c_register_driver(struct module      *owner,   
              struct i2c_driver    *driver) 
```

 函数参数和返回值含义如下： 
		owner：一般为 THIS_MODULE。 
		driver：要注册的 i2c_driver。 
		返回值：0，成功；负值，失败。 
		另外 i2c_add_driver 也常常用于注册 i2c_driver，i2c_add_driver 是一个宏，定义如下： 

