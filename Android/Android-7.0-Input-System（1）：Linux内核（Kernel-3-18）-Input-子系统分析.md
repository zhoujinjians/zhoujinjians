---
title: Android Input System（1）：Linux内核（Kernel-3.18） - Linux Input 子系统分析
cover: https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.09.jpg
categories: 
  - Input
tags:
  - Android
  - Input
toc: true
abbrlink: 20180208
date: 2018-02-08 09:25:00
---



源码（部分）：

**kernel/msm-3.18/include/linux**

- Input.h
- evdev.h

**kernel/msm-3.18/drivers/input**

- Input.c
- evdev.c
- gpio_keys.c

**kernel/msm-3.18/drivers/input/touchscreen/synaptics_dsx_htc_2.6**

- Makefile
- Kconfig
- synaptics_dsx_core.c


Google Pixel、Pixel XL 内核代码（Kernel-3.18）：

[Kernel source for Pixel and Pixel XL - Google](https://android.googlesource.com/kernel/msm/+/android-msm-marlin-3.18-nougat-mr2-pixel) [Kernel source for Pixel and Pixel XL - GitHub](https://github.com/matthewdalex/marlin)

--------------------------------------------------------------------------------

## （一）、Linux Input 子系统框架

输入(Input)子系统是分层架构的，总共分为5 层，从上到下分别是：用户空间层（User Space）事件处理层(Event Handler)、输入子系统核心层(Input Core)、硬件驱动层(Input Driver) 、硬件设备层（Hardware）。

驱动根据CORE提供的接口，向上报告发生的按键动作。然后CORE根据驱动的类型，分派这个报告给对应的事件处理层进行处理。事件处理层把数据变化反应到设备模型的文件中（事件缓冲区）。并通知在这些设备模型文件上等待的进程。

input子系统框架： 

![Markdown](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/linux.input/01-Linux-kernel-input-subsystem-framework.png)

(1) "硬件驱动层"负责操作具体的硬件设备，这层的代码是针对具体的驱动程序的，比如你的设备是触摸输入设备，还是鼠标输入设备，还是键盘输入设备，这些不同的设备，自然有不同的硬件操作，驱动工程师往往只需要完成这层的代码编写。

(2) "输入子系统核心层"是链接其他两层之间的纽带与桥梁，向下提供硬件驱动层的接口，向上提供事件处理层的接口。

(3) "事件处理层" 负责与用户程序打交道，将硬件驱动层传来的事件报告给用户程序。

各层之间通信的基本单位就是事件，任何一个输入设备的动作都可以抽象成一种事件，如键盘的按下，触摸屏的按下，鼠标的移动等。事件有三种属性：类型（type），编码(code)，值(value)， Input 子系统支持的所有事件都定义在 input.h中，包括所有支持的类型，所属类型支持的编码等。事件传送的方向是 硬件驱动层-->子系统核心-->事件处理层-->用户空间。

## （二）、Input 主要通用数据结构

## 2.1、input_dev

输入设备 input_dev，这是input设备基本的设备结构，每个input驱动程序中都必须分配初始化这样一个结构

```c
[->input.h]
struct input_dev {
    const char *name;    //输入设备的名称  
    const char *phys;    //输入设备节点名称  
    const char *uniq;    //指定唯一的ID号，就像MAC地址一样
    struct input_id id;    //输入设备标识ID，用于和事件处理层进行匹配

    unsigned long propbit[BITS_TO_LONGS(INPUT_PROP_CNT)];

    unsigned long evbit[BITS_TO_LONGS(EV_CNT)];    //位图，记录设备支持的事件类型
    unsigned long keybit[BITS_TO_LONGS(KEY_CNT)];    //位图，记录设备支持的按键类型      
    unsigned long relbit[BITS_TO_LONGS(REL_CNT)];    //位图，记录设备支持的相对坐标  
    unsigned long absbit[BITS_TO_LONGS(ABS_CNT)];    //位图，记录设备支持的绝对坐标
    unsigned long mscbit[BITS_TO_LONGS(MSC_CNT)];    //位图，记录设备支持的其他功能
    unsigned long ledbit[BITS_TO_LONGS(LED_CNT)];    //位图，记录设备支持的指示灯
    unsigned long sndbit[BITS_TO_LONGS(SND_CNT)];   //位图，记录设备支持的声音或警报
    unsigned long ffbit[BITS_TO_LONGS(FF_CNT)];     //位图，记录设备支持的作用力功能  
    unsigned long swbit[BITS_TO_LONGS(SW_CNT)];     //位图，记录设备支持的开关功能

    unsigned int hint_events_per_packet;

    unsigned int keycodemax;      //设备支持的最大按键值个数  
    unsigned int keycodesize;    //每个按键的字节大小
    void *keycode;      //指向按键池，即指向按键值数组首地址

    int (*setkeycode)(struct input_dev *dev,
              const struct input_keymap_entry *ke,
              unsigned int *old_keycode);    //修改按键值
    int (*getkeycode)(struct input_dev *dev,
              struct input_keymap_entry *ke);   //获取按键值

    struct ff_device *ff;     //用于强制更新输入设备的部分内容  

    unsigned int repeat_key;    //重复按键的键值
    struct timer_list timer;   //设置当有连击时的延时定时器  

    int rep[REP_CNT];

    struct input_mt *mt;

    struct input_absinfo *absinfo;

    unsigned long key[BITS_TO_LONGS(KEY_CNT)];    //位图，按键的状态  
    unsigned long led[BITS_TO_LONGS(LED_CNT)];    //位图，led的状态  
    unsigned long snd[BITS_TO_LONGS(SND_CNT)];    //位图，声音的状态  
    unsigned long sw[BITS_TO_LONGS(SW_CNT)];   //位图，开关的状态  

    int (*open)(struct input_dev *dev);
    void (*close)(struct input_dev *dev);
    int (*flush)(struct input_dev *dev, struct file *file);
    int (*event)(struct input_dev *dev, unsigned int type, unsigned int code, int value);

    struct input_handle __rcu *grab; //类似私有指针，可以直接访问到事件处理接口event  

    spinlock_t event_lock;
    struct mutex mutex;

    unsigned int users;
    bool going_away;

    struct device dev;

    struct list_head    h_list; //该链表头用于链接此设备所关联的input_handle   
    struct list_head    node; //用于将此设备链接到input_dev_list(链接了所有注册到内核的事件处理器)  

    unsigned int num_vals;
    unsigned int max_vals;
    struct input_value *vals;

    bool devres_managed;
};
```

## 2.1、input_handler

input_handler 这是事件处理器的数据结构，代表一个事件处理器

```c
[->input.h]
struct input_handler {

    void *private;
    /* 当事件处理器接收到来自Input设备传来的事件时调用的处理函数,
        event、events用于处理事件 */  
    void (*event)(struct input_handle *handle, unsigned int type, unsigned int code, int value);
    void (*events)(struct input_handle *handle,
               const struct input_value *vals, unsigned int count);
    bool (*filter)(struct input_handle *handle, unsigned int type, unsigned int code, int value);
    /* 比较 device's id with handler's id_table ，匹配device and handler*/
    bool (*match)(struct input_handler *handler, struct input_dev *dev);
    /* connect用于建立intput_handler和input_dev的联系,
       当一个Input设备注册到内核的时候被调用,将输入设备与事件处理器联结起来 */
    int (*connect)(struct input_handler *handler, struct input_dev *dev, const struct input_device_id *id);
    /* disconnect用于解除handler和device的联系 */
    void (*disconnect)(struct input_handle *handle);
    void (*start)(struct input_handle *handle);

    bool legacy_minors;
    int minor;    //次设备号
    const char *name;    //次设备号

    const struct input_device_id *id_table;    //用于和device匹配 ,这个是事件处理器所支持的input设备

    //这个链表用来链接他所支持的input_handle结构,input_dev与input_handler配对之后就会生成一个input_handle结构
    struct list_head    h_list;

    //链接到input_handler_list，这个链表链接了所有注册到内核的事件处理器
    struct list_head    node;
};
```

## 2.3、input_handle

```c
[->input.h]
struct input_handle {
    /*
    每个配对的事件处理器都会分配一个对应的设备结构，如evdev事件处理器的evdev结构，
    注意这个结构与设备驱动层的input_dev不同，初始化handle时，保存到这里。   
    */

    void *private;
    /*
    打开标志，每个input_handle 打开后才能操作，
    这个一般通过事件处理器的open方法间接设置  
    */  

    int open;
    const char *name;
    /* 指向Input_dev结构实体 */
    struct input_dev *dev;
    /* 指向Input_Hander结构实体 */
    struct input_handler *handler;
    /* input_handle通过d_node连接到了input_dev上的h_list链表上 */
    struct list_head    d_node;
    /* input_handle通过h_node连接到了input_handler的h_list链表上 */
    struct list_head    h_node;
};
```

## 2.4、三个数据结构之间的关系

> input_dev: 是硬件驱动层，代表一个input设备。 input_handler: 是事件处理层，代表一个事件处理器。 input_handle: 属于核心层，代表一个配对的input_dev与input_handler

input_dev 通过全局的input_dev_list链接在一起。设备注册的时候实现这个操作。注：（稍后详细分析）

```c
[->input.c]
static LIST_HEAD(input_dev_list);
static LIST_HEAD(input_handler_list);

int input_register_device(struct input_dev *dev)
{
    struct input_devres *devres = NULL;
    struct input_handler *handler;
    unsigned int packet_size;
    const char *path;
    int error;

  ......
    list_add_tail(&dev->node, &input_dev_list);

    list_for_each_entry(handler, &input_handler_list, node)
        input_attach_handler(dev, handler);

   ......
}
```

input_handler 通过全局的input_handler_list链接在一起。事件处理器注册的时候实现这个操作（事件处理器一般内核自带，一般不需要我们来写）注：（稍后详细分析）

```c
[->input.c]
static LIST_HEAD(input_dev_list);
static LIST_HEAD(input_handler_list);

int input_register_handler(struct input_handler *handler)
{
    struct input_dev *dev;
    int error;
    ......
    list_add_tail(&handler->node, &input_handler_list);

    list_for_each_entry(dev, &input_dev_list, node)
        input_attach_handler(dev, handler);
    ......
    return 0;
}
```

input_hande 没有一个全局的链表，它注册的时候将自己分别挂在了input_dev 和 input_handler 的h_list上了。通过input_dev 和input_handler就可以找到input_handle在设备注册和事件处理器，注册的时候都要进行配对工作**(input_match_device)**，配对后就会实现链接**(handler->connect)**通过input_handle也可以找到input_dev和input_handler。注：（稍后详细分析）

```c
[->input.c]
static int input_attach_handler(struct input_dev *dev, struct input_handler *handler)
{
    const struct input_device_id *id;
    int error;

    id = input_match_device(handler, dev);
    ......
    error = handler->connect(handler, dev, id);
    ......
    return error;
}
```

我们可以看到，input_device和input_handler中都有一个h_list,而input_handle拥有指向input_dev和input_handler的指针，也就是说input_handle是用来关联input_dev和input_handler的。 那么为什么一个input_device和input_handler中拥有的是h_list而不是一个handle呢？因为一个device可能对应多个handler,而一个handler也不能只处理一个device,比如说一个鼠标，它可以对应even handler，也可以对应mouse handler,因此当其注册时与系统中的handler进行匹配，就有可能产生两个实例，一个是evdev,另一个是mousedev,而任何一个实例中都只有一个handle。至于以何种方式来传递事件，就由用户程序打开哪个实例来决定。后面一个情况很容易理解，一个事件驱动不能只为一个甚至一种设备服务，系统中可能有多种设备都能使用这类handler,比如event handler就可以匹配所有的设备。在input子系统中，有8种事件驱动，每种事件驱动最多可以对应32个设备，因此dev实例总数最多可以达到256个。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/linux.input/02-Linux-kernel-input-dev-handler.png)

## （三）、Input 核心层（Input.c）

这一节主要介绍核心层的初始化，input_device、input_handle、input_handler之间的关系(稍后回头看更佳)。 总体概览图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/linux.input/03-Linux-kernel-input-core-h_list.png)

## 3.1、输入核心层：初始化

首先从驱动"入口函数"开始查看

```c
[->input.c]
static int __init input_init(void)
{
    int err;
    //注册input类，可在/sys/class下看到对应节点文件
    err = class_register(&input_class);
    ......
    err = input_proc_init();/*创建/proc中的项，查看/proc/bus/input  */
    ......
    /*注册设备/dev/input，主设备号为INPUT_MAJOR，就是13，后面注册的输入设备都使用该主设备号*/
    err = register_chrdev_region(MKDEV(INPUT_MAJOR, 0),
                     INPUT_MAX_CHAR_DEVICES, "input");
    ......
    return 0;
    ......
    return err;
}
```

在入口函数里面创建了一个input_class类，其实就在/sys/class下创建了一个目录input.当然对于一个新设备，可以注册进一个class也可以不注册进去，如果存在对应class的话注册进去更好，另外在/proc创建了入口项,这样就可以/proc目录看到input的信息，然后就注册设备，可以看出输入子系统的主设备号是13，在这里并没有生成设备文件。只是在/dev/目录下创建了input目录，以后所有注册进系统的输入设备文件都放在这个目录下。

相应的对应关系可以使用adb 命令进入文件系统之后，cat /proc/bus/input/devices ，查看各个设备对应的event多少，比如Google Pixel 手机：

```c
I: Bus=0000 Vendor=0000 Product=0003 Version=2066
N: Name="synaptics_dsxv26"
P: Phys=synaptics_dsx/touch_input
S: Sysfs=/devices/soc/7577000.i2c/i2c-3/3-0020/input/input3
U: Uniq=
H: Handlers=mdss_fb kgsl event3
B: PROP=2
B: EV=b
B: KEY=8000 0 0
B: ABS=663800000000000
```

event3 就是事件序号， 我们在调试的时候直接 adb shell getevent /dev/input/event3，来实时捕捉 event3 中储存的数据。

那么接下来看看怎么注册input设备的.我们需要在设备驱动层中完成输入设备的注册，通过调用input_register_device()函数来完成，该函数的一个重要任务就是完成设备与事件驱动的匹配

## 3.2、输入核心层：注册设备input_dev

```c
[->input.c]
int input_register_device(struct input_dev *dev)
{
    struct input_devres *devres = NULL;
    struct input_handler *handler;
    unsigned int packet_size;
    const char *path;
    int error;

    if (dev->devres_managed) {
        devres = devres_alloc(devm_input_device_unregister,
                      sizeof(struct input_devres), GFP_KERNEL);
        ......
        devres->input = dev;
    }
    //EN_SYN这个是设备都要支持的事件类型，所以要设置   
    /* Every input device generates EV_SYN/SYN_REPORT events. */
    __set_bit(EV_SYN, dev->evbit);

    /* KEY_RESERVED is not supposed to be transmitted to userspace. */
    __clear_bit(KEY_RESERVED, dev->keybit);

    /* Make sure that bitmasks not mentioned in dev->evbit are clean. */
    input_cleanse_bitmasks(dev);

    packet_size = input_estimate_events_per_packet(dev);
    if (dev->hint_events_per_packet < packet_size)
        dev->hint_events_per_packet = packet_size;

    dev->max_vals = dev->hint_events_per_packet + 2;
    dev->vals = kcalloc(dev->max_vals, sizeof(*dev->vals), GFP_KERNEL);
    ......

    /*
     * If delay and period are pre-set by the driver, then autorepeating
     * is handled by the driver itself and we don't do it in input.c.
     */
     // 这个定时器是为了重复按键而设置的
    if (!dev->rep[REP_DELAY] && !dev->rep[REP_PERIOD]) {
        dev->timer.data = (long) dev;
        dev->timer.function = input_repeat_key;
        dev->rep[REP_DELAY] = 250;
        dev->rep[REP_PERIOD] = 33;
    }
    /* 如果设备驱动没有设置自己的获取键值的函数，系统默认 */  
    if (!dev->getkeycode)
        dev->getkeycode = input_default_getkeycode;
    /* 如果设备驱动没有指定按键重置函数，系统默认 */  
    if (!dev->setkeycode)
        dev->setkeycode = input_default_setkeycode;

    error = device_add(&dev->dev);
    ......
    path = kobject_get_path(&dev->dev.kobj, GFP_KERNEL);
    pr_info("%s as %s\n",
        dev->name ? dev->name : "Unspecified device",
        path ? path : "N/A");
    kfree(path);

    error = mutex_lock_interruptible(&input_mutex);
    ......
    // 将新分配的input设备连接到input_dev_list链表上  
    list_add_tail(&dev->node, &input_dev_list);
    /* 核心重点，input设备在增加到input_dev_list链表上之后，会查找
     * input_handler_list事件处理链表上的handler进行匹配，这里的匹配
     * 方式与设备模型的device和driver匹配过程很相似，所有的input
     * 都挂在input_dev_list上，所有类型的事件都挂在input_handler_list
     * 上，进行“匹配相亲”，list_for_each_entry就是个for循环，跳出条件遍历了一遍，又回到链表头 */  
    list_for_each_entry(handler, &input_handler_list, node)
        input_attach_handler(dev, handler);

    input_wakeup_procfs_readers();

    mutex_unlock(&input_mutex);

    if (dev->devres_managed) {
        dev_dbg(dev->dev.parent, "%s: registering %s with devres.\n",
            __func__, dev_name(&dev->dev));
        devres_add(dev->dev.parent, devres);
    }
    return 0;

......
}
```

上面的代码主要的功能有以下几个功能，也是设备驱动注册为输入设备委托内核做的事情：

1、进一步初始化输入设备，例如连击事件 2、注册输入设备到input类中，把输入设备挂到输入设备链表input_dev_list中 3、查找并匹配输入设备对应的事件处理层，通过input_handler_list链表

我们需要再分析下这个匹配的过程，input_attach_handler()匹配过程如下：

```c
[->input.c]
static int input_attach_handler(struct input_dev *dev, struct input_handler *handler)
{
    const struct input_device_id *id;
    int error;
    /* input_dev 和 input_handler 进行匹配,返回匹配的id，类型是struct input_device_id  */  
    id = input_match_device(handler, dev);
    ......
    /* 匹配成功，调用handler里面的connect函数,这个函数在事件处理器中定义，主要生成一个input_handle结构，并初始化，还生成一个事件处理器相关的设备结构 */  
    error = handler->connect(handler, dev, id);
    ......
    return error;
}
```

我们先来看下input_match_device（）函数，看一下这个匹配的条件是什么，如何匹配的过程是怎样的，匹配的结果会是什么

```c
[->input.c]
static const struct input_device_id *input_match_device(struct input_handler *handler,
                            struct input_dev *dev)
{
    const struct input_device_id *id;

    for (id = handler->id_table; id->flags || id->driver_info; id++) {

        if (id->flags & INPUT_DEVICE_ID_MATCH_BUS)
            if (id->bustype != dev->id.bustype) //匹配总线id
                continue;

        if (id->flags & INPUT_DEVICE_ID_MATCH_VENDOR)
            if (id->vendor != dev->id.vendor)  //匹配生产商id
                continue;

        if (id->flags & INPUT_DEVICE_ID_MATCH_PRODUCT)
            if (id->product != dev->id.product)  //匹配产品id  
                continue;

        if (id->flags & INPUT_DEVICE_ID_MATCH_VERSION)
            if (id->version != dev->id.version) //匹配版本  
                continue;

        //匹配id的evbit和input_dev中evbit的各个位，如果不匹配则continue，数组中下一个设备  
        if (!bitmap_subset(id->evbit, dev->evbit, EV_MAX))
            continue;
        ......
        if (!bitmap_subset(id->swbit, dev->swbit, SW_MAX))
            continue;

        if (!handler->match || handler->match(handler, dev))
            return id;//匹配成功,返回id
    }

    return NULL;
}
```

input_match_device() 到最合适的事件处理层驱动时，便执行handler->connect() 函数进行连接了，看下面这部分代码（以evdev类型驱动为例，在input/evdev.c中）

```c
[->evdev.c]
static int evdev_connect(struct input_handler *handler, struct input_dev *dev,
             const struct input_device_id *id)
{
    struct evdev *evdev;
    int minor;
    int dev_no;
    int error;
    /* EVDEV_MINORS为32，代表共能容纳32个evdev事件层设备，下面代码在找到空的地方，用于保存evdev事件层的数据，即上面定义的evdev */
    minor = input_get_new_minor(EVDEV_MINOR_BASE, EVDEV_MINORS, true);
    ......
    /* 开始给evdev事件层驱动分配空间了 */
    evdev = kzalloc(sizeof(struct evdev), GFP_KERNEL);
    ......
    /* 初始化client_list列表和evdev_wait队列，后面介绍 */
    INIT_LIST_HEAD(&evdev->client_list);
    spin_lock_init(&evdev->client_lock);
    mutex_init(&evdev->mutex);
    init_waitqueue_head(&evdev->wait);
    evdev->exist = true;
    /* 初始化evdev结构体，其中handle为输入设备和事件处理的关联接口 */
    dev_no = minor;
    /* Normalize device number if it falls into legacy range */
    if (dev_no < EVDEV_MINOR_BASE + EVDEV_MINORS)
        dev_no -= EVDEV_MINOR_BASE;
    dev_set_name(&evdev->dev, "event%d", dev_no);
    /*这里就将handle的dev指针指向了input_dev*/
    evdev->handle.dev = input_get_device(dev);
    evdev->handle.name = dev_name(&evdev->dev);
    evdev->handle.handler = handler;/*这里将handle的handler指向了当前的input_handler.注意本函数evdev_connect,可能是在在输入设备注册的时候
38     在input_register_device函数中调用input_attach_handler的时候调用;也可能是在输入设备的处理方法input_handler时在input_register_handler
39     函数中也会用到input_attach_handler函数,就会调用本函数.这里就很明显了,本函数就将input_handler和input_dev都放在input_handle中统一管理*/
    evdev->handle.private = evdev;

    /*初始化evdev中的内嵌device*/
    evdev->dev.devt = MKDEV(INPUT_MAJOR, minor);
    evdev->dev.class = &input_class;
    evdev->dev.parent = &dev->dev;
    evdev->dev.release = evdev_free;
    device_initialize(&evdev->dev);
   /* input_dev设备驱动层和input_handler事件处理层的关联，由input_handle完成(不要和handler搞混淆了，这不是一个概念～) */  
    error = input_register_handle(&evdev->handle);
    ......

    cdev_init(&evdev->cdev, &evdev_fops);
    evdev->cdev.kobj.parent = &evdev->dev.kobj;
    error = cdev_add(&evdev->cdev, evdev->dev.devt, 1);
    ......
    error = device_add(&evdev->dev);
    ......
    return 0;
    ......
}
```

## 3.3、输入核心层：注册input_handler

为了逻辑更清新，我们稍后再来看input_register_handle() 程，先来了解input_handler的注册过程。 要了解input_handler的注册过程，又需要先了解evdev初始化过程（以evdev为例）： /kernel/drivers/input下众多事件处理器handler其中的一个，可以看下源码/kernel/drivers/input/evdev.c中的模块init

```c
[->edev.c]
static int __init evdev_init(void)
{
    return input_register_handler(&evdev_handler);
}
```

这个初始化就是往input核心中注册一个input_handler类型的evdev_handler，调用的是input.c提供的接口，input_handler结构前面有介绍，看下evdev_handler的赋值：

```c
[->edev.c]
static struct input_handler evdev_handler = {
    .event        = evdev_event,
    .events        = evdev_events,
    .connect    = evdev_connect,
    .disconnect    = evdev_disconnect,
    .legacy_minors    = true,
    .minor        = EVDEV_MINOR_BASE,
    .name        = "evdev",
    .id_table    = evdev_ids,
};
```

可以注意的是evdev是匹配所有设备的，因为：

```c
[->edev.c]
static const struct input_device_id evdev_ids[] = {
    { .driver_info = 1 },    /* Matches all devices */
    { },            /* Terminating zero entry */
};
```

```c
[->input.c]
int input_register_handler(struct input_handler *handler)
{
    struct input_dev *dev;
    int error;

    error = mutex_lock_interruptible(&input_mutex);
    ......

    INIT_LIST_HEAD(&handler->h_list);
    //添加进input_handler_list全局链表
    list_add_tail(&handler->node, &input_handler_list);
    //同样遍历input_dev这个链表，依次调用下面的input_attach_handler去匹配input_dev,这个跟input_dev注册的时候的情形类似  
    list_for_each_entry(dev, &input_dev_list, node)
        input_attach_handler(dev, handler);

    input_wakeup_procfs_readers();

    mutex_unlock(&input_mutex);
    return 0;
}
```

## 3.4、输入核心层：注册input_handle（不要和handler搞混淆了哦，这不是一个概念～）

input_handle关联匹配input_dev和input_handler 继续分析input_dev和input_handler 是如何关联上的

```c
[->input.c]
int input_register_handle(struct input_handle *handle)
{
    struct input_handler *handler = handle->handler;
    struct input_dev *dev = handle->dev;
    int error;

    ......
    error = mutex_lock_interruptible(&dev->mutex);
    ......

    /* 将d_node链接到输入设备的h_list，h_node链接到事件层的h_list链表上
    * 因此，在handle中是输入设备和事件层的关联结构体，通过输入设备可以
    * 找到对应的事件处理层接口，通过事件处理层也可找到匹配的输入设备
    */  

    //把这个handle的d_node 加到对应input_dev的h_list链表里面  
    if (handler->filter)
        list_add_rcu(&handle->d_node, &dev->h_list);
    else
        list_add_tail_rcu(&handle->d_node, &dev->h_list);

    mutex_unlock(&dev->mutex);

    ......
    //把这个handle的h_node 加到对应input_handler的h_list链表里面
    list_add_tail_rcu(&handle->h_node, &handler->h_list);

    if (handler->start)
        handler->start(handle);

    return 0;
}
```

这个注册是把handle 本身的链表加入到它自己的input_dev 以及 input_handler的h_list链表中，这样以后就可以通过h_list遍历到这个handle，这样就实现了三者的绑定联系。

以上是输入设备驱动注册的全过程，纵观整个过程，输入设备驱动最终的目的就是能够与事件处理层的事件驱动相互匹配，但是在drivers/input目录下有evdev.c事件驱动、mousedev.c事件驱动、joydev.c事件驱动等等，我们的输入设备产生的事件应该最终上报给谁，然后让事件被谁去处理呢？知道了这么个原因再看上面代码就会明白，其实evdev.c、mousedev.c等根据硬件输入设备的处理方式的不同抽象出了不同的事件处理接口帮助上层去调用，而我们写的设备驱动程序只不过是完成了硬件寄存器中数据的读写，但提交给用户的事件必须是经过事件处理层的封装和同步才能够完成的，事件处理层提供给用户一个统一的界面来操作。 整个关联注册的过程： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/linux.input/04-Linux-kernel-input-reg-device.png)

## （四）、Input 事件处理层 Event handler （以evdev事件处理器为例）

![Markdown](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/linux.input/05-Linux-kernel-input-event-hardware.png)

## 4.1、主要数据结构

**（1） evdev设备结构**

```c
[evdev.h]
struct evdev {   
    int exist;   
    int open;                     //打开标志   
    int minor;                    //次设备号   
    struct input_handle handle;   //关联的input_handle   
    wait_queue_head_t wait;       //等待队列，当进程读取设备，而没有事件产生的时候，进程就会睡在其上面   
    struct evdev_client *grab;    //强制绑定的evdev_client结构，这个结构后面再分析   
    struct list_head client_list; //evdev_client 链表，这说明一个evdev设备可以处理多个evdev_client，可以有多个进程访问evdev设备   
    spinlock_t client_lock;       /* protects client_list */   
    struct mutex mutex;   
    struct device dev;            //device结构，说明这是一个设备结构   
};
```

evdev结构体在配对成功的时候生成，由handler->connect生成，对应设备文件为/class/input/event(n)，如触摸屏驱动的event3，这个设备是用户空间要访问的设备，可以理解它是一个虚拟设备，因为没有对应的硬件，但是通过handle->dev 就可以找到input_dev结构，而它对应着触摸屏，设备文件为/class/input/input3。这个设备结构生成之后保存在evdev_table中，索引值是minor。 **（2）evdev用户端结构**

```c
[evdev.h]
struct evdev_client {  
    unsigned int head;  //buffer数组的索引头  
    unsigned int tail;   //buffer数组的索引尾  
    unsigned int packet_head; /* [future] position of the first element of next packet */  
    spinlock_t buffer_lock; /* protects access to buffer, head and tail */  
    struct wake_lock wake_lock;  
    bool use_wake_lock;  
    char name[28];  
    struct fasync_struct *fasync;    //异步通知函数  
    struct evdev *evdev;  //包含一个evdev变量  
    struct list_head node;  //链表  
    unsigned int bufsize;  
    struct input_event buffer[];   //input_event数据结构的数组，input_event代表一个事件，基本成员：类型（type），编码（code），值（value）  
};
```

这个结构在进程打开event3设备的时候调用evdev的open方法，在open中创建这个结构，并初始化。在关闭设备文件的时候释放这个结构。 **（3）input_event结构**

```c
[input.h]
struct input_event {
    struct timeval time;    //事件发生的时间   
    __u16 type;             //事件类型   
    __u16 code;             //子事件   
    __s32 value;            //事件的value  
};
```

## 4.2、事件处理层evdev

事件处理层与用户程序和输入子系统核心打交道，是他们两层的桥梁。一般内核有好几个事件处理器，像evdev mousedev jotdev。evdev事件处理器可以处理所有的事件，触摸屏驱动就是用的这个，所以下面分析这个事件处理器的实现。它也是作为模块注册到内核中的,前面已经分析过它的模块初始化函数

```c
[->evdev.c]
static const struct file_operations evdev_fops = {
    .owner        = THIS_MODULE,
    .read        = evdev_read,
    .write        = evdev_write,
    .poll        = evdev_poll,
    .open        = evdev_open,
    .release    = evdev_release,
    .unlocked_ioctl    = evdev_ioctl,
#ifdef CONFIG_COMPAT
    .compat_ioctl    = evdev_ioctl_compat,
#endif
    .fasync        = evdev_fasync,
    .flush        = evdev_flush,
    .llseek        = no_llseek,
};
```

如果匹配上了就会创建一个evdev，它里边封装了一个handle，会把input_dev和input_handler关联到一起。关系如下：

![Markdown](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/linux.input/06-Linux-kernel-input-evdev-connect.png)

## 4.3、evdev设备结点的open()操作

我们知道.对主设备号为INPUT_MAJOR的设备节点进行操作，会将操作集转换成handler的操作集。在evdev中，这个操作集就是evdev_fops。对应的open函数如下示：

首先来看打开event(x)设备文件，evdev_open函数.

```c
[->evdev.c]
static int evdev_open(struct inode *inode, struct file *file)
{
    struct evdev *evdev = container_of(inode->i_cdev, struct evdev, cdev);
    //evdev_client的buffer大小  
    unsigned int bufsize = evdev_compute_buffer_size(evdev->handle.dev);
    unsigned int size = sizeof(struct evdev_client) +
                    bufsize * sizeof(struct input_event);
    struct evdev_client *client;
    int error;
    //打开的时候创建一个evdev_client
    client = kzalloc(size, GFP_KERNEL | __GFP_NOWARN);
    ......

    client->bufsize = bufsize;
    spin_lock_init(&client->buffer_lock);
    snprintf(client->name, sizeof(client->name), "%s-%d",
            dev_name(&evdev->dev), task_tgid_vnr(current));
    client->evdev = evdev;
    evdev_attach_client(evdev, client);
    //调用打开真正的底层设备函数  
    error = evdev_open_device(evdev);
    ......

    file->private_data = client;
    nonseekable_open(inode, file);

    return 0;
    ......
}


static int evdev_open_device(struct evdev *evdev)
{
    int retval;

    retval = mutex_lock_interruptible(&evdev->mutex);
    if (retval)/*如果设备不存在，返回错误*/  
        return retval;

    if (!evdev->exist)
        retval = -ENODEV;
    else if (!evdev->open++) {//递增打开计数  
        retval = input_open_device(&evdev->handle);//如果是被第一次打开，则调用input_open_device
        if (retval)
            evdev->open--;
    }

    mutex_unlock(&evdev->mutex);
    return retval;
}


int input_open_device(struct input_handle *handle)
{
    struct input_dev *dev = handle->dev;//根据input_handle找到对应的input_dev设备  
    int retval;

    retval = mutex_lock_interruptible(&dev->mutex);
    ......

    handle->open++;//递增handle的打开计数  

    if (!dev->users++ && dev->open)
        retval = dev->open(dev);//如果是第一次打开.则调用input device的open()函数  

    if (retval) {
        dev->users--;
        if (!--handle->open) {
            synchronize_rcu();
        }
    }

 out:
    mutex_unlock(&dev->mutex);
    return retval;
}
```

## 4.4、用户进程读取event的底层实现

至于具体的如何初始化input_dev，这个是具体的输入设备去实现的，稍后具体实例再分析，现在来看看，对于一个event(x)设备文件的，应用程序来读，最终会导致"handler"里面的的"读函数"被调用。

evdev_fops 结 构 体 是 一 个 file_operations 的 类 型 。 当 用 户 层 调 用 类 似 代 码open("/dev/input/event3" , O_RDONLY) 函 数 打 开 设 备 结 点 时 , 会 调 用 evdev_fops 中 的evdev_read()函数,该函数的代码如下：

```c
[->evdev.c]
static ssize_t evdev_read(struct file *file, char __user *buffer,
              size_t count, loff_t *ppos)
{
    struct evdev_client *client = file->private_data;//就是刚才在open函数中保存的evdev_client  
    struct evdev *evdev = client->evdev;
    struct input_event event;
    size_t read = 0;
    int error;

    for (;;) {
        ......
        //如果获得了数据则取出来，调用evdev_fetch_next_event  
        while (read + input_event_size() <= count &&
               evdev_fetch_next_event(client, &event)) {

        //input_event_to_user调用copy_to_user传入用户程序中，这样读取完成  
            if (input_event_to_user(buffer + read, &event))
                return -EFAULT;

            read += input_event_size();
        }
        ......
        /*如果是可阻塞状态的话,则等待在wait队列上.直到有数据要被处理,当前进程才被唤醒.这很好理解,既然是
         输入设备,读的话比如读按键,那么必须要有硬件设备有按键按下才会返回按键值,这里还是处于事件处理层,应用程序在这里休眠,那么谁来唤醒?
         当然是有按键按下才去唤醒,因此这个工作就交给了设备驱动层,那么找到这个唤醒呢,直接去找不好找,那么可以直接搜索evdev->wait,搜索结果
         可知evdev->wait在evdev_event()函数中被唤醒*/
        if (!(file->f_flags & O_NONBLOCK)) {
            error = wait_event_interruptible(evdev->wait,
                    client->packet_head != client->tail ||
                    !evdev->exist || client->revoked);
            if (error)
                return error;
        }
    }

    return read;
}

static int evdev_fetch_next_event(struct evdev_client *client,
                  struct input_event *event)
{
    int have_event;

    spin_lock_irq(&client->buffer_lock);
    /*先判断一下是否有数据*/   
    have_event = client->packet_head != client->tail;
    /*如果有就从环形缓冲区的取出来，记得是从head存储，tail取出*/
    if (have_event) {
        *event = client->buffer[client->tail++];
        client->tail &= client->bufsize - 1;
        if (client->use_wake_lock &&
            client->packet_head == client->tail)
            wake_unlock(&client->wake_lock);
    }

    spin_unlock_irq(&client->buffer_lock);

    return have_event;
}

int input_event_to_user(char __user *buffer,
            const struct input_event *event)
{
    /*如果设置了标志INPUT_COMPAT_TEST就将事件event包装成结构体compat_event*/
    if (INPUT_COMPAT_TEST && !COMPAT_USE_64BIT_TIME) {
        struct input_event_compat compat_event;

        compat_event.time.tv_sec = event->time.tv_sec;
        compat_event.time.tv_usec = event->time.tv_usec;
        compat_event.type = event->type;
        compat_event.code = event->code;
        compat_event.value = event->value;
         /*将包装成的compat_event拷贝到用户空间*/  
        if (copy_to_user(buffer, &compat_event,
                 sizeof(struct input_event_compat)))
            return -EFAULT;

    } else {
       /*否则，将event拷贝到用户空间*/   
        if (copy_to_user(buffer, event, sizeof(struct input_event)))
            return -EFAULT;
    }

    return 0;
}
```

如果是可阻塞状态的话，则等待在wait队列上。直到有数据要被处理，当前进程才被唤醒。这很好理解，既然是输入设备，读的话比如读按键，那么必须要有硬件设备有按键按下才会返回按键值，这里还是处于事件处理层，应用程序在这里休眠，那么谁来唤醒?

当然是有按键按下才去唤醒，因此这个工作就交给了设备驱动层。那么找到这个唤醒呢，直接去找不好找。那么可以直接搜索evdev->wait，搜索结果可知evdev->wait在evdev_event()函数中被唤醒

注释中说的很清楚，evdev_event()会唤醒此处的读按键进程。那么evdev_event()又是被谁调用?显然是设备驱动层，现在看一个设备层例子，内核中有个按键的例子，gpiokey.c，这只是个例子不针对任何设备，在gpio_keys.c终端处理函数里面

```c
[->gpio_keys.c]
static irqreturn_t gpio_keys_irq_isr(int irq, void *dev_id)
{
    ......
    if (!bdata->key_pressed) {
        ......
        input_event(input, EV_KEY, button->code, 1);
        input_sync(input);

        ......
    }

    ......
}
```

如此可以看出 在设备的中断服务程序里面，确定事件是什么，然后调用相应的input_handler的event处理函数 实际上这就是我们的核心 input_event()是用来上报事件的

```c
[->input.c]
void input_event(struct input_dev *dev,
         unsigned int type, unsigned int code, int value)
{
    unsigned long flags;

    if (is_event_supported(type, dev->evbit, EV_MAX)) {

        spin_lock_irqsave(&dev->event_lock, flags);
        input_handle_event(dev, type, code, value);
        spin_unlock_irqrestore(&dev->event_lock, flags);
    }
}

static void input_handle_event(struct input_dev *dev,
                   unsigned int type, unsigned int code, int value)
{
    ......
    if (disposition & INPUT_FLUSH) {
        if (dev->num_vals >= 2)
            input_pass_values(dev, dev->vals, dev->num_vals);
        dev->num_vals = 0;
    } else if (dev->num_vals >= dev->max_vals - 2) {
        dev->vals[dev->num_vals++] = input_value_sync;
        input_pass_values(dev, dev->vals, dev->num_vals);
        dev->num_vals = 0;
    }

}

static void input_pass_values(struct input_dev *dev,
                  struct input_value *vals, unsigned int count)
{
    struct input_handle *handle;
    struct input_value *v;
    ......
    handle = rcu_dereference(dev->grab);
    if (handle) {
        count = input_to_handler(handle, vals, count);
    } else {
        list_for_each_entry_rcu(handle, &dev->h_list, d_node)
            if (handle->open)
                count = input_to_handler(handle, vals, count);
    }
    ......
}

static unsigned int input_to_handler(struct input_handle *handle,
            struct input_value *vals, unsigned int count)
{
    struct input_handler *handler = handle->handler;
    struct input_value *end = vals;
    struct input_value *v;
    ......

    if (handler->events)
        handler->events(handle, vals, count);
    else if (handler->event)
        for (v = vals; v != end; v++)
            handler->event(handle, v->type, v->code, v->value);

    return count;
}
```

可以看到最终调用handler->event()来处理，此处handler即对应evdev。

```c
[->input.c]
handler->event(handle, v->type, v->code, v->value)
```

所以会调用evdev_event()函数

```c
[->evdev.c]
static void evdev_pass_values(struct evdev_client *client,
            const struct input_value *vals, unsigned int count,
            ktime_t mono, ktime_t real)
{
    struct evdev *evdev = client->evdev;
    const struct input_value *v;
    struct input_event event;
    bool wakeup = false;

    if (client->revoked)
        return;

    event.time = ktime_to_timeval(client->clkid == CLOCK_MONOTONIC ?
                      mono : real);

    /* Interrupts are disabled, just acquire the lock. */
    spin_lock(&client->buffer_lock);

    for (v = vals; v != vals + count; v++) {
        event.type = v->type;
        event.code = v->code;
        event.value = v->value;
        __pass_event(client, &event);
        if (v->type == EV_SYN && v->code == SYN_REPORT)
            wakeup = true;
    }

    spin_unlock(&client->buffer_lock);

    if (wakeup)
        wake_up_interruptible(&evdev->wait);
}

static void evdev_events(struct input_handle *handle,
             const struct input_value *vals, unsigned int count)
{
    struct evdev *evdev = handle->private;
    struct evdev_client *client;
    ......
    if (client)
        evdev_pass_values(client, vals, count, time_mono, time_real);
    else
        list_for_each_entry_rcu(client, &evdev->client_list, node)
            evdev_pass_values(client, vals, count,
                      time_mono, time_real);

    rcu_read_unlock();
}


static void evdev_event(struct input_handle *handle,
            unsigned int type, unsigned int code, int value)
{
    struct input_value vals[] = { { type, code, value } };
    evdev_events(handle, vals, 1);
}
```

最终唤醒evdev_read()将数据拷贝到用户空间。

## （五）、Input 事件上报过程

## 5.1、Input 事件产生

当按下触摸屏时，进入触摸屏按下中断，开始ad转换，ad转换完成进入ad完成中断，在这个终端中将事件发送出去，会调用以下函数上报事件:

```c
    input_report_key(input_dev,
            BTN_TOUCH, 1);

    input_report_abs(input_dev,
            ABS_POSITION_X, x);

    input_report_abs(input_dev,
            ABS_POSITION_Y, y);

   input_sync(input_dev);
```

这两个函数调用了 input_event(dev, EV_ABS, code, value) 所有的事件报告函数都调用这个函数。

```c
[->input.h]
static inline void input_report_key(struct input_dev *dev, unsigned int code, int value)
{
    input_event(dev, EV_KEY, code, !!value);
}

static inline void input_report_abs(struct input_dev *dev, unsigned int code, int value)
{
    input_event(dev, EV_ABS, code, value);
}

static inline void input_sync(struct input_dev *dev)
{
    input_event(dev, EV_SYN, SYN_REPORT, 0);
}
```

## 5.2、Input 事件报告

input_event 函数前面已经分析过，这里不再分析。

```c
[->input.c:input_pass_values]
for (v = vals; v != end; v++)
    handler->event(handle, v->type, v->code, v->value);
```

最终会调用handler->event(handle, v->type, v->code, v->value) 来将数据 传递给用户空间等待读取数据的进程

```c
copy_to_user(buffer, event, sizeof(struct input_event))
```

## （六）、Android Input子系统

输入子系统的系统架构如下图所示： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/linux.input/07-Linux-kernel-android-input-system.png)

详细分析请参考：Android 7.1.2 (Android N) Android 输入子系统-Input System 分析

## （七）、Input 设备驱动层实例（Synaptics）

触摸屏也是用上面这一套框架来操作的。右边需要一个"evdev.c"文件。左边要分配一个"input_dev"结构。接着就看上图的硬件设备左边的过程：分配一个"input_dev"结构体 --> 设置这个"input_dev"结构体 --> 注册这个"input_dev"结构体 --> 硬件相关的操作。 
![Markdown](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/linux.input/08-Linux-kernel-input-drivers-fw.png)


编写Input驱动一般框架:

Google Pixel、Pixel XL 触控驱动模块型号为Synaptics（ClearPad S3708），源码：[Synaptics 触摸屏驱动源码](https://github.com/matthewdalex/marlin/tree/2f567606935d601f1391ad9575b103f35737a438/drivers/input/touchscreen/synaptics_dsx_htc_2.6)

Makefile：

```c
[->drivers/input/touchscreen/synaptics_dsx_htc_2.6/Makefile]
#
# Makefile for the Synaptics DSX touchscreen driver.
#

# Each configuration option enables a list of files.

obj-$(CONFIG_TOUCHSCREEN_SYNAPTICS_DSX_I2C_HTC_v26) += synaptics_dsx_i2c.o
obj-$(CONFIG_TOUCHSCREEN_SYNAPTICS_DSX_SPI_HTC_v26) += synaptics_dsx_spi.o
obj-$(CONFIG_TOUCHSCREEN_SYNAPTICS_DSX_CORE_HTC_v26) += synaptics_dsx_core.o
obj-$(CONFIG_TOUCHSCREEN_SYNAPTICS_DSX_RMI_DEV_HTC_v26) += synaptics_dsx_rmi_dev.o
......
```

抓取kernel log：可知input 驱动名为synaptics_dsxv26，全局搜索可知synaptics_rmi4_f12_init在[->synaptics_dsx_core.c]中。

```c
[    1.362728] c3      1 [TP]:synaptics_rmi4_f12_init: Function 12 max x = 1079 max y = 1919 Rx: 16 Tx: 28
[    1.363344] c3      1 [TP]synaptics_rmi4_f12_init:Wakeup Gesture range (0,0) -> (1079,1919)
[    1.363623] c3      1 [TP]:synaptics_rmi4_f12_init report data init done
[    1.371945] c3      1 [TP]:synaptics_rmi4_query_device: chip_id:3708, firmware_id:2433782
[    1.372865] c3      1 [TP]:synaptics_rmi4_query_device: config_version: 5331763200190000000000000000000000000000000000000000000000000000
[    1.373249] c3      1 input: synaptics_dsxv26 as /devices/soc/7577000.i2c/i2c-3/3-0020/input/input3
```

查看input设备：adb shell cat /proc/bus/input/devices

```c
I: Bus=0000 Vendor=0000 Product=0003 Version=2066
N: Name="synaptics_dsxv26"
P: Phys=synaptics_dsx/touch_input
S: Sysfs=/devices/soc/7577000.i2c/i2c-3/3-0020/input/input3
U: Uniq=
H: Handlers=mdss_fb kgsl event3
B: PROP=2
B: EV=b
B: KEY=8000 0 0
B: ABS=663800000000000

对应：/dev/input/event3
```

## 7.1、分配Input_dev结构体

## 7.1.1、synaptics_rmi4_f12_init()

首先看一下初始化过程：

```c
[->synaptics_dsx_core.c]
static struct platform_driver synaptics_rmi4_driver = {
    .driver = {
        .name = PLATFORM_DRIVER_NAME,
        .owner = THIS_MODULE,
#ifdef CONFIG_PM
        .pm = &synaptics_rmi4_dev_pm_ops,
#endif
    },
    .probe = synaptics_rmi4_probe,
    .remove = synaptics_rmi4_remove,
};

static int __init synaptics_rmi4_init(void)
{
    int retval;

    retval = synaptics_rmi4_bus_init();
    if (retval)
        return retval;

    return platform_driver_register(&synaptics_rmi4_driver);
}
module_init(synaptics_rmi4_init);
```

首先注册平台驱动，当驱动和设备匹配成功，继续看一下synaptics_rmi4_probe()函数

## 7.1.2、synaptics_rmi4_probe()

```c
[->synaptics_dsx_core.c]
static int synaptics_rmi4_probe(struct platform_device *pdev)
{
    int retval, len;
    unsigned char attr_count;
    struct synaptics_rmi4_data *rmi4_data;
    const struct synaptics_dsx_hw_interface *hw_if;
    const struct synaptics_dsx_board_data *bdata;
    struct dentry *temp;
    //初始化platform_data、board_data、rmi4_data
    hw_if = pdev->dev.platform_data;
    bdata = hw_if->board_data;
    rmi4_data = kzalloc(sizeof(*rmi4_data), GFP_KERNEL);

    rmi4_data->pdev = pdev;
    rmi4_data->current_page = MASK_8BIT;
    rmi4_data->hw_if = hw_if;
    rmi4_data->touch_stopped = false;
    rmi4_data->sensor_sleep = false;
    rmi4_data->irq_enabled = false;
    rmi4_data->fw_updating = false;
    rmi4_data->fingers_on_2d = false;
    rmi4_data->update_coords = true;

    rmi4_data->write_buf = devm_kzalloc(&pdev->dev, I2C_WRITE_BUF_MAX_LEN,
                    GFP_KERNEL);
    rmi4_data->write_buf_len = I2C_WRITE_BUF_MAX_LEN;

    rmi4_data->irq_enable = synaptics_rmi4_irq_enable;
    rmi4_data->reset_device = synaptics_rmi4_reset_device;

    mutex_init(&(rmi4_data->rmi4_io_ctrl_mutex));
    mutex_init(&(rmi4_data->rmi4_reset_mutex));

    retval = synaptics_dsx_regulator_configure(rmi4_data);
    retval = synaptics_dsx_regulator_enable(rmi4_data, true);

    platform_set_drvdata(pdev, rmi4_data);

    if (bdata->gpio_config) {
        retval = synaptics_rmi4_set_gpio(rmi4_data);
    } else {
        retval = synaptics_dsx_pinctrl_init(rmi4_data);
        if (!retval && rmi4_data->ts_pinctrl) {
            retval = pinctrl_select_state(rmi4_data->ts_pinctrl,
                    rmi4_data->pinctrl_state_active);
        }

        retval = synaptics_dsx_gpio_configure(rmi4_data, true);
    }

    if (bdata->fw_name) {
        len = strlen(bdata->fw_name);

        strlcpy(rmi4_data->fw_name, bdata->fw_name, len + 1);
    }
    //分配Input_dev结构体，设置，注册
    retval = synaptics_rmi4_set_input_dev(rmi4_data);

    ......

    rmi4_data->irq = gpio_to_irq(bdata->irq_gpio);
    //请求中断，并设置中断处理函数synaptics_rmi4_irq
    retval = synaptics_rmi4_irq_enable(rmi4_data, true);

    if (!exp_data.initialized) {
        mutex_init(&exp_data.mutex);
        INIT_LIST_HEAD(&exp_data.list);
        exp_data.initialized = true;
    }

    exp_data.workqueue = create_singlethread_workqueue("dsx_exp_workqueue");
    INIT_DELAYED_WORK(&exp_data.work, synaptics_rmi4_exp_fn_work);
    exp_data.rmi4_data = rmi4_data;
    exp_data.queue_work = true;
    queue_delayed_work(exp_data.workqueue,
            &exp_data.work,
            msecs_to_jiffies(EXP_FN_WORK_DELAY_MS));

    rmi4_data->dir = debugfs_create_dir(DEBUGFS_DIR_NAME, NULL);
    ......

    for (attr_count = 0; attr_count < ARRAY_SIZE(attrs); attr_count++) {
        retval = sysfs_create_file(&rmi4_data->input_dev->dev.kobj,
                &attrs[attr_count].attr);
        ......
    }

    synaptics_secure_touch_init(rmi4_data);
    synaptics_secure_touch_stop(rmi4_data, 1);

    return retval;
    .......
}
```

## 7.1.3、分配Input_dev结构体

```c
[->synaptics_dsx_core.c]
static int synaptics_rmi4_set_input_dev(struct synaptics_rmi4_data *rmi4_data)
{
    int retval;
    const struct synaptics_dsx_board_data *bdata =
                rmi4_data->hw_if->board_data;

    rmi4_data->input_dev = input_allocate_device();
    ......
}
```

## 7.2、设置支持事件类型 set_bit(EV_SYN, evbit)、set_bit(EV_KEY, evbit)、set_bit(EV_ABS,evbit) ......

```c
[->synaptics_dsx_core.c]
static int synaptics_rmi4_set_input_dev(struct synaptics_rmi4_data *rmi4_data)
{
    int retval;
    const struct synaptics_dsx_board_data *bdata =
                rmi4_data->hw_if->board_data;

    rmi4_data->input_dev = input_allocate_device();
    ......
    retval = synaptics_rmi4_query_device(rmi4_data);
    ....
    //#define PLATFORM_DRIVER_NAME "synaptics_dsxv26"(synaptics_dsx_v2_6.h)

    rmi4_data->input_dev->name = PLATFORM_DRIVER_NAME;
    rmi4_data->input_dev->phys = INPUT_PHYS_NAME;
    rmi4_data->input_dev->id.product = SYNAPTICS_DSX_DRIVER_PRODUCT;
    rmi4_data->input_dev->id.version = SYNAPTICS_DSX_DRIVER_VERSION;
    rmi4_data->input_dev->dev.parent = rmi4_data->pdev->dev.parent;
    input_set_drvdata(rmi4_data->input_dev, rmi4_data);

    set_bit(EV_SYN, rmi4_data->input_dev->evbit);
    set_bit(EV_KEY, rmi4_data->input_dev->evbit);
    set_bit(EV_ABS, rmi4_data->input_dev->evbit);
    set_bit(BTN_TOUCH, rmi4_data->input_dev->keybit);
    set_bit(BTN_TOOL_FINGER, rmi4_data->input_dev->keybit);

    synaptics_rmi4_set_params(rmi4_data);
    ......
}
```

## 7.3、注册设备input_register_device()

此处即与前面kernel log呼应：注册名为 synaptics_dsxv26 的输入设备

```c
[->synaptics_dsx_core.c]
static int synaptics_rmi4_set_input_dev(struct synaptics_rmi4_data *rmi4_data)
{
    int retval;
    const struct synaptics_dsx_board_data *bdata =
                rmi4_data->hw_if->board_data;

    rmi4_data->input_dev = input_allocate_device();
    ......
    retval = synaptics_rmi4_query_device(rmi4_data);
    ....
    //#define PLATFORM_DRIVER_NAME "synaptics_dsxv26"(synaptics_dsx_v2_6.h)

    rmi4_data->input_dev->name = PLATFORM_DRIVER_NAME;
    rmi4_data->input_dev->phys = INPUT_PHYS_NAME;
    rmi4_data->input_dev->id.product = SYNAPTICS_DSX_DRIVER_PRODUCT;
    rmi4_data->input_dev->id.version = SYNAPTICS_DSX_DRIVER_VERSION;
    rmi4_data->input_dev->dev.parent = rmi4_data->pdev->dev.parent;
    input_set_drvdata(rmi4_data->input_dev, rmi4_data);

    set_bit(EV_SYN, rmi4_data->input_dev->evbit);
    set_bit(EV_KEY, rmi4_data->input_dev->evbit);
    set_bit(EV_ABS, rmi4_data->input_dev->evbit);
    set_bit(BTN_TOUCH, rmi4_data->input_dev->keybit);
    set_bit(BTN_TOOL_FINGER, rmi4_data->input_dev->keybit);

    synaptics_rmi4_set_params(rmi4_data);
    ......
    retval = input_register_device(rmi4_data->input_dev);
}
```

## 7.4、硬件相关操作

当触摸屏按下，会产生中断，进而调用中断处理函数synaptics_rmi4_irq():

```c
[->synaptics_dsx_core.c]
static irqreturn_t synaptics_rmi4_irq(int irq, void *data)
{
    struct synaptics_rmi4_data *rmi4_data = data;
    const struct synaptics_dsx_board_data *bdata =
            rmi4_data->hw_if->board_data;

    if (IRQ_HANDLED == synaptics_filter_interrupt(data))
        return IRQ_HANDLED;

    if (gpio_get_value(bdata->irq_gpio) != bdata->irq_on_state)
        goto exit;

    synaptics_rmi4_sensor_report(rmi4_data, true);

exit:
    return IRQ_HANDLED;
}
```

进一步调用synaptics_rmi4_sensor_report(rmi4_data, true)处理数据：

```c
[->synaptics_dsx_core.c]
static void synaptics_rmi4_sensor_report(struct synaptics_rmi4_data *rmi4_data,
        bool report)
{
    int retval;
    unsigned char data[MAX_INTR_REGISTERS + 1];
    unsigned char *intr = &data[1];
    bool was_in_bl_mode;
    struct synaptics_rmi4_f01_device_status status;
    struct synaptics_rmi4_fn *fhandler;
    struct synaptics_rmi4_exp_fhandler *exp_fhandler;
    struct synaptics_rmi4_device_info *rmi;

    rmi = &(rmi4_data->rmi4_mod_info);

    ....

    retval = synaptics_rmi4_reg_read(rmi4_data,
            rmi4_data->f01_data_base_addr,
            data,
            rmi4_data->num_of_intr_regs + 1);
    ......
    //读取寄存器数据
    status.data[0] = data[0];
    if (status.status_code == STATUS_CRC_IN_PROGRESS) {
        retval = synaptics_rmi4_check_status(rmi4_data,
                &was_in_bl_mode);
        ....
        retval = synaptics_rmi4_reg_read(rmi4_data,
                rmi4_data->f01_data_base_addr,
                status.data,
                sizeof(status.data));
        ......
    }
    if (status.unconfigured && !status.flash_prog) {
        pr_notice("%s: spontaneous reset detected\n", __func__);
    }

    //synaptics_rmi4_report_touch()上报数据
    if (!list_empty(&rmi->support_fn_list)) {
        list_for_each_entry(fhandler, &rmi->support_fn_list, link) {
            if (fhandler->num_of_data_sources) {
                if (fhandler->intr_mask &
                        intr[fhandler->intr_reg_num]) {
                    synaptics_rmi4_report_touch(rmi4_data,
                            fhandler);
                }
            }
        }
    }

    mutex_lock(&exp_data.mutex);
    if (!list_empty(&exp_data.list)) {
        list_for_each_entry(exp_fhandler, &exp_data.list, link) {
            if (!exp_fhandler->insert &&
                    !exp_fhandler->remove &&
                    (exp_fhandler->exp_fn->attn != NULL))
                exp_fhandler->exp_fn->attn(rmi4_data, intr[0]);
        }
    }
    mutex_unlock(&exp_data.mutex);

    return;
}
```

## 7.4.1、Input数据上报：

```c
[->synaptics_dsx_core.c]
static void synaptics_rmi4_report_touch(struct synaptics_rmi4_data *rmi4_data,
        struct synaptics_rmi4_fn *fhandler)
{
    ......
    switch (fhandler->fn_number) {
    ......
    case SYNAPTICS_RMI4_F12:
        touch_count_2d = synaptics_rmi4_f12_abs_report(rmi4_data,
                fhandler);

        if (touch_count_2d)
            rmi4_data->fingers_on_2d = true;
        else
            rmi4_data->fingers_on_2d = false;
        break;
    ......
    default:
        break;
    }

    return;
}

static int synaptics_rmi4_f12_abs_report(struct synaptics_rmi4_data *rmi4_data,
        struct synaptics_rmi4_fn *fhandler)
{
    int retval;
    unsigned char touch_count = 0; /* number of touch points */
    unsigned char index;
    unsigned char finger;
    unsigned char fingers_to_process;
    unsigned char finger_status;
    unsigned char size_of_2d_data;
    unsigned char gesture_type;
    unsigned short data_addr;
    int x;
    int y;
    int wx;
    int wy;
    int temp;

    struct synaptics_rmi4_f12_extra_data *extra_data;
    struct synaptics_rmi4_f12_finger_data *data;
    struct synaptics_rmi4_f12_finger_data *finger_data;
    static unsigned char finger_presence;
    static unsigned char stylus_presence;

    fingers_to_process = fhandler->num_of_data_points;
    data_addr = fhandler->full_addr.data_base;
    extra_data = (struct synaptics_rmi4_f12_extra_data *)fhandler->extra;
    size_of_2d_data = sizeof(struct synaptics_rmi4_f12_finger_data);

    ......

    retval = synaptics_rmi4_reg_read(rmi4_data,
            data_addr + extra_data->data1_offset,
            (unsigned char *)fhandler->data,
            fingers_to_process * size_of_2d_data);
    if (retval < 0)
        return 0;

    data = (struct synaptics_rmi4_f12_finger_data *)fhandler->data;


    mutex_lock(&(rmi4_data->rmi4_report_mutex));
    //根据触摸点数量循环上报input数据
    for (finger = 0; finger < fingers_to_process; finger++) {
        finger_data = data + finger;
        finger_status = finger_data->object_type_and_status;


        x = (finger_data->x_msb << 8) | (finger_data->x_lsb);
        y = (finger_data->y_msb << 8) | (finger_data->y_lsb);

        if (rmi4_data->hw_if->board_data->swap_axes) {
            temp = x;
            x = y;
            y = temp;
            temp = wx;
            wx = wy;
            wy = temp;
        }

        if (rmi4_data->hw_if->board_data->x_flip)
            x = rmi4_data->sensor_max_x - x;
        if (rmi4_data->hw_if->board_data->y_flip)
            y = rmi4_data->sensor_max_y - y;

        switch (finger_status) {
        case F12_FINGER_STATUS:
        case F12_GLOVED_FINGER_STATUS:

            input_report_key(rmi4_data->input_dev,
                    BTN_TOUCH, 1);
            input_report_key(rmi4_data->input_dev,
                    BTN_TOOL_FINGER, 1);
            input_report_abs(rmi4_data->input_dev,
                    ABS_MT_POSITION_X, x);
            input_report_abs(rmi4_data->input_dev,
                    ABS_MT_POSITION_Y, y);
        ......
    }

    ......

    input_sync(rmi4_data->input_dev);

    mutex_unlock(&(rmi4_data->rmi4_report_mutex));

    return touch_count;
}
```

调用input_report_key()、input_report_abs()、input_sync() 上报、同步数据。

## （八）、参考文档(特别感谢各位前辈的分析和图示)：

[Linux/Android----Input系统](https://blog.csdn.net/column/details/input.html)
[Android Input子系统浅谈](https://blog.csdn.net/tiantangniaochao/article/details/50497353)
[Android(Linux) 输入子系统解析](http://huaqianlee.github.io/2017/11/23/Android/Android-Linux-input-system-analysis/)
[input子系统分析之三:驱动模块](http://www.cnblogs.com/jason-lu/p/3156411.html)
[Linux驱动框架之----Input子系统](https://blog.csdn.net/fanwenjieok/article/details/38503027)
[input子系统事件处理层(evdev)的环形缓冲区](https://www.zybuluo.com/zifehng/note/718523)
[linux input输入子系统分析《四》：input子系统整体流程全面分析](https://blog.csdn.net/ielife/article/details/7814108)
[Linux input子系统分析之二：深入剖析input_handler、input_core、input_device](https://blog.csdn.net/yueqian_scut/article/details/48792939)
