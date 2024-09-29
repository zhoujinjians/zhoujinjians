---
title: Linux学习系列3：pinctrl子系统
cover: https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.43.jpg
categories:
 - Linux
tags:
 - Linux
toc: true
abbrlink: 20220815
date: 2022-08-15 00:00:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Kernel-4.15**==）&&（==**文章基于 Android 10.0**==）

[【me.zhoujinjian.com博客原图链接】](https://github.com/zhoujinjiana) 

[【开发板 RockPi4bPlusV1.6】](https://shop.allnetchina.cn/collections/frontpage/products/rock-pi-4-model-b-board-only-2-4-5ghz-wlan-bluetooth-5-0)

[【开发板 RockPi4bPlusV1.6 Android 10.0 && Linux（Kernel 4.15）源码链接】](https://github.com/radxa/manifests/tree/rockchip-android-10)



正是由于前人（各位大神）的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

 [【嵌入式Linux全新系列教程之驱动大全(基于IMX6ULL开发板)】](https://www.100ask.net/detail/p_5ff2c46ce4b0c4f2bc4fa16d/8)    

# 1. Pinctrl作用

![image-20210430121123225](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/06_pinctrl_hardware_block.png)

Pinctrl：Pin Controller，顾名思义，就是用来控制引脚的：

* 引脚枚举与命名(Enumerating and naming)
* 引脚复用(Multiplexing)：比如用作GPIO、I2C或其他功能
* 引脚配置(Configuration)：比如上拉、下来、open drain、驱动强度等



Pinctrl驱动由芯片厂家的BSP工程师提供，一般的驱动工程师只需要在设备树里：

* 指明使用那些引脚
* 复用为哪些功能
* 配置为哪些状态

在一般的设备驱动程序里，甚至可以没有pinctrl的代码。

对于一般的驱动工程师，只需要知道“怎么使用pinctrl”即可。

# 2. Pinctrl子系统使用示例

* 查看原理图确定使用哪些引脚：比如pinA、pinB
* 生成pincontroller设备树信息
  * 选择功能：比如把pinA配置为I2C_SCL、把pinB配置为I2C_SDA
  * 配置：比如把pinA、pinB配置为open drain
* 使用pincontroller设备树信息：比如在i2c节点里定义"pinctrl-names"、"pinctrl-0"

## 2.1 pinctrl信息

```shell
	pinctrl: pinctrl {
		compatible = "rockchip,rk3399-pinctrl";
		rockchip,grf = <&grf>;
		rockchip,pmu = <&pmugrf>;
		#address-cells = <2>;
		#size-cells = <2>;
		ranges;
        ......
        i2c0 {
			i2c0_xfer: i2c0-xfer {
				rockchip,pins =
					<1 RK_PB7 RK_FUNC_2 &pcfg_pull_none>,
					<1 RK_PC0 RK_FUNC_2 &pcfg_pull_none>;
			};
		};
		......
    }
```

## 2.2. 在client节点使用pinctrl

```shell
	i2c0: i2c@ff3c0000 {
		compatible = "rockchip,rk3399-i2c";
		reg = <0x0 0xff3c0000 0x0 0x1000>;
		assigned-clocks = <&pmucru SCLK_I2C0_PMU>;
		assigned-clock-rates = <200000000>;
		clocks = <&pmucru SCLK_I2C0_PMU>, <&pmucru PCLK_I2C0_PMU>;
		clock-names = "i2c", "pclk";
		interrupts = <GIC_SPI 57 IRQ_TYPE_LEVEL_HIGH 0>;
		pinctrl-names = "default";
		pinctrl-0 = <&i2c0_xfer>;
		#address-cells = <1>;
		#size-cells = <0>;
		status = "disabled";
	};
```



## 2.3. 使用过程

这是透明的，我们的驱动基本不用管。当设备切换状态时，对应的pinctrl就会被调用。

比如在platform_device和platform_driver的枚举过程中，流程如下：

```c
[2021/8/30 14:49:52] [    1.021595] enable function i2c0 group i2c0-xfer
[2021/8/30 14:49:52] [    1.021615] CPU: 2 PID: 1 Comm: swapper/0 Not tainted 4.19.111 #20
[2021/8/30 14:49:52] [    1.021630] Hardware name: ROCKPI 4B (DT)
[2021/8/30 14:49:52] [    1.021639] Call trace:
[2021/8/30 14:49:52] [    1.021654]  dump_backtrace+0x0/0x178
[2021/8/30 14:49:52] [    1.021669]  show_stack+0x14/0x20
[2021/8/30 14:49:52] [    1.021684]  dump_stack+0x94/0xb4
[2021/8/30 14:49:52] [    1.021702]  rockchip_pmx_set+0x188/0x198
[2021/8/30 14:49:52] [    1.021721]  pinmux_enable_setting+0x1a8/0x270
[2021/8/30 14:49:52] [    1.021737]  pinctrl_commit_state+0xe4/0x168
[2021/8/30 14:49:52] [    1.021750]  pinctrl_select_state+0x18/0x28
[2021/8/30 14:49:52] [    1.021767]  pinctrl_bind_pins+0x13c/0x150
[2021/8/30 14:49:52] [    1.021782]  really_probe+0x6c/0x2b0
[2021/8/30 14:49:52] [    1.021798]  driver_probe_device+0x58/0x100
[2021/8/30 14:49:52] [    1.021813]  device_driver_attach+0x6c/0x78
[2021/8/30 14:49:52] [    1.021828]  __driver_attach+0xb0/0xf0
[2021/8/30 14:49:52] [    1.021843]  bus_for_each_dev+0x68/0xc8
[2021/8/30 14:49:52] [    1.021858]  driver_attach+0x20/0x28
[2021/8/30 14:49:52] [    1.021872]  bus_add_driver+0xf8/0x1f0
[2021/8/30 14:49:52] [    1.021884]  driver_register+0x60/0x110
[2021/8/30 14:49:52] [    1.021895]  __platform_driver_register+0x40/0x48
[2021/8/30 14:49:52] [    1.021912]  rk3x_i2c_driver_init+0x18/0x20
[2021/8/30 14:49:52] [    1.021928]  do_one_initcall+0x48/0x240
[2021/8/30 14:49:52] [    1.021944]  kernel_init_freeable+0x210/0x37c
[2021/8/30 14:49:52] [    1.021960]  kernel_init+0x10/0x108
[2021/8/30 14:49:52] [    1.021975]  ret_from_fork+0x10/0x18
```

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/09_pinctrl_really_probe.png)

# 3.Pinctrl子系统主要数据结构

## 3.1. 设备树

### 3.1.1 理想模型

![image-20210506093602409](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/12_pinctrl_dts_modules.png)

### 3.1.2 实际的例子

* RK3399

  ![image-20210831103556839](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/image-20210831103556839.png)


## 3.2. PinController的数据结构

记住pinctrl的三大作用，有助于理解所涉及的数据结构：

* 引脚枚举与命名(Enumerating and naming)
* 引脚复用(Multiplexing)：比如用作GPIO、I2C或其他功能
* 引脚配置(Configuration)：比如上拉、下来、open drain、驱动强度等

### 3.2.1 pinctrl_desc和pinctrl_dev

#### 1. 结构体引入

pincontroller虽然是一个软件的概念，但是它背后是有硬件支持的，所以可以使用一个结构体来表示它：pinctrl_dev。

怎么构造出pinctrl_dev？我们只需要描述它：提供一个pinctrl_desc，然后调用pinctrl_register就可以：

```c
struct pinctrl_dev *pinctrl_register(struct pinctrl_desc *pctldesc,
				    struct device *dev, void *driver_data);
```



怎么使用pinctrl_desc、pinctrl_dev来描述一个pin controller？这两个结构体定义如下：

![image-20210505153958014](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/16_pinctrl_desc_and_pinctrl_dev.png)

pinctrl_desc示例如下：
![image-20210831105701675](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/image-20210831105701675.png)


#### 2. 作用1：描述、获得引脚

使用结构体pinctrl_pin_desc来描述一个引脚，一个pin controller有多个引脚：

![image-20210419142714778](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/02_pinctrl_pin_desc.png)



使用pinctrl_ops来操作引脚，主要功能有二：

* 来取出某组的引脚：get_groups_count、get_group_pins
* 处理设备树中pin controller中的某个节点：dt_node_to_map，把device_node转换为一系列的pinctrl_map

![image-20210505155008286](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/03_pinctrl_ops.png)

#### 3. 作用2：引脚复用

![image-20210505161910767](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/04_pinmux_ops.png)

#### 4. 作用3：引脚配置

![](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/05_pinconf_ops.png)

#### 5. 使用pinctrl_desc注册得到pinctrl_dev

调用devm_pinctrl_register或pinctrl_register，就可以根据pinctrl_desc构造出pinctrl_dev，并且把pinctrl_dev放入链表：

```c
devm_pinctrl_register
    pinctrl_register
        pinctrl_init_controller
            struct pinctrl_dev *pctldev;
            pctldev = kzalloc(sizeof(*pctldev), GFP_KERNEL);

            pctldev->owner = pctldesc->owner;
            pctldev->desc = pctldesc;
            pctldev->driver_data = driver_data;

            /* check core ops for sanity */
            ret = pinctrl_check_ops(pctldev);

            /* If we're implementing pinmuxing, check the ops for sanity */
            ret = pinmux_check_ops(pctldev);

            /* If we're implementing pinconfig, check the ops for sanity */
            ret = pinconf_check_ops(pctldev);

            /* Register all the pins */
            ret = pinctrl_register_pins(pctldev, pctldesc->pins, pctldesc->npins);

            list_add_tail(&pctldev->node, &pinctrldev_list);
```



## 3.3 client的数据结构

在设备树中，使用pinctrl时格式如下：

```shell
/* For a client device requiring named states */
device {
    pinctrl-names = "active", "idle";
    pinctrl-0 = <&state_0_node_a>;
    pinctrl-1 = <&state_1_node_a &state_1_node_b>;
};
```

设备节点要么被转换为platform_device，或者其他结构体(比如i2c_client)，但是里面都会有一个device结构体，比如：

![image-20210505171819747](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/17_pinctrl_in_device.png)



### 3.3.1 dev_pin_info

每个device结构体里都有一个dev_pin_info结构体，用来保存设备的pinctrl信息：

![image-20210505173004090](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/18_dev_pin_info.png)

### 3.3.2 pinctrl

假设芯片上有多个pin controller，那么这个设备使用哪个pin controller？

这需要通过设备树来确定：

* 分析设备树，找到pin controller
* 对于每个状态，比如default、init，去分析pin controller中的设备树节点
  * 使用pin controller的pinctrl_ops.dt_node_to_map来处理设备树的pinctrl节点信息，得到一系列的pinctrl_map
  * 这些pinctrl_map放在pinctrl.dt_maps链表中
  * 每个pinctrl_map都被转换为pinctrl_setting，放在对应的pinctrl_state.settings链表中

![image-20210505182828324](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/19_pinctrl_maps.png)



### 3.3.3 pinctrl_map和pinctrl_setting

设备引用pin controller中的某个节点时，这个节点会被转换为一系列的pinctrl_map：

* 转换为多少个pinctrl_map，完全由具体的驱动决定
* 每个pinctrl_map，又被转换为一个pinctrl_setting
* 举例，设备节点里有：`pinctrl-0 = <&state_0_node_a>`
  * pinctrl-0对应一个状态，会得到一个pinctrl_state
  * state_0_node_a节点被解析为一系列的pinctrl_map
  * 这一系列的pinctrl_map被转换为一系列的pinctrl_setting
  * 这些pinctrl_setting被放入pinctrl_state的settings链表

![image-20210831104152437](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/image-20210831104152437.png)

3.4. 使用pinctrl_setting

调用过程：

```shell
really_probe
	pinctrl_bind_pins
		pinctrl_select_state
			/* Apply all the settings for the new state */
			list_for_each_entry(setting, &state->settings, node) {
				switch (setting->type) {
				case PIN_MAP_TYPE_MUX_GROUP:
					ret = pinmux_enable_setting(setting);
							ret = ops->set_mux(...);
				break;
				case PIN_MAP_TYPE_CONFIGS_PIN:
				case PIN_MAP_TYPE_CONFIGS_GROUP:
					ret = pinconf_apply_setting(setting);
							ret = ops->pin_config_group_set(...);
					break;
				default:
					ret = -EINVAL;
				break;
			}		
```

```c
really_probe
-->pinctrl_bind_pins
---->devm_pinctrl_get
------>pinctrl_get
-------->create_pinctrl(INIT_LIST_HEAD(&p->states))
                       (INIT_LIST_HEAD(&p->dt_maps))
---------->pinctrl_dt_to_map
               ->dt_to_map_one_config
                  ->rockchip_dt_node_to_map
---------->add_setting
------>devres_add
---->pinctrl_select_state
------>pinmux_enable_setting
-------->ops->set_mux
                ->rockchip_pmx_set
------>pinconf_apply_setting
-------->ops->pin_config_group_set
```

![image-20210831104315779](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/image-20210831104315779.png)

# 4.PinController构造过程情景分析

## 4.1. 设备树

![image-20210831103556839](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/image-20210831103556839.png)

## 4.2. 驱动代码执行流程

驱动程序位置：

```shell
F:\Rock4Plus_Android10\kernel\drivers\pinctrl\pinctrl-rockchip.c
```

调用过程：

```c
rockchip_pinctrl_probe
	rockchip_pinctrl_register(pdev, info);
        struct pinctrl_desc *ctrldesc = &info->pctl;
        struct pinctrl_pin_desc *pindesc, *pdesc;
        struct rockchip_pin_bank *pin_bank;
        int pin, bank, ret;
        int k;

        ctrldesc->name = "rockchip-pinctrl";
        ctrldesc->owner = THIS_MODULE;
        ctrldesc->pctlops = &rockchip_pctrl_ops;
        ctrldesc->pmxops = &rockchip_pmx_ops;
        ctrldesc->confops = &rockchip_pinconf_ops;

        pindesc = devm_kcalloc(&pdev->dev,
                       info->ctrl->nr_pins, sizeof(*pindesc),
                       GFP_KERNEL);
        ctrldesc->pins = pindesc;
        ctrldesc->npins = info->ctrl->nr_pins;

        pdesc = pindesc;
        for (bank = 0, k = 0; bank < info->ctrl->nr_banks; bank++) {
            pin_bank = &info->ctrl->pin_banks[bank];
            for (pin = 0; pin < pin_bank->nr_pins; pin++, k++) {
                pdesc->number = k;
                pdesc->name = kasprintf(GFP_KERNEL, "%s-%d",
                            pin_bank->name, pin);
                pdesc++;
            }
        }

        ret = rockchip_pinctrl_parse_dt(pdev, info);

        info->pctl_dev = devm_pinctrl_register(&pdev->dev, ctrldesc, info);
```



## 4.3. 作用1：描述、获得引脚：解析设备树

#### 3.1 单个引脚

```c
F:\Rock4Plus_Android10\kernel\drivers\pinctrl\pinctrl-rockchip.c
ctrldesc->pins = pindesc;
ctrldesc->npins = info->ctrl->nr_pins;
```



可以在开发板上查看：

```shell
rk3399_Android10:/sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl $ cat pins
registered pins: 160
pin 0 (gpio0-0)
......
pin 31 (gpio0-31)

pin 32 (gpio1-0)
......
pin 63 (gpio1-31)

pin 64 (gpio2-0)
......
pin 95 (gpio2-31)

pin 96 (gpio3-0)
......
pin 127 (gpio3-31)

pin 128 (gpio4-0)
......
pin 159 (gpio4-31)
```

#### 3.2 某组引脚

```c
F:\Rock4Plus_Android10\kernel\drivers\pinctrl\pinctrl-rockchip.c
static const struct pinctrl_ops rockchip_pctrl_ops = {
	.get_groups_count	= rockchip_get_groups_count,
	.get_group_name		= rockchip_get_group_name,
	.get_group_pins		= rockchip_get_group_pins,
	.dt_node_to_map		= rockchip_dt_node_to_map,
	.dt_free_map		= rockchip_dt_free_map,
};
```

某组引脚中，有哪些引脚？这要分析设备树：imx_pinctrl_probe_dt。

```shell
rk3399_Android10:/sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl $ cat pingroups
registered pin groups:
group: clk-32k
pin 0 (gpio0-0)
......
group: rgmii-pins
pin 113 (gpio3-17)
pin 110 (gpio3-14)
pin 109 (gpio3-13)
pin 108 (gpio3-12)
pin 107 (gpio3-11)
pin 105 (gpio3-9)
pin 104 (gpio3-8)
pin 103 (gpio3-7)
pin 102 (gpio3-6)
pin 101 (gpio3-5)
pin 100 (gpio3-4)
pin 99 (gpio3-3)
pin 98 (gpio3-2)
pin 97 (gpio3-1)
pin 96 (gpio3-0)
......
group: i2c0-xfer
pin 47 (gpio1-15)
pin 48 (gpio1-16)
......
group: uart4-xfer
pin 39 (gpio1-7)
pin 40 (gpio1-8)
......
```



#### 3.3 设备树解析情景分析

分析：rockchip_pinctrl_parse_dt

```c
F:\Rock4Plus_Android10\kernel\drivers\pinctrl\pinctrl-rockchip.c
static int rockchip_pinctrl_parse_dt(struct platform_device *pdev,
					      struct rockchip_pinctrl *info)
{
	struct device *dev = &pdev->dev;
	struct device_node *np = dev->of_node;
	struct device_node *child;
	int ret;
	int i;

	rockchip_pinctrl_child_count(info, np);

	dev_dbg(&pdev->dev, "nfunctions = %d\n", info->nfunctions);
	dev_dbg(&pdev->dev, "ngroups = %d\n", info->ngroups);

	info->functions = devm_kcalloc(dev,
					      info->nfunctions,
					      sizeof(struct rockchip_pmx_func),
					      GFP_KERNEL);
	......
	info->groups = devm_kcalloc(dev,
					    info->ngroups,
					    sizeof(struct rockchip_pin_group),
					    GFP_KERNEL);
	......
	i = 0;
	for_each_child_of_node(np, child) {
		......
		ret = rockchip_pinctrl_parse_functions(child, info, i++);
		.......
	}

	return 0;
}


static int rockchip_pinctrl_parse_functions(struct device_node *np,
						struct rockchip_pinctrl *info,
						u32 index)
{
	struct device_node *child;
	struct rockchip_pmx_func *func;
	struct rockchip_pin_group *grp;
	int ret;
	static u32 grp_index;
	u32 i = 0;
	dev_dbg(info->dev, "parse function(%d): %s\n", index, np->name);
	func = &info->functions[index];
	/* Initialise function */
	func->name = np->name;
	func->ngroups = of_get_child_count(np);
	......
	func->groups = devm_kcalloc(info->dev,
			func->ngroups, sizeof(char *), GFP_KERNEL);
	......
	for_each_child_of_node(np, child) {
		func->groups[i] = child->name;
		grp = &info->groups[grp_index++];
		ret = rockchip_pinctrl_parse_groups(child, grp, info, i++);
		......
	}

	return 0;
}
static int rockchip_pinctrl_parse_groups(struct device_node *np,
					      struct rockchip_pin_group *grp,
					      struct rockchip_pinctrl *info,
					      u32 index)
{
	struct rockchip_pin_bank *bank;
	int size;
	const __be32 *list;
	int num;
	int i, j;
	int ret;
	dev_dbg(info->dev, "group(%d): %s\n", index, np->name);
	/* Initialise group */
	grp->name = np->name;
	list = of_get_property(np, "rockchip,pins", &size);
	size /= sizeof(*list);
	......
	grp->npins = size / 4;

	grp->pins = devm_kcalloc(info->dev, grp->npins, sizeof(unsigned int),
						GFP_KERNEL);
	grp->data = devm_kcalloc(info->dev,
					grp->npins,
					sizeof(struct rockchip_pin_config),
					GFP_KERNEL);
	......
	for (i = 0, j = 0; i < size; i += 4, j++) {
		const __be32 *phandle;
		struct device_node *np_config;

		num = be32_to_cpu(*list++);
		bank = bank_num_to_bank(info, num);
		......

		grp->pins[j] = bank->pin_base + be32_to_cpu(*list++);
		grp->data[j].func = be32_to_cpu(*list++);

		phandle = list++;
		......

		np_config = of_find_node_by_phandle(be32_to_cpup(phandle));
		ret = pinconf_generic_parse_dt_config(np_config, NULL,
				&grp->data[j].configs, &grp->data[j].nconfigs);
		......
	}

	return 0;
}

```



## 4.4. 作用2：引脚复用

## 4.5. 作用3：引脚配置

# 5.client端使用pinctrl过程的情景分析

## 5.1. 回顾client的数据结构

在设备树中，使用pinctrl时格式如下：

![image-20210831103556839](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/image-20210831103556839.png)


设备节点要么被转换为platform_device，或者其他结构体(比如i2c_client)，但是里面都会有一个device结构体，比如：

![image-20210505171819747](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/17_pinctrl_in_device-16303802580763.png)



### 1.1 dev_pin_info

每个device结构体里都有一个dev_pin_info结构体，用来保存设备的pinctrl信息：

![image-20210505173004090](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/18_dev_pin_info-16303802580764.png)

### 1.2 pinctrl

假设芯片上有多个pin controller，那么这个设备使用哪个pin controller？

这需要通过设备树来确定：

* 分析设备树，找到pin controller
* 对于每个状态，比如default、init，去分析pin controller中的设备树节点
  * 使用pin controller的pinctrl_ops.dt_node_to_map来处理设备树的pinctrl节点信息，得到一系列的pinctrl_map
  * 这些pinctrl_map放在pinctrl.dt_maps链表中
  * 每个pinctrl_map都被转换为pinctrl_setting，放在对应的pinctrl_state.settings链表中

![image-20210505182828324](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/19_pinctrl_maps-16303802580765.png)



### 1.3 pinctrl_map和pinctrl_setting

设备引用pin controller中的某个节点时，这个节点会被转换为一些列的pinctrl_map：

* 转换为多少个pinctrl_map，完全由具体的驱动决定
* 每个pinctrl_map，又被转换为一个pinctrl_setting
* 举例，设备节点里有：`pinctrl-0 = <&state_0_node_a>`
  * pinctrl-0对应一个状态，会得到一个pinctrl_state
  * state_0_node_a节点被解析为一系列的pinctrl_map
  * 这一系列的pinctrl_map被转换为一系列的pinctrl_setting
  * 这些pinctrl_setting被放入pinctrl_state的settings链表

![image-20210831104152437](https://raw.githubusercontent.com/zhoujinjiana/zhoujinjian.com.images/master/kernel.pinctrl/image-20210831104152437.png)



## 5.2. client节点的pinctrl构造过程

#### 2.1 函数调用

```shell
really_probe
	pinctrl_bind_pins
		dev->pins = devm_kzalloc(dev, sizeof(*(dev->pins)), GFP_KERNEL);
		
		dev->pins->p = devm_pinctrl_get(dev);
							pinctrl_get
								create_pinctrl(dev);
									ret = pinctrl_dt_to_map(p);
									
                                    for_each_maps(maps_node, i, map) {
	                                    ret = add_setting(p, map);
                                    }
		
		dev->pins->default_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_DEFAULT);			
```



#### 2.2 情景分析

##### 1. 设备树转换为pinctrl_map

```c
really_probe
-->pinctrl_bind_pins(dev)
---->devm_pinctrl_get
------>pinctrl_get
-------->create_pinctrl(INIT_LIST_HEAD(&p->states))
                       (INIT_LIST_HEAD(&p->dt_maps))
---------->pinctrl_dt_to_map
               ->dt_to_map_one_config
                  ->rockchip_dt_node_to_map
---------->add_setting
------>devres_add

F:\Rock4Plus_Android10\kernel\drivers\pinctrl\core.c
    static struct pinctrl *create_pinctrl(struct device *dev,
				      struct pinctrl_dev *pctldev)
{
	struct pinctrl *p;
	const char *devname;
	struct pinctrl_maps *maps_node;
	int i;
	const struct pinctrl_map *map;
	int ret;
	/*
	 * create the state cookie holder struct pinctrl for each
	 * mapping, this is what consumers will get when requesting
	 * a pin control handle with pinctrl_get()
	 */
	p = kzalloc(sizeof(*p), GFP_KERNEL);
     ......
	p->dev = dev;
	INIT_LIST_HEAD(&p->states);
	INIT_LIST_HEAD(&p->dt_maps);

	ret = pinctrl_dt_to_map(p, pctldev);
    ......
	devname = dev_name(dev);

	mutex_lock(&pinctrl_maps_mutex);
	/* Iterate over the pin control maps to locate the right ones */
	for_each_maps(maps_node, i, map) {
        ......
		ret = add_setting(p, pctldev, map);
        ......
	}
	mutex_unlock(&pinctrl_maps_mutex);
    ......
	kref_init(&p->users);
	/* Add the pinctrl handle to the global list */
	mutex_lock(&pinctrl_list_mutex);
	list_add_tail(&p->node, &pinctrl_list);
	mutex_unlock(&pinctrl_list_mutex);
	return p;
}
F:\Rock4Plus_Android10\kernel\drivers\pinctrl\devicetree.c
    int pinctrl_dt_to_map(struct pinctrl *p, struct pinctrl_dev *pctldev)
{
	struct device_node *np = p->dev->of_node;
	int state, ret;
	char *propname;
	struct property *prop;
	const char *statename;
	const __be32 *list;
	int size, config;
	phandle phandle;
	struct device_node *np_config;
	......
	/* We may store pointers to property names within the node */
	of_node_get(np);
	/* For each defined state ID */
	for (state = 0; ; state++) {
		/* Retrieve the pinctrl-* property */
		propname = kasprintf(GFP_KERNEL, "pinctrl-%d", state);
		prop = of_find_property(np, propname, &size);
		.....
		/* For every referenced pin configuration node in it */
		for (config = 0; config < size; config++) {
			phandle = be32_to_cpup(list++);
			/* Look up the pin configuration node */
			np_config = of_find_node_by_phandle(phandle);
	        ......
			/* Parse the node */
			ret = dt_to_map_one_config(p, pctldev, statename,
						   np_config);
			of_node_put(np_config);
	        ......
		}
	        ......
	}

	return 0;

err:
	pinctrl_dt_free_maps(p);
	return ret;
}

dt_to_map_one_config
    ->rockchip_dt_node_to_map
```

##### 2. pinctrl_map转换为pinctrl_setting

```c
really_probe
-->pinctrl_bind_pins(dev)
---->devm_pinctrl_get
------>pinctrl_get
-------->create_pinctrl(INIT_LIST_HEAD(&p->states))
                       (INIT_LIST_HEAD(&p->dt_maps))
---------->pinctrl_dt_to_map
               ->dt_to_map_one_config
                  ->rockchip_dt_node_to_map
---------->add_setting
------>devres_add
F:\Rock4Plus_Android10\kernel\drivers\pinctrl\core.c
    static int add_setting(struct pinctrl *p, struct pinctrl_dev *pctldev,
		       const struct pinctrl_map *map)
{
	struct pinctrl_state *state;
	struct pinctrl_setting *setting;
	int ret;

	state = find_state(p, map->name);
	......
	setting = kzalloc(sizeof(*setting), GFP_KERNEL);
	......
	setting->type = map->type;

	if (pctldev)
		setting->pctldev = pctldev;
	else
		setting->pctldev =
			get_pinctrl_dev_from_devname(map->ctrl_dev_name);
    ......
	setting->dev_name = map->dev_name;

	switch (map->type) {
	case PIN_MAP_TYPE_MUX_GROUP:
		ret = pinmux_map_to_setting(map, setting);
		break;
	case PIN_MAP_TYPE_CONFIGS_PIN:
	case PIN_MAP_TYPE_CONFIGS_GROUP:
		ret = pinconf_map_to_setting(map, setting);
		break;
	......
	}
	......

	list_add_tail(&setting->node, &state->settings);

	return 0;
}
```



## 5.3. 切换state情景分析

#### 3.1 函数调用过程

涉及pinctrl子系统的其他2个作用：引脚复用、引脚配置

```shell
really_probe
	pinctrl_bind_pins
		pinctrl_select_state
			/* Apply all the settings for the new state */
			list_for_each_entry(setting, &state->settings, node) {
				switch (setting->type) {
				case PIN_MAP_TYPE_MUX_GROUP:
					ret = pinmux_enable_setting(setting);
							ret = ops->set_mux(...);
				break;
				case PIN_MAP_TYPE_CONFIGS_PIN:
				case PIN_MAP_TYPE_CONFIGS_GROUP:
					ret = pinconf_apply_setting(setting);
							ret = ops->pin_config_group_set(...);
					break;
				default:
					ret = -EINVAL;
				break;
			}		
```



#### 3.2 情景分析

```shell
[2021/8/30 14:49:52] [    1.021595] enable function i2c0 group i2c0-xfer
[2021/8/30 14:49:52] [    1.021615] CPU: 2 PID: 1 Comm: swapper/0 Not tainted 4.19.111 #20
[2021/8/30 14:49:52] [    1.021630] Hardware name: ROCKPI 4B (DT)
[2021/8/30 14:49:52] [    1.021639] Call trace:
[2021/8/30 14:49:52] [    1.021654]  dump_backtrace+0x0/0x178
[2021/8/30 14:49:52] [    1.021669]  show_stack+0x14/0x20
[2021/8/30 14:49:52] [    1.021684]  dump_stack+0x94/0xb4
[2021/8/30 14:49:52] [    1.021702]  rockchip_pmx_set+0x188/0x198
[2021/8/30 14:49:52] [    1.021721]  pinmux_enable_setting+0x1a8/0x270
[2021/8/30 14:49:52] [    1.021737]  pinctrl_commit_state+0xe4/0x168
[2021/8/30 14:49:52] [    1.021750]  pinctrl_select_state+0x18/0x28
[2021/8/30 14:49:52] [    1.021767]  pinctrl_bind_pins+0x13c/0x150
[2021/8/30 14:49:52] [    1.021782]  really_probe+0x6c/0x2b0
[2021/8/30 14:49:52] [    1.021798]  driver_probe_device+0x58/0x100
[2021/8/30 14:49:52] [    1.021813]  device_driver_attach+0x6c/0x78
[2021/8/30 14:49:52] [    1.021828]  __driver_attach+0xb0/0xf0
[2021/8/30 14:49:52] [    1.021843]  bus_for_each_dev+0x68/0xc8
[2021/8/30 14:49:52] [    1.021858]  driver_attach+0x20/0x28
[2021/8/30 14:49:52] [    1.021872]  bus_add_driver+0xf8/0x1f0
[2021/8/30 14:49:52] [    1.021884]  driver_register+0x60/0x110
[2021/8/30 14:49:52] [    1.021895]  __platform_driver_register+0x40/0x48
[2021/8/30 14:49:52] [    1.021912]  rk3x_i2c_driver_init+0x18/0x20
[2021/8/30 14:49:52] [    1.021928]  do_one_initcall+0x48/0x240
[2021/8/30 14:49:52] [    1.021944]  kernel_init_freeable+0x210/0x37c
[2021/8/30 14:49:52] [    1.021960]  kernel_init+0x10/0x108
[2021/8/30 14:49:52] [    1.021975]  ret_from_fork+0x10/0x18


[2021/8/30 14:50:19] [    0.763414] enable function gmac group rgmii-pins
[2021/8/30 14:50:19] [    0.763433] CPU: 4 PID: 1 Comm: swapper/0 Not tainted 4.19.111 #20
[2021/8/30 14:50:19] [    0.763445] Hardware name: ROCKPI 4B (DT)
[2021/8/30 14:50:19] [    0.763456] Call trace:
[2021/8/30 14:50:19] [    0.763480]  dump_backtrace+0x0/0x178
[2021/8/30 14:50:19] [    0.763492]  show_stack+0x14/0x20
[2021/8/30 14:50:19] [    0.763507]  dump_stack+0x94/0xb4
[2021/8/30 14:50:19] [    0.763522]  rockchip_pmx_set+0x188/0x198
[2021/8/30 14:50:19] [    0.763534]  pinmux_enable_setting+0x1a8/0x270
[2021/8/30 14:50:19] [    0.763544]  pinctrl_commit_state+0xe4/0x168
[2021/8/30 14:50:19] [    0.763554]  pinctrl_select_state+0x18/0x28
[2021/8/30 14:50:19] [    0.763567]  pinctrl_bind_pins+0x13c/0x150
[2021/8/30 14:50:19] [    0.763581]  really_probe+0x6c/0x2b0
[2021/8/30 14:50:19] [    0.763593]  driver_probe_device+0x58/0x100
[2021/8/30 14:50:19] [    0.763605]  device_driver_attach+0x6c/0x78
[2021/8/30 14:50:19] [    0.763618]  __driver_attach+0xb0/0xf0
[2021/8/30 14:50:19] [    0.763629]  bus_for_each_dev+0x68/0xc8
[2021/8/30 14:50:19] [    0.763641]  driver_attach+0x20/0x28
[2021/8/30 14:50:19] [    0.763653]  bus_add_driver+0xf8/0x1f0
[2021/8/30 14:50:19] [    0.763665]  driver_register+0x60/0x110
[2021/8/30 14:50:19] [    0.763679]  __platform_driver_register+0x40/0x48
[2021/8/30 14:50:19] [    0.763693]  rk_gmac_dwmac_driver_init+0x18/0x20
[2021/8/30 14:50:19] [    0.763706]  do_one_initcall+0x48/0x240
[2021/8/30 14:50:19] [    0.763721]  kernel_init_freeable+0x210/0x37c
[2021/8/30 14:50:19] [    0.763734]  kernel_init+0x10/0x108
[2021/8/30 14:50:19] [    0.763747]  ret_from_fork+0x10/0x18
```

