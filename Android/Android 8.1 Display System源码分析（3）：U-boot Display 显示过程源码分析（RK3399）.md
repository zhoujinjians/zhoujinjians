---
title: Android 8.1 Display System源码分析（3）：U-boot Display 显示过程源码分析（RK3399）
cover: https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/personal.website/post.cover.pictures.00012.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20200619
date: 2020-06-19 09:25:00
---

注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux @Rockchip版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-4.4**==）&&（==**文章基于 Android-8.1**==）
[【开发板 - Firefly-RK3399 7.85寸1024x768显示屏模组（MIPI接口）】](
http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/compile_android8.1_firmware.html#)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【[RK3399][Android7.1] Uboot display 加载过程小结】](
https://blog.csdn.net/kris_fei/article/details/79003925) 

[（2）【Connect to TS050 Touchscreen】](
https://docs.khadas.com/zh-cn/edge/ConnectLcd.html#3-%EF%BC%88MIPI-HDMI%EF%BC%89%E5%B1%8F%E5%B9%95%E9%85%8D%E7%BD%AE) 

[（3）【Firefly-RK3399 LCD使用】](
http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/driver_lcd.html) 

[（4）【RK3288——LCD裸机】](
https://hceng.cn/2018/07/19/RK3288%E2%80%94%E2%80%94LCD%E8%A3%B8%E6%9C%BA/) 

--------------------------------------------------------------------------------
==源码（部分）==

>U-boot

- I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\*
- I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\drm\*
- I:\RK3399_Android8.1_MIPI\u-boot\arch\arm\lib\board.c

--------------------------------------------------------------------------------

显示模块主要分 vop, dsi, panel三大模块，另加gpio, 背光的控制，另外还有logo的解析和加载。整个流程基本上就是解析各个模块参数，然后准备，使能各个模块。
![](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/zjj.sys.display.8.1.uboot/firefly-rk3399-mipi-dsi-framwork.jpg)

#### (一)、MIPI屏幕配置

#### （1）、LCD使用

Firefly-RK3399开发板外置了两个LCD屏接口，一个是EDP，一个是MIPI，接口对应板子上的位置如下图，我们这里是mipi：
![](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/zjj.sys.display.8.1.uboot/firefly-rk3399-lcd-edp-mipi.jpg)


##### 1.1、Config配置

以Android8.1配置MIPI屏为例，Firefly-RK3399默认的配置文件kernel/arch/arm64/configs/firefly_defconfig已经把LCD相关的配置设置好了，如果自己做了修改，请注意把以下配置加上：

>CONFIG_LCD_MIPI=y
>CONFIG_MIPI_DSI=y
>CONFIG_RK32_MIPI_DSI=y

##### 1.2、DTS配置

以rk3399-firefly-mipi.dt为例介绍：MIPI
###### 1.2.1、使能对应显示设备节点
``` c
I:\RK3399_Android8.1_MIPI\kernel\arch\arm64\boot\dts\rockchip\rk3399-firefly-mipi.dts

&dsi {
	status = "okay";
    ......
}
```

###### 1.2.2、绑定 VOP
``` c
I:\RK3399_Android8.1_MIPI\kernel\arch\arm64\boot\dts\rockchip\rk3399-firefly-mipi.dts

&dsi_in_vopl {
	status = "disabled";
};

&dsi_in_vopb {
	status = "okay";
};

&hdmi {
	status = "okay";
};

&hdmi_in_vopb {
	status = "disabled";
};
&hdmi_in_vopl {
	status = "okay";
};

&cdn_dp {
    status = "okay";
    extcon = <&fusb0>;
    phys = <&tcphy0_dp>;
};

&dp_in_vopb {
    status = "disabled";
};

```

###### 1.2.3、开机 logo

如果 uboot logo 未开启，那 kernel 阶段也无法显示开机 logo，只能等到 android 启动后才能看到显示。在 dts 里面将对应的 route 使能即可打开 uboot logo 支持，比如打开 hdmi 的uboot logo 显示:
``` c
I:\RK3399_Android8.1_MIPI\kernel\arch\arm64\boot\dts\rockchip\rk3399-firefly-mipi.dts

&route_hdmi {
	status = "disabled";
	logo,mode = "center";
};
&route_dsi {
	status = "okay";
	logo,mode = "center";
};
```

###### 1.2.4、绑定 PLL

rk3399 的 hdmi 所绑定的 vop 时钟需要挂载到 vpll 上，若是双显需将另一个 vop 时钟挂到cpll 这样可以分出任意 dclk 的频率。如当 hdmi 绑定到 vopb 时配置：

``` c
I:\RK3399_Android8.1_MIPI\kernel\arch\arm64\boot\dts\rockchip\rk3399-firefly-mipi.dts

&vopb {
	assigned-clocks = <&cru DCLK_VOP0_DIV>;
	assigned-clock-parents = <&cru PLL_CPLL>;
};

&vopl {
	assigned-clocks = <&cru DCLK_VOP1_DIV>;
	assigned-clock-parents = <&cru PLL_VPLL>;
};

```

###### 1.2.5、打开音频

``` c
I:\RK3399_Android8.1_MIPI\kernel\arch\arm64\boot\dts\rockchip\rk3399-firefly-mipi.dts

&hdmi_sound {
	status = "okay";
};
```
###### 1.2.6、配置Panel && timing

``` c
I:\RK3399_Android8.1_MIPI\kernel\arch\arm64\boot\dts\rockchip\rk3399-firefly-mipi.dts

&dsi {
	status = "okay";
	dsi_panel: panel {
		compatible ="simple-panel-dsi";
		reg = <0>;
		//ddc-i2c-bu
		//power-supply = <&vcc_lcd>;
		//pinctrl-0 = <&lcd_panel_reset &lcd_panel_enable>;
		backlight = <&backlight>;
		/*
		enable-gpios = <&gpio1 1 GPIO_ACTIVE_HIGH>;
		reset-gpios = <&gpio4 29 GPIO_ACTIVE_HIGH>;
		*/
		dsi,flags = <(MIPI_DSI_MODE_VIDEO | MIPI_DSI_MODE_VIDEO_BURST /*| MIPI_DSI_MODE_VIDEO_SYNC_PULSE*/)>;
		dsi,format = <MIPI_DSI_FMT_RGB888>;
		bus-format = <MEDIA_BUS_FMT_RGB666_1X18>;
		dsi,lanes = <4>;

        dsi,channel = <0>;

        enable-delay-ms = <35>;
        prepare-delay-ms = <6>;
		/*
        delay,power = <10>;
        delay,reset = <250>;

        delay,unreset = <0>;
        delay,unpower = <0>;
		*/
        unprepare-delay-ms = <0>;
        disable-delay-ms = <20>;
		
        size,width = <120>;
        size,height = <170>;

        status = "okay";
				

        panel-init-sequence = [
            05 20 01 29
            05 96 01 11
        ];

		panel-exit-sequence = [
			05 05 01 28
			05 78 01 10
		];
		
		power_ctr: power_ctr {
               rockchip,debug = <0>;
               lcd_en: lcd-en {
                       gpios = <&gpio1 1 GPIO_ACTIVE_HIGH>;
					   pinctrl-names = "default";
					   pinctrl-0 = <&lcd_panel_enable>;
                       rockchip,delay = <10>;
               };
               lcd_rst: lcd-rst {
                       gpios = <&gpio4 29 GPIO_ACTIVE_HIGH>;
					   pinctrl-names = "default";
					   pinctrl-0 = <&lcd_panel_reset>;
                       rockchip,delay = <6>;
               };
	   	};

		disp_timings: display-timings {
				          native-mode = <&timing0>;
				          timing0: timing0 {
				                       clock-frequency = <80000000>;
				                       hactive = <768>;
				                       vactive = <1024>;
				                       hsync-len = <20>;   //20, 50
				                       hback-porch = <130>; //50, 56
				                       hfront-porch = <150>;//50, 30
				                       vsync-len = <40>;
				                       vback-porch = <130>;
				                       vfront-porch = <136>;
				                       hsync-active = <0>;
				                       vsync-active = <0>;
				                       de-active = <0>;
				                       pixelclk-active = <0>;
				                   };
				      };
	};
};
```

这里定义了LCD的电源控制引脚：
>lcd_en:(GPIO1_A1)GPIO_ACTIVE_HIGH
>
>lcd_en:(GPIO4_D5)GPIO_ACTIVE_HIGH

都是高电平有效。

Firefly-RK3399硬件文档原理图：
![](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/zjj.sys.display.8.1.uboot/firefly-rk3399-sch-mipi-tx.jpg)

![](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/zjj.sys.display.8.1.uboot/firefly-rk3399-sch-u1e.jpg)

LCD的时序参数（timing），一般在屏的datasheet都可以找到，时序属性参考下图：
![](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/zjj.sys.display.8.1.uboot/firefly-rk3399-mipi-timing.jpg)

属性说明图：
![](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/zjj.sys.display.8.1.uboot/firefly-rk3399-dts-property.jpg)

###### 1.2.7、MIPI-DPHY 

``` c
&mipi_dphy {
	status = "okay";
};
```
Note：rk3288/rk3399没有该节点，不需要配置。



###### 1.2.8、Command

``` c
I:\RK3399_Android8.1_MIPI\kernel\arch\arm64\boot\dts\rockchip\rk3399-firefly-mipi.dts

        panel-init-sequence = [
            05 20 01 29
            05 96 01 11
        ];

		panel-exit-sequence = [
			05 05 01 28
			05 78 01 10
		];
```
说明：前3个字节（16进制），分别代表DataType，Delay，Payload Length。

从第四个字节开始的数据代表长度为Length的实际有效Payload。

panel-init-sequence 一条命令的解析如下：

Data Type：0x05(DCS Long Write)

Delay：0x20(32s)

Payload Length：0x01 (1 Bytes)

Payload：029

panel-init-sequence 后一条命令的解析如下：

Data Type：0x05(DCS Long Write)

Delay：0x96(150s)

Payload Length：0x01 (1 Bytes)

Payload：11


第一个字节代表的命令类型分两大类：DCS 命令和 Generic 命令。DCS 命令是有 mipi 标准协议里面指定的命令，具体可以参考《MIPI Alliance Specification for Display CommandSet.pdf 》。Generic 命令一般应用于厂商自定义的命令。具体使用哪种类型需要参考屏规格书或者咨询屏厂 FAE。如果没有特别指定，建议按 DCS 命令类型配置。DCS 命 令 的 类 型 有 三 种 ： 0x05/0x15/0x39 。 Generic 命 令 的 类 型 分 为 ：0x03/0x13/0x23/0x29。

###### 1.2.9、背光 backlight
``` c
I:\RK3399_Android8.1_MIPI\kernel\arch\arm64\boot\dts\rockchip\rk3399-firefly-mipi.dts

&backlight {
	status = "okay";
	pwms = <&pwm1 0 25000 0>;
};

&pwm1 {
	status = "okay";
};

&dsi {
	status = "okay";
	dsi_panel: panel {
		compatible ="simple-panel-dsi";
		reg = <0>;
		backlight = <&backlight>;
```

pwms属性：配置PWM，MIPI屏使用的pwm输出是pwm1，25000ns是周期(40 KHz)。

#### (二)、U-boot Display 显示过程分析

首先看看全部Log：

``` c
[09:50:13]zjj.sye.display board/rockchip/common/rkboot/fastboot.c board_fbt_preboot 474 
[09:50:13]normal boot.
[09:50:13]Rockchip UBOOT DRM driver version: develop-v1.0.0
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c init_display_buffer 55 
[09:50:13]zjj.sye.display memory_start = 2109734912 
[09:50:13]zjj.sye.display memory_end = 2109734912 
[09:50:13]zjj.sye.display drivers/video/rockchip_crtc.c rockchip_get_crtc 73 
[09:50:13]zjj.sye.display compatible namerockchip,rk3399-vop-big  
[09:50:13]zjj.sye.display drivers/video/rockchip_connector.c rockchip_get_connector 120 
[09:50:13]zjj.sye.display compatible namerockchip,rk3399-mipi-dsi  
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c connector_phy_init 188 
[09:50:13]failed to find phy node
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c connector_panel_init 230 
[09:50:13]zjj.sye.display drivers/video/rockchip_panel.c rockchip_get_panel 142 
[09:50:13]zjj.sye.display rockchip_get_panel simple-panel-dsi
[09:50:13]zjj.sye.display drivers/video/rockchip_panel.c rockchip_panel_init 161 
[09:50:13]zjj.sye.display drivers/video/rockchip_dsi_panel.c rockchip_dsi_panel_init 321 
[09:50:13]zjj.sye.display drivers/video/rockchip_dsi_panel.c rockchip_dsi_panel_parse_dt 255 
[09:50:13]zjj.sye.display drivers/video/rockchip_dsi_panel.c rockchip_dsi_panel_parse_cmds 77 
[09:50:13]zjj.sye.display drivers/video/rockchip_dsi_panel.c rockchip_dsi_panel_parse_cmds 77 
[09:50:13]zjj.sye.display drivers/video/backlight/pwm_bl.c rk_pwm_bl_config 197 
[09:50:13]zjj.sye.display rk_pwm_bl_config: 0 
[09:50:13]zjj.sye.display board/rockchip/common/rkboot/fastboot.c board_fbt_preboot 529 
[09:50:13]read logo on state from dts [1]
[09:50:13]no fuel gauge found
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c load_bmp_logo 952 
[09:50:13]zjj.sye.display bmp_name logo.bmp 
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c display_logo 814 
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c display_init 630 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c rockchip_dw_mipi_dsi_init 1204 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c rockchip_dsi_dual_channel_probe 1151 
[09:50:13]Using display timing dts
[09:50:13]zjj.sye.display Detailed mode clock 80000 kHz, flags[a]
[09:50:13]    H: 0768 0918 0938 1068
[09:50:13]    V: 1024 1160 1200 1330
[09:50:13]bus_format: 1009
[09:50:13]rk lcdc - 0 dclk set: dclk = 80000000HZ, pll select = 1, div = 1
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c display_logo 867 
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c display_set_plane 705 
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c display_init 630 
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c display_enable 727 
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c connector_panel_power_prepare 292 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c rockchip_dw_mipi_dsi_prepare 1358 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c rockchip_dw_dsi_pre_init 1268 
[09:50:13]final DSI-Link bandwidth: 532 Mbps x 4
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c rockchip_dw_dsi_host_init 1293 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_init 995 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_dpi_config 1011 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_video_mode_config 940 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_video_packet_config 1051 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_video_mode_config 940 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c rockchip_dw_dsi_controller_init 1342 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c mipi_dphy_init 1318 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_phy_init 540 
[09:50:13]failed to wait for phy clk lane stop state
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_set_mode 962 
[09:50:13]zjj.sye.display DSI_COMMAND_MODE 
[09:50:13]zjj.sye.display drivers/video/rockchip_panel.c rockchip_panel_prepare 187 
[09:50:13]zjj.sye.display drivers/video/rockchip_dsi_panel.c rockchip_dsi_panel_prepare 182 
[09:50:13]zjj.sye.display drivers/video/rockchip_dsi_panel.c rockchip_dsi_panel_send_cmds 147 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c rockchip_dw_mipi_dsi_transfer 911 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_short_write 784 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c rockchip_dw_mipi_dsi_transfer 911 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_short_write 784 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c rockchip_dw_mipi_dsi_enable 1377 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_set_mode 968 
[09:50:13]zjj.sye.display DSI_VIDEO_MODE 
[09:50:13]zjj.sye.display drivers/video/rockchip-dw-mipi-dsi.c dw_mipi_dsi_video_mode_config 940 
[09:50:13]zjj.sye.display drivers/video/rockchip_panel.c rockchip_panel_enable 213 
[09:50:13]zjj.sye.display drivers/video/rockchip_dsi_panel.c rockchip_dsi_panel_enable 226 
[09:50:13]zjj.sye.display drivers/video/backlight/pwm_bl.c rk_pwm_bl_config 197 
[09:50:13]zjj.sye.display rk_pwm_bl_config: -1 
[09:50:13]zjj.sye.display drivers/video/rockchip_display.c load_bmp_logo 952 
[09:50:13]zjj.sye.display bmp_name logo_kernel.bmp 

```


#### （2）、U-boot Display 显示过程流程图

![](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/zjj.sys.display.8.1.uboot/FireFly-Rk3399-Uboot-Display-Flow.jpg)

#### （三）、vop, dsi, panel三大模块init分析
#### 3.1、panel模块init分析
##### 3.1.1、rockchip_display_init()

```c

I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_display.c
int rockchip_display_init(void)
{
	const void *blob = gd->fdt_blob;
	int route, child, phandle, connect, crtc_node, conn_node;
	const struct rockchip_connector *conn;
	const struct rockchip_crtc *crtc;
	struct display_state *s;
	const char *name;

	printf("Rockchip UBOOT DRM driver version: %s\n", DRIVER_VERSION);

	route = fdt_path_offset(blob, "/display-subsystem/route");
	......
    
    //获取fb地址
	init_display_buffer();

	fdt_for_each_subnode(blob, child, route) {
		......
		phandle = fdt_getprop_u32_default_node(blob, child, 0,
						       "connect", -1);
		......
        
       //获取connect node.
		connect = fdt_node_offset_by_phandle(blob, phandle);
		......
      
      //获取对应的crtc_node.
		crtc_node = find_crtc_node(blob, connect);
		......
        
      //根据crtc_node从g_crtc数组中找到具体的元素，这里compatible是"rockchip,rk3399-vop-big"，在rk3399-firefly-mipi.dts中定义
		crtc = rockchip_get_crtc(blob, crtc_node);
		......
      
       //获取对应的connector node
		conn_node = find_connector_node(blob, connect);
		......

       //根据connector node从g_connector数组中找到具体的元素，这里compatible是"rockchip,rk3399-mipi"，在rk3399-firefly-mipi.dts中定义
		conn = rockchip_get_connector(blob, conn_node);
		......
        
       //display相关信息都放在struct display_state *s中
		s = malloc(sizeof(*s));
		......
		memset(s, 0, sizeof(*s));

		INIT_LIST_HEAD(&s->head);
        
       //分别获取uboot/kernel中的logo name以及模式
		fdt_get_string(blob, child, "logo,uboot", &s->ulogo_name);
		fdt_get_string(blob, child, "logo,kernel", &s->klogo_name);
		fdt_get_string(blob, child, "logo,mode", &name);
		if (!strcmp(name, "fullscreen"))
			s->logo_mode = ROCKCHIP_DISPLAY_FULLSCREEN;
		else
			s->logo_mode = ROCKCHIP_DISPLAY_CENTER;
		fdt_get_string(blob, child, "charge_logo,mode", &name);
		if (!strcmp(name, "fullscreen"))
			s->charge_logo_mode = ROCKCHIP_DISPLAY_FULLSCREEN;
		else
			s->charge_logo_mode = ROCKCHIP_DISPLAY_CENTER;

		s->blob = blob;
		s->conn_state.node = conn_node;
		s->conn_state.connector = conn;
		s->conn_state.overscan.left_margin = 100;
		s->conn_state.overscan.right_margin = 100;
		s->conn_state.overscan.top_margin = 100;
		s->conn_state.overscan.bottom_margin = 100;
		s->crtc_state.node = crtc_node;
		s->crtc_state.crtc = crtc;
		s->crtc_state.crtc_id = get_crtc_id(blob, connect);
		s->node = child;
      
       //无phy node
		connector_phy_init(s);
		connector_panel_init(s);
		list_add_tail(&s->head, &rockchip_display_list);
	}

	return 0;

```
##### 3.1.2、rockchip_get_crtc()
此函数主要获取crtc(即VOP)寄存器相关信息

```c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_crtc.c
const struct rockchip_crtc *
rockchip_get_crtc(const void *blob, int crtc_node)
{
	const char *name;
	int i;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	fdt_get_string(blob, crtc_node, "compatible", &name);
        printf("zjj.sye.display compatible name%s  \n", name);
	for (i = 0; i < ARRAY_SIZE(g_crtc); i++) {
		if (!strcmp(name, g_crtc[i].compatible))
			break;
	}
	if (i >= ARRAY_SIZE(g_crtc))
		return NULL;

	return &g_crtc[i];
}

```
我们这里compatible name是：
>compatible name rockchip,rk3399-vop-big
所以具体的寄存器信息是：


``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_crtc.c
static const struct rockchip_crtc g_crtc[] = {
    ......
	}, {
		.compatible = "rockchip,rk3399-vop-big",
		.funcs = &rockchip_vop_funcs,
		.data = &rk3399_vop_big,
	},
    
```

此处VOP的操作函数是：

``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_vop.c
const struct rockchip_crtc_funcs rockchip_vop_funcs = {
	.init = rockchip_vop_init,
	.set_plane = rockchip_vop_set_plane,
	.prepare = rockchip_vop_prepare,
	.enable = rockchip_vop_enable,
	.disable = rockchip_vop_disable,
	.fixup_dts = rockchip_vop_fixup_dts,
};

```
VOP寄存器信息是：
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_crtc.c
const struct vop_data rk3399_vop_big = {
	.version = VOP_VERSION(3, 5),
	.max_output = {4096, 2160},
	.feature = VOP_FEATURE_OUTPUT_10BIT,
	.ctrl = &rk3288_ctrl_data,
	.win = &rk3288_win01_data,
	.line_flag = &rk3366_vop_line_flag,
	.reg_len = RK3399_DSP_VACT_ST_END_F1 * 4,
};
static const struct vop_ctrl rk3288_ctrl_data = {
	.standby = VOP_REG(RK3288_SYS_CTRL, 0x1, 22),
	.axi_outstanding_max_num = VOP_REG(RK3288_SYS_CTRL1, 0x1f, 13),
	.axi_max_outstanding_en = VOP_REG(RK3288_SYS_CTRL1, 0x1, 12),
	.reg_done_frm = VOP_REG_VER(RK3288_SYS_CTRL1, 0x1, 24, 3, 7, -1),
	.htotal_pw = VOP_REG(RK3288_DSP_HTOTAL_HS_END, 0x1fff1fff, 0),
	.hact_st_end = VOP_REG(RK3288_DSP_HACT_ST_END, 0x1fff1fff, 0),
	.vtotal_pw = VOP_REG(RK3288_DSP_VTOTAL_VS_END, 0x1fff1fff, 0),
	.vact_st_end = VOP_REG(RK3288_DSP_VACT_ST_END, 0x1fff1fff, 0),
	.vact_st_end_f1 = VOP_REG(RK3288_DSP_VACT_ST_END_F1, 0x1fff1fff, 0),
	.vs_st_end_f1 = VOP_REG(RK3288_DSP_VS_ST_END_F1, 0x1fff1fff, 0),
	.hpost_st_end = VOP_REG(RK3288_POST_DSP_HACT_INFO, 0x1fff1fff, 0),
	.vpost_st_end = VOP_REG(RK3288_POST_DSP_VACT_INFO, 0x1fff1fff, 0),
	.vpost_st_end_f1 = VOP_REG(RK3288_POST_DSP_VACT_INFO_F1, 0x1fff1fff, 0),
	.post_scl_factor = VOP_REG(RK3288_POST_SCL_FACTOR_YRGB, 0xffffffff, 0),
	.post_scl_ctrl = VOP_REG(RK3288_POST_SCL_CTRL, 0x3, 0),

	.dsp_interlace = VOP_REG(RK3288_DSP_CTRL0, 0x1, 10),
	.auto_gate_en = VOP_REG(RK3288_SYS_CTRL, 0x1, 23),
	.dsp_layer_sel = VOP_REG(RK3288_DSP_CTRL1, 0xff, 8),
	.post_lb_mode = VOP_REG_VER(RK3288_SYS_CTRL, 0x1, 18, 3, 2, -1),
	.global_regdone_en = VOP_REG_VER(RK3288_SYS_CTRL, 0x1, 11, 3, 2, -1),
	.overlay_mode = VOP_REG_VER(RK3288_SYS_CTRL, 0x1, 16, 3, 2, -1),
	.core_dclk_div = VOP_REG_VER(RK3399_DSP_CTRL0, 0x1, 4, 3, 4, -1),
	.p2i_en = VOP_REG_VER(RK3399_DSP_CTRL0, 0x1, 5, 3, 4, -1),
	.dclk_ddr = VOP_REG_VER(RK3399_DSP_CTRL0, 0x1, 8, 3, 4, -1),
	.dp_en = VOP_REG(RK3399_SYS_CTRL, 0x1, 11),
	.rgb_en = VOP_REG(RK3288_SYS_CTRL, 0x1, 12),
	.hdmi_en = VOP_REG(RK3288_SYS_CTRL, 0x1, 13),
	.edp_en = VOP_REG(RK3288_SYS_CTRL, 0x1, 14),
	.mipi_en = VOP_REG(RK3288_SYS_CTRL, 0x1, 15),
	.mipi_dual_channel_en = VOP_REG(RK3288_SYS_CTRL, 0x1, 3),
	.data01_swap = VOP_REG_VER(RK3288_SYS_CTRL, 0x1, 17, 3, 5, -1),
	.pin_pol = VOP_REG_VER(RK3288_DSP_CTRL0, 0xf, 4, 3, 0, 1),
	.dp_pin_pol = VOP_REG_VER(RK3368_DSP_CTRL1, 0xf, 16, 3, 2, -1),
	.rgb_pin_pol = VOP_REG_VER(RK3368_DSP_CTRL1, 0xf, 16, 3, 2, -1),
	.tve_dclk_en = VOP_REG(RK3288_SYS_CTRL, 0x1, 24),
	.tve_dclk_pol = VOP_REG(RK3288_SYS_CTRL, 0x1, 25),
	.tve_sw_mode = VOP_REG(RK3288_SYS_CTRL, 0x1, 26),
	.sw_uv_offset_en  = VOP_REG(RK3288_SYS_CTRL, 0x1, 27),
	.sw_genlock   = VOP_REG(RK3288_SYS_CTRL, 0x1, 28),
	.hdmi_pin_pol = VOP_REG_VER(RK3368_DSP_CTRL1, 0xf, 20, 3, 2, -1),
	.edp_pin_pol = VOP_REG_VER(RK3368_DSP_CTRL1, 0xf, 24, 3, 2, -1),
	.mipi_pin_pol = VOP_REG_VER(RK3368_DSP_CTRL1, 0xf, 28, 3, 2, -1),

	.dither_down = VOP_REG(RK3288_DSP_CTRL1, 0xf, 1),
	.dither_up = VOP_REG(RK3288_DSP_CTRL1, 0x1, 6),

	.dsp_out_yuv = VOP_REG_VER(RK3399_POST_SCL_CTRL, 0x1, 2, 3, 5, -1),
	.dsp_data_swap = VOP_REG(RK3288_DSP_CTRL0, 0x1f, 12),
	.dsp_ccir656_avg = VOP_REG(RK3288_DSP_CTRL0, 0x1, 20),
	.dsp_blank = VOP_REG(RK3288_DSP_CTRL0, 0x3, 18),
	.dsp_lut_en = VOP_REG(RK3288_DSP_CTRL1, 0x1, 0),
	.update_gamma_lut = VOP_REG_VER(RK3288_DSP_CTRL1, 0x1, 7, 3, 5, -1),
	.out_mode = VOP_REG(RK3288_DSP_CTRL0, 0xf, 0),

	.bcsh_brightness = VOP_REG(RK3288_BCSH_BCS, 0xff, 0),
	.bcsh_contrast = VOP_REG(RK3288_BCSH_BCS, 0x1ff, 8),
	.bcsh_sat_con = VOP_REG(RK3288_BCSH_BCS, 0x3ff, 20),
	.bcsh_out_mode = VOP_REG(RK3288_BCSH_BCS, 0x3, 0),
	.bcsh_sin_hue = VOP_REG(RK3288_BCSH_H, 0x1ff, 0),
	.bcsh_cos_hue = VOP_REG(RK3288_BCSH_H, 0x1ff, 16),
	.bcsh_r2y_csc_mode = VOP_REG_VER(RK3368_BCSH_CTRL, 0x1, 6, 3, 1, -1),
	.bcsh_r2y_en = VOP_REG_VER(RK3368_BCSH_CTRL, 0x1, 4, 3, 1, -1),
	.bcsh_y2r_csc_mode = VOP_REG_VER(RK3368_BCSH_CTRL, 0x3, 2, 3, 1, -1),
	.bcsh_y2r_en = VOP_REG_VER(RK3368_BCSH_CTRL, 0x1, 0, 3, 1, -1),
	.bcsh_color_bar = VOP_REG(RK3288_BCSH_COLOR_BAR, 0xffffff, 8),
	.bcsh_en = VOP_REG(RK3288_BCSH_COLOR_BAR, 0x1, 0),

	.xmirror = VOP_REG(RK3288_DSP_CTRL0, 0x1, 22),
	.ymirror = VOP_REG(RK3288_DSP_CTRL0, 0x1, 23),

	.dsp_background = VOP_REG(RK3288_DSP_BG, 0xffffffff, 0),

	.cfg_done = VOP_REG(RK3288_REG_CFG_DONE, 0x1, 0),
	.win_gate[0] = VOP_REG(RK3288_WIN2_CTRL0, 0x1, 0),
	.win_gate[1] = VOP_REG(RK3288_WIN3_CTRL0, 0x1, 0),
};


static const struct vop_win rk3288_win01_data = {
	.scl = &rk3288_win_full_scl,
	.enable = VOP_REG(RK3288_WIN0_CTRL0, 0x1, 0),
	.format = VOP_REG(RK3288_WIN0_CTRL0, 0x7, 1),
	.rb_swap = VOP_REG(RK3288_WIN0_CTRL0, 0x1, 12),
	.ymirror = VOP_REG_VER(RK3368_WIN0_CTRL0, 0x1, 22, 3, 2, -1),
	.act_info = VOP_REG(RK3288_WIN0_ACT_INFO, 0x1fff1fff, 0),
	.dsp_info = VOP_REG(RK3288_WIN0_DSP_INFO, 0x0fff0fff, 0),
	.dsp_st = VOP_REG(RK3288_WIN0_DSP_ST, 0x1fff1fff, 0),
	.yrgb_mst = VOP_REG(RK3288_WIN0_YRGB_MST, 0xffffffff, 0),
	.uv_mst = VOP_REG(RK3288_WIN0_CBR_MST, 0xffffffff, 0),
	.yrgb_vir = VOP_REG(RK3288_WIN0_VIR, 0x3fff, 0),
	.uv_vir = VOP_REG(RK3288_WIN0_VIR, 0x3fff, 16),
	.src_alpha_ctl = VOP_REG(RK3288_WIN0_SRC_ALPHA_CTRL, 0xffffffff, 0),
	.dst_alpha_ctl = VOP_REG(RK3288_WIN0_DST_ALPHA_CTRL, 0xffffffff, 0),
};
```

##### 3.1.3、rockchip_get_connector()
>compatible name rockchip,rk3399-mipi-dsi  

``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_connector.c
const struct rockchip_connector *
rockchip_get_connector(const void *blob, int connector_node)
{
	const char *name;
	int i;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	fdt_get_string(blob, connector_node, "compatible", &name);
        printf("zjj.sye.display compatible name%s  \n", name);
	for (i = 0; i < ARRAY_SIZE(g_connector); i++) {
		if (!strcmp(name, g_connector[i].compatible))
			break;
	}
	if (i >= ARRAY_SIZE(g_connector))
		return NULL;

	return &g_connector[i];
}

```

此处DSI的操作函数是：
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
const struct rockchip_connector_funcs rockchip_dw_mipi_dsi_funcs = {
	.init = rockchip_dw_mipi_dsi_init,
	.deinit = rockchip_dw_mipi_dsi_deinit,
	.prepare = rockchip_dw_mipi_dsi_prepare,
	.enable = rockchip_dw_mipi_dsi_enable,
	.disable = rockchip_dw_mipi_dsi_disable,
	.transfer = rockchip_dw_mipi_dsi_transfer,
};

```

DSI寄存器信息是：
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static const u32 rk3399_dsi0_grf_reg_fields[MAX_FIELDS] = {
	[DPIUPDATECFG]		= GRF_REG_FIELD(0x6224, 15, 15),
	[DPISHUTDN]		= GRF_REG_FIELD(0x6224, 14, 14),
	[DPICOLORM]		= GRF_REG_FIELD(0x6224, 13, 13),
	[VOPSEL]		= GRF_REG_FIELD(0x6250,  0,  0),
	[TURNREQUEST]		= GRF_REG_FIELD(0x6258, 12, 15),
	[TURNDISABLE]		= GRF_REG_FIELD(0x6258,  8, 11),
	[FORCETXSTOPMODE]	= GRF_REG_FIELD(0x6258,  4,  7),
	[FORCERXMODE]		= GRF_REG_FIELD(0x6258,  0,  3),
};

static const u32 rk3399_dsi1_grf_reg_fields[MAX_FIELDS] = {
	[VOPSEL]		= GRF_REG_FIELD(0x6250,  4,  4),
	[DPIUPDATECFG]		= GRF_REG_FIELD(0x6250,  3,  3),
	[DPISHUTDN]		= GRF_REG_FIELD(0x6250,  2,  2),
	[DPICOLORM]		= GRF_REG_FIELD(0x6250,  1,  1),
	[TURNDISABLE]		= GRF_REG_FIELD(0x625c, 12, 15),
	[FORCETXSTOPMODE]	= GRF_REG_FIELD(0x625c,  8, 11),
	[FORCERXMODE]		= GRF_REG_FIELD(0x625c,  4,  7),
	[ENABLE_N]		= GRF_REG_FIELD(0x625c,  0,  3),
	[MASTERSLAVEZ]		= GRF_REG_FIELD(0x6260,  7,  7),
	[ENABLECLK]		= GRF_REG_FIELD(0x6260,  6,  6),
	[BASEDIR]		= GRF_REG_FIELD(0x6260,  5,  5),
	[TURNREQUEST]		= GRF_REG_FIELD(0x6260,  0,  3),
};

const struct dw_mipi_dsi_plat_data rk3399_mipi_dsi_drv_data = {
	.dsi0_grf_reg_fields = rk3399_dsi0_grf_reg_fields,
	.dsi1_grf_reg_fields = rk3399_dsi1_grf_reg_fields,
	.max_bit_rate_per_lane = 1500000000UL,
	.soc_type = RK3399,
};

```

##### 3.1.4、connector_panel_init()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_display.c
static int connector_panel_init(struct display_state *state)
{
	struct connector_state *conn_state = &state->conn_state;
	struct panel_state *panel_state = &state->panel_state;
	const void *blob = state->blob;
	int conn_node = conn_state->node;
	const struct rockchip_panel *panel;
	int panel_node, dsp_lut_node;
	int ret, len;
    printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	DBG("zjj.sye.display enter\n");
	panel_node = get_panel_node(state, conn_node);
	......

	if (!fdt_device_is_available(blob, panel_node)) {
		......
	}

	panel_state->node = panel_node;

	panel = rockchip_get_panel(blob, panel_node);
	......

	connector_pclist_parse_dt(conn_state, blob, panel_node);
	panel_state->panel = panel;

	ret = rockchip_panel_init(state);
	......

	return 0;
}

```
##### 3.1.5、rockchip_get_panel()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_panel.c
static const struct rockchip_panel g_panel[] = {
	  {
		.compatible = "simple-panel-dsi",
		.funcs = &rockchip_dsi_panel_funcs,
	  }
};
const struct rockchip_panel *rockchip_get_panel(const void *blob, int node)
{
	const char *name;
	int i, ret, index = 0;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	for (;;) {
		ret = fdt_get_string_index(blob, node, "compatible", index++, &name);
                printf("zjj.sye.display rockchip_get_panel %s\n", name);
		if (ret < 0)
			break;

		for (i = 0; i < ARRAY_SIZE(g_panel); i++)
			if (!strcmp(name, g_panel[i].compatible))
				return &g_panel[i];
	}

	return NULL;
}
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_dsi_panel.c
const struct rockchip_panel_funcs rockchip_dsi_panel_funcs = {
	.init		= rockchip_dsi_panel_init,
	.deinit		= rockchip_dsi_panel_deinit,
	.prepare	= rockchip_dsi_panel_prepare,
	.unprepare	= rockchip_dsi_panel_unprepare,
	.enable		= rockchip_dsi_panel_enable,
	.disable	= rockchip_dsi_panel_disable,
};


```

##### 3.1.6、rockchip_panel_init()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_panel.c
int rockchip_panel_init(struct display_state *state)
{
	struct panel_state *panel_state = &state->panel_state;
	const struct rockchip_panel *panel = panel_state->panel;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	if (!panel || !panel->funcs || !panel->funcs->init) {
		printf("%s: failed to find panel init funcs\n", __func__);
		return -ENODEV;
	}

	return panel->funcs->init(state);
}
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_dsi_panel.c
static int rockchip_dsi_panel_init(struct display_state *state)
{
	const void *blob = state->blob;
	struct connector_state *conn_state = &state->conn_state;
	struct panel_state *panel_state = &state->panel_state;
	int node = panel_state->node;
	struct rockchip_dsi_panel *panel;
	int ret;
    printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	panel = malloc(sizeof(*panel));
	if (!panel)
		return -ENOMEM;
	memset(panel, 0, sizeof(*panel));

	panel->blob = blob;
	panel->node = node;
	panel_state->private = panel;

	ret = rockchip_dsi_panel_parse_dt(blob, node, panel);
	......

	conn_state->bus_format = panel->bus_format;

	return 0;
}

static int rockchip_dsi_panel_parse_dt(const void *blob, int node, struct rockchip_dsi_panel *panel)
{
	struct fdt_gpio_state *enable_gpio = &panel->enable_gpio;
	struct fdt_gpio_state *reset_gpio = &panel->reset_gpio;
	const void *data;
	int len = 0;
	int ret = 0;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	fdtdec_decode_gpio(blob, node, "enable-gpios", enable_gpio);
	fdtdec_decode_gpio(blob, node, "reset-gpios", reset_gpio);

	panel->delay_prepare = fdtdec_get_int(blob, node, "prepare-delay-ms", 0);
	panel->delay_unprepare = fdtdec_get_int(blob, node, "unprepare-delay-ms", 0);
	panel->delay_enable = fdtdec_get_int(blob, node, "enable-delay-ms", 0);
	panel->delay_disable = fdtdec_get_int(blob, node, "disable-delay-ms", 0);
	panel->delay_init = fdtdec_get_int(blob, node, "init-delay-ms", 0);
	panel->delay_reset = fdtdec_get_int(blob, node, "reset-delay-ms", 0);
	panel->bus_format = fdtdec_get_int(blob, node, "bus-format", MEDIA_BUS_FMT_RBG888_1X24);

	data = fdt_getprop(blob, node, "panel-init-sequence", &len);
	if (data) {
		panel->on_cmds = malloc(sizeof(*panel->on_cmds));
		......

		ret = rockchip_dsi_panel_parse_cmds(blob, node, data, len,
						    panel->on_cmds);
		......
	}

	data = fdt_getprop(blob, node, "panel-exit-sequence", &len);
	if (data) {
		panel->off_cmds = malloc(sizeof(*panel->off_cmds));
		......

		ret = rockchip_dsi_panel_parse_cmds(blob, node, data, len,
						    panel->off_cmds);
		......
	}

	/* keep panel blank on init. */
	gpio_direction_output(enable_gpio->gpio, !!(enable_gpio->flags & OF_GPIO_ACTIVE_LOW));
	gpio_direction_output(reset_gpio->gpio, !(reset_gpio->flags & OF_GPIO_ACTIVE_LOW));

#ifdef CONFIG_RK_PWM_BL
	rk_pwm_bl_config(0);
#endif
	return 0;
......
	return ret;
}

```
##### 3.1.7、rk_pwm_bl_config()

``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_dsi_panel.c
int rk_pwm_bl_config(int brightness)
{
	u64 val, div, clk_rate;
	unsigned long prescale = 0, pv, dc;
	u32 on;
	int conf = 0;
	int duty_ns, period_ns;
	int ret;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
        printf("zjj.sye.display rk_pwm_bl_config: %d \n",brightness);
	if (!bl.node) {
#ifdef CONFIG_OF_LIBFDT
		debug("rk pwm parse dt start.\n");
		if (!gd->fdt_blob)
			return -1;

		ret = rk_bl_parse_dt(gd->fdt_blob);
		debug("rk pwm parse dt end.\n");
		if (ret < 0)
			return ret;
#endif
		rk_iomux_config(bl.id + RK_PWM0_IOMUX);
		gpio_direction_output(bl.bl_en.gpio, bl.bl_en.flags);
	}

	if (!bl.status)
		return -EPERM;

	if (brightness == 0)
		gpio_set_value(bl.bl_en.gpio, !(bl.bl_en.flags));
	else
		gpio_set_value(bl.bl_en.gpio, bl.bl_en.flags);

	if (brightness < 0)
		brightness = bl.dft_brightness;
	debug("%s: brightness: %d\n", __func__, brightness);
	brightness = bl.levels[brightness];
	duty_ns = (brightness * bl.period)/bl.max_brightness;
	period_ns = bl.period;
	on   =  RK_PWM_ENABLE;
	conf = PWM_OUTPUT_LEFT|PWM_LP_DISABLE|PWM_CONTINUMOUS;

	if (bl.polarity == PWM_POLARITY_INVERSED)
		conf |= PWM_DUTY_NEGATIVE | PWM_INACTIVE_POSTIVE;
	else
		conf |= PWM_DUTY_POSTIVE | PWM_INACTIVE_NEGATIVE;

	/*
	 * Find pv, dc and prescale to suit duty_ns and period_ns. This is done
	 * according to formulas described below:
	 *
	 * period_ns = 10^9 * (PRESCALE ) * PV / PWM_CLK_RATE
	 * duty_ns = 10^9 * (PRESCALE + 1) * DC / PWM_CLK_RATE
	 *
	 * PV = (PWM_CLK_RATE * period_ns) / (10^9 * (PRESCALE + 1))
	 * DC = (PWM_CLK_RATE * duty_ns) / (10^9 * (PRESCALE + 1))
	 */

	clk_rate = get_pclk_pwm(bl.id);

	while (1) {
		div = 1000000000;
		div *= 1 + prescale;
		val = clk_rate * period_ns;
		pv = div64_u64(val, div);
		val = clk_rate * duty_ns;
		dc = div64_u64(val, div);
		/* if duty_ns and period_ns are not achievable then return */
		if (pv < PWMPCR_MIN_PERIOD || dc < PWMDCR_MIN_DUTY)
			return -EINVAL;

		/*
		 * if pv and dc have crossed their upper limit, then increase
		 * prescale and recalculate pv and dc.
		 */
		if (pv > PWMPCR_MAX_PERIOD || dc > PWMDCR_MAX_DUTY) {
			if (++prescale > PWMCR_MAX_PRESCALE)
				return -EINVAL;
			continue;
		}
		break;
	}

	/*
	 * NOTE: the clock to PWM has to be enabled first before writing to the
	 * registers.
	 */

	conf |= (prescale << RK_PWM_PRESCALE);
	write_pwm_reg(&bl, PWM_REG_DUTY, dc);
	write_pwm_reg(&bl, PWM_REG_PERIOD, pv);
	#ifdef RK_VOP_PWM
	if (bl.id >= RK_VOP0_PWM) {
		write_pwm_reg(&bl, VOP_PWM_REG_CNTR, 0);
		write_pwm_reg(&bl, VOP_PWM_REG_CTRL, on|conf);
	} else
	#endif
	{
		write_pwm_reg(&bl, PWM_REG_CNTR, 0);
		write_pwm_reg(&bl, PWM_REG_CTRL, on|conf);
	}

	return 0;
}

```


#### 3.2、dsi模块init分析
##### 3.2.1、rockchip_show_logo()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_display.c
void rockchip_show_logo(void)
{
	struct display_state *s;

	list_for_each_entry(s, &rockchip_display_list, head) {
		s->logo.mode = s->logo_mode;
		if (load_bmp_logo(&s->logo, s->ulogo_name))
			printf("failed to display uboot logo\n");
		else
			display_logo(s);
		if (load_bmp_logo(&s->logo, s->klogo_name))
			printf("failed to display kernel logo\n");
	}
}
```
##### 3.2.2、load_bmp_logo()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_display.c
static int load_bmp_logo(struct logo_info *logo, const char *bmp_name)
{
	struct rockchip_logo_cache *logo_cache;
	struct bmp_header *header;
	void *dst = NULL, *pdst;
	int size;
    printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
    printf("zjj.sye.display bmp_name %s \n", bmp_name);

	logo_cache = find_or_alloc_logo_cache(bmp_name);
	if (logo_cache->logo.mem) {
		memcpy(logo, &logo_cache->logo, sizeof(*logo));
		return 0;
	}

	header = get_bmp_header(bmp_name);

	logo->bpp = get_unaligned_le16(&header->bit_count);
	logo->width = get_unaligned_le32(&header->width);
	logo->height = get_unaligned_le32(&header->height);
	size = get_unaligned_le32(&header->file_size);
	if (!can_direct_logo(logo->bpp)) {
		if (size > CONFIG_RK_BOOT_BUFFER_SIZE) {
			return -ENOMEM;
		}
		pdst = (void *)gd->arch.rk_boot_buf_addr;

	} else {
		pdst = get_display_buffer(size);
		dst = pdst;
	}

	if (load_bmp_content(bmp_name, pdst, size)) {
		return 0;
	}

	if (!can_direct_logo(logo->bpp)) {
		int dst_size;
		logo->bpp = (logo->bpp <= 16) ? 16 : logo->bpp;
		dst_size = logo->width * logo->height * logo->bpp >> 3;

		if (bmpdecoder(pdst, dst, logo->bpp)) {
			return 0;
		}
		logo->offset = 0;
		logo->ymirror = 0;
	} else {
		logo->offset = get_unaligned_le32(&header->data_offset);
		logo->ymirror = 1;
	}
	logo->mem = (u32)(unsigned long)dst;

	memcpy(&logo_cache->logo, logo, sizeof(*logo));

	return 0;
}
```
##### 3.2.3、display_logo()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_display.c
static int display_logo(struct display_state *state)
{
	struct crtc_state *crtc_state = &state->crtc_state;
	struct connector_state *conn_state = &state->conn_state;
	struct logo_info *logo = &state->logo;
	int hdisplay, vdisplay;
    printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	display_init(state);
	if (!state->is_init)
		return -ENODEV;

	switch (logo->bpp) {
	case 16:
		crtc_state->format = ROCKCHIP_FMT_RGB565;
		break;
	case 24:
		crtc_state->format = ROCKCHIP_FMT_RGB888;
		break;
	case 32:
		crtc_state->format = ROCKCHIP_FMT_ARGB8888;
		break;
	default:
		printf("can't support bmp bits[%d]\n", logo->bpp);
		return -EINVAL;
	}
	crtc_state->rb_swap = logo->bpp != 32;
	hdisplay = conn_state->mode.hdisplay;
	vdisplay = conn_state->mode.vdisplay;
	crtc_state->src_w = logo->width;
	crtc_state->src_h = logo->height;
	crtc_state->src_x = 0;
	crtc_state->src_y = 0;
	crtc_state->ymirror = logo->ymirror;

	crtc_state->dma_addr = logo->mem + logo->offset;
	crtc_state->xvir = ALIGN(crtc_state->src_w * logo->bpp, 32) >> 5;

	if (logo->mode == ROCKCHIP_DISPLAY_FULLSCREEN) {
		crtc_state->crtc_x = 0;
		crtc_state->crtc_y = 0;
		crtc_state->crtc_w = hdisplay;
		crtc_state->crtc_h = vdisplay;
	} else {
		if (crtc_state->src_w >= hdisplay) {
			crtc_state->crtc_x = 0;
			crtc_state->crtc_w = hdisplay;
		} else {
			crtc_state->crtc_x = (hdisplay - crtc_state->src_w) / 2;
			crtc_state->crtc_w = crtc_state->src_w;
		}

		if (crtc_state->src_h >= vdisplay) {
			crtc_state->crtc_y = 0;
			crtc_state->crtc_h = vdisplay;
		} else {
			crtc_state->crtc_y = (vdisplay - crtc_state->src_h) / 2;
			crtc_state->crtc_h = crtc_state->src_h;
		}
	}
    printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	display_set_plane(state);
	display_enable(state);

	return 0;
}
```

##### 3.2.4、display_init()
``` c
static int display_init(struct display_state *state)
{
	const struct rockchip_connector *conn = state->conn_state.connector;
	const struct rockchip_connector_funcs *conn_funcs = conn->funcs;
	struct rockchip_crtc *crtc = state->crtc_state.crtc;
	const struct rockchip_crtc_funcs *crtc_funcs = crtc->funcs;
	const struct connector_state *conn_state = &state->conn_state;
	struct drm_display_mode *mode = &conn_state->mode;
	int ret = 0;

    printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);

	if (state->is_init)
		return 0;

	if (!conn_funcs || !crtc_funcs) {
		printf("failed to find connector or crtc functions\n");
		return -ENXIO;
	}

	if (conn_funcs->init) {
		ret = conn_funcs->init(state);
		......
	}

	if (conn_funcs->get_timing) {
		ret = conn_funcs->get_timing(state);
	} else {
		ret = display_get_timing(state);
	}
	drm_mode_set_crtcinfo(mode, CRTC_INTERLACE_HALVE_V);

	if (crtc_funcs->init) {
		ret = crtc_funcs->init(state);
	}

	state->is_init = 1;

	return 0;
......
	return ret;
}

```

##### 3.2.5、rockchip_dw_mipi_dsi_init()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static int rockchip_dw_mipi_dsi_init(struct display_state *state)
{
	struct connector_state *conn_state = &state->conn_state;
	const struct rockchip_connector *connector = conn_state->connector;
	const struct dw_mipi_dsi_plat_data *pdata = connector->data;
	int mipi_node = conn_state->node;
	struct dw_mipi_dsi *dsi;
	int panel;
	static int id = 0;
	int ret;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	dsi = malloc(sizeof(*dsi));
	if (!dsi)
		return -ENOMEM;
	memset(dsi, 0, sizeof(*dsi));

	dsi->base = (void *)fdtdec_get_addr_size_auto_noparent(state->blob,
						mipi_node, "reg", 0, NULL);
	dsi->pdata = pdata;
	dsi->id = id++;
	dsi->blob = state->blob;
	dsi->node = mipi_node;
	conn_state->private = dsi;
	conn_state->output_mode = ROCKCHIP_OUT_MODE_P888;
	conn_state->color_space = V4L2_COLORSPACE_DEFAULT;

	panel = fdt_subnode_offset(state->blob, mipi_node, "panel");
	......

	FDT_GET_INT(dsi->lanes, "dsi,lanes");
	FDT_GET_INT(dsi->format, "dsi,format");
	FDT_GET_INT(dsi->mode_flags, "dsi,flags");
	FDT_GET_INT(dsi->channel, "reg");
	dsi->lvds_force_clk = fdtdec_get_int(state->blob, panel, "dsi,lvds-force-clk", -1);
	......

	ret = rockchip_dsi_dual_channel_probe(dsi);
	......

	conn_state->type = DRM_MODE_CONNECTOR_DSI;
	if (dsi->slave)
		conn_state->output_type = ROCKCHIP_OUTPUT_DSI_DUAL_CHANNEL;

	return 0;
}
```

##### 3.2.6、display_get_timing()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_display.c
static int display_get_timing(struct display_state *state)
{
	const struct rockchip_connector *conn = state->conn_state.connector;
	struct connector_state *conn_state = &state->conn_state;
	const struct rockchip_connector_funcs *conn_funcs = conn->funcs;
	struct drm_display_mode *mode = &conn_state->mode;
	const struct drm_display_mode *m;
	const void *blob = state->blob;
	int conn_node = conn_state->node;
	int panel;

	panel = get_panel_node(state, conn_node);
	......

	if (!display_get_timing_from_dts(panel, blob, mode)) {
		printf("Using display timing dts\n");
		goto done;
	}

	m = rockchip_get_display_mode_from_panel(state);
	if (m) {
		printf("Using display timing from compatible panel driver\n");
		memcpy(mode, m, sizeof(*m));
		goto done;
	}

	rockchip_panel_prepare(state);

	......
	return -ENODEV;
done:
	printf("zjj.sye.display Detailed mode clock %u kHz, flags[%x]\n"
	       "    H: %04d %04d %04d %04d\n"
	       "    V: %04d %04d %04d %04d\n"
	       "bus_format: %x\n",
	       mode->clock, mode->flags,
	       mode->hdisplay, mode->hsync_start,
	       mode->hsync_end, mode->htotal,
	       mode->vdisplay, mode->vsync_start,
	       mode->vsync_end, mode->vtotal,
	       conn_state->bus_format);

	return 0;
}
```

#### 3.3、vop模块init分析
##### 3.3.1、rockchip_vop_init()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_vop.c
static int rockchip_vop_init(struct display_state *state)
{
	struct crtc_state *crtc_state = &state->crtc_state;
	struct connector_state *conn_state = &state->conn_state;
	struct drm_display_mode *mode = &conn_state->mode;
	const struct rockchip_crtc *crtc = crtc_state->crtc;
	const struct vop_data *vop_data = crtc->data;
	struct vop *vop;
	u16 hsync_len = mode->crtc_hsync_end - mode->crtc_hsync_start;
	u16 hdisplay = mode->crtc_hdisplay;
	u16 htotal = mode->crtc_htotal;
	u16 hact_st = mode->crtc_htotal - mode->crtc_hsync_start;
	u16 hact_end = hact_st + hdisplay;
	u16 vdisplay = mode->crtc_vdisplay;
	u16 vtotal = mode->crtc_vtotal;
	u16 vsync_len = mode->crtc_vsync_end - mode->crtc_vsync_start;
	u16 vact_st = mode->crtc_vtotal - mode->crtc_vsync_start;
	u16 vact_end = vact_st + vdisplay;
	u32 val, act_end;
	int rate;
	bool yuv_overlay = false, post_r2y_en = false, post_y2r_en = false;
	u16 post_csc_mode;
	int for_ddr_freq = 0;

	vop = malloc(sizeof(*vop));
	if (!vop)
		return -ENOMEM;
	memset(vop, 0, sizeof(*vop));

	crtc_state->private = vop;
	vop->regs = (void *)fdtdec_get_addr_size_auto_noparent(state->blob,
					crtc_state->node, "reg", 0, NULL);
	vop->regsbak = malloc(vop_data->reg_len);
	vop->win = vop_data->win;
	vop->win_offset = vop_data->win_offset;
	vop->ctrl = vop_data->ctrl;
	vop->line_flag = vop_data->line_flag;
	vop->version = vop_data->version;
	vop->max_output = vop_data->max_output;

#ifdef CONFIG_RKCHIP_RK3399
	/* Set Dclk pll parent */
	if (conn_state->type == DRM_MODE_CONNECTOR_HDMIA)
		rkclk_lcdc_dclk_pll_sel(crtc_state->crtc_id, 0);
	else
		rkclk_lcdc_dclk_pll_sel(crtc_state->crtc_id, 1);
#endif

	/* Set aclk hclk and dclk */

	rate = rkclk_lcdc_clk_set(crtc_state->crtc_id, mode->crtc_clock * 1000);
	if (rate != mode->clock * 1000) {
		printf("Warn: vop clk request %dhz, but real clock is %dhz\n",
		       mode->clock * 1000, rate);
	}
	memcpy(vop->regsbak, vop->regs, vop_data->reg_len);

	rockchip_vop_init_gamma(vop, state);

	VOP_CTRL_SET(vop, global_regdone_en, 1);
	VOP_CTRL_SET(vop, axi_outstanding_max_num, 30);
	VOP_CTRL_SET(vop, axi_max_outstanding_en, 1);
	VOP_CTRL_SET(vop, reg_done_frm, 1);
	VOP_CTRL_SET(vop, win_gate[0], 1);
	VOP_CTRL_SET(vop, win_gate[1], 1);
	VOP_CTRL_SET(vop, win_channel[0], 0x12);
	VOP_CTRL_SET(vop, win_channel[1], 0x34);
	VOP_CTRL_SET(vop, win_channel[2], 0x56);
	VOP_CTRL_SET(vop, dsp_blank, 0);

	val = 0x8;
	val |= (mode->flags & DRM_MODE_FLAG_NHSYNC) ? 0 : 1;
	val |= (mode->flags & DRM_MODE_FLAG_NVSYNC) ? 0 : (1 << 1);
	VOP_CTRL_SET(vop, pin_pol, val);

	switch (conn_state->type) {
   ......
	case DRM_MODE_CONNECTOR_DSI:
		VOP_CTRL_SET(vop, mipi_en, 1);
		VOP_CTRL_SET(vop, mipi_pin_pol, val);
		VOP_CTRL_SET(vop, mipi_dual_channel_en,
			!!(conn_state->output_type & ROCKCHIP_OUTPUT_DSI_DUAL_CHANNEL));
		VOP_CTRL_SET(vop, data01_swap,
			!!(conn_state->output_type & ROCKCHIP_OUTPUT_DSI_DUAL_LINK));
		break;
	......
	default:
		printf("unsupport connector_type[%d]\n", conn_state->type);
	}

	if (conn_state->output_mode == ROCKCHIP_OUT_MODE_AAAA &&
	    !(vop_data->feature & VOP_FEATURE_OUTPUT_10BIT))
		conn_state->output_mode = ROCKCHIP_OUT_MODE_P888;

	switch (conn_state->bus_format) {
	case MEDIA_BUS_FMT_RGB565_1X16:
		val = DITHER_DOWN_EN(1) | DITHER_DOWN_MODE(RGB888_TO_RGB565);
		break;
	case MEDIA_BUS_FMT_RGB666_1X18:
	case MEDIA_BUS_FMT_RGB666_1X24_CPADHI:
		val = DITHER_DOWN_EN(1) | DITHER_DOWN_MODE(RGB888_TO_RGB666);
		break;
	case MEDIA_BUS_FMT_YUV8_1X24:
	case MEDIA_BUS_FMT_UYYVYY8_0_5X24:
		val = DITHER_DOWN_EN(0) | PRE_DITHER_DOWN_EN(1);
		break;
	case MEDIA_BUS_FMT_YUV10_1X30:
	case MEDIA_BUS_FMT_UYYVYY10_0_5X30:
		val = DITHER_DOWN_EN(0) | PRE_DITHER_DOWN_EN(0);
		break;
	case MEDIA_BUS_FMT_RGB888_1X24:
	default:
		val = DITHER_DOWN_EN(0) | PRE_DITHER_DOWN_EN(0);
		break;
	}
	if (conn_state->output_mode == ROCKCHIP_OUT_MODE_AAAA)
		val |= PRE_DITHER_DOWN_EN(0);
	else
		val |= PRE_DITHER_DOWN_EN(1);
	val |= DITHER_DOWN_MODE_SEL(DITHER_DOWN_ALLEGRO);
	VOP_CTRL_SET(vop, dither_down, val);

	VOP_CTRL_SET(vop, dclk_ddr,
		     conn_state->output_mode == ROCKCHIP_OUT_MODE_YUV420 ? 1 : 0);
	VOP_CTRL_SET(vop, hdmi_dclk_out_en,
		     conn_state->output_mode == ROCKCHIP_OUT_MODE_YUV420 ? 1 : 0);

	if (is_uv_swap(conn_state->bus_format, conn_state->output_mode))
		VOP_CTRL_SET(vop, dsp_data_swap, DSP_RB_SWAP);
	else
		VOP_CTRL_SET(vop, dsp_data_swap, 0);

	VOP_CTRL_SET(vop, out_mode, conn_state->output_mode);

	if (VOP_CTRL_SUPPORT(vop, overlay_mode)) {
		yuv_overlay = is_yuv_output(conn_state->bus_format);
		VOP_CTRL_SET(vop, overlay_mode, yuv_overlay);
	}
	/*
	 * todo: r2y for win csc
	 */
	VOP_CTRL_SET(vop, dsp_out_yuv, is_yuv_output(conn_state->bus_format));

	if (yuv_overlay) {
		if (!is_yuv_output(conn_state->bus_format))
			post_y2r_en = true;
	} else {
		if (is_yuv_output(conn_state->bus_format))
			post_r2y_en = true;
	}

	post_csc_mode = to_vop_csc_mode(conn_state->color_space);
	VOP_CTRL_SET(vop, bcsh_r2y_en, post_r2y_en);
	VOP_CTRL_SET(vop, bcsh_y2r_en, post_y2r_en);
	VOP_CTRL_SET(vop, bcsh_r2y_csc_mode, post_csc_mode);
	VOP_CTRL_SET(vop, bcsh_y2r_csc_mode, post_csc_mode);

	/*
	 * Background color is 10bit depth if vop version >= 3.5
	 */
	if (!is_yuv_output(conn_state->bus_format))
		val = 0;
	else if (VOP_MAJOR(vop->version) == 3 &&
		 VOP_MINOR(vop->version) >= 5)
		val = 0x20010200;
	else
		val = 0x801080;
	VOP_CTRL_SET(vop, dsp_background, val);

	VOP_CTRL_SET(vop, htotal_pw, (htotal << 16) | hsync_len);
	val = hact_st << 16;
	val |= hact_end;
	VOP_CTRL_SET(vop, hact_st_end, val);
	val = vact_st << 16;
	val |= vact_end;
	VOP_CTRL_SET(vop, vact_st_end, val);
	if (mode->flags & DRM_MODE_FLAG_INTERLACE) {
		u16 vact_st_f1 = vtotal + vact_st + 1;
		u16 vact_end_f1 = vact_st_f1 + vdisplay;

		val = vact_st_f1 << 16 | vact_end_f1;
		VOP_CTRL_SET(vop, vact_st_end_f1, val);

		val = vtotal << 16 | (vtotal + vsync_len);
		VOP_CTRL_SET(vop, vs_st_end_f1, val);
		VOP_CTRL_SET(vop, dsp_interlace, 1);
		VOP_CTRL_SET(vop, p2i_en, 1);
		vtotal += vtotal + 1;
		act_end = vact_end_f1;
	} else {
		VOP_CTRL_SET(vop, dsp_interlace, 0);
		VOP_CTRL_SET(vop, p2i_en, 0);
		act_end = vact_end;
	}
	VOP_CTRL_SET(vop, vtotal_pw, (vtotal << 16) | vsync_len);
	vop_post_config(state, vop);
	VOP_CTRL_SET(vop, core_dclk_div,
		     !!(mode->flags & DRM_MODE_FLAG_DBLCLK));

	VOP_CTRL_SET(vop, standby, 1);

	if (VOP_MAJOR(vop->version) == 3 &&
	    (VOP_MINOR(vop->version) == 2 || VOP_MINOR(vop->version) == 8))
		for_ddr_freq = 1000;
	VOP_LINE_FLAG_SET(vop, line_flag_num[0], act_end - 1);
	VOP_LINE_FLAG_SET(vop, line_flag_num[1],
			  act_end - us_to_vertical_line(mode, for_ddr_freq));
	vop_cfg_done(vop);

	return 0;
}
```

##### 3.3.2、display_set_plane()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_display.c

static int display_set_plane(struct display_state *state)
{
	const struct rockchip_crtc *crtc = state->crtc_state.crtc;
	const struct rockchip_crtc_funcs *crtc_funcs = crtc->funcs;
	int ret;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	if (!state->is_init)
		return -EINVAL;

	if (crtc_funcs->set_plane) {
		ret = crtc_funcs->set_plane(state);
		if (ret)
			return ret;
	}

	return 0;
}

I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_vop.c
static int rockchip_vop_set_plane(struct display_state *state)
{
	struct crtc_state *crtc_state = &state->crtc_state;
	struct connector_state *conn_state = &state->conn_state;
	struct drm_display_mode *mode = &conn_state->mode;
	u32 act_info, dsp_info, dsp_st, dsp_stx, dsp_sty;
	struct vop *vop = crtc_state->private;
	int src_w = crtc_state->src_w;
	int src_h = crtc_state->src_h;
	int crtc_x = crtc_state->crtc_x;
	int crtc_y = crtc_state->crtc_y;
	int crtc_w = crtc_state->crtc_w;
	int crtc_h = crtc_state->crtc_h;
	int xvir = crtc_state->xvir;

	act_info = (src_h - 1) << 16;
	act_info |= (src_w - 1) & 0xffff;

	dsp_info = (crtc_h - 1) << 16;
	dsp_info |= (crtc_w - 1) & 0xffff;

	dsp_stx = crtc_x + mode->htotal - mode->hsync_start;
	dsp_sty = crtc_y + mode->vtotal - mode->vsync_start;
	dsp_st = dsp_sty << 16 | (dsp_stx & 0xffff);

	if (crtc_state->ymirror)
		crtc_state->dma_addr += (src_h - 1) * xvir * 4;
	VOP_WIN_SET(vop, ymirror, crtc_state->ymirror);
	VOP_WIN_SET(vop, format, crtc_state->format);
	VOP_WIN_SET(vop, yrgb_vir, xvir);
	VOP_WIN_SET(vop, yrgb_mst, crtc_state->dma_addr);

	scl_vop_cal_scl_fac(vop, src_w, src_h, crtc_w, crtc_h,
			    crtc_state->format);

	VOP_WIN_SET(vop, act_info, act_info);
	VOP_WIN_SET(vop, dsp_info, dsp_info);
	VOP_WIN_SET(vop, dsp_st, dsp_st);
	VOP_WIN_SET(vop, rb_swap, crtc_state->rb_swap);

	VOP_WIN_SET(vop, src_alpha_ctl, 0);

	VOP_WIN_SET(vop, enable, 1);
	vop_cfg_done(vop);

	return 0;
}
```


#### （四）、vop, dsi, panel三大模块prepare分析
#### 4.1、vop模块prepare分析

##### 4.1.1、display_enable()


``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_display.c
static int display_enable(struct display_state *state)
{
	const struct rockchip_connector *conn = state->conn_state.connector;
	const struct rockchip_crtc *crtc = state->crtc_state.crtc;
	const struct rockchip_connector_funcs *conn_funcs = conn->funcs;
	const struct rockchip_crtc_funcs *crtc_funcs = crtc->funcs;
	int ret = 0;

	display_init(state);
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	if (!state->is_init)
		return -EINVAL;

	if (state->is_enable)
		return 0;

	connector_panel_power_prepare(state);

	if (crtc_funcs->prepare) {
		ret = crtc_funcs->prepare(state);
		if (ret)
			return ret;
	}

	if (conn_funcs->prepare) {
		ret = conn_funcs->prepare(state);
		if (ret)
			goto unprepare_crtc;
	}

	rockchip_panel_prepare(state);

	if (crtc_funcs->enable) {
		ret = crtc_funcs->enable(state);
		if (ret)
			goto unprepare_conn;
	}

	if (conn_funcs->enable) {
		ret = conn_funcs->enable(state);
		if (ret)
			goto disable_crtc;
	}

	rockchip_panel_enable(state);

	state->is_enable = true;

	return 0;
......
	return ret;
}

```
##### 4.1.2、connector_panel_power_prepare()

``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_display.c
void connector_panel_power_prepare(struct display_state *state)
{
	struct panel_state *panel_state = &state->panel_state;
	const struct rockchip_panel *panel = panel_state->panel;
	struct connector_state *conn_state = &state->conn_state;
	struct rockchip_pwrctr *pwrctr = &(conn_state->pwrctr);
	struct list_head *pclist_head = &(pwrctr->pclist_head); 
	struct list_head *pos;
	struct rockchip_pwrctr_list *pclist;
	struct pwrctr *pc;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
    if(!panel)
        return 0;

	if (list_empty(pclist_head))
		return 0;
	list_for_each(pos, pclist_head) {
		pclist = list_entry(pos, struct rockchip_pwrctr_list,
				list);
		pc = &pclist->pc;
		if(pwrctr->debug > 0) {
			printf("pwrctr: set %s(%d)=%d,delay:%dms\n",
					pc->name, pc->gpio.gpio, pc->atv_val, pc->delay);
		}
		gpio_set_value(pc->gpio.gpio, pc->atv_val);
		mdelay(pc->delay);
	}

}

```

##### 4.1.3、rockchip_vop_prepare()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_vop.c
static int rockchip_vop_prepare(struct display_state *state)
{
	return 0;
}

```

#### 4.2、dsi模块prepare分析

##### 4.2.1、rockchip_dw_mipi_dsi_prepare()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static int rockchip_dw_mipi_dsi_prepare(struct display_state *state)
{
	struct connector_state *conn_state = &state->conn_state;
	struct crtc_state *crtc_state = &state->crtc_state;
	const struct rockchip_connector *connector = conn_state->connector;
	const struct dw_mipi_dsi_plat_data *pdata = connector->data;
	struct dw_mipi_dsi *dsi = conn_state->private;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	dw_mipi_dsi_vop_routing(dsi, crtc_state->crtc_id);

	rockchip_dw_dsi_pre_init(state, dsi);

	rockchip_dw_dsi_controller_init(dsi);

	dsi_write(dsi, DSI_PWR_UP, POWERUP);
	dw_mipi_dsi_wait_for_two_frames(dsi);

	dw_mipi_dsi_set_mode(dsi, DSI_COMMAND_MODE);

	return 0;
}
```
##### 4.2.2、dw_mipi_dsi_vop_routing()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static void dw_mipi_dsi_vop_routing(struct dw_mipi_dsi *dsi, int vop_id)
{
	grf_field_write(dsi, VOPSEL, vop_id);
	if (dsi->slave)
		grf_field_write(dsi->slave, VOPSEL, vop_id);
}


```
##### 4.2.3、rockchip_dw_dsi_pre_init()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static void rockchip_dw_dsi_pre_init(struct display_state *state,
				     struct dw_mipi_dsi *dsi)
{
	struct connector_state *conn_state = &state->conn_state;
	unsigned long bw, rate;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	dsi->mode = &conn_state->mode;

	dw_mipi_dsi_clk_enable(dsi);

	rockchip_phy_power_on(state);

	if (conn_state->phy) {
		bw = rockchip_dsi_calc_bandwidth(dsi);
		rate = rockchip_phy_set_pll(state, bw * USEC_PER_SEC);
		dsi->lane_mbps = rate / USEC_PER_SEC;
		//rockchip_phy_power_on(state);
	} else {
		dw_mipi_dsi_get_lane_bps(dsi);
	}

	printf("final DSI-Link bandwidth: %u Mbps x %d\n",
	       dsi->lane_mbps, dsi->lanes);

	if (dsi->slave)
		rockchip_dw_dsi_pre_init(state, dsi->slave);
}

```


##### 4.2.4、rockchip_dw_dsi_controller_init()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static void rockchip_dw_dsi_controller_init(struct dw_mipi_dsi *dsi)
{
	rockchip_dw_dsi_host_init(dsi);
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	mdelay(10);
	mipi_dphy_init(dsi);
	dw_mipi_dsi_phy_init(dsi);

	if (dsi->slave)
		rockchip_dw_dsi_controller_init(dsi->slave);
}
```


##### 4.2.5、mipi_dphy_init()

``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static void mipi_dphy_init(struct dw_mipi_dsi *dsi)
{
	u32 map[] = {0x1, 0x3, 0x7, 0xf};
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	/* Configures DPHY to work as a Master */
	grf_field_write(dsi, MASTERSLAVEZ, 1);

	/* Configures lane as TX */
	grf_field_write(dsi, BASEDIR, 0);

	/* Set all REQUEST inputs to zero */
	grf_field_write(dsi, TURNREQUEST, 0);
	grf_field_write(dsi, TURNDISABLE, 0);
	grf_field_write(dsi, FORCETXSTOPMODE, 0);
	grf_field_write(dsi, FORCERXMODE, 0);
	udelay(1);

	/* Enable Data Lane Module */
	grf_field_write(dsi, ENABLE_N, map[dsi->lanes - 1]);

	/* Enable Clock Lane Module */
	grf_field_write(dsi, ENABLECLK, 1);
}

```
##### 4.2.6、dw_mipi_dsi_phy_init()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static int dw_mipi_dsi_phy_init(struct dw_mipi_dsi *dsi)
{
	int ret, testdin, vco, val;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	vco = (dsi->lane_mbps < 200) ? 0 : (dsi->lane_mbps + 100) / 200;

	testdin = max_mbps_to_testdin(dsi->lane_mbps);
	if (testdin < 0) {
		printf("failed to get testdin for %dmbps lane clock\n",
		       dsi->lane_mbps);
		return testdin;
	}

	dsi_write(dsi, DSI_PWR_UP, POWERUP);

	dw_mipi_dsi_phy_write(dsi, 0x10, BYPASS_VCO_RANGE |
					 VCO_RANGE_CON_SEL(vco) |
					 VCO_IN_CAP_CON_LOW |
					 REF_BIAS_CUR_SEL);

	dw_mipi_dsi_phy_write(dsi, 0x11, CP_CURRENT_3MA);
	dw_mipi_dsi_phy_write(dsi, 0x12, CP_PROGRAM_EN | LPF_PROGRAM_EN |
					 LPF_RESISTORS_20_KOHM);

	dw_mipi_dsi_phy_write(dsi, 0x44, HSFREQRANGE_SEL(testdin));

         dw_mipi_dsi_phy_write(dsi, 0x19, PLL_LOOP_DIV_EN | PLL_INPUT_DIV_EN);
	dw_mipi_dsi_phy_write(dsi, 0x17, INPUT_DIVIDER(dsi->dphy.input_div));
	val = LOOP_DIV_LOW_SEL(dsi->dphy.feedback_div) | LOW_PROGRAM_EN;
	dw_mipi_dsi_phy_write(dsi, 0x18, val);
	dw_mipi_dsi_phy_write(dsi, 0x19, PLL_LOOP_DIV_EN | PLL_INPUT_DIV_EN);
	val = LOOP_DIV_HIGH_SEL(dsi->dphy.feedback_div) | HIGH_PROGRAM_EN;
	dw_mipi_dsi_phy_write(dsi, 0x18, val);
	

	dw_mipi_dsi_phy_write(dsi, 0x20, POWER_CONTROL | INTERNAL_REG_CURRENT |
					 BIAS_BLOCK_ON | BANDGAP_ON);

	dw_mipi_dsi_phy_write(dsi, 0x21, TER_RESISTOR_LOW | TER_CAL_DONE |
					 SETRD_MAX | TER_RESISTORS_ON);
	dw_mipi_dsi_phy_write(dsi, 0x21, TER_RESISTOR_HIGH | LEVEL_SHIFTERS_ON |
					 SETRD_MAX | POWER_MANAGE |
					 TER_RESISTORS_ON);

	dw_mipi_dsi_phy_write(dsi, 0x22, LOW_PROGRAM_EN |
					 BIASEXTR_SEL(BIASEXTR_127_7));
	dw_mipi_dsi_phy_write(dsi, 0x22, HIGH_PROGRAM_EN |
					 BANDGAP_SEL(BANDGAP_96_10));

	dw_mipi_dsi_phy_write(dsi, 0x70, TLP_PROGRAM_EN | 0xf);
	dw_mipi_dsi_phy_write(dsi, 0x71, THS_PRE_PROGRAM_EN | 0x2d);
	dw_mipi_dsi_phy_write(dsi, 0x72, THS_ZERO_PROGRAM_EN | 0xa);

	dsi_write(dsi, DSI_PHY_RSTZ, PHY_ENFORCEPLL | PHY_ENABLECLK |
				     PHY_UNRSTZ | PHY_UNSHUTDOWNZ);

	ret = readl_poll_timeout(dsi->base + DSI_PHY_STATUS,
				 val, val & LOCK, 1000, PHY_STATUS_TIMEOUT_US);
	if (ret < 0) {
		printf("failed to wait for phy lock state\n");
		return ret;
	}

	ret = readl_poll_timeout(dsi->base + DSI_PHY_STATUS,
				 val, val & STOP_STATE_CLK_LANE, 1000,
				 PHY_STATUS_TIMEOUT_US);
	if (ret < 0)
		printf("failed to wait for phy clk lane stop state\n");

	return ret;
}

```


##### 4.2.7、dw_mipi_dsi_set_mode()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static void dw_mipi_dsi_set_mode(struct dw_mipi_dsi *dsi,
				 enum dw_mipi_dsi_mode mode)
{
	if (mode == DSI_COMMAND_MODE) {
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
        printf("zjj.sye.display %s \n", "DSI_COMMAND_MODE");
		dsi_write(dsi, DSI_PWR_UP, RESET);
		dsi_write(dsi, DSI_MODE_CFG, ENABLE_CMD_MODE);
		dsi_write(dsi, DSI_PWR_UP, POWERUP);
	} else {
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
        printf("zjj.sye.display %s \n", "DSI_VIDEO_MODE");
		dsi_write(dsi, DSI_PWR_UP, RESET);
		dsi_write(dsi, DSI_MODE_CFG, ENABLE_VIDEO_MODE);
		dw_mipi_dsi_video_mode_config(dsi);
		dsi_write(dsi, DSI_PWR_UP, POWERUP);
	}
}

```
#### 4.3、panel模块prepare分析
##### 4.3.1、rockchip_panel_prepare()

``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_panel.c
int rockchip_panel_prepare(struct display_state *state)
{
	struct panel_state *panel_state = &state->panel_state;
	const struct rockchip_panel *panel = panel_state->panel;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	if (!panel || !panel->funcs || !panel->funcs->prepare) {
		printf("%s: failed to find panel prepare funcs\n", __func__);
		return -ENODEV;
	}

	return panel->funcs->prepare(state);
}

I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_dsi_panel.c
static int rockchip_dsi_panel_prepare(struct display_state *state)
{
	struct panel_state *panel_state = &state->panel_state;
	struct rockchip_dsi_panel *panel = panel_state->private;
	int ret;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	fdtdec_set_gpio(&panel->enable_gpio, 1);
	msleep(panel->delay_prepare);

	fdtdec_set_gpio(&panel->reset_gpio, 1);
	msleep(panel->delay_reset);
	fdtdec_set_gpio(&panel->reset_gpio, 0);

	msleep(panel->delay_init);

	if (panel->on_cmds) {
		ret = rockchip_dsi_panel_send_cmds(state, panel->on_cmds);
		if (ret)
			printf("failed to send on cmds: %d\n", ret);
	}

	return 0;
}

```


#### （五）、vop, dsi, panel三大模块enable分析
#### 5.1、vop模块enable分析
##### 5.1.1、rockchip_display_init()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_vop.c
static int rockchip_vop_enable(struct display_state *state)
{
	struct crtc_state *crtc_state = &state->crtc_state;
	struct vop *vop = crtc_state->private;

	VOP_CTRL_SET(vop, standby, 0);
	vop_cfg_done(vop);

	return 0;
}


```
#### 5.2、dsi模块enable分析
##### 5.2.1、rockchip_dw_mipi_dsi_enable()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip-dw-mipi-dsi.c
static int rockchip_dw_mipi_dsi_enable(struct display_state *state)
{
	struct connector_state *conn_state = &state->conn_state;
	struct dw_mipi_dsi *dsi = conn_state->private;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	dw_mipi_dsi_set_mode(dsi, DSI_VIDEO_MODE);
	if (dsi->slave)
		dw_mipi_dsi_set_mode(dsi->slave, DSI_VIDEO_MODE);

	return 0;
}

```


#### 5.3、panel模块enable分析
##### 5.3.1、rockchip_dw_mipi_dsi_enable()
``` c
I:\RK3399_Android8.1_MIPI\u-boot\drivers\video\rockchip_dsi_panel.c
static int rockchip_dsi_panel_enable(struct display_state *state)
{
	struct panel_state *panel_state = &state->panel_state;
	struct rockchip_dsi_panel *panel = panel_state->private;
        printf("zjj.sye.display %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
	msleep(panel->delay_enable);

#ifdef CONFIG_RK_PWM_BL
	rk_pwm_bl_config(-1);
#endif
	return 0;
}

```
#### (六)、参考资料(特别感谢)：


[（1）【[RK3399][Android7.1] Uboot display 加载过程小结】](
https://blog.csdn.net/kris_fei/article/details/79003925) 

[（2）【Connect to TS050 Touchscreen】](
https://docs.khadas.com/zh-cn/edge/ConnectLcd.html#3-%EF%BC%88MIPI-HDMI%EF%BC%89%E5%B1%8F%E5%B9%95%E9%85%8D%E7%BD%AE) 

[（3）【Firefly-RK3399 LCD使用】](
http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/driver_lcd.html) 

[（4）【RK3288——LCD裸机】](
https://hceng.cn/2018/07/19/RK3288%E2%80%94%E2%80%94LCD%E8%A3%B8%E6%9C%BA/)
