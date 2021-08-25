``` C
TimerInit()
GoDNetInitializeAll()
	UsrDNetInitEvent()
	_NASMInit()						//Initialize the Network Access State Machine
	CANInit()						//Initialize the CAN Driver
    CANopen()
	CANSetBitRate()					//Changes the bitrate of the node
	mConnInit()						//Initialize all connection stuff
    	_Conn6Create				//创建conn6
    	_Conn7Create				//创建conn7
GoDNetProcessAllMsgEvents()			//Process all DeviceNet Messaging events
    _ConnErrorManager()
    
```

正常終了した場合はNO_ERRORを返します。

LED番号が違う時に、LED_NO_ERRORを返します。

制御コードが違う時に、PARAM_ERRORを返します。

他のエラーを発生時に、GENERAL_ERRORを返します。

スイッチ番号が違う時に、DIPSW_NO_ERRORを返します。

GPIO状態取得エラー発生時に、GPIO_READ_ERRORを返します。

SPIオープン時エラーを発生に、SPI_OPEN_ERRORを返します。

SPIクロース時エラーを発生時に、SPI_CLOSE_ERRORを返します。

SPIでデータリード時にエラーを発生時に、SPI_READ_ERRORを返します。

SPIでデータライト時にエラーを発生時に、SPI_WRITE_ERRORを返します。

CAN通信エラーを発生にCAN_READ_ERRORを返します。

I2Cでデータリード時にエラーを発生時に、I2C_READ_ERRORを返します。

メモリ不足の場合に、SYS_ENOMEM_ERRORを返します。

権限ので、エラーを発生時に、SYS_EPERM_ERRORを返します。



conn.c 此文件包含几个连接管理功能，以捕获通信事件并将它们分配到适当的实例或其他管理功能。

conn1.c此文件提供了预定义的显式消息传递连接功能。

conn2.c此文件提供了预定义的轮询/更改状态/循环I/O消息传递连接功能。

conn3.c此文件提供预定义位频点I/O消息连接功能

conn4.c此文件提供了预定义的状态更改/循环I/O消息传递连接功能

conn5.c此文件提供了预定义的多播轮询的I/O消息传递连接功能。

conn6.c此文件提供了未连接的显式消息传递功能，它看起来类似于其他常规I/O连接，但不支持所有事件和碎片化。

conn7.c此文件提供了重复的MACID消息传递功能，它看起来类似于其他常规的I/O连接，但不支持所有的事件和碎片化。

mDNetSetMACID	此函数设置MACID。在初始化时使用它

mDNetSetBaudRate	此函数设置波特率。有效值分别为0、1和2。在初始化时使用这个程序

mDNetSetBOI	设置总线关闭中断操作。这应该在初始化时断言，并且可以在处理“设置属性事件”时在正常操作时断言。

mDNetSetMACSwChange	如果得到支持，请设置MACID开关更改指示。应用程序应该使用此功能来通知开发网络对象的更改。通常，如果应用程序有交换机，它应该通知DeviceNet固件交换机自上次重置以来发生更改

mDNetSetBaudSwChange	如果有支持，请设置波特率开关的变化指示。应用程序应该使用此功能来通知开发网络对象的更改。通常，如果应用程序有交换机，它应该通知DeviceNet固件，该交换机自上次重置以来已经发生了更改。

mDNetSetMACSwValue	如果受到支持，请设置MACID开关值。应用程序应使用此功能通知开发网对象交换机值

mDNetSetBaudSwValue	如果有支持，请设置波特率开关值。应用程序应该使用此功能来通知服务网对象交换机值。

mDNetGetBusOffCount	获取存储在DeviceNet对象中的当前总线的计数值。此值将由“连接对象错误管理”功能进行更新

mDNetGetAllocChoice	获取当前的分配选择字节。此值将根据来自服务器和内部监视器计时器的请求而更改。这可以在内部用于获取已分配的连接的指示。

