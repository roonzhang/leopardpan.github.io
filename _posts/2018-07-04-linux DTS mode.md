---
layout: post
title: "Linux的设备树模型"
date: 2018-07-04
description: "设备树模型"
tag: 设备驱动
---   
------

最近一直从事**VxWorks**的驱动开发，其与**Linux**下设备驱动的开发有很大的相似性。正巧最近在csdn上看到刘盼的课程有介绍驱动模型，现结合Linux和vxWorks两者的设备驱动模型在此做一个总结。

-----
### 设备驱动模型
-----

设备驱动的主要功能就是操作硬件，而操作硬件最关键的便是设备的`基地址和中断号`了，而世界上的板子千千万，每个板子的信息也不一样，站在驱动开发的角度来看，每次重新更换板卡时，基地址和中断号不一样，则驱动程序也需要更改为相对应的参数，如此一来，千千万的板卡就需要千千万的驱动程序，这种方式肯定是不可取的。驱动程序要做的就是以不变应万变，这个时候，设备驱动模型就显得尤其重要了。


而驱动程序想要实现不变应万变，那么CPU板级信息就应与驱动程序分离，这是我们设计的前提。但是驱动是一定要去取基地址和中断号的，而基地址和中断号又与硬件又是密切相关的，中间这层耦合关系似乎很难隔离。


最简单的办法，**轮询匹配**，驱动程序满世界去询问各个板卡，你的基地址是多少，中断号是多少，或者通过设定的几个预设值去匹配。


**VxWorks传统驱动模型**和**Linux2.6**之前便是通过这种类似的形式去匹配的，比如pci设备的配置空间查询接口`pciFindDevice`，pci是有这样的一个机制，而其他板卡呢？比如与pci对应的`isa`，还有其他板级总线，`i2c、spi`呢？他们与基地址和中断号的联系更紧密，而且在不知道基地址的情形下他们无法给你回应，这个时候一般在创建设备的接口中预留这两个参数作为输入。


其实，你可以发现，这还是一个耦合的情况。


那么，我们就换一个思路，设计一个接口适配器（adapter）的类去适配不同的板级信息，基地址、中断号等硬件信息全部放到`adpter`里去维护，而驱动则通过`adpter`的接口去获取相对应的硬件信息。vxWorks6.6之后**vxBus**驱动模型和**Linux2.6**之后采用的便是这种方法，只不过不叫`adpter`，而叫做**总线**。通过总线（软件意义上的虚拟总线）去匹配设备和驱动，把设备和驱动绑定起来。

| 项目        |    功能    		|
| --------    | :-----:  			|
| 设备        | 描述硬件信息（基地址、中断号、时钟等）|
| 驱动        | 完成外设的具体功能   							 |
| 总线        | 完成设备和驱动的匹配 								|

其中Linux下项目目录如下：板级设备信息`arch`，驱动`drivers`，总线`drivers/base/platform.c`，vxWorks下项目目录如下：板级设备信息`target/config`，驱动`target/src/drv`，总线`target/src/hwif/vxbus`。

-----
### 设备驱动模型的实例
-----

按照我们上面的模型，首先就是硬件信息要录入到设备端，也就是在bsp中注册硬件设备相关信息，假设一个叫做roon的设备，**Linux**下示例如下：
```c
static struct resource roon_resources[] =
{
	[0] = {
		.start = ...,
		.end = ...,
		.flags = IORESOURCE_IO,
	},
};

static struct platform_device roon_platform_device =
{
	.name = "roon",
	.id = 0,
	.num_resources = ARRAY_SIZE(roon_resources),
	.resource = roon_resources,
	.dev =
	{
		.release = roon_platform_release,
		.platform_data = NULL,
	},
};
```
然后让设备向总线注册：
```c
if ((ret = platform_device_register(&roon_platform_device)) < 0)
{
	platform_device_put(&roon_platform_device);
	return ret;
}
```
如此一来，platform总线就知道了roon这个设备的硬件相关信息，但是驱动还是不知道，那我们同样把驱动注册进总线：
```c
static int roon_probe(struct platform_device *pdev)
{
	int ret;
	struct resource *res_io;

	/* 从设备资源获取GPIO编号 */
	res_io = platform_get_resource(pdev, IORESOURCE_IO, 0);
	rts_io = res_io->start;
	...
}
```
现在通过上面的处理，设备向总线注册了硬件信息，驱动也向总线注册了驱动模块，但是总线又是怎样把驱动好设备绑定起来的呢？内核中代码如下:
```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdrv = to_platform_driver(drv);

    /* When driver_override is set, only bind to the matching driver */
    if (pdev->driver_override)
        return !strcmp(pdev->driver_override, drv->name);

    /* Attempt an OF style match first */
    if (of_driver_match_device(dev, drv))
        return 1;

    /* Then try ACPI style match */
    if (acpi_driver_match_device(dev, drv))
        return 1;

    /* Then try to match against the id table */
    if (pdrv->id_table)
        return platform_match_id(pdrv->id_table, pdev) != NULL;

    /* fall-back to driver name match */
    return (strcmp(pdev->name, drv->name) == 0);
}
```
从代码中可以看出，platform总线下，设备与驱动的匹配是通过总线去匹配他们的名字来实现的，如果是相同的名字，则匹配成功。

在**vxWorks**下也大同小异，示例如下：
```c
const struct hcfResource roonResources[] =
{
    { "regBase", HCF_RES_INT, {(void *)ROON_BASE_ADR} },
    { "irq",     HCF_RES_INT, {(void *)(INUM_TO_IVEC(INT_NUM_ROON))} },
    { "clkFreq", HCF_RES_INT, {(void *)PCI_CLK_FREQ} },
    ...
};

#define roonNum NELEMENTS(roonResources)

struct hcfDevice hcfDeviceList[] =
{
	{ "roon", 0, VXB_BUSID_PLB, 0, roonNum, roonResources },
	...
}
```
通过代码不难分析，硬件设备信息已经注册到PLB总线上，同样，接下来是驱动也要注册进总线：
```c
LOCAL void roonvxbInstInit(struct vxbDev * pDev)
{
	if (pDev->busID == VXB_BUSID_PLB)
	{
		/* get the HCF_DEVICE address */
		pHcf = hcfDeviceGet(pDev);
		...
		if (devResourceGet (pHcf, "irq", HCF_RES_INT, (void *)&val) == OK)
		pChan->irq = (UINT16) val;
		...
	}
}

LOCAL struct drvBusFuncs roonvxbFuncs =
{
  roonvxbInstInit,        /* devInstanceInit */
  roonvxbInstInit2,       /* devInstanceInit2 */
  roonvxbInstConnect      /* devConnect */
};

LOCAL struct vxbDevRegInfo roonDrvRegistration =
{
    NULL,					/* pNext */
    VXB_DEVID_DEVICE,		/* devID */
    VXB_BUSID_PLB,		/* busID = PLB */
    VXB_VER_4_0_0,		/* vxbVersion */
    "roon",				/* drvName */
    &roonvxbFuncs,		/* pDrvBusFuncs */
    NULL,					/* pMethods */
    roonProbe,			/* devProbe */
    NULL					/* pParamDefaults */
};

void vxbRoonDrvRegister(void)
{
  /* call the vxBus routine to register the driver */
  vxbDevRegister ((struct vxbDevRegInfo *) &roonDrvRegistration);
}
```

同理，总线匹配其实也是通过名字去搜索匹配的，增加设备只需要在hcfDeviceList数组中添加即可，匹配后，驱动与设备形成一个实例，通过`vxbDevMethodGet()`接口来查询系统中的每一个实例。

-----
### 总结
-----

如上面分析的，最底层是不同硬件的板级文件，中间层是内核的虚拟总线，上面才是设备的驱动程序，现在板级文件与设备驱动程序已经解耦，但是随着时代的发展，设备越来越多，如果每次设备改动都要去修改板级文件(bsp)，那维护板级文件又是一个很大头的问题了。


时代在发展，设备在增多，人类的智慧也在增长，所以我们是绝对不允许这样的问题存在的，其实现在**Linux**在`arm体系`内核版本3.x后引入了原本`power pc`下的设备树(`DTS`)模型。vxWorks7.0也已经采用`DTS`来配置硬件设备了。关于设备树，有机会再详谈。


在软件编程当中，高内聚、低耦合一直是指导思想，所谓高内聚低耦合是模块内各元素联系越紧密就代表内聚性越高，模块间联系越不紧密就代表耦合性越低。所以高内聚、低耦合强调的就是内部要紧紧抱团。如上分析，设备和驱动就是基于这种模型去实现彼此隔离不相干的。

------

转载请注明原地址，迷死她张：[www.roon.pro](https://www.roon.pro) 谢谢！

作者 [迷死她张](https://www.roon.pro)

微信公众号 [迷死她张](http://mp.weixin.qq.com/mp/homepage?__biz=MzIxOTYyNjQ4Mg==&hid=4&sn=36c28244f4b4a44604719c1059709b7a#wechat_redirect)

个人微信号 roon93

2018 年 06月 28日
