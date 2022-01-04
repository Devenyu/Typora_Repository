## 一、Linux USB Gadget Driver功能

为了与主机端驱动设备的USB Device Driver概念进行区别，将在外围器件中运行的驱动程序称为USB Gadget Driver。其中，Host端驱动设备的驱动程序是master或者client driver，设备端gadget driver是slave或者function driver。
		Gadget Driver和USB Host端驱动程序类似，都是使用请求队列来对I/O包进行缓冲，这些请求可以被提交和取消。它们的结构、消息和常量的定义也和USB技术规范第九章的内容一致。同时也是通过bind和unbind将driver与device建立关系。

## 二、Linux USB Gadget Driver核心数据结构

1． USB_Gadget对象

```c
struct usb_gadget {
/* readonly to gadget driver */
const struct usb_gadget_ops *ops; //Gadget设备操作函数集
struct usb_ep *ep0; //控制端点，只对setup包响应
struct list_head ep_list;//将设备的所有端点连成链表，ep0不在其中
enum usb_device_speed speed;//高速、全速和低速
unsigned is_dualspeed:1; //是否同时支持高速和全速
unsigned is_otg:1; //是否支持OTG(On-To-Go)
unsigned is_a_peripheral:1;
unsigned b_hnp_enable:1;
unsigned a_hnp_support:1;
unsigned a_alt_hnp_support:1;
const char *name; //器件名称
struct device dev; //内核设备模型使用
};
```

2． Gadget器件操作函数集
操作UDC硬件的API，但操作端点的函数由端点操作函数集完成

```c
struct usb_gadget_ops {
int (*get_frame)(struct usb_gadget *);
int (*wakeup)(struct usb_gadget *);
int (*set_selfpowered) (struct usb_gadget *, int is_selfpowered);
int (*vbus_session) (struct usb_gadget *, int is_active);
int (*vbus_draw) (struct usb_gadget *, unsigned mA);
int (*pullup) (struct usb_gadget *, int is_on);
int (*ioctl)(struct usb_gadget *, unsigned code, unsigned long param);
};
```

3． USB Gadget driver对象

```c
struct usb_gadget_driver {
char *function; //驱动名称
enum usb_device_speed speed; //USB设备速度类型
int (*bind)(struct usb_gadget *); //将驱动和设备绑定，一般在驱动注册时调用
void (*unbind)(struct usb_gadget *);//卸载驱动时调用，rmmod时调用
int (*setup)(struct usb_gadget *, const struct usb_ctrlrequest *); //处理ep0的控制请求，在中断中调用，不能睡眠
void (*disconnect)(struct usb_gadget *); //可能在中断中调用不能睡眠
void (*suspend)(struct usb_gadget *); //电源管理模式相关，设备挂起
void (*resume)(struct usb_gadget *);//电源管理模式相关，设备恢复
/* FIXME support safe rmmod */
struct device_driver driver; //内核设备管理使用
};
```

4． 描述一个I/O请求

```c
struct usb_request {
void *buf; //数据缓存区
unsigned length; //数据长度
dma_addr_t dma; //与buf关联的DMA地址，DMA传输时使用
unsigned no_interrupt:1;//当为true时，表示没有完成函数，则通过中断通知传输完成，这个由DMA控制器直接控制
unsigned zero:1; //当输出的最后一个数据包不够长度是是否填充0
unsigned short_not_ok:1; //当接收的数据不够指定长度时，是否报错
void (*complete)(struct usb_ep *ep, struct usb_request *req);//请求完成函数
void *context;//被completion回调函数使用
struct list_head list; //被Gadget Driver使用，插入队列
int status;//返回完成结果，0表示成功
unsigned actual;//实际传输的数据长度
};
```

**成员**

- `buf`

  用于数据的缓冲区。始终提供此内容;某些控制器仅使用 PIO，或者不对某些终结点使用 DMA。

- `length`

  该数据的长度

- `dma`

  对应于"buf"的 DMA 地址。如果未设置此字段，并且 USB 控制器需要一个字段，则它负责映射和取消映射缓冲区。

- `sg`

  支持 SG 的控制器的散点列表。

- `num_sgs`

  SG 条目数

- `num_mapped_sgs`

  映射到 DMA 的 SG 条目数（内部）

- `stream_id`

  使用 USB3.0 批量流时的流 ID

- `no_interrupt`

  如果为 true，则提示不需要完成 irq。有时对由 DMA 控制器直接处理的深度请求队列很有用。

- `zero`

  如果为 true，则在写入数据时，根据需要添加零长度数据包，使最后一个数据包变为"短";

- `short_not_ok`

  读取数据时，使短数据包被视为错误（队列停止前进，直到清理）。

- `dma_mapped`

  指示请求是否已映射到 DMA（内部）

- `complete`

  请求完成时调用的函数，因此可以重用此请求及其缓冲区。将始终在禁用中断的情况下调用该函数，并且它不得处于睡眠状态。读取以短数据包终止，或在缓冲区填满时终止，以先到者为准。当写入终止时，某些数据字节通常仍处于传输状态（通常采用硬件 fifo）。错误（对于读取或写入）会阻止队列前进，直到完成函数返回，因此，任何因错误而失效的传输都可以首先取消排队。

- `context`

  供完成回调使用

- `list`

  供小工具驱动程序使用。

- `status`

  报告完成代码，零或负 errno。通常，故障会阻止传输队列前进，直到返回完成回调。代码"-ESHUTDOWN"表示由设备断开连接或驱动程序禁用终结点时导致的完成。

- `actual`

  报告传入/传出缓冲区的字节数。对于读取（OUT 传输），这可能小于请求的长度。如果设置了 short_not_ok 标志，则短读取将被视为错误，即使状态以其他方式指示成功完成也是如此。请注意，对于写入（IN 传输），当请求报告为完成时，某些数据字节可能仍驻留在设备端 FIFO 中。

**描述**

这些是通过使用它们的终结点分配/释放的。硬件的驱动程序可以将额外的每个请求数据添加到它返回的内存中，这通常避免了在请求排队时单独的内存分配（潜在故障）。

请求标志会影响请求处理，例如是否写入了零长度的数据包（"零"标志），短读是否应被视为错误（阻止请求队列前进，"short_not_ok"标志），或暗示不需要中断（"no_interrupt"标志，用于深度请求队列）。

批量终结点可以使用任何大小的缓冲区，也可用于中断传输。仅中断端点的功能可能要少得多。

5． 端点

```c
struct usb_ep {
void *driver_data;  //端点私有数据
const char *name; //端点名称
const struct usb_ep_ops *ops; //端点操作函数集
struct usb_ep_caps      caps;
bool claimed;
bool enabled;
struct list_head ep_list; //Gadget设备建立所有端点的链表
unsigned maxpacket:16;//这个端点使用的最大包长度
unsigned maxpacket_limit:16;
unsigned max_streams:16;
unsigned mult:2;
unsigned maxburst:5;
u8 address;
const struct usb_endpoint_descriptor    *desc;
const struct usb_ss_ep_comp_descriptor  *comp_desc;
};
```

**成员**

- `driver_data`

  供小工具驱动程序使用。

- `name`

  终结点的标识符，例如"ep-a"或"ep9in-bulk"

- `ops`

  用于访问特定于硬件的操作的函数指针。

- `ep_list`

  小工具的ep_list保存其所有端点

- `caps`

  描述 endoint 支持的类型和方向的结构。

- `claimed`

  如果此终结点由函数声明，则为 True。

- `enabled`

  当前终结点已启用/已禁用状态。

- `maxpacket`

  此终结点上使用的最大数据包大小。初始值有时可以根据用于配置终结点的终结点描述符进行减小（硬件允许）。

- `maxpacket_limit`

  此终结点可以处理的最大数据包大小值。它在终结点初始化时由 UDC 驱动程序设置一次，不应更改。不应与 maxpacket 混淆。

- `max_streams`

  此 EP 支持的最大流数（0 - 16，实际数为 2^n）

- `mult`

  乘数，SS Isoc EP的"多重"值

- `maxburst`

  此 EP 支持的最大突发数（对于 usb3）

- `address`

  用于在查找与连接速度匹配的描述符时标识终结点

- `desc`

  端点描述符。此指针在启用终结点之前设置，并在禁用终结点之前保持有效。

- `comp_desc`

  在超高速支持的情况下，这是用于配置端点的端点伴随描述符

6． 端点操作函数集

```c
struct usb_ep_ops {
int (*enable) (struct usb_ep *ep,  const struct usb_endpoint_descriptor *desc);
int (*disable) (struct usb_ep *ep);
struct usb_request *(*alloc_request) (struct usb_ep *ep, gfp_t gfp_flags);
void (*free_request) (struct usb_ep *ep, struct usb_request *req);
int (*queue) (struct usb_ep *ep, struct usb_request *req, gfp_t gfp_flags);
int (*dequeue) (struct usb_ep *ep, struct usb_request *req);
int (*set_halt) (struct usb_ep *ep, int value);
int (*set_wedge) (struct usb_ep *ep);
int (*fifo_status) (struct usb_ep *ep);
void (*fifo_flush) (struct usb_ep *ep);
};
```

7． 字符串结构

```c
struct usb_gadget_strings {
u16 language; /* 0x0409 for en-us */
struct usb_string *strings;
};
struct usb_string {
u8 id; //索引
const char *s;
};
```

8． UDC驱动程序需要实现的上层调用接口
int usb_gadget_register_driver(struct usb_gadget_driver *driver);
int usb_gadget_unregister_driver(struct usb_gadget_driver *driver);

![The USB driver structure under Linux system](https://img.devenyu.top/img/image-20220104143520515.png)

![The structure of Linux gadget driver](https://img.devenyu.top/img/image-20220104143804301.png)