# Linux 控制LED及其驱动开发

## 一、在linux上控制LED

```
cd /sys/class/gpio/
echo 906 > export
echo out > gpio906/direction
echo 1 > gpio906/value		//led熄灭
```

## 二、驱动开发

- 查看硬件原理图以及数据手册

这里我们以AX7020开发板为例，打开AX7020开发板原理图，查看PS LED1的连接方式。

![image-20210908165542579](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210908165542579.png)

可知目标led连接在开发板PS端的MIO0引脚上。

再打开数据手册《ug585-Zynq-7000-TRM.pdf》。根据关键词MIO我们能找到一张MIO对应表在2.4.4节。

![image-20210908165851591](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210908165851591.png)

我们可以知道led的操作对应的引脚功能为GPIO0，MIO0对应GPIO0。我们再到寄存器设置"Appendix B: Register Details"章节中找到GPIO的寄存器设置。

![image-20210908170110064](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210908170110064.png)

GPIO寄存器的基地址为0xE000A000，对于GPIO的设置，需要使能、设置方向以及控制

![image-20210908170351325](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210908170351325.png)

由此得使能寄存器的地址为0xE000A208 、方向寄存器的地址为0xE000A204

![image-20210908170605698](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210908170605698.png)

由此得控制（数据）寄存器的地址为0xE000A040

- 编写字符设备驱动程序

  上面我们已绊得到了 led 驱劢的大体框架，接下来要做的实际就是晚上讴备操作凼数中的

内容，也是再对应的凼数中讴置相关的寄存器。在 open()凼数中实现 led 初始化，在 write()凼数

中实现 led 的控制，在 release()函数中完成 led 的使能。最终的带代码如下：

```c
 #include <linux/module.h>
 #include <linux/kernel.h>
 #include <linux/fs.h>
 #include <linux/init.h>
 #include <linux/ide.h>
 #include <linux/types.h>

 /* 驱动名称 */
 #define DEVICE_NAME "gpio_leds"
 /* 驱动主设备号 */
 #define GPIO_LED_MAJOR 200

 /* gpio 寄存器虚拟地址 */
 static unsigned int gpio_add_minor;
 /* gpio 寄存器物理基地址 */
 #define GPIO_BASE 0xE000A000
 /* gpio 寄存器所占空间大小 */
 #define GPIO_SIZE 0x1000
 /* gpio 方向寄存器 */
 #define GPIO_DIRM_0 (unsigned int *)(0xE000A204 - GPIO_BASE + gpio_add_minor)
 /* gpio 使能寄存器 */
 #define GPIO_OEN_0 (unsigned int *)(0xE000A208 - GPIO_BASE + gpio_add_minor)
 /* gpio 控制寄存器 */
 #define GPIO_DATA_0 (unsigned int *)(0xE000A040 - GPIO_BASE + gpio_add_minor)

 /* 时钟使能寄存器虚拟地址 */
 static unsigned int clk_add_minor;
 /* 时钟使能寄存器物理基地址 */
 #define CLK_BASE 0xF8000000
 /* 时钟使能寄存器所占空间大小 */
 #define CLK_SIZE 0x1000
 /* AMBA 外设时钟使能寄存器 */
 #define APER_CLK_CTRL (unsigned int *)(0xF800012C - CLK_BASE + clk_add_minor)

 /* open 函数实现, 对应到 Linux 系统调用函数的 open 函数 */
 static int gpio_leds_open(struct inode *inode_p, struct file *file_p)
 {
 /* 把需要修改的物理地址映射到虚拟地址 */
 gpio_add_minor = (unsigned int)ioremap(GPIO_BASE, GPIO_SIZE);
 clk_add_minor = (unsigned int)ioremap(CLK_BASE, CLK_SIZE);

 /* MIO_0 时钟使能 */
 *APER_CLK_CTRL |= 0x00400000;
 /* MIO_0 设置成输出 */
 *GPIO_DIRM_0 |= 0x00000001;
 /* MIO_0 使能 */
 *GPIO_OEN_0 |= 0x00000001;
     
 printk("gpio_test module open\n");

 return 0;
 }


 /* write 函数实现, 对应到 Linux 系统调用函数的 write 函数 */
 static ssize_t gpio_leds_write(struct file *file_p, const char __user *buf, size_t len, loff_t *
loff_t_p)
 {
 int rst;
 char writeBuf[5] = {0};

 printk("gpio_test module write\n");

 rst = copy_from_user(writeBuf, buf, len);
 if(0 != rst)
 {
 return -1;
 }

 if(1 != len)
 {
 printk("gpio_test len err\n");
 return -2;
 }
 if(1 == writeBuf[0])
 {
 *GPIO_DATA_0 &= 0xFFFFFFFE;
 printk("gpio_test ON\n");
 }
 else if(0 == writeBuf[0])
 {
 *GPIO_DATA_0 |= 0x00000001;
 printk("gpio_test OFF\n");
 }
 else
 {
 printk("gpio_test para err\n");
 return -3;
 }

 return 0;
 }

 /* release 函数实现, 对应到 Linux 系统调用函数的 close 函数 */
 static int gpio_leds_release(struct inode *inode_p, struct file *file_p)
 {
 printk("gpio_test module release\n");
 return 0;
 }

 /* file_operations 结构体声明, 是上面 open、write 实现函数与系统调用函数对应的关键 */
 static struct file_operations gpio_leds_fops = {
 .owner = THIS_MODULE,
 .open = gpio_leds_open,
 .write = gpio_leds_write,
 .release = gpio_leds_release,
 };

 /* 模块加载时会调用的函数 */
 static int __init gpio_led_init(void)
 {
 int ret;

 /* 通过模块主设备号、名称、模块带有的功能函数(及 file_operations 结构体)来注册模块 */
 ret = register_chrdev(GPIO_LED_MAJOR, DEVICE_NAME, &gpio_leds_fops);
 if (ret < 0)
 {
 printk("gpio_led_dev_init_ng\n");
 return ret;
 }
 else
 {
 /* 注册成功 */
 printk("gpio_led_dev_init_ok\n");
 }
     return 0;
 }

 /* 卸载模块 */
 static void __exit gpio_led_exit(void)
 {
 /* 释放对虚拟地址的占用 */
 iounmap((unsigned int *)gpio_add_minor);
 iounmap((unsigned int *)clk_add_minor);
 /* 注销模块, 释放模块对这个设备号和名称的占用 */
 unregister_chrdev(GPIO_LED_MAJOR, DEVICE_NAME);

 printk("gpio_led_dev_exit_ok\n");
 }

 /* 标记加载、卸载函数 */
 module_init(gpio_led_init);
 module_exit(gpio_led_exit);

 /* 驱动描述信息 */
 MODULE_AUTHOR("Alinx");
 MODULE_ALIAS("gpio_led");
 MODULE_DESCRIPTION("GPIO LED driver");
 MODULE_VERSION("v1.0");
 MODULE_LICENSE("GPL");
```

**49** 行出现的 printk()函数，是内核态输出字符串到控制台的函数，这里用于调试，相当于应

用程序中的 printf()。printk()函数存在消息级别，定义在头文件 include/linux/kern_levels.h 中，

如下：

```c
#define KERN_EMERG KERN_SOH "" /* system is unusable */
#define KERN_ALERT KERN_SOH "" /* action must be taken immediately */
#define KERN_CRIT KERN_SOH "" /* critical conditions */
#define KERN_ERR KERN_SOH "" /* error conditions */
#define KERN_WARNING KERN_SOH "" /* warning conditions */
#define KERN_NOTICE KERN_SOH "" /* normal but significant condition */
#define KERN_INFO KERN_SOH "" /* informational */
#define KERN_DEBUG KERN_SOH "" /* debug-level messages */
```

其中0的优先级最高，7的优先级最低。如果要设置消息级别，可设置如下：

`		printk(KERN_INFO"gpio_test module open\n");`

如果不设置消息级别，那么printk()会采样默认级别MESSAGE_LOGLEVEL_DEFAULT。只有消息级别高于头文件include/linux/printk.h中定义的宏CONSOLE_LOGLEVEL_DEFAULT,消息才会被打印。

39行出现的函数ioremap()，用于把物理地址映射到虚拟地址。在Linux中由于MMU内存映射的关系，我们无法直接操作物理地址，而需要把物理地址映射到虚拟地址上再操作对应的虚拟地址。ioremap()定义在头文件arch/arm/include/asm/io.h中，如下：

`#define ioremap(cookie,size) __arm_ioremap((cookie), (size),MT_DEVICE)`

cookie指代物理地址，size为需要映射的地址长度。124行的代码即是把GPIO寄存器基地址范围是整个GPIO寄存器映射到虚拟地址并赋值给全局变量gpio_add_minor，之后可以直接对变量gpio_add_minor进行读写操作。

97行的iounmap()函数是与ioremap()相对的释放虚拟地址函数。输入参数为ioremap()函数返回得到的虚拟地址首地址，比如这里的gpio_add_minor。

剩下的就是寄存器操作了。对于虚拟地址的操作，我们这里都是直接通过指针访问的，但是Linux有推荐的读写方法，而不是使用指针，如下：

读函数：

`u8 readb(const volatile void __iomem *addr);`

`u16 readw(const volatile void __iomem *addr);`

`u32 readl(const volatile void __iomem *addr);`

写函数：

`void writeb(u8 value, volatile void __iomem *addr);`

`void writew(u16 value, volatile void __iomem *addr);`

`void writel(u32 value, volatile void __iomem *addr);`

value为需要写入的值，addr为需要操作的地址。

56行的write()函数中通过判断用户输入的__buf值为0还是1相应的执行点灯和灭灯的操作。
