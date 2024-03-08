---
title: Linux学习系列1：clk子系统
cover: https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.41.jpg
categories:
 - Linux
tags:
 - Linux
toc: true
abbrlink: 20220615
date: 2022-06-15 00:00:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Kernel-4.15**==）&&（==**文章基于 Android 10.0**==）

[【me.zhoujinjian.com博客原图链接】](https://github.com/zhoujinjianzjj) 

[【开发板 RockPi4bPlusV1.6】](https://shop.allnetchina.cn/collections/frontpage/products/rock-pi-4-model-b-board-only-2-4-5ghz-wlan-bluetooth-5-0)

[【开发板 RockPi4bPlusV1.6 Android 10.0 && Linux（Kernel 4.15）源码链接】](https://github.com/radxa/manifests/tree/rockchip-android-10)



正是由于前人（各位大神）的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

 [【linux中common clock framework】](https://blog.csdn.net/qq_16777851/category_7923311.html)    

 [【CLOCK子系统】](http://www.mysixue.com/?p=129)    



1.系统概述
===================================

Clock子系统是Linux内核中专门管理时钟的子系统。时钟在嵌入式系统中很重要，它就像人的脉搏一样，驱动器件工作。

任何一个CPU， 都需要给它提供一个外部晶振， 这个晶振就是用来提供时钟的；任何一个CPU内部的片上外设， 也需要工作时钟： 例如GPIO控制器， 首先得给它提供工作时钟， 然后才能访问它的寄存器。

1.1 clk的种类说明
--------------------------------

![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/clk_source.png)

如上图所示，时钟源大概可分为如下几种：

- 提供基础时钟源的晶振（可分为有源晶振、无源晶振两种）；

- 用于倍频的锁相环；

- 用于分频的divider；

- 用于多路时钟源选择的mux；

- 用于时钟使能的与门电路等。

而在CCF(Common Clock Framework，以下简称CCF)子系统的抽象中，这五种均抽象为clk。

但是针对这5种类型的时钟也提供了单独的时钟注册函数（也就是对clk_register函数的封装，并针对不同的时钟类型定义了不同的结构体）。

1. fixed-clock：clk_register_fixed_rate（\kernel\drivers\clk\clk-fixed-clock.c）
2. pll：clk_register(\kernel\drivers\clk\clk.c)
3. divider：clk_register_divider（\kernel\drivers\clk\clk-divider.c）
4. mux：clk_register_mux（\kernel\drivers\clk\clk-mux.c）
5. gate：clk_register_gate（\kernel\drivers\clk\clk-gate.c）

在CCF子系统中，针对硬件时钟的操作接口，也抽象了对应的结构体struct clk_ops。

包含时钟的使能接口、时钟频率的修改接口等等。而针对上述所说的不同种类的时钟源，其并不需要实现所有struct clk_ops中定义的接口。如下图所示，针对“时钟使能的与门电路”而言，仅需要实现enabel、disable、is_enable接口即可；针对多路时钟源选择的mux而言，则需要实现父时钟源的设置及获取的接口set_parent、get_parent等；对于倍频、分频而言，则需要实现时钟频率相关的接口set_rate、recalc_rate等。

> 图来源：https://www.kernel.org/doc/html/latest/driver-api/clk.html

![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/clk_char.png)

1.2 CCF子系统的框架说明
--------------------------------

 Linux内核的clock子系统, 按照其职能, 可以大致分为3部分:

1. 向下提供注册接口, 以便各个clocks能注册进clock子系统
2. 在核心层维护池子, 管理所有注册进来的clocks. 这一部分实现的是通用逻辑, 与具体硬件无关
3. 向上, 也就是像各个消费clocks的模块的device driver, 提供获取/使能/配置/关闭clock的通用API



如图所示，是CCF子系统的软件框架，其对各设备驱动子系统提供统一时钟源操作的接口，实现为具体的硬件设备获取其对应的输入时钟源，并可通过统一的接口实现对时钟源的配置（如时钟的使能、时钟频率的配置等等）；在CCF内部，针对每一个时钟源，抽象结构体struct clk_hw，该结构体中包含每一个硬件时钟源的操作接口（struct clk_ops），当时钟源驱动程序完成时钟操作接口的定义，并调用clk_register完成注册后，则CCF即可借助clk_hw的clk_ops完成对时钟源的参数配置等操作。


--------------------------------

2.Clock Provider — 如何注册Clocks
===================================

2.1 编写clock driver的大致步骤
--------------------------------

首先你得准备一个platform_device, 引入device tree的机制后, platform_device被dts替代了, 因此我们就需要在dts里面描述时钟树。

**编写platform_device(DTS node)**

将时钟树中所有的clock，抽象为一个虚拟的设备，用一个DTS node表示.

```dtd
F:\Rock4Plus_Android10\u-boot\arch\arm\dts\rk3399.dtsi
pmucru: pmu-clock-controller@ff750000 {
		compatible = "rockchip,rk3399-pmucru";
		reg = <0x0 0xff750000 0x0 0x1000>;
		rockchip,grf = <&pmugrf>;
		#clock-cells = <1>;
		#reset-cells = <1>;
		assigned-clocks = <&pmucru PLL_PPLL>;
		assigned-clock-rates = <676000000>;
	};

	cru: clock-controller@ff760000 {
		compatible = "rockchip,rk3399-cru";
		reg = <0x0 0xff760000 0x0 0x1000>;
		rockchip,grf = <&grf>;
		#clock-cells = <1>;
		#reset-cells = <1>;
		assigned-clocks =
			<&cru PLL_GPLL>, <&cru PLL_CPLL>,
			<&cru PLL_NPLL>,
			<&cru ACLK_PERIHP>, <&cru HCLK_PERIHP>,
			<&cru PCLK_PERIHP>,
			<&cru ACLK_PERILP0>, <&cru HCLK_PERILP0>,
			<&cru PCLK_PERILP0>, <&cru ACLK_CCI>,
			<&cru HCLK_PERILP1>, <&cru PCLK_PERILP1>;
		assigned-clock-rates =
			 <594000000>,  <800000000>,
			<1000000000>,
			 <150000000>,   <75000000>,
			  <37500000>,
			 <100000000>,  <100000000>,
			  <50000000>, <600000000>,
			 <100000000>,   <50000000>;
	};
```

这种方式跟编写一个普通的片上外设的DTS很类似, 比如GPIO控制器的DTS, 对比一下, 是不是很类似.

从这个例子里面, 我们可以看出一个clock的DTS node的基本语法:

- compatible , 决定了与这个node匹配的driver

- reg就是用来描述PCM的寄存器

- #clock-cells, 这个是clock provider node独有的, 表明在引用此clock时, 需要用几个32位来描述. 什么意思呢?

假设这个clock有多个输出时钟, 那么这个时候#clock–cells 应该为 <1>, 因为我们在引用此clock的时候, 需要指明到底用哪一个输出时钟.在引用此clock, 就得这样写:

```dtd
F:\Rock4Plus_Android10\u-boot\arch\arm\dts\rk3399.dtsi
i2c0: i2c@ff3c0000 {
		compatible = "rockchip,rk3399-i2c";
		reg = <0x0 0xff3c0000 0x0 0x1000>;
		assigned-clocks = <&pmucru SCLK_I2C0_PMU>;
		assigned-clock-rates = <200000000>;
		clocks = <&pmucru SCLK_I2C0_PMU>, <&pmucru PCLK_I2C0_PMU>;
        /* 指明引用pmucru clock,SCLK_I2C0_PMU/PCLK_I2C0_PMU是一个32位的整数, 表明到底用哪一个输出clock */
		clock-names = "i2c", "pclk";
		interrupts = <GIC_SPI 57 IRQ_TYPE_LEVEL_HIGH 0>;
		pinctrl-names = "default";
		pinctrl-0 = <&i2c0_xfer>;
		#address-cells = <1>;
		#size-cells = <0>;
		status = "disabled";
	};
```

## 2.2 编写platform_driver

有了platform_device之后, 接下来就得编写platform_driver, 在driver里面最重要的事情就是向clock子系统注册.

如何注册呢?

clock子系统定义了clock driver需要实现的数据结构, 同时提供了注册的函数. 我们只需要准备好相关的数据结构, 然后调用注册函数进行注册即可.

这些数据结构和接口函数的定义是在: include/linux/clk-provider.h

需要实现的数据结构是 struct clk_hw, 需要调用的注册函数是struct clk *clk_register(struct device *dev, struct clk_hw *hw). 数据结构和注册函数的细节我们在后文说明.

注册函数会返回给你一个struct clk类型的指针, 在Linux的clock子系统中, 用一个struct clk代表一个clock. 你的CPU的时钟树里有多少个clock, 就会有多少个对应的struct clk.

当你拿到返回结果之后, 你需要调用另外一个API : of_clk_add_provider, 把刚刚拿到的返回结果通过此API丢到池子里面, 好让consumer从这个池子里获取某一个clock.

```C
F:\Rock4Plus_Android10\kernel\drivers\clk\rockchip\clk.c
void __init rockchip_clk_of_add_provider(struct device_node *np,
				struct rockchip_clk_provider *ctx)
{
	if (of_clk_add_provider(np, of_clk_src_onecell_get,
				&ctx->clk_data))
		pr_err("%s: could not register clk provider\n", __func__);
}
```

池子的概念我们在前文讲述过, 除了上述方法, 你还可以用另外一种方法把struct clk添加到池子里面.

如果你要用这种方法, 你会用到clock子系统提供给你的另外几个API, 这些API的定义在: include/linux/clkdev.h

其中最主要的一个API是int clk_register_clkdev(struct clk *, const char *, const char *, …), 你可以在你的clock driver里面调用这个API, 把刚刚拿到的返回结果通过此API丢到池子里面.

为什么会存在这两种方式呢? 得从consumer的角度来解答这个问题.

我们用GPIO来举个例子, GPIO控制器需要工作时钟, 这个时钟假设叫gpio_clk, 它是一个provider. 你需要把这个provider注册进clock子系统, 并把用于描述这个gpio_clk的struct clk添加到池子里面.

在GPIO控制器的driver代码里, 我们需要获取到gpio_clk这个时钟并使能它, 获取的过程就是向池子查询.

怎么查询? 你可以直接给定一个name, 然后通过这个name向池子查询; 你也可以在GPIO的DTS node里面用clocks = <&theclock>;方式指明使用哪一个clock, 然后通过这种方式向池子查询.

如果consumer是通过name查询, 则对应的添加到池子的API就是clk_register_clkdev

如果consumer是通过DTS查询, 则对应的添加到池子的API就是of_clk_add_provider

那么我在我的clock driver里面到底应该用哪个API向池子添加clk呢?

两者你都应该同时使用, 这样consumer端不管用哪种查询方式都能工作.

读到这里, 建议你回头看看clock子系统的系统框图, 结合框图在琢磨琢磨.

接下来, 我们就会详细介绍这些数据结构和相关的API了.

## **2.3**  **主要数据结构**

通过前文, 我们知道了clock driver需要实现的一个主要的数据结构struct clk_hw.

与之相关的还有另外几个重要数据结构: struct clk_init_data和struct clk_ops.

下面我们看看这几个数据结构.

### **struct** clk_hw

>include/linux/clk-provider.h

![image-20210824162026412](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/image-20210824162026412.png)

### **struct** clk_init_data 

>include/linux/clk-provider.h

![image-20210824162137515](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/image-20210824162137515.png)

### **struct**  clk_ops

>include/linux/clk-provider.h

![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/struct_clk_ops-16297933295211.png)

## **2.4** 主要 API 说明

Linux clock子系统向下提供几个重要的API, 下面我挨个看下这些API的细节.

### clk_register /devm_clk_register

```C
头文件: include/linux/clk-provider.h
实现文件: drivers/clk/clk.c
/**
 * clk_register - allocate a new clock, register it and return an opaque cookie
 * @dev: device that is registering this clock
 * @hw: link to hardware-specific clock data
 *
 * clk_register is the primary interface for populating the clock tree with new
 * clock nodes.  It returns a pointer to the newly allocated struct clk which
 * cannot be dereferenced by driver code but may be used in conjuction with the
 * rest of the clock API.  In the event of an error clk_register will return an
 * error code; drivers must test for an error code after calling clk_register.
 */
struct clk *clk_register(struct device *dev, struct clk_hw *hw);
struct clk *devm_clk_register(struct device *dev, struct clk_hw *hw);

void clk_unregister(struct clk *clk);
void devm_clk_unregister(struct device *dev, struct clk *clk);

```

clk_register是clock子系统提供的注册clock的最基础的API函数, 后文描述的其它APIs都是对它的封装.

devm_clk_register是clk_register的devm版本, devm机制在《设备模型》一文中有详述.

要向系统注册一个clock也很简单, 准备好clk_hw结构体, 然后调用clk_register接口即可.

不过, clock framework所做的远比这周到, 它基于clk_register, 又封装了其它接口, 在向clock子系统注册时, 连struct clk_hw都不需要关心, 而是直接使用类似人类语言的方式.

也就是说, 实际在编写clock driver的时候, 我们不会直接使用clk_register接口, 只需要调用下面的某个API即可.

下文我们一一介绍这些API.

### **clk_register_fixed_rate**

clock有不同的类型, 有的clock频率是固定的, 不可调整的, 例如外部晶振, 频率就是固定的(24M / 25M). 这种类型的clock就是fixed_rate clock.

fixed_rate clock具有固定的频率, 不能开关、不能调整频率、不能选择parent、不需要提供任何的clk_ops回调函数, 是最简单的一类clock.

如果要注册这种类型的clock, 直接调用本API, 传递相应的参数给本API即可. API的细节如下:

```c
头文件: include/linux/clk-provider.h

实现文件: drivers/clk/clk-fixed-rate.c

/*
 * DOC: Basic clock implementations common to many platforms
 *
 * Each basic clock hardware type is comprised of a structure describing the
 * clock hardware, implementations of the relevant callbacks in struct clk_ops,
 * unique flags for that hardware type, a registration function and an
 * alternative macro for static initialization
 */

/**
 * struct clk_fixed_rate - fixed-rate clock
 * @hw:		handle between common and hardware-specific interfaces
 * @fixed_rate:	constant frequency of clock
 */
struct clk_fixed_rate {
	struct		clk_hw hw;
	unsigned long	fixed_rate;
	unsigned long	fixed_accuracy;
	u8		flags;
};

#define to_clk_fixed_rate(_hw) container_of(_hw, struct clk_fixed_rate, hw)

extern const struct clk_ops clk_fixed_rate_ops;
struct clk *clk_register_fixed_rate(struct device *dev, const char *name,
		const char *parent_name, unsigned long flags,
		unsigned long fixed_rate);
struct clk_hw *clk_hw_register_fixed_rate(struct device *dev, const char *name,
		const char *parent_name, unsigned long flags,
		unsigned long fixed_rate);
struct clk *clk_register_fixed_rate_with_accuracy(struct device *dev,
		const char *name, const char *parent_name, unsigned long flags,
		unsigned long fixed_rate, unsigned long fixed_accuracy);
void clk_unregister_fixed_rate(struct clk *clk);
struct clk_hw *clk_hw_register_fixed_rate_with_accuracy(struct device *dev,
		const char *name, const char *parent_name, unsigned long flags,
		unsigned long fixed_rate, unsigned long fixed_accuracy);
void clk_hw_unregister_fixed_rate(struct clk_hw *hw);

void of_fixed_clk_setup(struct device_node *np);
```

若想注册一个fixed rate clock, 除了你可以在自己的clock driver里面手动调用clk_register_fixed_rate之外, 还有一种更简单的方式, 那就是用DTS.

你可以在DTS里面用如下方式描述一个fixed rate clock:

```dtd
F:\Rock4Plus_Android10\u-boot\arch\arm\dts\rk3399.dtsi
xin24m: xin24m {
		compatible = "fixed-clock";
		clock-frequency = <24000000>;
		clock-output-names = "xin24m";
		#clock-cells = <0>;
	};
```

- 关键地方在于compatible一定要是”fixed-clock”

- “drivers/clk/clk-fixed-rate.c”中的of_fixed_clk_setup会负责匹配这个compatible, 这个C文件是clock子系统实现的.

- of_fixed_clk_setup会解析3个参数: clock-frequency, clock-accuracy, clock-output-names

clock-frequency是必须的, 另外2个参数可选.

参数解析完毕之后, 会调用clk_register_fixed_rate_with_accuracy向系统注册一个clock.

### **clk_register_gate**

这一类clock只可开关（会提供.enable/.disable回调）, 可使用下面接口注册:

 

```c
头文件: include/linux/clk-provider.h

实现文件: drivers/clk/clk-gate.c
/**
 * struct clk_gate - gating clock
 *
 * @hw:		handle between common and hardware-specific interfaces
 * @reg:	register controlling gate
 * @bit_idx:	single bit controlling gate
 * @flags:	hardware-specific flags
 * @lock:	register lock
 *
 * Clock which can gate its output.  Implements .enable & .disable
 *
 * Flags:
 * CLK_GATE_SET_TO_DISABLE - by default this clock sets the bit at bit_idx to
 *	enable the clock.  Setting this flag does the opposite: setting the bit
 *	disable the clock and clearing it enables the clock
 * CLK_GATE_HIWORD_MASK - The gate settings are only in lower 16-bit
 *	of this register, and mask of gate bits are in higher 16-bit of this
 *	register.  While setting the gate bits, higher 16-bit should also be
 *	updated to indicate changing gate bits.
 */
struct clk_gate {
	struct clk_hw hw;
	void __iomem	*reg;
	u8		bit_idx;
	u8		flags;
	spinlock_t	*lock;
};

#define to_clk_gate(_hw) container_of(_hw, struct clk_gate, hw)

#define CLK_GATE_SET_TO_DISABLE		BIT(0)
#define CLK_GATE_HIWORD_MASK		BIT(1)

extern const struct clk_ops clk_gate_ops;
struct clk *clk_register_gate(struct device *dev, const char *name,
		const char *parent_name, unsigned long flags,
		void __iomem *reg, u8 bit_idx,
		u8 clk_gate_flags, spinlock_t *lock);
struct clk_hw *clk_hw_register_gate(struct device *dev, const char *name,
		const char *parent_name, unsigned long flags,
		void __iomem *reg, u8 bit_idx,
		u8 clk_gate_flags, spinlock_t *lock);
void clk_unregister_gate(struct clk *clk);
void clk_hw_unregister_gate(struct clk_hw *hw);
int clk_gate_is_enabled(struct clk_hw *hw);
```

**clk_register_gate**, 它的参数列表如下:

- name: clock的名称

- parent_name: parent clock的名称, 没有的话可留空

- flags : flags描述

- reg: 控制该clock开关的寄存器地址（虚拟地址）

- bit_idx: reg中, 第几个bit是控制clock开/关的

- clk_gate_flags: gate clock特有的参数. 其中一个可选值是CLK_GATE_SET_TO_DISABLE, 它的意思是1表示开还是0表示开, 类似于翻转位.

- lock: 如果clock开关时需要互斥, 可提供一个spinlock.



### clk_register_divider/clk_register_divider_table

这一类clock可以设置分频值（因而会提供.recalc_rate/.set_rate/.round_rate回调）, 可通过下面两个接口注册：

 

```c
头文件: include/linux/clk-provider.h
实现文件: drivers/clk/clk-divider.c
struct clk_divider {
	struct clk_hw	hw;
	void __iomem	*reg;
	u8		shift;
	u8		width;
	u8		flags;
	unsigned long	max_prate;
	const struct clk_div_table	*table;
	spinlock_t	*lock;
};

#define clk_div_mask(width)	((1 << (width)) - 1)
#define to_clk_divider(_hw) container_of(_hw, struct clk_divider, hw)

#define CLK_DIVIDER_ONE_BASED		BIT(0)
#define CLK_DIVIDER_POWER_OF_TWO	BIT(1)
#define CLK_DIVIDER_ALLOW_ZERO		BIT(2)
#define CLK_DIVIDER_HIWORD_MASK		BIT(3)
#define CLK_DIVIDER_ROUND_CLOSEST	BIT(4)
#define CLK_DIVIDER_READ_ONLY		BIT(5)
#define CLK_DIVIDER_MAX_AT_ZERO		BIT(6)

extern const struct clk_ops clk_divider_ops;
extern const struct clk_ops clk_divider_ro_ops;

unsigned long divider_recalc_rate(struct clk_hw *hw, unsigned long parent_rate,
		unsigned int val, const struct clk_div_table *table,
		unsigned long flags, unsigned long width);
long divider_round_rate_parent(struct clk_hw *hw, struct clk_hw *parent,
			       unsigned long rate, unsigned long *prate,
			       const struct clk_div_table *table,
			       u8 width, unsigned long flags);
long divider_ro_round_rate_parent(struct clk_hw *hw, struct clk_hw *parent,
				  unsigned long rate, unsigned long *prate,
				  const struct clk_div_table *table, u8 width,
				  unsigned long flags, unsigned int val);
int divider_get_val(unsigned long rate, unsigned long parent_rate,
		const struct clk_div_table *table, u8 width,
		unsigned long flags);

struct clk *clk_register_divider(struct device *dev, const char *name,
		const char *parent_name, unsigned long flags,
		void __iomem *reg, u8 shift, u8 width,
		u8 clk_divider_flags, spinlock_t *lock);
struct clk_hw *clk_hw_register_divider(struct device *dev, const char *name,
		const char *parent_name, unsigned long flags,
		void __iomem *reg, u8 shift, u8 width,
		u8 clk_divider_flags, spinlock_t *lock);
struct clk *clk_register_divider_table(struct device *dev, const char *name,
		const char *parent_name, unsigned long flags,
		void __iomem *reg, u8 shift, u8 width,
		u8 clk_divider_flags, const struct clk_div_table *table,
		spinlock_t *lock);
struct clk_hw *clk_hw_register_divider_table(struct device *dev,
		const char *name, const char *parent_name, unsigned long flags,
		void __iomem *reg, u8 shift, u8 width,
		u8 clk_divider_flags, const struct clk_div_table *table,
		spinlock_t *lock);
void clk_unregister_divider(struct clk *clk);
void clk_hw_unregister_divider(struct clk_hw *hw);

```

**lk_register_divider** : 该接口用于注册分频比规则的clock

- reg: 控制clock分频比的寄存器

- shift: 控制分频比的bit在寄存器中的偏移

- width: 控制分频比的bit位数, 默认情况下, 实际的divider值是寄存器值加1.

如果有其它例外, 可使用下面的的flag指示。

- clk_divider_flags: divider clock特有的flag, 包括:
  CLK_DIVIDER_ONE_BASED:

实际的divider值就是寄存器值（0是无效的，除非设置CLK_DIVIDER_ALLOW_ZERO flag）

CLK_DIVIDER_POWER_OF_TWO:

实际的divider值是寄存器值得2次方

CLK_DIVIDER_ALLOW_ZERO:

divider值可以为0（不改变，视硬件支持而定）



**clk_register_divider_table** : 该接口用于注册分频比不规则的clock，和上面接口比较，差别在于divider值和寄存器值得对应关系由一个table决定. table的原型如下:

```c
struct clk_div_table {
  unsigned int  val;
  unsigned int  div;
};
```

val代表寄存器值, div代表对应的分频值.

同样, clock子系统并没有实现DTS相关接口, 不过你可以自己编写driver去解析DTS并调用API注册.

### **clk_register_mux**

这一类clock可以选择多个parent, 因为会实现.get_parent/.set_parent/.recalc_rate回调, 可通过下面两个接口注册:

 

```c
头文件: include/linux/clk-provider.h

实现文件: drivers/clk/clk-mux.c
    
struct clk_mux {
	struct clk_hw	hw;
	void __iomem	*reg;
	u32		*table;
	u32		mask;
	u8		shift;
	u8		flags;
	spinlock_t	*lock;
};

#define to_clk_mux(_hw) container_of(_hw, struct clk_mux, hw)

#define CLK_MUX_INDEX_ONE		BIT(0)
#define CLK_MUX_INDEX_BIT		BIT(1)
#define CLK_MUX_HIWORD_MASK		BIT(2)
#define CLK_MUX_READ_ONLY		BIT(3) /* mux can't be changed */
#define CLK_MUX_ROUND_CLOSEST		BIT(4)

extern const struct clk_ops clk_mux_ops;
extern const struct clk_ops clk_mux_ro_ops;

struct clk *clk_register_mux(struct device *dev, const char *name,
		const char * const *parent_names, u8 num_parents,
		unsigned long flags,
		void __iomem *reg, u8 shift, u8 width,
		u8 clk_mux_flags, spinlock_t *lock);
struct clk_hw *clk_hw_register_mux(struct device *dev, const char *name,
		const char * const *parent_names, u8 num_parents,
		unsigned long flags,
		void __iomem *reg, u8 shift, u8 width,
		u8 clk_mux_flags, spinlock_t *lock);

struct clk *clk_register_mux_table(struct device *dev, const char *name,
		const char * const *parent_names, u8 num_parents,
		unsigned long flags,
		void __iomem *reg, u8 shift, u32 mask,
		u8 clk_mux_flags, u32 *table, spinlock_t *lock);
struct clk_hw *clk_hw_register_mux_table(struct device *dev, const char *name,
		const char * const *parent_names, u8 num_parents,
		unsigned long flags,
		void __iomem *reg, u8 shift, u32 mask,
		u8 clk_mux_flags, u32 *table, spinlock_t *lock);

int clk_mux_val_to_index(struct clk_hw *hw, u32 *table, unsigned int flags,
			 unsigned int val);
unsigned int clk_mux_index_to_val(u32 *table, unsigned int flags, u8 index);

void clk_unregister_mux(struct clk *clk);
void clk_hw_unregister_mux(struct clk_hw *hw);

```

**clk_register_mux** : 该接口可注册mux控制比较规则的clock（类似divider clock）

- parent_names: 一个字符串数组，用于描述所有可能的parent clock

- num_parents: parent clock的个数

- reg、shift、width: 选择parent的寄存器、偏移、宽度

- clk_mux_flags: mux clock特有的flag

CLK_MUX_INDEX_ONE : 寄存器值不是从0开始，而是从1开始

CLK_MUX_INDEX_BIT : 寄存器值为2的幂

 

**clk_register_mux_table** : 该接口通过一个table, 注册mux控制不规则的clock, 原理和divider clock类似, 不再详细介绍

 

同样, clock子系统并没有实现DTS相关接口, 不过你可以自己编写driver去解析DTS并调用API注册.

### **clk_register_fixed_factor**

这一类clock具有固定的factor（即multiplier和divider）, clock的频率是由parent clock的频率, 乘以mul, 除以div, 多用于一些具有固定分频系数的clock.

由于parent clock的频率可以改变, 因而fix factor clock也可以改变频率, 因此也会提供.recalc_rate/.set_rate/.round_rate等回调.

 

可通过下面接口注册:

 

```c
头文件: include/linux/clk-provider.h

实现文件: drivers/clk/clk-fixed-factor.c

void of_fixed_factor_clk_setup(struct device_node *node);

/**
 * struct clk_fixed_factor - fixed multiplier and divider clock
 *
 * @hw:		handle between common and hardware-specific interfaces
 * @mult:	multiplier
 * @div:	divider
 *
 * Clock with a fixed multiplier and divider. The output frequency is the
 * parent clock rate divided by div and multiplied by mult.
 * Implements .recalc_rate, .set_rate and .round_rate
 */

struct clk_fixed_factor {
	struct clk_hw	hw;
	unsigned int	mult;
	unsigned int	div;
};

#define to_clk_fixed_factor(_hw) container_of(_hw, struct clk_fixed_factor, hw)

extern const struct clk_ops clk_fixed_factor_ops;
struct clk *clk_register_fixed_factor(struct device *dev, const char *name,
		const char *parent_name, unsigned long flags,
		unsigned int mult, unsigned int div);
void clk_unregister_fixed_factor(struct clk *clk);
struct clk_hw *clk_hw_register_fixed_factor(struct device *dev,
		const char *name, const char *parent_name, unsigned long flags,
		unsigned int mult, unsigned int div);
void clk_hw_unregister_fixed_factor(struct clk_hw *hw);

```

API 参数比较简单, 不多说了.

 

另外, clock子系统还提供了此种类型clock的DTS接口.

clk-fixed-factor.c中的of_fixed_factor_clk_setup函数会负责解析DTS. 关于DTS的匹配规则和相关参数, 自己看看源码吧.

### **clk_register_fractional_divider**

这一类和divider clock很像, 唯一的不同在于它可支持到更细的粒度 (可用小数表示分频因子) . 例如 clk = parent / 1.5

 

API说明如下:

 

```c
头文件: include/linux/clk-provider.h

实现文件: drivers/clk/clk-fractional-divider.c
    
/**
 * struct clk_fractional_divider - adjustable fractional divider clock
 *
 * @hw:		handle between common and hardware-specific interfaces
 * @reg:	register containing the divider
 * @mshift:	shift to the numerator bit field
 * @mwidth:	width of the numerator bit field
 * @nshift:	shift to the denominator bit field
 * @nwidth:	width of the denominator bit field
 * @max_parent:	the maximum frequency of fractional divider parent clock
 * @lock:	register lock
 *
 * Clock with adjustable fractional divider affecting its output frequency.
 */
struct clk_fractional_divider {
	struct clk_hw	hw;
	void __iomem	*reg;
	u8		mshift;
	u8		mwidth;
	u32		mmask;
	u8		nshift;
	u8		nwidth;
	u32		nmask;
	u8		flags;
	unsigned long	max_prate;
	void		(*approximation)(struct clk_hw *hw,
				unsigned long rate, unsigned long *parent_rate,
				unsigned long *m, unsigned long *n);
	spinlock_t	*lock;
};

#define to_clk_fd(_hw) container_of(_hw, struct clk_fractional_divider, hw)

extern const struct clk_ops clk_fractional_divider_ops;
struct clk *clk_register_fractional_divider(struct device *dev,
		const char *name, const char *parent_name, unsigned long flags,
		void __iomem *reg, u8 mshift, u8 mwidth, u8 nshift, u8 nwidth,
		u8 clk_divider_flags, spinlock_t *lock);
struct clk_hw *clk_hw_register_fractional_divider(struct device *dev,
		const char *name, const char *parent_name, unsigned long flags,
		void __iomem *reg, u8 mshift, u8 mwidth, u8 nshift, u8 nwidth,
		u8 clk_divider_flags, spinlock_t *lock);
void clk_hw_unregister_fractional_divider(struct clk_hw *hw);

```

### **clk_register_composite**

顾名思义, 就是mux、divider、gate等clock的组合, 可通过下面接口注册:

 

```c
头文件: include/linux/clk-provider.h

实现文件: drivers/clk/clk-composite.c
/***
 * struct clk_composite - aggregate clock of mux, divider and gate clocks
 *
 * @hw:		handle between common and hardware-specific interfaces
 * @mux_hw:	handle between composite and hardware-specific mux clock
 * @rate_hw:	handle between composite and hardware-specific rate clock
 * @gate_hw:	handle between composite and hardware-specific gate clock
 * @brother_hw: a member of clk_composite who has the common parent clocks
 *              with another clk_composite, and it's also a handle between
 *              common and hardware-specific interfaces
 * @mux_ops:	clock ops for mux
 * @rate_ops:	clock ops for rate
 * @gate_ops:	clock ops for gate
 */
struct clk_composite {
	struct clk_hw	hw;
	struct clk_ops	ops;

	struct clk_hw	*mux_hw;
	struct clk_hw	*rate_hw;
	struct clk_hw	*gate_hw;
	struct clk_hw	*brother_hw;

	const struct clk_ops	*mux_ops;
	const struct clk_ops	*rate_ops;
	const struct clk_ops	*gate_ops;
};

#define to_clk_composite(_hw) container_of(_hw, struct clk_composite, hw)

struct clk *clk_register_composite(struct device *dev, const char *name,
		const char * const *parent_names, int num_parents,
		struct clk_hw *mux_hw, const struct clk_ops *mux_ops,
		struct clk_hw *rate_hw, const struct clk_ops *rate_ops,
		struct clk_hw *gate_hw, const struct clk_ops *gate_ops,
		unsigned long flags);
void clk_unregister_composite(struct clk *clk);
struct clk_hw *clk_hw_register_composite(struct device *dev, const char *name,
		const char * const *parent_names, int num_parents,
		struct clk_hw *mux_hw, const struct clk_ops *mux_ops,
		struct clk_hw *rate_hw, const struct clk_ops *rate_ops,
		struct clk_hw *gate_hw, const struct clk_ops *gate_ops,
		unsigned long flags);
void clk_hw_unregister_composite(struct clk_hw *hw);

```

### **clk_register_gpio_gate**

把某个gpio当做一个gate clock. 也就是说可以通过clock子系统来控制这个gpio 开/关, 也就是控制这个GPIO输出高/低电平.

 

某些情况下, 有的模块需要通过某个GPIO来控制其使能/禁止, 例如蓝牙模块. 而且你也不需要向此模块提供时钟, 模板本身就有晶振存在, 可以自己给自己供时钟.

这个时候我们就可以把该GPIO抽象成gpio gate clock, 借助clock子系统, 来控制模块的enable/disable.

 

```c
头文件: include/linux/clk-provider.h

实现文件: drivers/clk/clk-gpio-gate.c
/***
 * struct clk_gpio_gate - gpio gated clock
 *
 * @hw:		handle between common and hardware-specific interfaces
 * @gpiod:	gpio descriptor
 *
 * Clock with a gpio control for enabling and disabling the parent clock.
 * Implements .enable, .disable and .is_enabled
 */

struct clk_gpio {
	struct clk_hw	hw;
	struct gpio_desc *gpiod;
};

#define to_clk_gpio(_hw) container_of(_hw, struct clk_gpio, hw)

extern const struct clk_ops clk_gpio_gate_ops;
struct clk *clk_register_gpio_gate(struct device *dev, const char *name,
		const char *parent_name, struct gpio_desc *gpiod,
		unsigned long flags);
struct clk_hw *clk_hw_register_gpio_gate(struct device *dev, const char *name,
		const char *parent_name, struct gpio_desc *gpiod,
		unsigned long flags);
void clk_hw_unregister_gpio_gate(struct clk_hw *hw);

/**
 * struct clk_gpio_mux - gpio controlled clock multiplexer
 *
 * @hw:		see struct clk_gpio
 * @gpiod:	gpio descriptor to select the parent of this clock multiplexer
 *
 * Clock with a gpio control for selecting the parent clock.
 * Implements .get_parent, .set_parent and .determine_rate
 */

extern const struct clk_ops clk_gpio_mux_ops;
struct clk *clk_register_gpio_mux(struct device *dev, const char *name,
		const char * const *parent_names, u8 num_parents, struct gpio_desc *gpiod,
		unsigned long flags);
struct clk_hw *clk_hw_register_gpio_mux(struct device *dev, const char *name,
		const char * const *parent_names, u8 num_parents, struct gpio_desc *gpiod,
		unsigned long flags);
void clk_hw_unregister_gpio_mux(struct clk_hw *hw);
```

API 参数比较简单, 不多说了.

 

另外, clock子系统还提供了此种类型clock的DTS接口.

### xxx_unregister

上述xxx_register函数都有对应的xxx_unregister.

unregister的函数原型在上述代码中都有提及, 这里不多说了, 有兴趣可以自行阅读源代码.

### **clk_register_clkdev**

头文件: include/linux/clkdev.h

实现文件: drivers/clk/clkdev.c

原型: int clk_register_clkdev**(**struct clk ***,** const char ***,** const char ***,** **…);**

 

上述的clk_register_xxx接口会向clock子系统注册一个clock, 并返回一个代表该clock的struct clk结构体.

clk_register_clkdev的作用就是把返回的这个struct clk添加到某个池子里面.

 

这个池子其实就是个链表啦, 链表定义在clkdev.c里面.

池子里面主要是用name做为关键字, 区分不同的clk.

 

当consumer端想要查询某个clk的时候, 就会尝试从这个链表里面通过clock name去检索.

### **of_clk_add_provider**

```c
头文件: include/linux/clk-provider.h
实现文件: drivers/clk/clk.c
int of_clk_add_provider(struct device_node *np,
			struct clk *(*clk_src_get)(struct of_phandle_args *args,
						   void *data),
			void *data);
```

此API的主要目的也是把得到的struct clk结构体添加到某个池子里面.

 

这个池子也是个链表, 定义在clk.c里面.

与上面那个池子的不同之处在于, 这里是用device_node做为关键字来区分不同的clk.

 

当consumer端想要查询某个clk的时候, 会在DTS node里面通过clocks = <&xxxx>来引用某个clock.

通过引用的这个clock的phandle, 就能找到对应的device_node. 然后通过device_node就能从池子里面检索出需要的clk.

 

其实, 这两种池子对应了我们的consumer端的两种方式: 一种是通过name获取clk, 不需要DTS; 另外一种就是通过DTS.

随着Linux内核大力推行DTS, 方式二会逐渐成为主流.

# **3.**  **clock** **core —** 如何管理Clocks

## **3.1**       **简介**

第2章我们描述了clock子系统的功能之一 : 向底层的clock driver提供注册接口.

本章我们描述clock子系统的功能之二 : 如何管理这些clocks.

当clock driver向子系统注册某一个clock的时候, 子系统内部会创建一个数据结构来表示这个clock. 这个数据结构在前文提过, 就是struct clk.

一个struct clk就对应一个clock. 到目前为止, 我们还没见过这个数据结构的庐山真面目呢! 本章会围绕这个数据结构展开, 充分理解它, 你就里面本章了.

## **3.2**       **主要数据结构**

### **struct** **clk**

结构体定义在一个C文件里面 : drivers/clk/clk.c

![](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/struct_clk.png)

### **struct** **clk_core**

前文我们说过, 一个struct clk就代表一个clock. 一个clk_core与clock也是一一对应的关系. 那它俩有什么区别呢?

 

struct clk更像是对外的接口, 例如当provider向clock子系统注册时, 它会得到一个struct clk*的返回结果; 当consumer想要使用某个clock时, 也会首先获取struct clk*这个结构体.

 

struct clk_core则是clock子系统核心层的一个私有数据结构, 在核心层代描述某个具体的clock. provider或consumer不会触碰这个数据结构.

 

结构体定义在一个C文件里面 : drivers/clk/clk.c

![image-20210824164801849](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/image-20210824164801849.png)

## **3.3** **关键代码分析**

### **clk_register**

我们在2.4节介绍了clock子系统提供给provider端的总多clk_register_xxx函数. 这些函数都是对clk_register的封装, 基础都是clk_register.

 

因此我们本章只重点讲解这一个API, 其它API有兴趣可以自己阅读源码.

 

在开始分析之前, 结合之前讲解的内容, 你能猜到clk_register会做哪些事情吗?

首先, 它得创建struct clk和struct clk_core这两个数据结构.

然后, 它得维护clk_core的树形关系, 就像时钟数的硬件形态那样:

有一个或多个ROOT_CLOCK, ROOT_CLOCK没有parent, 下面挂载的是children, children下面在挂载children.

为什么要维护这样的树形结构呢? 因为我们在操作某个clk时, 往往会跟它的parent有关: 例如要使能某个clk, 必须保证它的parent也使能; 或者parent的rate改变了, 那它的children的rate都有可能改变. 因此clock子系统必须维护好树形结构, 才能方便的处理这些相关性.

下面我们来看看代码:

```c
实现文件: drivers/clk/clk.c
/**
 * clk_register - allocate a new clock, register it and return an opaque cookie
 * @dev: device that is registering this clock
 * @hw: link to hardware-specific clock data
 *
 * clk_register is the primary interface for populating the clock tree with new
 * clock nodes.  It returns a pointer to the newly allocated struct clk which
 * cannot be dereferenced by driver code but may be used in conjunction with the
 * rest of the clock API.  In the event of an error clk_register will return an
 * error code; drivers must test for an error code after calling clk_register.
 */
struct clk *clk_register(struct device *dev, struct clk_hw *hw)
{
	int i, ret;
	struct clk_core *core;

	core = kzalloc(sizeof(*core), GFP_KERNEL);
	......

	core->name = kstrdup_const(hw->init->name, GFP_KERNEL);
	......
	core->ops = hw->init->ops;

	if (dev && pm_runtime_enabled(dev))
		core->rpm_enabled = true;
	core->dev = dev;
	if (dev && dev->driver)
		core->owner = dev->driver->owner;
	core->hw = hw;
	core->flags = hw->init->flags;
	core->num_parents = hw->init->num_parents;
	core->min_rate = 0;
	core->max_rate = ULONG_MAX;
	hw->core = core;

	/* allocate local copy in case parent_names is __initdata */
	core->parent_names = kcalloc(core->num_parents, sizeof(char *),
					GFP_KERNEL);
    ......


	/* copy each string name in case parent_names is __initdata */
	for (i = 0; i < core->num_parents; i++) {
		core->parent_names[i] = kstrdup_const(hw->init->parent_names[i],
						GFP_KERNEL);
		......
	}

	/* avoid unnecessary string look-ups of clk_core's possible parents. */
	core->parents = kcalloc(core->num_parents, sizeof(*core->parents),
				GFP_KERNEL);
	......

	INIT_HLIST_HEAD(&core->clks);

	hw->clk = clk_hw_create_clk(hw, NULL, NULL);
	......

	ret = __clk_core_init(core);
	......

	__clk_free_clk(hw->clk);
	hw->clk = NULL;
    ......
}
EXPORT_SYMBOL_GPL(clk_register);
```

上面这段代码的逻辑比较简单:

- 首先, 创建了clk_core结构体, 然后用provider提供的clk_hw, 填充clk_core中的各个字段.

- 然后, 调用 clk_hw_create_clk, 在clk_hw_create_clk里面会创建struct clk这个结构体并初始化其相关字段. clk_register会返回给provider一个struct clk*的结构体指针, 这个clk*就是这里创建的.

下文我们会单独介绍clk_hw_create_clk这个函数.

- 最后, 会调用__clk_core_init, 在__clk_core_init里面会处理clk_core之间的树形结构关系.

下文我们会单独介绍__clk_init这个函数.



这个API会创建一个struct clk结构体.

要理解此API的细节, 首先得理清楚struct clk和struct clk_core的关系.

 

我们在3.2节《struct clk_core》中已经初步介绍了clk和clk_core的关系, 可以回头看看.

这里我们在做进一步的说明.clk和clk_core的关系与上面很类型, clk_core代表一个硬件上的clock, 一个clock对应唯一一个clk_core; 而clk则代表被使用的clock, 例如GPIO driver想要获取某个clock, 它会得到一个struct clk, SPI driver想要获取同一个clock, 它也会得到一个struct clk, 另外, 当此clock的provider向clock子系统注册时, provider也会得到一个struct clk.

```c
static struct clk *clk_hw_create_clk(struct clk_hw *hw, const char *dev_id,
				     const char *con_id)
{
	struct clk *clk;

	clk = kzalloc(sizeof(*clk), GFP_KERNEL);
	if (!clk)
		return ERR_PTR(-ENOMEM);

	clk->core = hw->core;
	clk->dev_id = dev_id;
	clk->con_id = kstrdup_const(con_id, GFP_KERNEL);
	clk->max_rate = ULONG_MAX;

	clk_prepare_lock();
	hlist_add_head(&clk->clks_node, &hw->core->clks);
	clk_prepare_unlock();

	return clk;
}

```

逻辑比较简单:

- 首先创建struct clk结构体, 然后初始化结构体的相关参数, 最后把这个struct clk挂载到clk_core的clks链表头下面.

### **__clk_core_init**

这个API主要是处理clock之间的树形关系.

当任何一个clock调用clk_register接口进行注册时, __clk_init都会负责把这个clock放在树形结构的恰当位置.

```c
/**
 * __clk_core_init - initialize the data structures in a struct clk_core
 * @core:	clk_core being initialized
 *
 * Initializes the lists in struct clk_core, queries the hardware for the
 * parent and rate and sets them both.
 */
static int __clk_core_init(struct clk_core *core)
{
	int i, ret;
	struct clk_core *orphan;
	struct hlist_node *tmp2;
	unsigned long rate;

	if (!core)
		return -EINVAL;

	clk_prepare_lock();

	ret = clk_pm_runtime_get(core);
	if (ret)
		goto unlock;

	/* check to see if a clock with this name is already registered */
	if (clk_core_lookup(core->name)) {
		pr_debug("%s: clk %s already initialized\n",
				__func__, core->name);
		ret = -EEXIST;
		goto out;
	}

	/* check that clk_ops are sane.  See Documentation/driver-api/clk.rst */
	if (core->ops->set_rate &&
	    !((core->ops->round_rate || core->ops->determine_rate) &&
	      core->ops->recalc_rate)) {
		pr_err("%s: %s must implement .round_rate or .determine_rate in addition to .recalc_rate\n",
		       __func__, core->name);
		ret = -EINVAL;
		goto out;
	}

	if (core->ops->set_parent && !core->ops->get_parent) {
		pr_err("%s: %s must implement .get_parent & .set_parent\n",
		       __func__, core->name);
		ret = -EINVAL;
		goto out;
	}

	if (core->num_parents > 1 && !core->ops->get_parent) {
		pr_err("%s: %s must implement .get_parent as it has multi parents\n",
		       __func__, core->name);
		ret = -EINVAL;
		goto out;
	}

	if (core->ops->set_rate_and_parent &&
			!(core->ops->set_parent && core->ops->set_rate)) {
		pr_err("%s: %s must implement .set_parent & .set_rate\n",
				__func__, core->name);
		ret = -EINVAL;
		goto out;
	}

	/* throw a WARN if any entries in parent_names are NULL */
	for (i = 0; i < core->num_parents; i++)
		WARN(!core->parent_names[i],
				"%s: invalid NULL in %s's .parent_names\n",
				__func__, core->name);

	core->parent = __clk_init_parent(core);

	/*
	 * Populate core->parent if parent has already been clk_core_init'd. If
	 * parent has not yet been clk_core_init'd then place clk in the orphan
	 * list.  If clk doesn't have any parents then place it in the root
	 * clk list.
	 *
	 * Every time a new clk is clk_init'd then we walk the list of orphan
	 * clocks and re-parent any that are children of the clock currently
	 * being clk_init'd.
	 */
	if (core->parent) {
		hlist_add_head(&core->child_node,
				&core->parent->children);
		core->orphan = core->parent->orphan;
	} else if (!core->num_parents) {
		hlist_add_head(&core->child_node, &clk_root_list);
		core->orphan = false;
	} else {
		hlist_add_head(&core->child_node, &clk_orphan_list);
		core->orphan = true;
	}

	/*
	 * optional platform-specific magic
	 *
	 * The .init callback is not used by any of the basic clock types, but
	 * exists for weird hardware that must perform initialization magic.
	 * Please consider other ways of solving initialization problems before
	 * using this callback, as its use is discouraged.
	 */
	if (core->ops->init)
		core->ops->init(core->hw);

	/*
	 * Set clk's accuracy.  The preferred method is to use
	 * .recalc_accuracy. For simple clocks and lazy developers the default
	 * fallback is to use the parent's accuracy.  If a clock doesn't have a
	 * parent (or is orphaned) then accuracy is set to zero (perfect
	 * clock).
	 */
	if (core->ops->recalc_accuracy)
		core->accuracy = core->ops->recalc_accuracy(core->hw,
					__clk_get_accuracy(core->parent));
	else if (core->parent)
		core->accuracy = core->parent->accuracy;
	else
		core->accuracy = 0;

	/*
	 * Set clk's phase.
	 * Since a phase is by definition relative to its parent, just
	 * query the current clock phase, or just assume it's in phase.
	 */
	if (core->ops->get_phase)
		core->phase = core->ops->get_phase(core->hw);
	else
		core->phase = 0;

	/*
	 * Set clk's duty cycle.
	 */
	clk_core_update_duty_cycle_nolock(core);

	/*
	 * Set clk's rate.  The preferred method is to use .recalc_rate.  For
	 * simple clocks and lazy developers the default fallback is to use the
	 * parent's rate.  If a clock doesn't have a parent (or is orphaned)
	 * then rate is set to zero.
	 */
	if (core->ops->recalc_rate)
		rate = core->ops->recalc_rate(core->hw,
				clk_core_get_rate_nolock(core->parent));
	else if (core->parent)
		rate = core->parent->rate;
	else
		rate = 0;
	core->rate = core->req_rate = rate;

	core->boot_enabled = clk_core_is_enabled(core);

	/*
	 * Enable CLK_IS_CRITICAL clocks so newly added critical clocks
	 * don't get accidentally disabled when walking the orphan tree and
	 * reparenting clocks
	 */
	if (core->flags & CLK_IS_CRITICAL) {
		unsigned long flags;

		ret = clk_core_prepare(core);
		if (ret)
			goto out;

		flags = clk_enable_lock();
		ret = clk_core_enable(core);
		clk_enable_unlock(flags);
		if (ret) {
			clk_core_unprepare(core);
			goto out;
		}
	}

	clk_core_hold_state(core);

	/*
	 * walk the list of orphan clocks and reparent any that newly finds a
	 * parent.
	 */
	hlist_for_each_entry_safe(orphan, tmp2, &clk_orphan_list, child_node) {
		struct clk_core *parent = __clk_init_parent(orphan);

		/*
		 * We need to use __clk_set_parent_before() and _after() to
		 * to properly migrate any prepare/enable count of the orphan
		 * clock. This is important for CLK_IS_CRITICAL clocks, which
		 * are enabled during init but might not have a parent yet.
		 */
		if (parent) {
			/* update the clk tree topology */
			__clk_set_parent_before(orphan, parent);
			__clk_set_parent_after(orphan, parent, NULL);
			__clk_recalc_accuracies(orphan);
			__clk_recalc_rates(orphan, 0);
			__clk_core_update_orphan_hold_state(orphan);
		}
	}

	kref_init(&core->ref);
out:
	clk_pm_runtime_put(core);
unlock:
	clk_prepare_unlock();

	if (!ret)
		clk_debug_register(core);

	return ret;
}
```

# **4.**  **clock** consumer — 如何使用Clocks

##  **4.1**    **简介**

clock 子系统管理clocks的最终目的, 是让device driver可以方便的获取并使用这些clocks.

我们知道clock子系统用一个struct clk结构体来抽象某一个clock.

当device driver要操作某个clock时, 它需要做两件事情:

- 首先, 获取clock. 也叫clk_get.

- 然后, 操作这个clock. 如 clk_prepare/ clk_enable/ clk_disable/ clk_set_rate/ …

## **4.2**    **APIs** **–** 获取clk

```c
头文件: include/linux/clk.h

实现文件: drivers/clk/clkdev.c

最主要的两个API:
/**
 * clk_get - lookup and obtain a reference to a clock producer.
 * @dev: device for clock "consumer"
 * @id: clock consumer ID
 *
 * Returns a struct clk corresponding to the clock producer, or
 * valid IS_ERR() condition containing errno.  The implementation
 * uses @dev and @id to determine the clock consumer, and thereby
 * the clock producer.  (IOW, @id may be identical strings, but
 * clk_get may return different clock producers depending on @dev.)
 *
 * Drivers must assume that the clock source is not enabled.
 *
 * clk_get should not be called from within interrupt context.
 */
struct clk *clk_get(struct device *dev, const char *id);
 
/**
 * devm_clk_get - lookup and obtain a managed reference to a clock producer.
 * @dev: device for clock "consumer"
 * @id: clock consumer ID
 *
 * Returns a struct clk corresponding to the clock producer, or
 * valid IS_ERR() condition containing errno.  The implementation
 * uses @dev and @id to determine the clock consumer, and thereby
 * the clock producer.  (IOW, @id may be identical strings, but
 * clk_get may return different clock producers depending on @dev.)
 *
 * Drivers must assume that the clock source is not enabled.
 *
 * devm_clk_get should not be called from within interrupt context.
 *
 * The clock will automatically be freed when the device is unbound
 * from the bus.
 */
struct clk *devm_clk_get(struct device *dev, const char *id);

```

devm_clk_get是devm版本的clk_get. 用于自动释放资源。

除了这两个用的最多的API之外, 还有一些其他的API, 如下:

```c
void clk_put(struct clk *clk);
void devm_clk_put(struct device *dev, struct clk *clk);
struct clk *clk_get_sys(const char *dev_id, const char *con_id);
struct clk *of_clk_get(struct device_node *np, int index);
struct clk *of_clk_get_by_name(struct device_node *np, const char *name);
struct clk *of_clk_get_from_provider(struct of_phandle_args *clkspec);
```

clk_get相当于一个总逻辑, 它会根据不同的情况调用上述这些API, 在实际代码中, 基本上我们只会用到clk_get或者devm_clk_get.

 

接下来, 我们会仔细看看clk_get的内部逻辑.

### **clk_get / devm_clk_get**

该API的主要作用就是根据参数, 从clock子系统中获取一个clk给到consumer, 然后consumer就可以操作该clk了. 因此该API的返回值就是用于描述某个clock的数据结构: struct clk.



```c
头文件: include/linux/clk.h

实现文件: drivers/clk/clkdev.c


struct clk *clk_get(struct device *dev, const char *con_id)
{
	const char *dev_id = dev ? dev_name(dev) : NULL;
	struct clk *clk;
 
	if (dev) {
		clk = of_clk_get_by_name(dev->of_node, con_id);        /* 从设备树节点得到clk */
		if (!IS_ERR(clk))            /* 注意这里对错误判断取反了,即没错的话会直接返回clk */
			return clk;
		if (PTR_ERR(clk) == -EPROBE_DEFER)    /* 重新获取 */
			return clk;
	}
 
	return clk_get_sys(dev_id, con_id);    /* 设备树节点没找到,从系统注册链表中搜索得到 */
}
 
struct clk *clk_get_sys(const char *dev_id, const char *con_id)
{
	struct clk_lookup *cl;
 
	mutex_lock(&clocks_mutex);
	cl = clk_find(dev_id, con_id);        /* 根据设备名或链接名在clock链表中查找到对应的cl*/
	if (cl && !__clk_get(cl->clk))        /* 这里对找到的时钟的引用计数+1 */
		cl = NULL;
	mutex_unlock(&clocks_mutex);
 
	return cl ? cl->clk : ERR_PTR(-ENOENT);
}
 
 
/*
 * Find the correct struct clk for the device and connection ID.
 * We do slightly fuzzy matching here:
 *  An entry with a NULL ID is assumed to be a wildcard.
 *  If an entry has a device ID, it must match
 *  If an entry has a connection ID, it must match
 * Then we take the most specific entry - with the following
 * order of precedence: dev+con > dev only > con only. 这是重点,即匹配优先级
 */
static struct clk_lookup *clk_find(const char *dev_id, const char *con_id)
{
	struct clk_lookup *p, *cl = NULL;
	int match, best_found = 0, best_possible = 0;
 
	if (dev_id)        /* 设备名称 */
		best_possible += 2;
	if (con_id)        /* 连接名称(可以是pclk,uart_clk,phy,pci,总是一般都是总线的时钟名称) */
		best_possible += 1;
 
	list_for_each_entry(p, &clocks, node) {    /* clocks链表中查找,根据dev+con > dev only > con优先级来匹配 */
		match = 0;
		if (p->dev_id) {
			if (!dev_id || strcmp(p->dev_id, dev_id))
				continue;
			match += 2;
		}
		if (p->con_id) {
			if (!con_id || strcmp(p->con_id, con_id))
				continue;
			match += 1;
		}
 
		if (match > best_found) {
			cl = p;
			if (match != best_possible)
				best_found = match;
			else
				break;
		}
	}
	return cl;
}
```



如果你已经充分理解了1.2节的那个系统框图, 那么此处的逻辑就很简单了:

- 如果dev不为空, 那么就以dev->of_node为参数, 调用of_XXX那一套, 从LIST_HEAD(**of_clk_providers**)这个池子里面查询某个clk.

如果查询到了, 则返回该clk. 这种情况其实对应5.1节中描述的DTS node方式.

- 如果上面没有获取到clk, 则调用clk_get_sys, 从LIST_HEAD(clocks)这个池子里面查询clk. 查询的关键字是con_id, 其实就是clock的name.

```c

struct clk *devm_clk_get(struct device *dev, const char *id)
{
	struct clk **ptr, *clk;
 
     /* 设备资源自动管理,卸载设备时会调用devm_clk_release,做清理工作 */
	ptr = devres_alloc(devm_clk_release, sizeof(*ptr), GFP_KERNEL);   
	if (!ptr)
		return ERR_PTR(-ENOMEM);
 
	clk = clk_get(dev, id);        /* 这里的核心和就是clk_get */
	if (!IS_ERR(clk)) {
		*ptr = clk;
		devres_add(dev, ptr);        /* 资源绑定到这设备资源链表上  */
	} else {
		devres_free(ptr);
	}
 
	return clk;

```



## **4.3**    **APIs** **–** **操作**clk

```c
头文件: include/linux/clk.h

实现文件: drivers/clk/clk.c
int clk_prepare(struct clk *clk);
void clk_unprepare(struct clk *clk);
 
int clk_enable(struct clk *clk);
void clk_disable(struct clk *clk);
 
unsigned long clk_get_rate(struct clk *clk);
int clk_set_rate(struct clk *clk, unsigned long rate);
long clk_round_rate(struct clk *clk, unsigned long rate);
 
struct clk *clk_get_parent(struct clk *clk);
int clk_set_parent(struct clk *clk, struct clk *parent);
 
static inline int clk_prepare_enable(struct clk *clk);
static inline void clk_disable_unprepare(struct clk *clk);
```



- clk_enable/clk_disable: 启动/停止clock. 不会睡眠

- clk_prepare/clk_unprepare: 启动clock前的准备工作/停止clock后的善后工作. 可能会睡眠

- clk_get_rate/clk_set_rate/clk_round_rate: clock频率的获取和设置, 其中clk_set_rate可能会不成功（例如没有对应的分频比）, 此时会返回错误. 如果要确保设置成功, 则需要先调用clk_round_rate接口, 得到和需要设置的rate比较接近的那个值

- clk_set_parent / clk_get_parent: 获取/选择clock的parent clock

- clk_prepare_enable: 将clk_prepare和clk_enable组合起来，一起调用

- clk_disable_unprepare: 将clk_disable和clk_unprepare组合起来，一起调用

**prepare/unprepare，enable/disable**的说明:

这两套API的本质，是把clock的启动/停止分为atomic和non-atomic两个阶段，以方便实现和调用。

因此上面所说的“不会睡眠/可能会睡眠”，有两个角度的含义：

一是告诉底层的clock driver，请把可能引起睡眠的操作，放到prepare/unprepare中实现，一定不能放到enable/disable中

二是提醒上层使用clock的driver，调用prepare/unprepare接口时可能会睡眠哦，千万不能在atomic上下文（例如中断处理中）调用哦，而调用enable/disable接口则可放心。

另外，clock的开关为什么需要睡眠呢？这里举个例子，例如enable PLL clk，在启动PLL后，需要等待它稳定。而PLL的稳定时间是很长的，这段时间要把CPU交出（进程睡眠），不然就会浪费CPU。

最后，为什么会有合在一起的clk_prepare_enable/clk_disable_unprepare接口呢？如果调用者能确保是在non-atomic上下文中调用，就可以顺序调用prepare/enable、disable/unprepared，为了简单，framework就帮忙封装了这两个接口。

## **4.4** 其它APIs

```c
头文件: include/linux/clk.h

实现文件: drivers/clk/clk.c
/**
 * clk_notifier_register: register a clock rate-change notifier callback
 * @clk: clock whose rate we are interested in
 * @nb: notifier block with callback function pointer
 *
 * ProTip: debugging across notifier chains can be frustrating. Make sure that
 * your notifier callback function prints a nice big warning in case of
 * failure.
 */
int clk_notifier_register(struct clk *clk, struct notifier_block *nb);

/**
 * clk_notifier_unregister: unregister a clock rate-change notifier callback
 * @clk: clock whose rate we are no longer interested in
 * @nb: notifier block which will be unregistered
 */
int clk_notifier_unregister(struct clk *clk, struct notifier_block *nb);

```

这两个API与内核提供的通知链机制有关.

这两个notify接口, 用于注册/注销 clock rate改变的通知.

例如某个driver关心某个clock, 期望这个clock的rate改变时, 通知到自己, 就可以注册一个notify.

# **5.**  RK3399 Clk分析

代码结构 CLOCK 的软件框架由 CLK 的 clk-rk3399.c（clk 的寄存器描述、clk 之间的树状关系等）、 Device driver 的 CLK 配置和 CLK API 三部分构成。这三部分的功能、CLK 代码路径如表 1-1 所 示。

| 项目                      | 功能                                                         | 路径                                                         |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Clk-rk3399.c Rk3399-cru.h | .c 中主要是 clk 的寄存器描述、 clk 之间的树状关系描述等。 .h 中是 clk 的 ID 定义，通过 ID 匹配 clk name。 | Drivers/clk/rockchip/clk-rk3399.c Include/dt-bindings/clock/rk3399-cru.h |
| RK 特别的处理             | 1、处理 RK 的 PLL 时钟 2、处理 RK 的 CPU 时钟等              | Drivers/clk/rockchip/clk-xxx.c                               |
| CLK API                   | 提供 linux环境下供 driver 调用 的接口                        | Drivers/clk/clk-xxx.c                                        |

## **5.1** **时钟的相关概念** 

**1，PLL** 锁相环，是由 24M 的晶振输入，然后内部锁相环锁出相应的频率。这个是 SOC 所有 CLOCK 的时钟的源。SOC 的所有总线及设备的时钟都是从 PLL 分频下来的。RK 平台主要 PLL 有:

![image-20210826102009375](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/image-20210826102009375.png)

**2，ACLK、PCLK、HCLK**

ACLK 是设备的总线的 CLK，PCLK 跟 HCLK 一般是用于寄存器读写的。而像 CLK_GPU 是 GPU 的控制器的时钟。 我们 SOC 的总线有 ACLK_PERI、HCLK_PERI、PCLK_PERI、ACLK_BUS、HCLK_BUS、 PCLK_BUS.各个设备的总线时钟会挂在上面这些时钟下面，如下图结构：

![image-20210826102204721](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/image-20210826102204721.png)

（如：EMMC 想提高自己设备的总线频率以实现其快速的数据拷贝或者搬移，可以提高 ACLK_PERI 来实现）

RK3399 上设计将高速和低速总线彻底分开，分成高速：ACLK_PERIHP、HCLK_PERIHP、 PCLK_PERIHP；低速：ACLK_PERILP0、HCLK_PERILP0、PCLK_PERILP0、 HCLK_PERILP1、PCLK_PERILP1。这样做是为了功耗最优，根据不同的需求可以设置不同的总 线频率。（具体每个设备在哪条总线下详细见时钟图） 可以参考如下（EMMC、GMAC、USB 等有自己的 ACLK）：

![image-20210826102226714](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/image-20210826102226714.png)

**3，GATING** Clock 的框架中有很多的 Gating，这个主要是为了降低功耗使用，在一些设备关闭，Clock 不 需要维持的时候就可以关闭 Gating，来节省功耗。 我们 Clock 的框架的 Gating 是按照树的结构，有父子属性。Gating 的开关是有一个引用计数 机制的，使用这个计数来实现 Clock 打开时，会遍历打开其父 Clock。在子 Clock 关闭时，父 Clock 会遍历所有的子 Clock，在所有的子都关闭的时候才会关闭父 Clock。 （如：I²S2 在使用的时候，必须要打开下面这三个 Gating，但是软件上只需要开最后一级的 Gating，我们的时钟结构会自动的打开其 Parent 的 Gating）

![image-20210826102344032](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/image-20210826102344032.png)

## **5.2** 时钟配置

**1，时钟初始化配置** 

```c
F:\Rock4Plus_Android10\u-boot\arch\arm\dts\rk3399.dtsi
	pmucru: pmu-clock-controller@ff750000 {
		compatible = "rockchip,rk3399-pmucru";
		reg = <0x0 0xff750000 0x0 0x1000>;
		rockchip,grf = <&pmugrf>;
		#clock-cells = <1>;
		#reset-cells = <1>;
		assigned-clocks = <&pmucru PLL_PPLL>;
		assigned-clock-rates = <676000000>; //CLOCK TREE 初始化时设置的频率：
	};

	cru: clock-controller@ff760000 {
		compatible = "rockchip,rk3399-cru";
		reg = <0x0 0xff760000 0x0 0x1000>;
		rockchip,grf = <&grf>;
		#clock-cells = <1>;
		#reset-cells = <1>;
		assigned-clocks =
			<&cru PLL_GPLL>, <&cru PLL_CPLL>,
			<&cru PLL_NPLL>,
			<&cru ACLK_PERIHP>, <&cru HCLK_PERIHP>,
			<&cru PCLK_PERIHP>,
			<&cru ACLK_PERILP0>, <&cru HCLK_PERILP0>,
			<&cru PCLK_PERILP0>, <&cru ACLK_CCI>,
			<&cru HCLK_PERILP1>, <&cru PCLK_PERILP1>;
		assigned-clock-rates = //CLOCK TREE 初始化时设置的频率：
			 <594000000>,  <800000000>,
			<1000000000>,
			 <150000000>,   <75000000>,
			  <37500000>,
			 <100000000>,  <100000000>,
			  <50000000>, <600000000>,
			 <100000000>,   <50000000>;
	};

```

**2，Gating CLOCK TREE** 

初始化时是否默认 enable： （1） 需要在 clk-rk3399.c 中增加 critical 配置，主要在 rk3399_cru_critical_clocks 中 增加需要默认打开的 CLK name，一旦增加 CLK 的计数被加 1，后面这个 CLK 将不能被 关闭。

```c
static const char *const rk3399_cru_critical_clocks[] __initconst = {
	"aclk_usb3_noc",
	"aclk_gmac_noc",
	"pclk_gmac_noc",
	......
	"aclk_perihp_noc",
	"hclk_perihp_noc",
	.......
	"hclk_sdioaudio_noc",
	"pclk_perilp1_noc",
	......
};

static const char *const rk3399_pmucru_critical_clocks[] __initconst = {
	"pclk_noc_pmu",
	"hclk_noc_pmu",
	"ppll",
	"pclk_pmu_src",
	"fclk_cm0s_src_pmu",
	"clk_timer_src_pmu",
	"pclk_rkpwm_pmu",
};

```

（2） CLK 的定义时候增加 flag 属性 CLK_IGNORE_UNUSED，这样即使这个 CLK 没有使 用，在最后 CLK 关闭没有用的 CLK 时也不会关闭这个。但是在 CLK TREE 上看到的 enable  cnt 还是 0，但是 CLK 是开启的。 GATE(PCLK_PMUGRF_PMU, "pclk_pmugrf_pmu", "pclk_pmu_src", CLK_IGNORE_UNUSED, RK3399_PMU_CLKGATE_CON(1), 1, GFLAGS)

**3，主要的 CLK 注册类型函数** 

常用的有如下几种： 

- GATE：描述 GATING，主要包括 CLK ID、类型、GATING 的寄存器偏移地址、BIT 位等。 

- MUX：描述 SLECT，主要包括 CLK ID、类型、MUX 的寄存器偏移地址、BIT 位等。 

- COMPOSITE：描述有 MUX、DIV、GATING 的 CLK，主要包括 CLK ID、类型、MUX、 DIV、GARING 的寄存器偏移地址、BIT 位等。

![image-20210826110739981](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/image-20210826110739981.png)

Clk-rk3xxx.c 中的使用，使用这些 CLK 的注册函数，描述此 CLK 的类型，寄存器及父子关系 等。

![image-20210826110855862](https://raw.githubusercontent.com/zhoujinjianzjj/zhoujinjian.com.images/master/kernel.clk/image-20210826110855862.png)

##  **5.3** CLK_OF_DECLARE 解析

```c
F:\Rock4Plus_Android10\kernel\init\main.c
asmlinkage __visible void __init start_kernel(void)
{
    ......
	time_init();
    ......
}
F:\Rock4Plus_Android10\kernel\arch\arm\kernel\time.c
void __init time_init(void)
{
	if (machine_desc->init_time) {
		machine_desc->init_time();
	} else {
#ifdef CONFIG_COMMON_CLK
		of_clk_init(NULL);
#endif
		timer_probe();
	}
}
F:\Rock4Plus_Android10\kernel\drivers\clk\clk.c

void __init of_clk_init(const struct of_device_id *matches)
{
.......
	if (!matches)
		matches = &__clk_of_table;
.......
}
_OF_DECLARE 的定义：

/*
 * Struct used for matching a device
 */
struct of_device_id {
	char	name[32];
	char	type[32];
	char	compatible[128];
	const void *data;
};
 
#ifdef CONFIG_OF                                        //CONFIG_OF的定义
#define _OF_DECLARE(table, name, compat, fn, fn_type)			\
	static const struct of_device_id __of_table_##name		\
		__used __section(__##table##_of_table)			\
		 = { .compatible = compat,				\
		     .data = (fn == (fn_type)NULL) ? fn : fn  }
#else
#define _OF_DECLARE(table, name, compat, fn, fn_type)			\
	static const struct of_device_id __of_table_##name		\
		__attribute__((unused))					\
		 = { .compatible = compat,				\
		     .data = (fn == (fn_type)NULL) ? fn : fn }
#endif
 
 
定义了一个 struct of_device_id 的结构体变量
如果定义CONFIG_OF，就将 struct of_device_id 变量放到 __clk_of_table 下；
```

Linux下系统时钟在初始化时经常用到 CLK_OF_DECLARE 这个宏，现在以RK3399为例做分析：

```c
F:\Rock4Plus_Android10\kernel\drivers\clk\rockchip\clk-rk3399.c
CLK_OF_DECLARE(rk3399_cru, "rockchip,rk3399-cru", rk3399_clk_init);
CLK_OF_DECLARE(rk3399_cru_pmu, "rockchip,rk3399-pmucru", rk3399_pmu_clk_init);
[2021/8/23 14:22:34] [    0.000000] Call trace:
[2021/8/23 14:22:34] [    0.000000]  dump_backtrace+0x0/0x178
[2021/8/23 14:22:34] [    0.000000]  show_stack+0x14/0x20
[2021/8/23 14:22:34] [    0.000000]  dump_stack+0x94/0xb4
[2021/8/23 14:22:34] [    0.000000]  rk3399_pmu_clk_init+0x60/0x120
[2021/8/23 14:22:34] [    0.000000]  of_clk_init+0x194/0x25c
[2021/8/23 14:22:34] [    0.000000]  time_init+0x10/0x44
[2021/8/23 14:22:34] [    0.000000]  start_kernel+0x390/0x4ec
[2021/8/23 14:22:34] [    0.000000] Call trace:
[2021/8/23 14:22:34] [    0.000000]  dump_backtrace+0x0/0x178
[2021/8/23 14:22:34] [    0.000000]  show_stack+0x14/0x20
[2021/8/23 14:22:34] [    0.000000]  dump_stack+0x94/0xb4
[2021/8/23 14:22:34] [    0.000000]  rk3399_clk_init+0x5c/0x1c4
[2021/8/23 14:22:34] [    0.000000]  of_clk_init+0x194/0x25c
[2021/8/23 14:22:34] [    0.000000]  time_init+0x10/0x44
[2021/8/23 14:22:34] [    0.000000]  start_kernel+0x390/0x4ec
```



##  **5.4** rk3399_pmu_clk_init分析

> [2021/8/23 14:22:34] [    0.000000] regs phy start(0xff750000) name(pmu-clock-controller@ff750000)

```c
F:\Rock4Plus_Android10\kernel\drivers\clk\rockchip\clk-rk3399.c
static void __init rk3399_pmu_clk_init(struct device_node *np)
{
	struct rockchip_clk_provider *ctx;
	void __iomem *reg_base;

	reg_base = of_iomap(np, 0);
	......

	rk3399_pmucru_base = reg_base;
	printk("rk3399_pmucru_base regs map(0x%p)\n", rk3399_cru_base);
	dump_stack();

	ctx = rockchip_clk_init(np, reg_base, CLKPMU_NR_CLKS);
	......

	rockchip_clk_register_plls(ctx, rk3399_pmu_pll_clks,
				   ARRAY_SIZE(rk3399_pmu_pll_clks), -1);//rk3399_pmu_pll_clks注册

	rockchip_clk_register_branches(ctx, rk3399_clk_pmu_branches,
				  ARRAY_SIZE(rk3399_clk_pmu_branches));//rk3399_clk_pmu_branches注册

	rockchip_clk_protect_critical(rk3399_pmucru_critical_clocks,
				  ARRAY_SIZE(rk3399_pmucru_critical_clocks));//rk3399_pmucru_critical_clocks注册

	rockchip_register_softrst(np, 2, reg_base + RK3399_PMU_SOFTRST_CON(0),
				  ROCKCHIP_SOFTRST_HIWORD_MASK);

	rockchip_clk_of_add_provider(np, ctx);

	atomic_notifier_chain_register(&panic_notifier_list,
				       &rk3399_clk_panic_block);
}
CLK_OF_DECLARE(rk3399_cru_pmu, "rockchip,rk3399-pmucru", rk3399_pmu_clk_init);
```

## **5.5** rk3399_pmu_clk_init分析

>[2021/8/23 14:22:34] [    0.000000] regs phy start(0xff760000) name(clock-controller@ff760000)

```c
static void __init rk3399_clk_init(struct device_node *np)
{
	struct rockchip_clk_provider *ctx;
	void __iomem *reg_base;
	struct clk *clk;

	reg_base = of_iomap(np, 0);
	......

	rk3399_cru_base = reg_base;
	printk("rk3399_cru_base regs map(0x%p)\n", rk3399_cru_base);
	dump_stack();

	ctx = rockchip_clk_init(np, reg_base, CLK_NR_CLKS);
	......

	/* Watchdog pclk is controlled by RK3399 SECURE_GRF_SOC_CON3[8]. */
	clk = clk_register_fixed_factor(NULL, "pclk_wdt", "pclk_alive", 0, 1, 1);
	if (IS_ERR(clk))
		pr_warn("%s: could not register clock pclk_wdt: %ld\n",
			__func__, PTR_ERR(clk));
	else
		rockchip_clk_add_lookup(ctx, clk, PCLK_WDT);

	rockchip_clk_register_plls(ctx, rk3399_pll_clks,
				   ARRAY_SIZE(rk3399_pll_clks), -1); //rk3399_pll_clks注册

	rockchip_clk_register_branches(ctx, rk3399_clk_branches,
				  ARRAY_SIZE(rk3399_clk_branches));//rk3399_clk_branches注册

	rockchip_clk_protect_critical(rk3399_cru_critical_clocks,
				      ARRAY_SIZE(rk3399_cru_critical_clocks));//rk3399_cru_critical_clocks注册

	rockchip_clk_register_armclk(ctx, ARMCLKL, "armclkl",
			mux_armclkl_p, ARRAY_SIZE(mux_armclkl_p),
			&rk3399_cpuclkl_data, rk3399_cpuclkl_rates,
			ARRAY_SIZE(rk3399_cpuclkl_rates));

	rockchip_clk_register_armclk(ctx, ARMCLKB, "armclkb",
			mux_armclkb_p, ARRAY_SIZE(mux_armclkb_p),
			&rk3399_cpuclkb_data, rk3399_cpuclkb_rates,
			ARRAY_SIZE(rk3399_cpuclkb_rates));

	rockchip_register_softrst(np, 21, reg_base + RK3399_SOFTRST_CON(0),
				  ROCKCHIP_SOFTRST_HIWORD_MASK);

	rockchip_register_restart_notifier(ctx, RK3399_GLB_SRST_FST, NULL);

	rockchip_clk_of_add_provider(np, ctx);
}
CLK_OF_DECLARE(rk3399_cru, "rockchip,rk3399-cru", rk3399_clk_init);

```

## **5.6** rockchip_clk_of_add_provider分析

```c
F:\Rock4Plus_Android10\kernel\drivers\clk\rockchip\clk.c
void __init rockchip_clk_of_add_provider(struct device_node *np,
				struct rockchip_clk_provider *ctx)
{
	if (of_clk_add_provider(np, of_clk_src_onecell_get,
				&ctx->clk_data))
		pr_err("%s: could not register clk provider\n", __func__);
}
F:\Rock4Plus_Android10\kernel\drivers\clk\clk.c
struct clk *of_clk_src_onecell_get(struct of_phandle_args *clkspec, void *data)
{
	struct clk_onecell_data *clk_data = data;
	unsigned int idx = clkspec->args[0];
	//F:\Rock4Plus_Android10\kernel\include\dt-bindings\clock\rk3399-cru.h
	//#define SCLK_I2C0_PMU			9
    //#define PCLK_I2C0_PMU			27
    if(idx == 9 || idx == 27) {
		printk("of_clk_src_onecell_get clock index %u\n", idx);
		dump_stack();
	}
	if (idx >= clk_data->clk_num) {
		pr_err("%s: invalid clock index %u\n", __func__, idx);
		return ERR_PTR(-EINVAL);
	}

	return clk_data->clks[idx];
}
EXPORT_SYMBOL_GPL(of_clk_src_onecell_get);
```

## **5.7** of_clk_add_provider分析

```c
F:\Rock4Plus_Android10\kernel\drivers\clk\clk.c

/**
 * of_clk_add_provider() - Register a clock provider for a node
 * @np: Device node pointer associated with clock provider
 * @clk_src_get: callback for decoding clock
 * @data: context pointer for @clk_src_get callback.
 */
int of_clk_add_provider(struct device_node *np,
			struct clk *(*clk_src_get)(struct of_phandle_args *clkspec,
						   void *data),
			void *data)
{
	struct of_clk_provider *cp;
	int ret;

	cp = kzalloc(sizeof(*cp), GFP_KERNEL);//创建of_clk_provider对象
	if (!cp)
		return -ENOMEM;

	cp->node = of_node_get(np);
	cp->data = data;//初始化data，即rk3399所有clk
	cp->get = clk_src_get;//clk_src_get即of_clk_src_onecell_get

	mutex_lock(&of_clk_mutex);
	list_add(&cp->link, &of_clk_providers);//加入of_clk_providers列表
	mutex_unlock(&of_clk_mutex);
	pr_debug("Added clock from %pOF\n", np);

	ret = of_clk_set_defaults(np, true);//默认clk设置
	if (ret < 0)
		of_clk_del_provider(np);

	return ret;
}
EXPORT_SYMBOL_GPL(of_clk_add_provider);
```

## **5.8** of_clk_set_defaults分析

```c
/**
 * of_clk_set_defaults() - parse and set assigned clocks configuration
 * @node: device node to apply clock settings for
 * @clk_supplier: true if clocks supplied by @node should also be considered
 *
 * This function parses 'assigned-{clocks/clock-parents/clock-rates}' properties
 * and sets any specified clock parents and rates. The @clk_supplier argument
 * should be set to true if @node may be also a clock supplier of any clock
 * listed in its 'assigned-clocks' or 'assigned-clock-parents' properties.
 * If @clk_supplier is false the function exits returning 0 as soon as it
 * determines the @node is also a supplier of any of the clocks.
 */
int of_clk_set_defaults(struct device_node *node, bool clk_supplier)
{
	int rc;

	if (!node)
		return 0;

	rc = __set_clk_parents(node, clk_supplier);
	if (rc < 0)
		return rc;

	return __set_clk_rates(node, clk_supplier);
}
EXPORT_SYMBOL_GPL(of_clk_set_defaults);
//解析“assigned-{clocks/clock-parents/clock-rates}”属性,并设置任何指定的时钟父级和速率。
```

# **6.**  RK3399 User Driver(i2c0)使用Clk分析

## 1,获取 CLK 指针

```c
F:\Rock4Plus_Android10\kernel\drivers\i2c\busses\i2c-rk3399.c
static int rk3x_i2c_probe(struct platform_device *pdev)
{
    ......
	i2c->clk = devm_clk_get(&pdev->dev, "i2c");
	i2c->pclk = devm_clk_get(&pdev->dev, "pclk");
    ......
}
[2021/8/23 14:22:36] [    1.081112] Call trace:
[2021/8/23 14:22:36] [    1.081127]  dump_backtrace+0x0/0x178
[2021/8/23 14:22:36] [    1.081138]  show_stack+0x14/0x20
[2021/8/23 14:22:36] [    1.081149]  dump_stack+0x94/0xb4
[2021/8/23 14:22:36] [    1.081161]  of_clk_src_onecell_get+0x54/0x80
[2021/8/23 14:22:36] [    1.081175]  __of_clk_get_from_provider.part.31+0x118/0x138
[2021/8/23 14:22:36] [    1.081186]  __of_clk_get_from_provider+0x1c/0x28
[2021/8/23 14:22:36] [    1.081197]  __of_clk_get_by_name+0xa8/0x148
[2021/8/23 14:22:36] [    1.081207]  clk_get+0x30/0x80
[2021/8/23 14:22:36] [    1.081220]  devm_clk_get+0x48/0xa0
[2021/8/23 14:22:36] [    1.081234]  rk3x_i2c_probe+0x164/0x1e0
[2021/8/23 14:22:36] [    1.081247]  platform_drv_probe+0x50/0xa8
[2021/8/23 14:22:36] [    1.081259]  really_probe+0x200/0x2b0
[2021/8/23 14:22:36] [    1.081271]  driver_probe_device+0x58/0x100
[2021/8/23 14:22:36] [    1.081283]  device_driver_attach+0x6c/0x78
[2021/8/23 14:22:36] [    1.081294]  __driver_attach+0xb0/0xf0
[2021/8/23 14:22:36] [    1.081306]  bus_for_each_dev+0x68/0xc8
[2021/8/23 14:22:36] [    1.081318]  driver_attach+0x20/0x28
[2021/8/23 14:22:36] [    1.081327]  bus_add_driver+0xf8/0x1f0
[2021/8/23 14:22:36] [    1.081339]  driver_register+0x60/0x110
[2021/8/23 14:22:36] [    1.081352]  __platform_driver_register+0x40/0x48
[2021/8/23 14:22:36] [    1.081365]  rk3x_i2c_driver_init+0x18/0x20
[2021/8/23 14:22:36] [    1.081378]  do_one_initcall+0x48/0x240
[2021/8/23 14:22:36] [    1.081391]  kernel_init_freeable+0x210/0x37c
[2021/8/23 14:22:36] [    1.081404]  kernel_init+0x10/0x108
[2021/8/23 14:22:36] [    1.081416]  ret_from_fork+0x10/0x18
[2021/8/23 14:22:36] [    1.081444] __of_clk_get_from_provider dev_id(ff3c0000.i2c) con_id(pclk) idx(27) 
[2021/8/23 14:22:36] [    1.081458] of_clk_src_onecell_get clock index 27
```

## 2,clk_enable

```c
[2021/8/23 9:50:17] [    1.389825] Call trace:
[2021/8/23 9:50:17] [    1.397107]  dump_stack+0x94/0xb4
[2021/8/23 9:50:17] [    1.397117]  clk_gate_endisable+0x78/0xc8
[2021/8/23 9:50:17] [    1.397136]  clk_gate_enable+0x10/0x20
[2021/8/23 9:50:17] [    1.397148]  clk_core_enable+0x78/0x280
[2021/8/23 9:50:17] [    1.397163]  clk_core_enable_lock+0x20/0x40
[2021/8/23 9:50:17] [    1.397176]  clk_enable+0x14/0x28
[2021/8/23 9:50:17] [    1.397189]  rk3x_i2c_xfer+0x64/0x460
```

