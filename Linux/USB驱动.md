# Linux usb gadget驱动理解

## 1、USB设备的初始化

```c
#mass_storage.c line:242
/****************************** Some noise ******************************/
 
static struct usb_composite_driver msg_driver = {
	.name		= "g_mass_storage",
	.dev		= &msg_device_desc,
	.max_speed	= USB_SPEED_SUPER,
	.needs_serial	= 1,
	.strings	= dev_strings,
	.bind		= msg_bind,
	.unbind		= msg_unbind,
};
 
MODULE_DESCRIPTION(DRIVER_DESC);
MODULE_AUTHOR("Michal Nazarewicz");
MODULE_LICENSE("GPL");
 
static int __init msg_init(void)
{
	return usb_composite_probe(&msg_driver);
}
module_init(msg_init);
 
static void msg_cleanup(void)
{
	if (test_and_clear_bit(0, &msg_registered))
		usb_composite_unregister(&msg_driver);
}
module_exit(msg_cleanup);
```

驱动入口在static int __init msg_init(void);其中init修饰表明这个函数仅在初始化期间使用，在模块被装载之后，它占用的资源就会释放掉。

在msg_init中调用了usb_composite_probe函数，它在composite.c中实现：

```c
#composite.c line:2198
int usb_composite_probe(struct usb_composite_driver *driver)
{
	struct usb_gadget_driver *gadget_driver;
 
	if (!driver || !driver->dev || !driver->bind)
		return -EINVAL;
 
	if (!driver->name)
		driver->name = "composite";
 
	driver->gadget_driver = composite_driver_template;
	gadget_driver = &driver->gadget_driver;
 
	gadget_driver->function =  (char *) driver->name;
	gadget_driver->driver.name = driver->name;
	gadget_driver->max_speed = driver->max_speed;
 
	return usb_gadget_probe_driver(gadget_driver);
}
EXPORT_SYMBOL_GPL(usb_composite_probe);
```

在msg_init()函数中用到的struct usb_composite_driver结构体对象的一个实例——msg_driver:

```c
struct usb_composite_driver {
	const char				*name;
	const struct usb_device_descriptor	*dev;
	struct usb_gadget_strings		**strings;
	enum usb_device_speed			max_speed;
	unsigned		needs_serial:1;
 
	int			(*bind)(struct usb_composite_dev *cdev);
	int			(*unbind)(struct usb_composite_dev *);
 
	void			(*disconnect)(struct usb_composite_dev *);
 
	/* global suspend hooks */
	void			(*suspend)(struct usb_composite_dev *);
	void			(*resume)(struct usb_composite_dev *);
	struct usb_gadget_driver		gadget_driver;
};
 
static struct usb_composite_driver msg_driver = {
	.name		= "g_mass_storage",
	.dev		= &msg_device_desc,
	.max_speed	= USB_SPEED_SUPER,
	.needs_serial	= 1,
	.strings	= dev_strings,
	.bind		= msg_bind,
	.unbind		= msg_unbind,
};
```

在此结构体中我们定义了U盘驱动的名称（name）、设备描述符（dev）、设备速度（usb_device_speed）、字符串描述符（strings)以及注册bind和unbind回调函数，这些参数和回调函数最终会在libcomposite.ko驱动（composite.c）中回调。

其中每一个USB设备均有一个设备描述符：

```c
static struct usb_device_descriptor msg_device_desc = {
	.bLength =		sizeof msg_device_desc,
	.bDescriptorType =	USB_DT_DEVICE,
 
	.bcdUSB =		cpu_to_le16(0x0200),
	.bDeviceClass =		USB_CLASS_PER_INTERFACE,
 
	/* Vendor and product id can be overridden by module parameters.  */
	.idVendor =		cpu_to_le16(FSG_VENDOR_ID),
	.idProduct =		cpu_to_le16(FSG_PRODUCT_ID),
	.bNumConfigurations =	1,
};
```

上述的bNumConfigurations字段值为1，代表该U盘设备驱动只有一个配置。

```c
#composite.c line:2198
int usb_composite_probe(struct usb_composite_driver *driver)
{
	struct usb_gadget_driver *gadget_driver;
 
	if (!driver || !driver->dev || !driver->bind)
		return -EINVAL;
 
	if (!driver->name)
		driver->name = "composite";
 
	driver->gadget_driver = composite_driver_template;
	gadget_driver = &driver->gadget_driver;
 
	gadget_driver->function =  (char *) driver->name;
	gadget_driver->driver.name = driver->name;
	gadget_driver->max_speed = driver->max_speed;
 
	return usb_gadget_probe_driver(gadget_driver);
}
EXPORT_SYMBOL_GPL(usb_composite_probe);
```

现在继续分析usb_composite_probe()函数，usb_composite_probe主要是填充好了struct usb_composite_driver结构体中的成员gadget_driver。

gadget_driver在这里被赋值为composite_driver_template实例了，在函数的最后将工作交给了usb_gadget_probe_driver函数。

```c
#linux-4.4.19/drivers/usb/gadget/udc/udc-core.c
int usb_gadget_probe_driver(struct usb_gadget_driver *driver)
{
	struct usb_udc		*udc = NULL;
	int			ret;
 
	if (!driver || !driver->bind || !driver->setup)
		return -EINVAL;
 
	mutex_lock(&udc_lock);
	list_for_each_entry(udc, &udc_list, list) {
		/* For now we take the first one */
		if (!udc->driver)
			goto found;
	}
 
	pr_debug("couldn't find an available UDC\n");
	mutex_unlock(&udc_lock);
	return -ENODEV;
found:
	ret = udc_bind_to_driver(udc, driver);
	mutex_unlock(&udc_lock);
	return ret;
}
EXPORT_SYMBOL_GPL(usb_gadget_probe_driver);
```

从usb_gadget_probe_driver函数的可知，它在遍历udc_list链表，找到一个udc没有对应驱动的实例，然后通过udc_bind_to_driver（）将udc驱动与上述的composite_driver_template模板实例绑定在一起，即能通过udc找到composite_driver_template实例，也能通过composite_driver_template实例找到绑定它的udc驱动。

接下来在udc_bind_to_driver中：

```c
#linux-4.4.19/drivers/usb/gadget/udc/udc-core.c
/* udc_lock must be held */
static int udc_bind_to_driver(struct usb_udc *udc, struct usb_gadget_driver *driver)
{
	int ret;
 
	dev_dbg(&udc->dev, "registering UDC driver [%s]\n",
			driver->function);
 
	udc->driver = driver;
	udc->dev.driver = &driver->driver;
	udc->gadget->dev.driver = &driver->driver;
 
	ret = driver->bind(udc->gadget, driver);
	if (ret)
		goto err1;
 
	/* If OTG, the otg core starts the UDC when needed */
	mutex_unlock(&udc_lock);
	udc->is_otg = !usb_otg_register_gadget(udc->gadget, &otg_gadget_intf);
	mutex_lock(&udc_lock);
	if (!udc->is_otg) {
		ret = usb_gadget_udc_start(udc);
		if (ret) {
			driver->unbind(udc->gadget);
			goto err1;
		}
		usb_udc_connect_control(udc);
	}
 
	kobject_uevent(&udc->dev.kobj, KOBJ_CHANGE);
	return 0;
err1:
	if (ret != -EISNAM)
		dev_err(&udc->dev, "failed to start %s: %d\n",
			udc->driver->function, ret);
	udc->driver = NULL;
	udc->dev.driver = NULL;
	udc->gadget->dev.driver = NULL;
	return ret;
}
```

回调了driver的bind函数，这个bind回调函数在composite_driver_template实例中赋值。最后调用usb_otg_register_gadget将gadget注册到otg core去，然后 开启udc功能(usb_gadget_udc_start(udc))。（涉及到了udc驱动)

>如果USB控制器不支持otg，kernel也没有加入CONFIG_USB_OTG的话，usb_otg_register_gadget()其实是空函数，返回不支持（return -ENOTSUPP;）。

下面看composite_bind函数

```c
static int composite_bind(struct usb_gadget *gadget,
		struct usb_gadget_driver *gdriver)
{
	struct usb_composite_dev	*cdev;
	struct usb_composite_driver	*composite = to_cdriver(gdriver);
	int				status = -ENOMEM;
 
	cdev = kzalloc(sizeof *cdev, GFP_KERNEL);
	if (!cdev)
		return status;
 
	spin_lock_init(&cdev->lock);
	cdev->gadget = gadget;
	set_gadget_data(gadget, cdev);
	INIT_LIST_HEAD(&cdev->configs);
	INIT_LIST_HEAD(&cdev->gstrings);
 
	status = composite_dev_prepare(composite, cdev);
	if (status)
		goto fail;
 
	/* composite gadget needs to assign strings for whole device (like
	 * serial number), register function drivers, potentially update
	 * power state and consumption, etc
	 */
	status = composite->bind(cdev);
	if (status < 0)
		goto fail;
 
	if (cdev->use_os_string) {
		status = composite_os_desc_req_prepare(cdev, gadget->ep0);
		if (status)
			goto fail;
	}
 
	update_unchanged_dev_desc(&cdev->desc, composite->dev);
 
	/* has userspace failed to provide a serial number? */
	if (composite->needs_serial && !cdev->desc.iSerialNumber)
		WARNING(cdev, "userspace failed to provide iSerialNumber\n");
 
	INFO(cdev, "%s ready\n", composite->name);
	return 0;
 
fail:
	__composite_unbind(gadget, false);
	return status;
}
```

其中，composite_dev_prepare()主要是初始化ep0端，紧跟着，status = composite->bind(cdev);实际上是回调mass_storage.c的int msg_bind(struct usb_composite_dev *cdev)，而cdev在composite_bind()函数开头通过kzalloc创建，并把gadget驱动等填充好struct usb_composite_dev结构体对象。update_unchanged_dev_desc()主要用于填充新的设备描述符信息，因为调用了msg_bind()后信息可能会有改变。

```c
int composite_dev_prepare(struct usb_composite_driver *composite,
		struct usb_composite_dev *cdev)
{
	struct usb_gadget *gadget = cdev->gadget;
	int ret = -ENOMEM;
 
	/* preallocate control response and buffer */
	cdev->req = usb_ep_alloc_request(gadget->ep0, GFP_KERNEL);
	if (!cdev->req)
		return -ENOMEM;
 
	cdev->req->buf = kmalloc(USB_COMP_EP0_BUFSIZ, GFP_KERNEL);
	if (!cdev->req->buf)
		goto fail;
 
	ret = device_create_file(&gadget->dev, &dev_attr_suspended);
	if (ret)
		goto fail_dev;
 
	cdev->req->complete = composite_setup_complete;
	cdev->req->context = cdev;
	gadget->ep0->driver_data = cdev;
 
	cdev->driver = composite;
 
	/*
	 * As per USB compliance update, a device that is actively drawing
	 * more than 100mA from USB must report itself as bus-powered in
	 * the GetStatus(DEVICE) call.
	 */
	if (CONFIG_USB_GADGET_VBUS_DRAW <= USB_SELF_POWER_VBUS_MAX_DRAW)
		usb_gadget_set_selfpowered(gadget);
 
	/* interface and string IDs start at zero via kzalloc.
	 * we force endpoints to start unassigned; few controller
	 * drivers will zero ep->driver_data.
	 */
	usb_ep_autoconfig_reset(gadget);
	return 0;
fail_dev:
	kfree(cdev->req->buf);
fail:
	usb_ep_free_request(gadget->ep0, cdev->req);
	cdev->req = NULL;
	return ret;
}
```

composite_dev_prepare主要是申请了ep0的请求队列（struct usb_request *req）、创建sysfs文件系统的属性文件、最后填充struct usb_request        *req的complete完成函数。我们知道ep0是设备控制传输用到的端点，尤其在设备枚举、属性配置上起关键作用，所以我们要首先初始化。

## 2、USB数据的传输

usb数据的传输是在端点fifo初始化完成后，利用usb_ep_queue函数来实现的。U盘gadget驱动就利用usb_ep_queue()封装而成以下两个函数用于传输U盘数据：

static bool start_in_transfer(struct fsg_common *common, struct fsg_buffhd *bh);
		static bool start_out_transfer(struct fsg_common *common, struct fsg_buffhd *bh);
		阅读SCSI命令文档，发现gadget只是实现其中一些常用的SCSI命令子集：

```c
static int do_scsi_command(struct fsg_common *common)
{
	struct fsg_buffhd	*bh;
	int			rc;
	int			reply = -EINVAL;
	int			i;
	static char		unknown[16];

	dump_cdb(common);

	/* Wait for the next buffer to become available for data or status */
	bh = common->next_buffhd_to_fill;
	common->next_buffhd_to_drain = bh;
	rc = sleep_thread(common, false, bh);
	if (rc)
		return rc;

	common->phase_error = 0;
	common->short_packet_received = 0;

	down_read(&common->filesem);	/* We're using the backing file */
	switch (common->cmnd[0]) {

	case INQUIRY:
		common->data_size_from_cmnd = common->cmnd[4];
		reply = check_command(common, 6, DATA_DIR_TO_HOST,
				      (1<<4), 0,
				      "INQUIRY");
		if (reply == 0)
			reply = do_inquiry(common, bh);
		break;

	case MODE_SELECT:
		common->data_size_from_cmnd = common->cmnd[4];
		reply = check_command(common, 6, DATA_DIR_FROM_HOST,
				      (1<<1) | (1<<4), 0,
				      "MODE SELECT(6)");
		if (reply == 0)
			reply = do_mode_select(common, bh);
		break;

	case MODE_SELECT_10:
		common->data_size_from_cmnd =
			get_unaligned_be16(&common->cmnd[7]);
		reply = check_command(common, 10, DATA_DIR_FROM_HOST,
				      (1<<1) | (3<<7), 0,
				      "MODE SELECT(10)");
		if (reply == 0)
			reply = do_mode_select(common, bh);
		break;

	case MODE_SENSE:
		common->data_size_from_cmnd = common->cmnd[4];
		reply = check_command(common, 6, DATA_DIR_TO_HOST,
				      (1<<1) | (1<<2) | (1<<4), 0,
				      "MODE SENSE(6)");
		if (reply == 0)
			reply = do_mode_sense(common, bh);
		break;

	case MODE_SENSE_10:
		common->data_size_from_cmnd =
			get_unaligned_be16(&common->cmnd[7]);
		reply = check_command(common, 10, DATA_DIR_TO_HOST,
				      (1<<1) | (1<<2) | (3<<7), 0,
				      "MODE SENSE(10)");
		if (reply == 0)
			reply = do_mode_sense(common, bh);
		break;

	case ALLOW_MEDIUM_REMOVAL:
		common->data_size_from_cmnd = 0;
		reply = check_command(common, 6, DATA_DIR_NONE,
				      (1<<4), 0,
				      "PREVENT-ALLOW MEDIUM REMOVAL");
		if (reply == 0)
			reply = do_prevent_allow(common);
		break;

	case READ_6:
		i = common->cmnd[4];
		common->data_size_from_cmnd = (i == 0) ? 256 : i;
		reply = check_command_size_in_blocks(common, 6,
				      DATA_DIR_TO_HOST,
				      (7<<1) | (1<<4), 1,
				      "READ(6)");
		if (reply == 0)
			reply = do_read(common);
		break;

	case READ_10:
		common->data_size_from_cmnd =
				get_unaligned_be16(&common->cmnd[7]);
		reply = check_command_size_in_blocks(common, 10,
				      DATA_DIR_TO_HOST,
				      (1<<1) | (0xf<<2) | (3<<7), 1,
				      "READ(10)");
		if (reply == 0)
			reply = do_read(common);
		break;

	case READ_12:
		common->data_size_from_cmnd =
				get_unaligned_be32(&common->cmnd[6]);
		reply = check_command_size_in_blocks(common, 12,
				      DATA_DIR_TO_HOST,
				      (1<<1) | (0xf<<2) | (0xf<<6), 1,
				      "READ(12)");
		if (reply == 0)
			reply = do_read(common);
		break;

	case READ_CAPACITY:
		common->data_size_from_cmnd = 8;
		reply = check_command(common, 10, DATA_DIR_TO_HOST,
				      (0xf<<2) | (1<<8), 1,
				      "READ CAPACITY");
		if (reply == 0)
			reply = do_read_capacity(common, bh);
		break;

	case READ_HEADER:
		if (!common->curlun || !common->curlun->cdrom)
			goto unknown_cmnd;
		common->data_size_from_cmnd =
			get_unaligned_be16(&common->cmnd[7]);
		reply = check_command(common, 10, DATA_DIR_TO_HOST,
				      (3<<7) | (0x1f<<1), 1,
				      "READ HEADER");
		if (reply == 0)
			reply = do_read_header(common, bh);
		break;

	case READ_TOC:
		if (!common->curlun || !common->curlun->cdrom)
			goto unknown_cmnd;
		common->data_size_from_cmnd =
			get_unaligned_be16(&common->cmnd[7]);
		reply = check_command(common, 10, DATA_DIR_TO_HOST,
				      (7<<6) | (1<<1), 1,
				      "READ TOC");
		if (reply == 0)
			reply = do_read_toc(common, bh);
		break;

	case READ_FORMAT_CAPACITIES:
		common->data_size_from_cmnd =
			get_unaligned_be16(&common->cmnd[7]);
		reply = check_command(common, 10, DATA_DIR_TO_HOST,
				      (3<<7), 1,
				      "READ FORMAT CAPACITIES");
		if (reply == 0)
			reply = do_read_format_capacities(common, bh);
		break;

	case REQUEST_SENSE:
		common->data_size_from_cmnd = common->cmnd[4];
		reply = check_command(common, 6, DATA_DIR_TO_HOST,
				      (1<<4), 0,
				      "REQUEST SENSE");
		if (reply == 0)
			reply = do_request_sense(common, bh);
		break;

	case START_STOP:
		common->data_size_from_cmnd = 0;
		reply = check_command(common, 6, DATA_DIR_NONE,
				      (1<<1) | (1<<4), 0,
				      "START-STOP UNIT");
		if (reply == 0)
			reply = do_start_stop(common);
		break;

	case SYNCHRONIZE_CACHE:
		common->data_size_from_cmnd = 0;
		reply = check_command(common, 10, DATA_DIR_NONE,
				      (0xf<<2) | (3<<7), 1,
				      "SYNCHRONIZE CACHE");
		if (reply == 0)
			reply = do_synchronize_cache(common);
		break;

	case TEST_UNIT_READY:
		common->data_size_from_cmnd = 0;
		reply = check_command(common, 6, DATA_DIR_NONE,
				0, 1,
				"TEST UNIT READY");
		break;

	/*
	 * Although optional, this command is used by MS-Windows.  We
	 * support a minimal version: BytChk must be 0.
	 */
	case VERIFY:
		common->data_size_from_cmnd = 0;
		reply = check_command(common, 10, DATA_DIR_NONE,
				      (1<<1) | (0xf<<2) | (3<<7), 1,
				      "VERIFY");
		if (reply == 0)
			reply = do_verify(common);
		break;

	case WRITE_6:
		i = common->cmnd[4];
		common->data_size_from_cmnd = (i == 0) ? 256 : i;
		reply = check_command_size_in_blocks(common, 6,
				      DATA_DIR_FROM_HOST,
				      (7<<1) | (1<<4), 1,
				      "WRITE(6)");
		if (reply == 0)
			reply = do_write(common);
		break;

	case WRITE_10:
		common->data_size_from_cmnd =
				get_unaligned_be16(&common->cmnd[7]);
		reply = check_command_size_in_blocks(common, 10,
				      DATA_DIR_FROM_HOST,
				      (1<<1) | (0xf<<2) | (3<<7), 1,
				      "WRITE(10)");
		if (reply == 0)
			reply = do_write(common);
		break;

	case WRITE_12:
		common->data_size_from_cmnd =
				get_unaligned_be32(&common->cmnd[6]);
		reply = check_command_size_in_blocks(common, 12,
				      DATA_DIR_FROM_HOST,
				      (1<<1) | (0xf<<2) | (0xf<<6), 1,
				      "WRITE(12)");
		if (reply == 0)
			reply = do_write(common);
		break;

	/*
	 * Some mandatory commands that we recognize but don't implement.
	 * They don't mean much in this setting.  It's left as an exercise
	 * for anyone interested to implement RESERVE and RELEASE in terms
	 * of Posix locks.
	 */
	case FORMAT_UNIT:
	case RELEASE:
	case RESERVE:
	case SEND_DIAGNOSTIC:
		/* Fall through */

	default:
unknown_cmnd:
		common->data_size_from_cmnd = 0;
		sprintf(unknown, "Unknown x%02x", common->cmnd[0]);
		reply = check_command(common, common->cmnd_size,
				      DATA_DIR_UNKNOWN, ~0, 0, unknown);
		if (reply == 0) {
			common->curlun->sense_data = SS_INVALID_COMMAND;
			reply = -EINVAL;
		}
		break;
	}
	up_read(&common->filesem);

	if (reply == -EINTR || signal_pending(current))
		return -EINTR;

	/* Set up the single reply buffer for finish_reply() */
	if (reply == -EINVAL)
		reply = 0;		/* Error reply length */
	if (reply >= 0 && common->data_dir == DATA_DIR_TO_HOST) {
		reply = min((u32)reply, common->data_size_from_cmnd);
		bh->inreq->length = reply;
		bh->state = BUF_STATE_FULL;
		common->residue -= reply;
	}				/* Otherwise it's already set */

	return 0;
}

```

可以看到主要是do_read和do_write，do_write()是通过start_out_transfer()从usb host获取到文件数据，然后调用vfs_write()写入文件系统，完成了将文件写入U盘的过程；而do_read()则是先通过vfs_read()从文件系统（加载驱动时指定的文件路径file=filename[,filename...]）中读取文件，然后调用start_in_transfer()写入usb host，完成了读取U盘内的文件到PC。

```c
static int do_write(struct fsg_common *common)
{
	struct fsg_lun		*curlun = common->curlun;
	u32			lba;
	struct fsg_buffhd	*bh;
	int			get_some_more;
	u32			amount_left_to_req, amount_left_to_write;
	loff_t			usb_offset, file_offset, file_offset_tmp;
	unsigned int		amount;
	ssize_t			nwritten;
	int			rc;

	if (curlun->ro) {
		curlun->sense_data = SS_WRITE_PROTECTED;
		return -EINVAL;
	}
	spin_lock(&curlun->filp->f_lock);
	curlun->filp->f_flags &= ~O_SYNC;	/* Default is not to wait */
	spin_unlock(&curlun->filp->f_lock);

	/*
	 * Get the starting Logical Block Address and check that it's
	 * not too big
	 */
	if (common->cmnd[0] == WRITE_6)
		lba = get_unaligned_be24(&common->cmnd[1]);
	else {
		lba = get_unaligned_be32(&common->cmnd[2]);

		/*
		 * We allow DPO (Disable Page Out = don't save data in the
		 * cache) and FUA (Force Unit Access = write directly to the
		 * medium).  We don't implement DPO; we implement FUA by
		 * performing synchronous output.
		 */
		if (common->cmnd[1] & ~0x18) {
			curlun->sense_data = SS_INVALID_FIELD_IN_CDB;
			return -EINVAL;
		}
		if (!curlun->nofua && (common->cmnd[1] & 0x08)) { /* FUA */
			spin_lock(&curlun->filp->f_lock);
			curlun->filp->f_flags |= O_SYNC;
			spin_unlock(&curlun->filp->f_lock);
		}
	}
	if (lba >= curlun->num_sectors) {
		curlun->sense_data = SS_LOGICAL_BLOCK_ADDRESS_OUT_OF_RANGE;
		return -EINVAL;
	}

	/* Carry out the file writes */
	get_some_more = 1;
	file_offset = usb_offset = ((loff_t) lba) << curlun->blkbits;
	amount_left_to_req = common->data_size_from_cmnd;
	amount_left_to_write = common->data_size_from_cmnd;

	while (amount_left_to_write > 0) {

		/* Queue a request for more data from the host */
		bh = common->next_buffhd_to_fill;
		if (bh->state == BUF_STATE_EMPTY && get_some_more) {

			/*
			 * Figure out how much we want to get:
			 * Try to get the remaining amount,
			 * but not more than the buffer size.
			 */
			amount = min(amount_left_to_req, FSG_BUFLEN);

			/* Beyond the end of the backing file? */
			if (usb_offset >= curlun->file_length) {
				get_some_more = 0;
				curlun->sense_data =
					SS_LOGICAL_BLOCK_ADDRESS_OUT_OF_RANGE;
				curlun->sense_data_info =
					usb_offset >> curlun->blkbits;
				curlun->info_valid = 1;
				continue;
			}

			/* Get the next buffer */
			usb_offset += amount;
			common->usb_amount_left -= amount;
			amount_left_to_req -= amount;
			if (amount_left_to_req == 0)
				get_some_more = 0;

			/*
			 * Except at the end of the transfer, amount will be
			 * equal to the buffer size, which is divisible by
			 * the bulk-out maxpacket size.
			 */
			set_bulk_out_req_length(common, bh, amount);
			if (!start_out_transfer(common, bh))
				/* Dunno what to do if common->fsg is NULL */
				return -EIO;
			common->next_buffhd_to_fill = bh->next;
			continue;
		}

		/* Write the received data to the backing file */
		bh = common->next_buffhd_to_drain;
		if (bh->state == BUF_STATE_EMPTY && !get_some_more)
			break;			/* We stopped early */

		/* Wait for the data to be received */
		rc = sleep_thread(common, false, bh);
		if (rc)
			return rc;

		common->next_buffhd_to_drain = bh->next;
		bh->state = BUF_STATE_EMPTY;

		/* Did something go wrong with the transfer? */
		if (bh->outreq->status != 0) {
			curlun->sense_data = SS_COMMUNICATION_FAILURE;
			curlun->sense_data_info =
					file_offset >> curlun->blkbits;
			curlun->info_valid = 1;
			break;
		}

		amount = bh->outreq->actual;
		if (curlun->file_length - file_offset < amount) {
			LERROR(curlun, "write %u @ %llu beyond end %llu\n",
				       amount, (unsigned long long)file_offset,
				       (unsigned long long)curlun->file_length);
			amount = curlun->file_length - file_offset;
		}

		/*
		 * Don't accept excess data.  The spec doesn't say
		 * what to do in this case.  We'll ignore the error.
		 */
		amount = min(amount, bh->bulk_out_intended_length);

		/* Don't write a partial block */
		amount = round_down(amount, curlun->blksize);
		if (amount == 0)
			goto empty_write;

		/* Perform the write */
		file_offset_tmp = file_offset;
		nwritten = kernel_write(curlun->filp, bh->buf, amount,
				&file_offset_tmp);
		VLDBG(curlun, "file write %u @ %llu -> %d\n", amount,
				(unsigned long long)file_offset, (int)nwritten);
		if (signal_pending(current))
			return -EINTR;		/* Interrupted! */

		if (nwritten < 0) {
			LDBG(curlun, "error in file write: %d\n",
					(int) nwritten);
			nwritten = 0;
		} else if (nwritten < amount) {
			LDBG(curlun, "partial file write: %d/%u\n",
					(int) nwritten, amount);
			nwritten = round_down(nwritten, curlun->blksize);
		}
		file_offset += nwritten;
		amount_left_to_write -= nwritten;
		common->residue -= nwritten;

		/* If an error occurred, report it and its position */
		if (nwritten < amount) {
			curlun->sense_data = SS_WRITE_ERROR;
			curlun->sense_data_info =
					file_offset >> curlun->blkbits;
			curlun->info_valid = 1;
			break;
		}

 empty_write:
		/* Did the host decide to stop early? */
		if (bh->outreq->actual < bh->bulk_out_intended_length) {
			common->short_packet_received = 1;
			break;
		}
	}

	return -EIO;		/* No default reply */
}

static int do_read(struct fsg_common *common)
{
	struct fsg_lun		*curlun = common->curlun;
	u32			lba;
	struct fsg_buffhd	*bh;
	int			rc;
	u32			amount_left;
	loff_t			file_offset, file_offset_tmp;
	unsigned int		amount;
	ssize_t			nread;

	/*
	 * Get the starting Logical Block Address and check that it's
	 * not too big.
	 */
	if (common->cmnd[0] == READ_6)
		lba = get_unaligned_be24(&common->cmnd[1]);
	else {
		lba = get_unaligned_be32(&common->cmnd[2]);

		/*
		 * We allow DPO (Disable Page Out = don't save data in the
		 * cache) and FUA (Force Unit Access = don't read from the
		 * cache), but we don't implement them.
		 */
		if ((common->cmnd[1] & ~0x18) != 0) {
			curlun->sense_data = SS_INVALID_FIELD_IN_CDB;
			return -EINVAL;
		}
	}
	if (lba >= curlun->num_sectors) {
		curlun->sense_data = SS_LOGICAL_BLOCK_ADDRESS_OUT_OF_RANGE;
		return -EINVAL;
	}
	file_offset = ((loff_t) lba) << curlun->blkbits;

	/* Carry out the file reads */
	amount_left = common->data_size_from_cmnd;
	if (unlikely(amount_left == 0))
		return -EIO;		/* No default reply */

	for (;;) {
		/*
		 * Figure out how much we need to read:
		 * Try to read the remaining amount.
		 * But don't read more than the buffer size.
		 * And don't try to read past the end of the file.
		 */
		amount = min(amount_left, FSG_BUFLEN);
		amount = min((loff_t)amount,
			     curlun->file_length - file_offset);

		/* Wait for the next buffer to become available */
		bh = common->next_buffhd_to_fill;
		rc = sleep_thread(common, false, bh);
		if (rc)
			return rc;

		/*
		 * If we were asked to read past the end of file,
		 * end with an empty buffer.
		 */
		if (amount == 0) {
			curlun->sense_data =
					SS_LOGICAL_BLOCK_ADDRESS_OUT_OF_RANGE;
			curlun->sense_data_info =
					file_offset >> curlun->blkbits;
			curlun->info_valid = 1;
			bh->inreq->length = 0;
			bh->state = BUF_STATE_FULL;
			break;
		}

		/* Perform the read */
		file_offset_tmp = file_offset;
		nread = kernel_read(curlun->filp, bh->buf, amount,
				&file_offset_tmp);
		VLDBG(curlun, "file read %u @ %llu -> %d\n", amount,
		      (unsigned long long)file_offset, (int)nread);
		if (signal_pending(current))
			return -EINTR;

		if (nread < 0) {
			LDBG(curlun, "error in file read: %d\n", (int)nread);
			nread = 0;
		} else if (nread < amount) {
			LDBG(curlun, "partial file read: %d/%u\n",
			     (int)nread, amount);
			nread = round_down(nread, curlun->blksize);
		}
		file_offset  += nread;
		amount_left  -= nread;
		common->residue -= nread;

		/*
		 * Except at the end of the transfer, nread will be
		 * equal to the buffer size, which is divisible by the
		 * bulk-in maxpacket size.
		 */
		bh->inreq->length = nread;
		bh->state = BUF_STATE_FULL;

		/* If an error occurred, report it and its position */
		if (nread < amount) {
			curlun->sense_data = SS_UNRECOVERED_READ_ERROR;
			curlun->sense_data_info =
					file_offset >> curlun->blkbits;
			curlun->info_valid = 1;
			break;
		}

		if (amount_left == 0)
			break;		/* No more left to read */

		/* Send this buffer and go read some more */
		bh->inreq->zero = 0;
		if (!start_in_transfer(common, bh))
			/* Don't know what to do if common->fsg is NULL */
			return -EIO;
		common->next_buffhd_to_fill = bh->next;
	}

	return -EIO;		/* No default reply */
}
static int do_read_capacity(struct fsg_common *common, struct fsg_buffhd *bh)
{
	struct fsg_lun	*curlun = common->curlun;
	u32		lba = get_unaligned_be32(&common->cmnd[2]);
	int		pmi = common->cmnd[8];
	u8		*buf = (u8 *)bh->buf;

	/* Check the PMI and LBA fields */
	if (pmi > 1 || (pmi == 0 && lba != 0)) {
		curlun->sense_data = SS_INVALID_FIELD_IN_CDB;
		return -EINVAL;
	}

	put_unaligned_be32(curlun->num_sectors - 1, &buf[0]);
						/* Max logical block */
	put_unaligned_be32(curlun->blksize, &buf[4]);/* Block length */
	return 8;
}

static int do_read_header(struct fsg_common *common, struct fsg_buffhd *bh)
{
	struct fsg_lun	*curlun = common->curlun;
	int		msf = common->cmnd[1] & 0x02;
	u32		lba = get_unaligned_be32(&common->cmnd[2]);
	u8		*buf = (u8 *)bh->buf;

	if (common->cmnd[1] & ~0x02) {		/* Mask away MSF */
		curlun->sense_data = SS_INVALID_FIELD_IN_CDB;
		return -EINVAL;
	}
	if (lba >= curlun->num_sectors) {
		curlun->sense_data = SS_LOGICAL_BLOCK_ADDRESS_OUT_OF_RANGE;
		return -EINVAL;
	}

	memset(buf, 0, 8);
	buf[0] = 0x01;		/* 2048 bytes of user data, rest is EC */
	store_cdrom_address(&buf[4], msf, lba);
	return 8;
}

static int do_read_toc(struct fsg_common *common, struct fsg_buffhd *bh)
{
	struct fsg_lun	*curlun = common->curlun;
	int		msf = common->cmnd[1] & 0x02;
	int		start_track = common->cmnd[6];
	u8		*buf = (u8 *)bh->buf;

	if ((common->cmnd[1] & ~0x02) != 0 ||	/* Mask away MSF */
			start_track > 1) {
		curlun->sense_data = SS_INVALID_FIELD_IN_CDB;
		return -EINVAL;
	}

	memset(buf, 0, 20);
	buf[1] = (20-2);		/* TOC data length */
	buf[2] = 1;			/* First track number */
	buf[3] = 1;			/* Last track number */
	buf[5] = 0x16;			/* Data track, copying allowed */
	buf[6] = 0x01;			/* Only track is number 1 */
	store_cdrom_address(&buf[8], msf, 0);

	buf[13] = 0x16;			/* Lead-out track is data */
	buf[14] = 0xAA;			/* Lead-out track number */
	store_cdrom_address(&buf[16], msf, curlun->num_sectors);
	return 20;
}

static int do_mode_sense(struct fsg_common *common, struct fsg_buffhd *bh)
{
	struct fsg_lun	*curlun = common->curlun;
	int		mscmnd = common->cmnd[0];
	u8		*buf = (u8 *) bh->buf;
	u8		*buf0 = buf;
	int		pc, page_code;
	int		changeable_values, all_pages;
	int		valid_page = 0;
	int		len, limit;

	if ((common->cmnd[1] & ~0x08) != 0) {	/* Mask away DBD */
		curlun->sense_data = SS_INVALID_FIELD_IN_CDB;
		return -EINVAL;
	}
	pc = common->cmnd[2] >> 6;
	page_code = common->cmnd[2] & 0x3f;
	if (pc == 3) {
		curlun->sense_data = SS_SAVING_PARAMETERS_NOT_SUPPORTED;
		return -EINVAL;
	}
	changeable_values = (pc == 1);
	all_pages = (page_code == 0x3f);

	/*
	 * Write the mode parameter header.  Fixed values are: default
	 * medium type, no cache control (DPOFUA), and no block descriptors.
	 * The only variable value is the WriteProtect bit.  We will fill in
	 * the mode data length later.
	 */
	memset(buf, 0, 8);
	if (mscmnd == MODE_SENSE) {
		buf[2] = (curlun->ro ? 0x80 : 0x00);		/* WP, DPOFUA */
		buf += 4;
		limit = 255;
	} else {			/* MODE_SENSE_10 */
		buf[3] = (curlun->ro ? 0x80 : 0x00);		/* WP, DPOFUA */
		buf += 8;
		limit = 65535;		/* Should really be FSG_BUFLEN */
	}

	/* No block descriptors */

	/*
	 * The mode pages, in numerical order.  The only page we support
	 * is the Caching page.
	 */
	if (page_code == 0x08 || all_pages) {
		valid_page = 1;
		buf[0] = 0x08;		/* Page code */
		buf[1] = 10;		/* Page length */
		memset(buf+2, 0, 10);	/* None of the fields are changeable */

		if (!changeable_values) {
			buf[2] = 0x04;	/* Write cache enable, */
					/* Read cache not disabled */
					/* No cache retention priorities */
			put_unaligned_be16(0xffff, &buf[4]);
					/* Don't disable prefetch */
					/* Minimum prefetch = 0 */
			put_unaligned_be16(0xffff, &buf[8]);
					/* Maximum prefetch */
			put_unaligned_be16(0xffff, &buf[10]);
					/* Maximum prefetch ceiling */
		}
		buf += 12;
	}

	/*
	 * Check that a valid page was requested and the mode data length
	 * isn't too long.
	 */
	len = buf - buf0;
	if (!valid_page || len > limit) {
		curlun->sense_data = SS_INVALID_FIELD_IN_CDB;
		return -EINVAL;
	}

	/*  Store the mode data length */
	if (mscmnd == MODE_SENSE)
		buf0[0] = len - 1;
	else
		put_unaligned_be16(len - 2, buf0);
	return len;
}

static int do_start_stop(struct fsg_common *common)
{
	struct fsg_lun	*curlun = common->curlun;
	int		loej, start;

	if (!curlun) {
		return -EINVAL;
	} else if (!curlun->removable) {
		curlun->sense_data = SS_INVALID_COMMAND;
		return -EINVAL;
	} else if ((common->cmnd[1] & ~0x01) != 0 || /* Mask away Immed */
		   (common->cmnd[4] & ~0x03) != 0) { /* Mask LoEj, Start */
		curlun->sense_data = SS_INVALID_FIELD_IN_CDB;
		return -EINVAL;
	}

	loej  = common->cmnd[4] & 0x02;
	start = common->cmnd[4] & 0x01;

	/*
	 * Our emulation doesn't support mounting; the medium is
	 * available for use as soon as it is loaded.
	 */
	if (start) {
		if (!fsg_lun_is_open(curlun)) {
			curlun->sense_data = SS_MEDIUM_NOT_PRESENT;
			return -EINVAL;
		}
		return 0;
	}

	/* Are we allowed to unload the media? */
	if (curlun->prevent_medium_removal) {
		LDBG(curlun, "unload attempt prevented\n");
		curlun->sense_data = SS_MEDIUM_REMOVAL_PREVENTED;
		return -EINVAL;
	}

	if (!loej)
		return 0;

	up_read(&common->filesem);
	down_write(&common->filesem);
	fsg_lun_close(curlun);
	up_write(&common->filesem);
	down_read(&common->filesem);

	return 0;
}

static int do_prevent_allow(struct fsg_common *common)
{
	struct fsg_lun	*curlun = common->curlun;
	int		prevent;

	if (!common->curlun) {
		return -EINVAL;
	} else if (!common->curlun->removable) {
		common->curlun->sense_data = SS_INVALID_COMMAND;
		return -EINVAL;
	}

	prevent = common->cmnd[4] & 0x01;
	if ((common->cmnd[4] & ~0x01) != 0) {	/* Mask away Prevent */
		curlun->sense_data = SS_INVALID_FIELD_IN_CDB;
		return -EINVAL;
	}

	if (curlun->prevent_medium_removal && !prevent)
		fsg_lun_fsync_sub(curlun);
	curlun->prevent_medium_removal = prevent;
	return 0;
}

static int do_read_format_capacities(struct fsg_common *common,
			struct fsg_buffhd *bh)
{
	struct fsg_lun	*curlun = common->curlun;
	u8		*buf = (u8 *) bh->buf;

	buf[0] = buf[1] = buf[2] = 0;
	buf[3] = 8;	/* Only the Current/Maximum Capacity Descriptor */
	buf += 4;

	put_unaligned_be32(curlun->num_sectors, &buf[0]);
						/* Number of blocks */
	put_unaligned_be32(curlun->blksize, &buf[4]);/* Block length */
	buf[4] = 0x02;				/* Current capacity */
	return 12;
}

static int do_mode_select(struct fsg_common *common, struct fsg_buffhd *bh)
{
	struct fsg_lun	*curlun = common->curlun;

	/* We don't support MODE SELECT */
	if (curlun)
		curlun->sense_data = SS_INVALID_COMMAND;
	return -EINVAL;
}
```



------

## Mass Storage设备所使用的SCSI命令集

![](https://img.devenyu.top/img/image-20211231140300715.png)

### 1、inquiry

上位机发送的：

![](https://img.devenyu.top/img/159396252edc9f15e0a5ab237a1088b5.png)

下位机发送的ack：

![](https://img.devenyu.top/img/1f61d8242d2c28e5c96cbf0825e8e416.png)

### 2、READ FORMATCAPACITIES

![](https://img.devenyu.top/img/fd699309e50393b210f065fb0f33c27c.png)

![](https://img.devenyu.top/img/367b74fe9acbdcbaa42d84cd9386a365.png)

![](https://img.devenyu.top/img/b059c53aa258d72cde367d5fe95b71c0.png)

![](https://img.devenyu.top/img/baab1a276d291d6049ad2280198a50c3.png)

### 3、READ CAPACITY

![](https://img.devenyu.top/img/98d1a1ccd61d37a243c84d536cbd97ab.png)

![](https://img.devenyu.top/img/6b5d5f4463fc1d2b6a048e5d3da64f8f.png)

### 4、READ(10)

![](https://img.devenyu.top/img/b5e2efaf814ce45a75eb51d22fea49d3.png)

### 5、SENSE6

![](https://img.devenyu.top/img/59696d31df0e4a9da24737a0997270cd.png)

![](https://img.devenyu.top/img/4ea57387276273e0bd6685dac965f2f0.png)





------



1、获取PartConfig.cfg中的分区名称，起始位置，结束位置, PartFS

 2、创建空的磁盘镜像文件，这里创建一个8G的软盘

 3、查找可用的Loop Num

 4、使用 losetup将磁盘镜像文件虚拟成块设备

 5、对块设备进行分区

 6、对PartFS进行判断，如果是bin文件则直接拷贝到分区中，如果是File则先格式化分区并挂载，最后将文件拷贝到该分区

 7、删除之前虚拟的块设备，将镜像拷贝到OutputDir/linux_burn_image

  	将outputdir中的文件打包成镜像



kernel 33792s 164863s boot.img
		program 164864s 689151s ext4/program
		data 689152s 7180799s ext4
		backup 7180800s 13672446s ext4
		others 13672447s 15245311s ext4



```markdown
Default Firmware location: ./FIRMWARE
folder mnt/ not exist so create now
PartNum=5
7456+0 records in
7456+0 records out
7818182656 bytes (7.8 GB, 7.3 GiB) copied, 471.894 s, 16.6 MB/s
fakeMMC.img
[sudo] password for devenyu: 
Available LoopNum is 23
Warning: The resulting partition is not properly aligned for best performance.
Warning: The resulting partition is not properly aligned for best performance.
Warning: The resulting partition is not properly aligned for best performance.
Warning: The resulting partition is not properly aligned for best performance.
Warning: The resulting partition is not properly aligned for best performance.
Model: Loopback device (loopback)
Disk /dev/loop23: 7818MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
 1      17.3MB  84.4MB  67.1MB               kernel
 2      84.4MB  353MB   268MB                program
 3      353MB   3677MB  3324MB               data
 4      3677MB  7000MB  3324MB               backup
 5      7000MB  7806MB  805MB                others

30847+1 records in
30847+1 records out
15794031 bytes (16 MB, 15 MiB) copied, 4.62495 s, 3.4 MB/s
mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 262144 1k blocks and 65536 inodes
Filesystem UUID: 0d4ac8ba-e5ce-47fb-9f4b-d47de9e3fb3b
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729, 204801, 221185

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 811456 4k blocks and 203200 inodes
Filesystem UUID: 361bb644-e0d3-4927-b6e1-ed6157eef318
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 811455 4k blocks and 203200 inodes
Filesystem UUID: 5e87625b-2fb7-420e-b7a2-5c9fcd4c8990
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 196608 4k blocks and 49152 inodes
Filesystem UUID: ea67b881-77da-4318-b790-4684dbd6c97e
Superblock backups stored on blocks: 
	32768, 98304, 163840

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

  adding: bootimg_AutoBoot.bin	(in=11504) (out=8490) (deflated 26%)
  adding: linux_burn_image ...........................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................	(in=7818182656) (out=23742433) (deflated 100%)
  adding: PartList.cfg	(in=140) (out=103) (deflated 26%)
  adding: spi_part.cfg	(in=69) (out=48) (deflated 30%)
  adding: u-boot_AutoBoot.bin 	(in=534896) (out=268203) (deflated 50%)
total bytes=7818729265, compressed=24019277 -> 100% savings
```

​			
