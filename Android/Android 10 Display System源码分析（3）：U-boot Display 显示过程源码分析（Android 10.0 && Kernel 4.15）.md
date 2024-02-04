---
title: Android 10 Display System源码分析（3）：U-boot Display 显示过程源码分析（Android 10.0 && Kernel 4.15）
cover: https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.24.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20210510
date: 2021-05-10 09:25:00
---

== 源码（部分）==

> U-boot

- X:\u-boot\drivers\video\drm\rockchip_*.c
- X:\u-boot\drivers\video\drm\dw_mipi_dsi.c
- X:\u-boot\drivers\pinctrl\pinctrl-rockchip.c
- X:\u-boot\drivers\video\pwm_backlight.c

![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/Android10.Display.3/Android10.Display.3.TestColor.gif)

显示模块主要分 vop, dsi, panel 三大模块，另加 gpio, 背光的控制，另外还有 logo 的解析和加载。整个流程基本上就是解析各个模块参数，然后准备，使能各个模块。

![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/Android10.Display.3/khadas-edge-v-rk3399-mipi-dsi-framwork.jpg)


## (一)、MIPI 屏幕配置
### （1）、LCD 使用
Edge-V 开发板外置了 3 个 LCD 屏接口：HDMI + MIPI + EDP。接口对应板子上的位置如下图，我们这里是 mipi：
![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/Android10.Display.3/edge-v_display_interfaces.jpg)
### （2）,（MIPI + HDMI）屏幕配置
![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/Android10.Display.3/edge_v_ts050.jpg)
### （3）,配置 dts
#### 3.1,使能对应显示设备节点

``` bash
&dp_in_vopl {
    status = "disabled";
};
​
&dp_in_vopb {
    status = "okay";
};
​
&hdmi_in_vopb {
    status = "okay";
};
​
&hdmi_in_vopl {
    status = "disabled";
};
​
&route_hdmi {
    status = "okay";
    connect = <&vopb_out_hdmi>;
};
​
&route_dsi {
    status = "okay";
    connect = <&vopl_out_dsi>;
};
​
&dsi_in_vopb {
    status = "disabled";
};
​
&dsi_in_vopl {
    status = "okay";
};
​
```

#### 3.2, 绑定 PLL
rk3399 的 hdmi 所绑定的 vop 时钟需要挂载到 vpll 上，若是双显需将另一个 vop 时钟挂到 cpll 这样可以分出任意 dclk 的频率。

``` bash
&vopb {
    status = "okay";
    assigned-clocks = <&cru DCLK_VOP0_DIV>;
    assigned-clock-parents = <&cru PLL_VPLL>;
};
​
&vopl {
    status = "okay";
    assigned-clocks = <&cru DCLK_VOP1_DIV>;
    assigned-clock-parents = <&cru PLL_CPLL>;
};
```
#### 3.3,配置 timing
![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/Android10.Display.3/Android10.Display.3.MIPI-DSI.png)

``` bash
&dsi {
    status = "okay";
    rockchip,lane-rate = <1000>;
    dsi_panel: panel@0 {
        status = "okay";
        compatible = "simple-panel-dsi";
        reg = <0>;
        backlight = <&backlight>;
        reset-gpios = <&gpio4 RK_PD4 GPIO_ACTIVE_HIGH>; /* GPIO4_D4 */
        enable-gpios = <&gpio4 RK_PD5 GPIO_ACTIVE_HIGH>; /* GPIO4_D5 */
//      pinctrl-names = "default";
//      pinctrl-0 = <&lcd_reset_gpio>, <&lcd_enable_gpio>;
        reset-delay-ms = <10>;
        enable-delay-ms = <60>;
        prepare-delay-ms = <60>;
        unprepare-delay-ms = <60>;
        disable-delay-ms = <60>;
        init-delay-ms = <20>;
        dsi,flags = <(MIPI_DSI_MODE_VIDEO | MIPI_DSI_MODE_VIDEO_BURST |
                  MIPI_DSI_MODE_LPM | MIPI_DSI_MODE_EOT_PACKET)>;
​
        dsi,format = <MIPI_DSI_FMT_RGB888>;
        dsi,lanes  = <4>;
        panel-init-sequence = [
            15 00 02  FF 05
            ......
            05 0A 01  29
        ];
        panel-exit-sequence = [
            05 05 01 28
            05 78 01 10
        ];
​
        disp_timings: display-timings {
            native-mode = <&timing0>;
            timing0: timing0 {
                clock-frequency = <120000000>;
                hactive = <1088>; /* default 1080, but afbdc must 16 align */
                hfront-porch = <104>;
                hback-porch = <127>;
                hsync-len = <4>; /////
                vactive = <1920>;
                vfront-porch = <4>;
                vback-porch = <3>;
                vsync-len = <2>; /////
                hsync-active = <0>;
                vsync-active = <0>;
                de-active = <0>;
                pixelclk-active = <0>;
            };
        };
    };
};
​
```
![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/Android10.Display.3/timing_attribute_reference_figure.png)
![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/Android10.Display.3/timing_attribute_reference_figure_2.png)

**命令格式说明**

> panel-exit-sequence = [ 
> 05 05 01 28 
> 05 78 01 10
> ];

命令的前面三个字节分别表示命令类型、延时和命令净荷长度。从第四个字节开始表示命令的有效 payload。这个字节数需要与第三个字节一致。
第一个字节代表的命令类型分两大类：DCS 命令和 Generic 命令。DCS 命令是有 mipi 标准协议里面指定的命令，具体可以参考《MIPI Alliance Specification for Display Command Set.pdf 》。Generic 命令一般应用于厂商自定义的命令。具体使用哪种类型需要参考屏规格书或者咨询屏厂 FAE。如果没有特别指定，建议按 DCS 命令类型配置。DCS 命 令 的 类 型 有 三 种 ： 0x05/0x15/0x39 。 Generic 命 令 的 类 型 分 为 ：
0x03/0x13/0x23/0x29。

### 3.4, 背光 backlight
pwms 属性：配置 PWM，MIPI 屏使用的 pwm 输出是 pwm1，25000ns 是周期 (40 KHz)。

``` bash
&backlight {
    pwms = <&pwm1 0 25000 0>;
    status = "okay";
};
​
​
&pwm1 {
    status = "okay";
};
```

## (二)、U-boot Display 显示过程分析
首先看看全部 Log：

``` xml
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_display_probe 1531 
[2020/9/11 15:42:21] init_display_buffer 0x7df00000zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_of_find_connector 1473 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_of_find_bridge 1416 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_of_find_panel 1355 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c connector_panel_init 244 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c connector_phy_init 212 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_of_find_connector 1473 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c dw_mipi_dsi_probe 606 
[2020/9/11 15:42:21] zjj.rk3399.uboot dw_mipi_dsi_probe dsi->id 0x0 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_of_find_bridge 1416 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_of_find_panel 1355 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c dw_mipi_dsi_child_pre_probe 729 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c dw_mipi_dsi_host_attach 675 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_panel.c rockchip_panel_ofdata_to_platdata 361 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_panel.c rockchip_panel_parse_cmds 99 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_panel.c rockchip_panel_parse_cmds 99 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_panel.c rockchip_panel_probe 422 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c connector_panel_init 244 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c connector_phy_init 212 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/clk/rockchip/clk_rk3399.c rk3399_clk_set_rate 1233 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c display_logo 956 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c display_init 701 
[2020/9/11 15:42:21] Rockchip UBOOT DRM driver version: v1.0.1
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_vop_preinit 585 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c display_logo 956 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c display_init 701 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_vop_preinit 585 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c panel_simple_init 577 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c dw_mipi_dsi_connector_init 380 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c display_get_timing 541 
[2020/9/11 15:42:21] Using display timing dts
[2020/9/11 15:42:21] Detailed mode clock 120000 kHz, flags[a]
[2020/9/11 15:42:21]     H: 1088 1192 1196 1323
[2020/9/11 15:42:21]     V: 1920 1924 1926 1929
[2020/9/11 15:42:21] bus_format: 100e
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_vop_init 616 
[2020/9/11 15:42:21] vop->regs ret=00000000ff8f0000
[2020/9/11 15:42:21] vop->grf ret=00000000ff770000
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/clk/rockchip/clk_rk3399.c rk3399_dclk_vop_set_parent 1347 
[2020/9/11 15:42:21] mode->clock ret=120000
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/clk/rockchip/clk_rk3399.c rk3399_clk_set_rate 1233 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/clk/rockchip/clk_rk3399.c rk3399_vop_set_clk 761 
[2020/9/11 15:42:21] zjj.rk3399.uboot aclkreg_addr 00000000ff7601c0 
[2020/9/11 15:42:21] zjj.rk3399.uboot dclkreg_addr 00000000ff7601c8 
[2020/9/11 15:42:21] PLL at 00000000ff760060: fbdiv=35, refdiv=1, postdiv1=7, postdiv2=1, vco=840000 khz, output=120000 khz
[2020/9/11 15:42:21] PLL at &pll_con[3] : 00000000ff76006c PLL at &pll_con[0] : 00000000ff760060 PLL at &pll_con[1] : 00000000ff760064 PLL at &pll_con[2] : 00000000ff760068 zjj.rk3399.uboot &cru->cpll_con[0] 00000000ff760060 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/zhoujinjian_mipi_rockchip_display.c zjj_rockchip_vop_init 602 
[2020/9/11 15:42:21] dma_addr 7df06000
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_vop_set_plane 670 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/zhoujinjian_mipi_rockchip_display.c zjj_rockchip_vop_set_plane 695 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c display_enable 870 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c dw_mipi_dsi_connector_prepare 541 
[2020/9/11 15:42:21] final DSI-Link bandwidth: 0 Mbps x 4
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write name VOPSEL
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write vop->grf:00000000ff770000 + reg:0x6250  GENMASK(msb:0x0, lsb:0x0) val:0x1 << lsb:0x0
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c dw_mipi_dsi_pre_enable 519 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c dw_mipi_dsi_host_init 400 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c mipi_dphy_init 458 
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write name TURNREQUEST
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write vop->grf:00000000ff770000 + reg:0x6258  GENMASK(msb:0xf, lsb:0xc) val:0x0 << lsb:0xc
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write name TURNDISABLE
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write vop->grf:00000000ff770000 + reg:0x6258  GENMASK(msb:0xb, lsb:0x8) val:0x0 << lsb:0x8
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write name FORCETXSTOPMODE
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write vop->grf:00000000ff770000 + reg:0x6258  GENMASK(msb:0x7, lsb:0x4) val:0x0 << lsb:0x4
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write name FORCERXMODE
[2020/9/11 15:42:21] zjj.rk3399.uboot grf_field_write vop->grf:00000000ff770000 + reg:0x6258  GENMASK(msb:0x3, lsb:0x0) val:0x0 << lsb:0x0
[2020/9/11 15:42:21] test_code=0x44, test_data=0x34, monitor_data=0x34
[2020/9/11 15:42:21] test_code=0x60, test_data=0x87, monitor_data=0x87
[2020/9/11 15:42:21] test_code=0x70, test_data=0x87, monitor_data=0x87
[2020/9/11 15:42:21] test_code=0x19, test_data=0x30, monitor_data=0x30
[2020/9/11 15:42:21] test_code=0x17, test_data=0x03, monitor_data=0x03
[2020/9/11 15:42:21] test_code=0x18, test_data=0x05, monitor_data=0x05
[2020/9/11 15:42:21] test_code=0x18, test_data=0x85, monitor_data=0x05
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c mipi_dphy_power_on 188 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_panel.c panel_simple_prepare 189 
[2020/9/11 15:42:21] zjj.rk3399.uboot drivers/video/drm/rockchip_panel.c rockchip_panel_send_dsi_cmds 145 
[2020/9/11 15:42:22] zjj.rk3399.uboot drivers/video/drm/rockchip_display.c rockchip_vop_enable 677 
[2020/9/11 15:42:22] zjj.rk3399.uboot drivers/video/drm/zhoujinjian_mipi_rockchip_display.c zjj_rockchip_vop_enable 723 
[2020/9/11 15:42:22] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c dw_mipi_dsi_connector_enable 574 
[2020/9/11 15:42:22] zjj.rk3399.uboot drivers/video/drm/dw_mipi_dsi.c dw_mipi_dsi_enable 340 
[2020/9/11 15:42:22] zjj.rk3399.uboot drivers/video/drm/rockchip_panel.c panel_simple_enable 269 
[2020/9/11 15:42:22] zjj rk_pwm_bl_config.
[2020/9/11 15:42:22] pwm_1->base 0xff420010
​
[END] 2020/9/11 15:42:59
​
```
#### U-boot Display 显示过程流程图


----------

![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/Android10.Display.3/Android10.Display.3.UBoot.Display.Flow.png)

## （三）、vop, dsi, panel 三大模块 init 分析

``` c
X:\u-boot\drivers\video\drm\rockchip_display.c
static int display_init(struct display_state *state)
{
    struct connector_state *conn_state = &state->conn_state;
    struct panel_state *panel_state = &state->panel_state;
    const struct rockchip_connector *conn = conn_state->connector;
    const struct rockchip_connector_funcs *conn_funcs = conn->funcs;
    struct crtc_state *crtc_state = &state->crtc_state;
    struct rockchip_crtc *crtc = crtc_state->crtc;
    const struct rockchip_crtc_funcs *crtc_funcs = crtc->funcs;
    struct drm_display_mode *mode = &conn_state->mode;
    int bpc;
    int ret = 0;
    static bool __print_once = false;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    ......
    if (crtc_funcs->preinit) {
        ret = rockchip_vop_preinit(state);
        if (ret)
            return ret;
    }
​
    if (panel_state->panel)
        panel_simple_init(panel_state->panel);
​
    if (conn_funcs->init) {
        ret = conn_funcs->init(state);
        if (ret)
            goto deinit;
    }
​
    if (conn_state->phy)
        rockchip_phy_init(conn_state->phy);
   ......
​
    if (panel_state->panel) {
        ret = display_get_timing(state);
    } else if (conn_state->bridge) {
        ret = video_bridge_read_edid(conn_state->bridge->dev,
                         conn_state->edid, EDID_SIZE);
        if (ret > 0) {
            ret = edid_get_drm_mode(conn_state->edid, ret, mode,
                        &bpc);
            if (!ret)
                edid_print_info((void *)&conn_state->edid);
        } else {
            ret = video_bridge_get_timing(conn_state->bridge->dev);
        }
    } else if (conn_funcs->get_timing) {
        ret = conn_funcs->get_timing(state);
    } else if (conn_funcs->get_edid) {
        ret = conn_funcs->get_edid(state);
        if (!ret) {
            ret = edid_get_drm_mode((void *)&conn_state->edid,
                        sizeof(conn_state->edid), mode,
                        &bpc);
            if (!ret)
                edid_print_info((void *)&conn_state->edid);
        }
    }
​
    if (ret)
        goto deinit;
​
    drm_mode_set_crtcinfo(mode, CRTC_INTERLACE_HALVE_V);
​
    if (crtc_funcs->init) {
        //zjj_rockchip_vop_init();
        ret = rockchip_vop_init(state);
        if (ret)
            goto deinit;
    }
    state->is_init = 1;
​
    crtc_state->crtc->active = true;
    memcpy(&crtc_state->crtc->active_mode,
           &conn_state->mode, sizeof(struct drm_display_mode));
​
    return 0;
​
deinit:
    if (conn_funcs->deinit)
        conn_funcs->deinit(state);
    return ret;
}
```

### 3.1、Panel 模块 init 分析
#### 3.1.1、panel_simple_init()

``` c
X:\u-boot\drivers\video\drm\rockchip_panel.c
static void panel_simple_init(struct rockchip_panel *panel)
{
    struct display_state *state = panel->state;
    struct connector_state *conn_state = &state->conn_state;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    conn_state->bus_format = panel->bus_format;
}
​
```

### 3.2、DSI 模块 init 分析
#### 3.2.1、dw_mipi_dsi_connector_init()

``` c
X:\u-boot\drivers\video\drm\dw_mipi_dsi.c
static int dw_mipi_dsi_connector_init(struct display_state *state)
{
    struct connector_state *conn_state = &state->conn_state;
    //struct dw_mipi_dsi *dsi = dev_get_priv(conn_state->dev);
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    //dsi->dphy.phy = conn_state->phy;
​
    conn_state->output_mode = ROCKCHIP_OUT_MODE_P888;
    conn_state->color_space = V4L2_COLORSPACE_DEFAULT;
    conn_state->type = DRM_MODE_CONNECTOR_DSI;
    //printf("zjj.rk3399.uboot dsi:%p , dsi->id %d \n",&dsi , dsi->id);
​
    return 0;
}
```

### 3.3、Vop 模块 init 分析
#### 3.3.1、rockchip_vop_init()

``` c
static int rockchip_vop_init(struct display_state *state)
{
    struct crtc_state *crtc_state = &state->crtc_state;
    struct connector_state *conn_state = &state->conn_state;
    struct drm_display_mode *mode = &conn_state->mode;
    const struct rockchip_crtc *crtc = crtc_state->crtc;
    const struct vop_data *vop_data = crtc->data;
    struct vop *vop;
    struct clk dclk;
    int ret;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    vop = malloc(sizeof(*vop));
​
    memset(vop, 0, sizeof(*vop));
​
    crtc_state->private = vop;
​
    vop->regs = dev_read_addr_ptr(crtc_state->dev);
    printf("vop->regs ret=%p\n", vop->regs);
    ......
    /* Process 'assigned-{clocks/clock-parents/clock-rates}' properties */
    ret = clk_set_defaults(crtc_state->dev);
    if (ret)
        debug("%s clk_set_defaults failed %d\n", __func__, ret);
​
    ret = clk_get_by_name(crtc_state->dev, "dclk_vop", &dclk);
    printf("mode->clock ret=%d\n",mode->clock);
    if (!ret)
        ret = clk_set_rate(&dclk, mode->clock * 1000);
    if (IS_ERR_VALUE(ret)) {
        printf("%s: Failed to set dclk: ret=%d\n", __func__, ret);
        return ret;
    }
​
    memcpy(vop->regsbak, vop->regs, vop_data->reg_len);
​
    //rockchip_vop_init_gamma(vop, state);
​
    zjj_rockchip_vop_init(vop->regs, vop->regsbak, vop_data->reg_len);
​
    return 0;
}
```

#### 3.3.2、zjj_rockchip_vop_init()

``` c
X:\u-boot\drivers\video\drm\zhoujinjian_mipi_rockchip_display.c
struct zjj_vop_reg {
    uint32_t mask;
    uint32_t offset:12;
    uint32_t shift:5;
    uint32_t begin_minor:4;
    uint32_t end_minor:4;
    uint32_t major:3;
    uint32_t write_mask:1;
};
/* zjj_rockchip_vop_init */
struct zjj_vop_reg global_regdone_en = VOP_REG_VER(RK3399_SYS_CTRL, 0x1, 11, 3, 2, -1);
struct zjj_vop_reg axi_outstanding_max_num = VOP_REG(RK3399_SYS_CTRL1, 0x1f, 13);
struct zjj_vop_reg axi_max_outstanding_en = VOP_REG(RK3399_SYS_CTRL1, 0x1, 12);
struct zjj_vop_reg win_gate0 = VOP_REG(RK3399_WIN2_CTRL0, 0x1, 0);
struct zjj_vop_reg win_gate1 = VOP_REG(RK3399_WIN3_CTRL0, 0x1, 0);
struct zjj_vop_reg dsp_blank = VOP_REG(RK3399_DSP_CTRL0, 0x3, 18);
struct zjj_vop_reg mipi_en = VOP_REG(RK3399_SYS_CTRL, 0x1, 15);
struct zjj_vop_reg mipi_pin_pol = VOP_REG_VER(RK3399_DSP_CTRL1, 0x7, 28, 3, 2, -1);
struct zjj_vop_reg mipi_dclk_pol = VOP_REG_VER(RK3399_DSP_CTRL1, 0x1, 31, 3, 2, -1);
struct zjj_vop_reg mipi_dual_channel_en = VOP_REG(RK3399_SYS_CTRL, 0x1, 3);
struct zjj_vop_reg data01_swap = VOP_REG_VER(RK3399_SYS_CTRL, 0x1, 17, 3, 5, -1);
struct zjj_vop_reg dither_down = VOP_REG(RK3399_DSP_CTRL1, 0xf, 1);
struct zjj_vop_reg dclk_ddr = VOP_REG_VER(RK3399_DSP_CTRL0, 0x1, 8, 3, 4, -1);
struct zjj_vop_reg dsp_data_swap = VOP_REG(RK3399_DSP_CTRL0, 0x1f, 12);
struct zjj_vop_reg out_mode = VOP_REG(RK3399_DSP_CTRL0, 0xf, 0);
struct zjj_vop_reg overlay_mode = VOP_REG_VER(RK3399_SYS_CTRL, 0x1, 16, 3, 2, -1);
struct zjj_vop_reg dsp_out_yuv = VOP_REG_VER(RK3399_POST_SCL_CTRL, 0x1, 2, 3, 5, -1);
struct zjj_vop_reg bcsh_r2y_en = VOP_REG_VER(RK3399_BCSH_CTRL, 0x1, 4, 3, 1, -1);
struct zjj_vop_reg bcsh_y2r_en = VOP_REG_VER(RK3399_BCSH_CTRL, 0x1, 0, 3, 1, -1);
struct zjj_vop_reg bcsh_r2y_csc_mode = VOP_REG_VER(RK3399_BCSH_CTRL, 0x1, 6, 3, 1, -1);
struct zjj_vop_reg bcsh_y2r_csc_mode = VOP_REG_VER(RK3399_BCSH_CTRL, 0x3, 2, 3, 1, -1);
struct zjj_vop_reg dsp_background = VOP_REG(RK3399_DSP_BG, 0xffffffff, 0);
struct zjj_vop_reg htotal_pw = VOP_REG(RK3399_DSP_HTOTAL_HS_END, 0x1fff1fff, 0);
struct zjj_vop_reg hact_st_end = VOP_REG(RK3399_DSP_HACT_ST_END, 0x1fff1fff, 0);
struct zjj_vop_reg vact_st_end = VOP_REG(RK3399_DSP_VACT_ST_END, 0x1fff1fff, 0);
struct zjj_vop_reg dsp_interlace = VOP_REG(RK3399_DSP_CTRL0, 0x1, 10);
struct zjj_vop_reg p2i_en = VOP_REG_VER(RK3399_DSP_CTRL0, 0x1, 5, 3, 4, -1);
struct zjj_vop_reg vtotal_pw = VOP_REG(RK3399_DSP_VTOTAL_VS_END, 0x1fff1fff, 0);
struct zjj_vop_reg hpost_st_end = VOP_REG(RK3399_POST_DSP_HACT_INFO, 0x1fff1fff, 0);
struct zjj_vop_reg vpost_st_end = VOP_REG(RK3399_POST_DSP_VACT_INFO, 0x1fff1fff, 0);
struct zjj_vop_reg post_scl_factor = VOP_REG(RK3399_POST_SCL_FACTOR_YRGB, 0xffffffff, 0);
struct zjj_vop_reg post_scl_ctrl = VOP_REG(RK3399_POST_SCL_CTRL, 0x3, 0);
struct zjj_vop_reg core_dclk_div = VOP_REG_VER(RK3399_DSP_CTRL0, 0x1, 4, 3, 4, -1);
struct zjj_vop_reg line_flag_num0 = VOP_REG(RK3399_LINE_FLAG, 0xffff, 0);
struct zjj_vop_reg line_flag_num1 = VOP_REG(RK3399_LINE_FLAG, 0xffff, 16);
​
void zjj_rockchip_vop_init(void * reg, u32 * rbak, int len)
{
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    regsbak = rbak;
    regs = reg;
​
    vop_mask_write(regs, global_regdone_en.offset, global_regdone_en.mask, global_regdone_en.shift, 
        1, global_regdone_en.write_mask);
    vop_mask_write(regs, axi_outstanding_max_num.offset, axi_outstanding_max_num.mask, axi_outstanding_max_num.shift, 
        30, axi_outstanding_max_num.write_mask);
    vop_mask_write(regs, axi_max_outstanding_en.offset, axi_max_outstanding_en.mask, axi_max_outstanding_en.shift, 
        1, axi_max_outstanding_en.write_mask);
    vop_mask_write(regs, win_gate0.offset, win_gate0.mask, win_gate0.shift, 
        1, win_gate0.write_mask);
    vop_mask_write(regs, win_gate1.offset, win_gate1.mask, win_gate1.shift, 
        1, win_gate0.write_mask);
    vop_mask_write(regs, dsp_blank.offset, dsp_blank.mask, dsp_blank.shift, 
        0, dsp_blank.write_mask);
    vop_mask_write(regs, mipi_en.offset, mipi_en.mask, mipi_en.shift, 
        1, mipi_en.write_mask);
    vop_mask_write(regs, mipi_pin_pol.offset, mipi_pin_pol.mask, mipi_pin_pol.shift, 
        8, mipi_pin_pol.write_mask);
    vop_mask_write(regs, mipi_dclk_pol.offset, mipi_dclk_pol.mask, mipi_dclk_pol.shift, 
        1, mipi_dclk_pol.write_mask);
    vop_mask_write(regs, mipi_dual_channel_en.offset, mipi_dual_channel_en.mask, mipi_dual_channel_en.shift, 
        0, mipi_dual_channel_en.write_mask);
​
    vop_mask_write(regs, data01_swap.offset, data01_swap.mask, data01_swap.shift, 
        0, data01_swap.write_mask);
​
    vop_mask_write(regs, dither_down.offset, dither_down.mask, dither_down.shift, 
        1, dither_down.write_mask);
​
    vop_mask_write(regs, dclk_ddr.offset, dclk_ddr.mask, dclk_ddr.shift, 
        0, dclk_ddr.write_mask);
    vop_mask_write(regs, dsp_data_swap.offset, dsp_data_swap.mask, dsp_data_swap.shift, 
        0, dsp_data_swap.write_mask);
    vop_mask_write(regs, dsp_data_swap.offset, dsp_data_swap.mask, dsp_data_swap.shift, 
        0, dsp_data_swap.write_mask);
​
    vop_mask_write(regs, out_mode.offset, out_mode.mask, out_mode.shift, 
        0, out_mode.write_mask);
    vop_mask_write(regs, overlay_mode.offset, overlay_mode.mask, overlay_mode.shift, 
        0, overlay_mode.write_mask);
​
    vop_mask_write(regs, dsp_out_yuv.offset, dsp_out_yuv.mask, dsp_out_yuv.shift, 
        0, dsp_out_yuv.write_mask);
​
    vop_mask_write(regs, bcsh_r2y_en.offset, bcsh_r2y_en.mask, bcsh_r2y_en.shift, 
        0, bcsh_r2y_en.write_mask);
    vop_mask_write(regs, bcsh_y2r_en.offset, bcsh_y2r_en.mask, bcsh_y2r_en.shift, 
        0, bcsh_y2r_en.write_mask);
    vop_mask_write(regs, bcsh_r2y_csc_mode.offset, bcsh_r2y_csc_mode.mask, bcsh_r2y_csc_mode.shift, 
        1, bcsh_r2y_csc_mode.write_mask);
    vop_mask_write(regs, bcsh_y2r_csc_mode.offset, bcsh_y2r_csc_mode.mask, bcsh_y2r_csc_mode.shift, 
        1, bcsh_y2r_csc_mode.write_mask);
    vop_mask_write(regs, htotal_pw.offset, htotal_pw.mask, htotal_pw.shift, 
        0x52b0004, htotal_pw.write_mask);
    vop_mask_write(regs, hact_st_end.offset, hact_st_end.mask, hact_st_end.shift, 
        0x8304c3, hact_st_end.write_mask);
    vop_mask_write(regs, vact_st_end.offset, vact_st_end.mask, vact_st_end.shift, 
        0x50785, vact_st_end.write_mask);
    vop_mask_write(regs, dsp_interlace.offset, dsp_interlace.mask, dsp_interlace.shift, 
        0, dsp_interlace.write_mask);
    vop_mask_write(regs, p2i_en.offset, p2i_en.mask, p2i_en.shift, 
        0, p2i_en.write_mask);
    vop_mask_write(regs, vtotal_pw.offset, vtotal_pw.mask, vtotal_pw.shift, 
         0x7890002, vtotal_pw.write_mask);
​
    vop_mask_write(regs, hpost_st_end.offset, hpost_st_end.mask, hpost_st_end.shift, 
         0x8304c3, hpost_st_end.write_mask);
​
    vop_mask_write(regs, vpost_st_end.offset, vpost_st_end.mask, vpost_st_end.shift, 
         0x50785, vpost_st_end.write_mask);
​
    vop_mask_write(regs, post_scl_factor.offset, post_scl_factor.mask, post_scl_factor.shift, 
         0x10001000 , post_scl_factor.write_mask);
    vop_mask_write(regs, post_scl_ctrl.offset, post_scl_ctrl.mask, post_scl_ctrl.shift, 
         0 , post_scl_ctrl.write_mask);
    
    vop_mask_write(regs, core_dclk_div.offset, core_dclk_div.mask, core_dclk_div.shift, 
         0 , core_dclk_div.write_mask);
    
    vop_mask_write(regs, line_flag_num0.offset, line_flag_num0.mask, line_flag_num0.shift, 
         0x782 , line_flag_num0.write_mask);
    vop_mask_write(regs, line_flag_num1.offset, line_flag_num1.mask, line_flag_num1.shift, 
         0x72b , line_flag_num1.write_mask);
    vop_mask_write(regs, cfg_done.offset, cfg_done.mask, cfg_done.shift, 
         1 , cfg_done.write_mask);
​
}
​
```

#### 3.3.3、zjj_rockchip_vop_set_plane()​​​​​​​

``` c
X:\u-boot\drivers\video\drm\zhoujinjian_mipi_rockchip_display.c
/* zjj_rockchip_vop_set_plane*/
struct zjj_vop_reg ymirror = VOP_REG(RK3399_DSP_CTRL0, 0x1, 23);
struct zjj_vop_reg xmirror = VOP_REG(RK3399_DSP_CTRL0, 0x1, 22);
struct zjj_vop_reg format = VOP_REG(RK3399_WIN2_CTRL0, 0x3, 5);
struct zjj_vop_reg yrgb_vir = VOP_REG(RK3399_WIN2_VIR0_1, 0x1fff, 0);
struct zjj_vop_reg yrgb_mst = VOP_REG(RK3399_WIN2_MST0, 0xffffffff, 0);
struct zjj_vop_reg act_info = VOP_REG(RK3399_WIN0_ACT_INFO, 0x1fff1fff, 0);
struct zjj_vop_reg dsp_info = VOP_REG(RK3399_WIN2_DSP_INFO0, 0x0fff0fff, 0);
struct zjj_vop_reg dsp_st = VOP_REG(RK3399_WIN2_DSP_ST0, 0x1fff1fff, 0);
struct zjj_vop_reg rb_swap = VOP_REG(RK3399_WIN2_CTRL0, 0x1, 20);
struct zjj_vop_reg src_alpha_ctl = VOP_REG(RK3399_WIN2_SRC_ALPHA_CTRL, 0xffff, 0);
struct zjj_vop_reg enable = VOP_REG(RK3399_WIN2_CTRL0, 0x1, 4);
​
​
void zjj_rockchip_vop_set_plane()
{
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
    vop_mask_write(regs, ymirror.offset, ymirror.mask, ymirror.shift, 
         0 , ymirror.write_mask);
    vop_mask_write(regs, xmirror.offset, xmirror.mask, xmirror.shift, 
         0 , xmirror.write_mask);
    vop_mask_write(regs, format.offset, format.mask, format.shift, 
         2 , format.write_mask);
    vop_mask_write(regs, yrgb_vir.offset, yrgb_vir.mask, yrgb_vir.shift, 
         0x12c , yrgb_vir.write_mask);
    vop_mask_write(regs, yrgb_mst.offset, yrgb_mst.mask, yrgb_mst.shift, 
         0x7df06000 , yrgb_mst.write_mask);
    vop_mask_write(regs, act_info.offset, act_info.mask, act_info.shift, 
         ((1920-1) << 16) |  ((1080-1) & 0xffff), act_info.write_mask);
    vop_mask_write(regs, dsp_info.offset, dsp_info.mask, dsp_info.shift, 
         ((1920-1) << 16) |  ((1080-1) & 0xffff), dsp_info.write_mask);
    vop_mask_write(regs, dsp_st.offset, dsp_st.mask, dsp_st.shift, 
         ((1929 - 1924) << 16) | ((1323 - 1192) & 0xffff)  , dsp_st.write_mask);
    vop_mask_write(regs, rb_swap.offset, rb_swap.mask, rb_swap.shift, 
         1 , rb_swap.write_mask);
    vop_mask_write(regs, src_alpha_ctl.offset, src_alpha_ctl.mask, src_alpha_ctl.shift, 
         0 , src_alpha_ctl.write_mask);
    vop_mask_write(regs, enable.offset, enable.mask, enable.shift, 
         1 , enable.write_mask);
​
}
```

## （四）、vop, dsi, panel 三大模块 init 分析
### 4.1、vop 模块 prepare 分析
#### 4.1.1、rockchip_vop_prepare()

``` c
static int rockchip_vop_prepare(struct display_state *state)
{
    return 0;
}
```

### 4.2、DSI 模块 prepare 分析
#### 4.2.1、dw_mipi_dsi_connector_prepare()

``` c
X:\u-boot\drivers\video\drm\dw_mipi_dsi.c
static int dw_mipi_dsi_connector_prepare(struct display_state *state)
{
    struct connector_state *conn_state = &state->conn_state;
    //struct crtc_state *crtc_state = &state->crtc_state;
    struct dw_mipi_dsi *dsi = dev_get_priv(conn_state->dev);
    //unsigned long lane_rate;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    memcpy(&dsi->mode, &conn_state->mode, sizeof(struct drm_display_mode));
    //if (dsi->dphy.phy)
    //  dw_mipi_dsi_set_hs_clk(dsi, lane_rate);
    //else
    //dw_mipi_dsi_set_pll(dsi, 1000000000);
​
    //printf("final DSI-Link lane_rate: lane_rate %ld crtc_id %d\n",lane_rate,crtc_state->crtc_id);
​
    printf("final DSI-Link bandwidth: %u Mbps x %d\n",
           dsi->lane_mbps, dsi->slave ? dsi->lanes * 2 : dsi->lanes);
​
    //dw_mipi_dsi_vop_routing(dsi, 1);
    grf_field_write(dsi, VOPSEL, 1);
​
    dw_mipi_dsi_pre_enable(dsi);
​
    return 0;
}
​
static void dw_mipi_dsi_pre_enable(struct dw_mipi_dsi *dsi)
{
    if (dsi->prepared)
        return;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    if (dsi->master)
        dw_mipi_dsi_pre_enable(dsi->master);
​
    dw_mipi_dsi_host_init(dsi);
    mipi_dphy_init(dsi);
    mipi_dphy_power_on(dsi);
    dsi_write(dsi, DSI_PWR_UP, POWERUP);
​
    dsi->prepared = true;
​
    //if (dsi->slave)
    //  dw_mipi_dsi_pre_enable(dsi->slave);
}
​
static void dw_mipi_dsi_host_init(struct dw_mipi_dsi *dsi)
{
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    //dw_mipi_dsi_init(dsi);
    dsi_write(dsi, DSI_PWR_UP, RESET);
    dsi_write(dsi, DSI_CLKMGR_CFG, 0xa07);
​
    //dw_mipi_dsi_dpi_config(dsi, &dsi->mode);
    dsi_write(dsi, DSI_DPI_VCID, 0x0);
    dsi_write(dsi, DSI_DPI_COLOR_CODING, 0x05);//MIPI_DSI_FMT_RGB888:DPI_COLOR_CODING_24BIT
    dsi_write(dsi, DSI_DPI_CFG_POL, 0x06);
    dsi_write(dsi, DSI_DPI_LP_CMD_TIM, OUTVACT_LPCMD_TIME(4)| INVACT_LPCMD_TIME(4));
    //dw_mipi_dsi_packet_handler_config(dsi);
    dsi_write(dsi, DSI_PCKHDL_CFG, 0x1c);
​
    //dw_mipi_dsi_video_mode_config(dsi);
    dsi_write(dsi, DSI_VID_MODE_CFG, 0x3f02);
    //dw_mipi_dsi_video_packet_config(dsi, &dsi->mode);
    dsi_write(dsi, DSI_VID_PKT_SIZE, VID_PKT_SIZE(1088));
​
    //dw_mipi_dsi_command_mode_config(dsi);
    dsi_write(dsi, DSI_TO_CNT_CFG, HSTX_TO_CNT(1000) | LPRX_TO_CNT(1000));
    dsi_write(dsi, DSI_BTA_TO_CNT, 0xd00);
​
    dsi_update_bits(dsi, DSI_MODE_CFG, CMD_VIDEO_MODE, COMMAND_MODE);
​
    //dw_mipi_dsi_line_timer_config(dsi);
    dsi_write(dsi, DSI_VID_HLINE_TIME, 0x55d);
    dsi_write(dsi, DSI_VID_HSA_TIME, 0x04);
    dsi_write(dsi, DSI_VID_HBP_TIME, 0x84);
​
    //dw_mipi_dsi_vertical_timing_config(dsi);
    dsi_write(dsi, DSI_VID_VACTIVE_LINES, 1920);
    dsi_write(dsi, DSI_VID_VSA_LINES, 0x02);
    dsi_write(dsi, DSI_VID_VFP_LINES, 0x04);
    dsi_write(dsi, DSI_VID_VBP_LINES, 0x03);
​
    //dw_mipi_dsi_dphy_timing_config(dsi);
    dsi_write(dsi, DSI_PHY_TMR_CFG, PHY_HS2LP_TIME(0x14)
          | PHY_LP2HS_TIME(0x10) | MAX_RD_TIME(10000));
​
    dsi_write(dsi, DSI_PHY_TMR_LPCLK_CFG, PHY_CLKHS2LP_TIME(0x40)
          | PHY_CLKLP2HS_TIME(0x40));
​
    //dw_mipi_dsi_dphy_interface_config(dsi);
    dsi_write(dsi, DSI_PHY_IF_CFG, PHY_STOP_WAIT_TIME(0x20) |
          N_LANES(4));
​
    //dw_mipi_dsi_clear_err(dsi);
    dsi_read(dsi, DSI_INT_ST0);
    dsi_read(dsi, DSI_INT_ST1);
    dsi_write(dsi, DSI_INT_MSK0, 0);
    dsi_write(dsi, DSI_INT_MSK1, 0);
​
}
​
static void mipi_dphy_init(struct dw_mipi_dsi *dsi)
{
    u32 map[] = {0x0, 0x1, 0x3, 0x7, 0xf};
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    //mipi_dphy_enableclk_deassert(dsi);
    dsi_update_bits(dsi, DSI_PHY_RSTZ, PHY_ENABLECLK, 0);
    udelay(1);
​
    //mipi_dphy_shutdownz_assert(dsi);
    dsi_update_bits(dsi, DSI_PHY_RSTZ, PHY_SHUTDOWNZ, 0);
    udelay(1);
​
    //mipi_dphy_rstz_assert(dsi);
    dsi_update_bits(dsi, DSI_PHY_RSTZ, PHY_RSTZ, 0);
    udelay(1);
​
    //testif_testclr_assert(dsi);
    dsi_update_bits(dsi, DSI_PHY_TST_CTRL0, PHY_TESTCLR, PHY_TESTCLR);
    udelay(1);
​
    /* Configures DPHY to work as a Master */
    grf_field_write(dsi, MASTERSLAVEZ, 1);
​
    /* Configures lane as TX */
    grf_field_write(dsi, BASEDIR, 0);
​
    /* Set all REQUEST inputs to zero */
    grf_field_write(dsi, TURNREQUEST, 0);
    grf_field_write(dsi, TURNDISABLE, 0);
    grf_field_write(dsi, FORCETXSTOPMODE, 0);
    grf_field_write(dsi, FORCERXMODE, 0);
    udelay(1);
​
    //testif_testclr_deassert(dsi);
    dsi_update_bits(dsi, DSI_PHY_TST_CTRL0, PHY_TESTCLR, 0);
    udelay(1);
​
    //if (!dsi->dphy.phy)
    //  dw_mipi_dsi_phy_init(dsi);
    testif_write(dsi, 0x44, HSFREQRANGE(26));
    testif_write(dsi, 0x60, 0x80 | (996 >> 3) * 60 / 1000);
    testif_write(dsi, 0x70, 0x80 | (996 >> 3) * 60 / 1000);
    testif_write(dsi, 0x19, 0x30);
    testif_write(dsi, 0x17, INPUT_DIV(3));
    testif_write(dsi, 0x18, FEEDBACK_DIV_LO(165));
    testif_write(dsi, 0x18, FEEDBACK_DIV_HI(165 >> 5));
​
    /* Enable Data Lane Module */
    grf_field_write(dsi, ENABLE_N, map[dsi->lanes]);
​
    /* Enable Clock Lane Module */
    grf_field_write(dsi, ENABLECLK, 1);
​
    //mipi_dphy_enableclk_assert(dsi);
    dsi_update_bits(dsi, DSI_PHY_RSTZ, PHY_ENABLECLK, PHY_ENABLECLK);
    udelay(1);
​
}
static int mipi_dphy_power_on(struct dw_mipi_dsi *dsi)
{
    u32 mask, val;
    int ret;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    //mipi_dphy_shutdownz_deassert(dsi);
    dsi_update_bits(dsi, DSI_PHY_RSTZ, PHY_SHUTDOWNZ, PHY_SHUTDOWNZ);
    udelay(1);
​
    //mipi_dphy_rstz_deassert(dsi);
    dsi_update_bits(dsi, DSI_PHY_RSTZ, PHY_RSTZ, PHY_RSTZ);
    udelay(1);
    mdelay(2);
​
    //if (dsi->dphy.phy) {
    //  rockchip_phy_set_mode(dsi->dphy.phy, PHY_MODE_VIDEO_MIPI);
    //  rockchip_phy_power_on(dsi->dphy.phy);
    //}
​
    ret = readl_poll_timeout((void*)0xff960000 + DSI_PHY_STATUS,
                 val, val & PHY_LOCK, PHY_STATUS_TIMEOUT_US);
    if (ret < 0) {
        printf("PHY is not locked\n");
        return ret;
    }
​
    udelay(200);
​
    mask = PHY_STOPSTATELANE;
    ret = readl_poll_timeout((void*)0xff960000 + DSI_PHY_STATUS,
                 val, (val & mask) == mask,
                 PHY_STATUS_TIMEOUT_US);
    if (ret < 0) {
        printf("lane module is not in stop state\n");
        return ret;
    }
​
    udelay(10);
​
    return 0;
}
```

### 4.3、panel 模块 prepare 分析
#### 4.3.1、rockchip_panel_prepare()

``` c
X:\u-boot\drivers\video\drm\rockchip_panel.c
​
static void panel_simple_prepare(struct rockchip_panel *panel)
{
    struct rockchip_panel_plat *plat = dev_get_platdata(panel->dev);
    struct rockchip_panel_priv *priv = dev_get_priv(panel->dev);
    struct mipi_dsi_device *dsi = dev_get_parent_platdata(panel->dev);
    int ret;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    if (priv->prepared)
        return;
​
    //if (priv->power_supply)
    //  regulator_set_enable(priv->power_supply, !plat->power_invert);
​
    //if (dm_gpio_is_valid(&priv->enable_gpio))
    //  dm_gpio_set_value(&priv->enable_gpio, 1);
    clrsetbits_le32(0xff790000, OFFSET_TO_BIT(29), 1 ? OFFSET_TO_BIT(29) : 0);
​
    //if (plat->delay.prepare)
        mdelay(60);
​
    //if (dm_gpio_is_valid(&priv->reset_gpio))
    //  dm_gpio_set_value(&priv->reset_gpio, 1);
    clrsetbits_le32(0xff790000, OFFSET_TO_BIT(28), 1 ? OFFSET_TO_BIT(28) : 0);
​
    //if (plat->delay.reset)
        mdelay(10);
​
    //if (dm_gpio_is_valid(&priv->reset_gpio))
    //  dm_gpio_set_value(&priv->reset_gpio, 0);    
    clrsetbits_le32(0xff790000, OFFSET_TO_BIT(28), 0 ? OFFSET_TO_BIT(28) : 0);
​
    //if (plat->delay.reset)
        mdelay(10);
​
    //if (dm_gpio_is_valid(&priv->reset_gpio))
    //  dm_gpio_set_value(&priv->reset_gpio, 1);
    clrsetbits_le32(0xff790000, OFFSET_TO_BIT(28), 1 ? OFFSET_TO_BIT(28) : 0);
​
    if (plat->delay.init)
        mdelay(20);
​
    if (plat->on_cmds) {
            ret = rockchip_panel_send_dsi_cmds(dsi, plat->on_cmds);
        if (ret)
            printf("failed to send on cmds: %d\n", ret);
    }
​
    priv->prepared = true;
}
​
static int rockchip_panel_send_dsi_cmds(struct mipi_dsi_device *dsi,
                    struct rockchip_panel_cmds *cmds)
{
    int i, ret;
​
    if (!cmds)
        return -EINVAL;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    for (i = 0; i < cmds->cmd_cnt; i++) {
        struct rockchip_cmd_desc *desc = &cmds->cmds[i];
        const struct rockchip_cmd_header *header = &desc->header;
​
        switch (header->data_type) {
        case MIPI_DSI_GENERIC_SHORT_WRITE_0_PARAM:
        case MIPI_DSI_GENERIC_SHORT_WRITE_1_PARAM:
        case MIPI_DSI_GENERIC_SHORT_WRITE_2_PARAM:
        case MIPI_DSI_GENERIC_LONG_WRITE:
            ret = mipi_dsi_generic_write(dsi, desc->payload,
                             header->payload_length);
            break;
        case MIPI_DSI_DCS_SHORT_WRITE:
        case MIPI_DSI_DCS_SHORT_WRITE_PARAM:
        case MIPI_DSI_DCS_LONG_WRITE:
            ret = mipi_dsi_dcs_write_buffer(dsi, desc->payload,
                            header->payload_length);
            break;
        default:
            printf("unsupport command data type: %d\n",
                   header->data_type);
            return -EINVAL;
        }
​
        if (ret < 0) {
            printf("failed to write cmd%d: %d\n", i, ret);
            return ret;
        }
​
        if (header->delay_ms)
            mdelay(header->delay_ms);
    }
​
    return 0;
}
​
X:\u-boot\drivers\video\drm\dw_mipi_dsi.c
static ssize_t dw_mipi_dsi_transfer(struct dw_mipi_dsi *dsi,
                    const struct mipi_dsi_msg *msg)
{
    struct mipi_dsi_packet packet;
    int ret;
    int val;
    //printf("zjj.rk3399.uboot  msg->type:0x%x mode_flags:0x%x\n", msg->type ,dsi->mode_flags);
​
    dsi_update_bits(dsi, DSI_VID_MODE_CFG, LP_CMD_EN, LP_CMD_EN);
    dsi_update_bits(dsi, DSI_LPCLK_CTRL, PHY_TXREQUESTCLKHS, 0);
​
​
    switch (msg->type) {
    case MIPI_DSI_DCS_SHORT_WRITE:
​
        dsi_update_bits(dsi, DSI_CMD_MODE_CFG, DCS_SW_0P_TX,
                dsi->mode_flags & MIPI_DSI_MODE_LPM ?
                DCS_SW_0P_TX : 0);
        break;
    case MIPI_DSI_DCS_SHORT_WRITE_PARAM:
        dsi_update_bits(dsi, DSI_CMD_MODE_CFG, DCS_SW_1P_TX,
                dsi->mode_flags & MIPI_DSI_MODE_LPM ?
                DCS_SW_1P_TX : 0);
        break;
    default:
        return -EINVAL;
    }
​
    dsi_update_bits(dsi, DSI_CMD_MODE_CFG,
                ACK_RQST_EN, ACK_RQST_EN);
​
    /* create a packet to the DSI protocol */
    ret = mipi_dsi_create_packet(&packet, msg);
​
​
    ret = genif_wait_cmd_fifo_not_full(dsi);
​
​
    /* Send packet header */
    val = get_unaligned_le32(packet.header);
    dsi_write(dsi, DSI_GEN_HDR, val);
​
    ret = genif_wait_write_fifo_empty(dsi);
    if (ret)
        return ret;
​
    if (msg->rx_len) {
        ret = dw_mipi_dsi_read_from_fifo(dsi, msg);
        if (ret < 0)
            return ret;
    }
​
    if (dsi->slave) {
        ret = dw_mipi_dsi_transfer(dsi->slave, msg);
        if (ret < 0)
            return ret;
    }
​
    return msg->rx_len ? msg->rx_len : msg->tx_len;
}
```

## （五）、vop, dsi, panel 三大模块 enable 分析
### 5.1、vop 模块 enable 分析
#### 5.1.1、rockchip_vop_enable()

``` c
X:\u-boot\drivers\video\drm\rockchip_display.c
static int rockchip_vop_enable(struct display_state *state)
{
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
    zjj_rockchip_vop_enable();
    return 0;
}
X:\u-boot\drivers\video\drm\zhoujinjian_mipi_rockchip_display.c
void zjj_rockchip_vop_enable()
{
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
    vop_mask_write(regs, standby.offset, standby.mask, standby.shift, 
         0 , standby.write_mask);
    vop_mask_write(regs, cfg_done.offset, cfg_done.mask, cfg_done.shift, 
         1 , cfg_done.write_mask);
​
}
```

### 5.2、dsi 模块 enable 分析
#### 5.2.1、rockchip_dw_mipi_dsi_enable()

``` c
static int dw_mipi_dsi_connector_enable(struct display_state *state)
{
    struct connector_state *conn_state = &state->conn_state;
    struct dw_mipi_dsi *dsi = dev_get_priv(conn_state->dev);
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    dw_mipi_dsi_enable(dsi);
​
    return 0;
}
​
static void dw_mipi_dsi_enable(struct dw_mipi_dsi *dsi)
{
    //const struct drm_display_mode *mode = &dsi->mode;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    dsi_update_bits(dsi, DSI_LPCLK_CTRL,
            PHY_TXREQUESTCLKHS, PHY_TXREQUESTCLKHS);
    dsi_write(dsi, DSI_PWR_UP, RESET);
    dsi_update_bits(dsi, DSI_MODE_CFG, CMD_VIDEO_MODE, VIDEO_MODE);
    dsi_write(dsi, DSI_PWR_UP, POWERUP);
​
}
```

### 5.3、panel 模块 enable 分析
#### 5.3.1、panel_simple_enable()

``` c
X:\u-boot\drivers\video\drm\rockchip_panel.c
static void panel_simple_enable(struct rockchip_panel *panel)
{
    struct rockchip_panel_plat *plat = dev_get_platdata(panel->dev);
    struct rockchip_panel_priv *priv = dev_get_priv(panel->dev);
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    if (priv->enabled)
        return;
​
    if (plat->delay.enable)
        mdelay(plat->delay.enable);
​
    //D:\RK3399_Android8.1_MIPI\u-boot\arch\arm\include\asm\arch-rk33xx\io-rk3399.h
#define RKIO_GRF_PHYS    0xFF770000
    
    //D:\RK3399_Android8.1_MIPI\u-boot\arch\arm\include\asm\arch-rk33xx\grf-rk3399.h
#define GRF_GPIO4C_IOMUX    0xe028
​
    //if (priv->backlight)
    //  backlight_enable(priv->backlight);
      //#define RKIO_PWM_PHYS                 0xFF420000
      //#define RKIO_PWM1_PHYS                 (RKIO_PWM_PHYS + 0x10)
      u32 bl_base = 0xff420010;
      printf("zjj rk_pwm_bl_config.\n");
      int brightness = 255;
      printf("pwm_1->base 0x%x\n", bl_base);
    
      //使能pwm_1,GRF_GPIO4C_IOMUX == 0xe028
      //grf_writel((3 << 28) | (1 << 12) , GRF_GPIO4C_IOMUX);
      volatile void *addr = (void*)(RKIO_GRF_PHYS + GRF_GPIO4C_IOMUX);
      *(u32 *)addr = (3 << 28) | (1 << 12);
    
      //Datasheet：Rockchip RK3399TRM V1.3 Part1 P903
      //PMU_PWRDN_ST
      //Address: Operational Base + offset (0x0018)
    
      if (brightness == 0) {
          //PWM_REG_DUTY == 0x08 即bl.base + PWM_REG_DUTY= 0xff420018
          //write_pwm_reg(&bl, PWM_REG_DUTY, 0);
          *(u32 *)(0xff420018) = 0;
      } else {
          //PWM_REG_DUTY == 0x08 即bl.base + PWM_REG_DUTY= 0xff420018
          //write_pwm_reg(&bl, PWM_REG_DUTY, 1);
          *(u32 *)(0xff420018) = 1;
      }
    
      //write_pwm_reg(&bl, PWM_REG_CNTR, 0);
      //Datasheet：Rockchip RK3399TRM V1.3 Part1 P906
      //0xff42001c hardware pull up.0x000f
      //write_pwm_reg(&bl, PWM_REG_CTRL, 0x000f);
      *(u32 *)(0xff42001c) = 0x000f;
​
    priv->enabled = true;
}
```

**循环刷新Buffer：**

``` c
X:\u-boot\drivers\video\drm\rockchip_display.c
​
static int display_logo(struct display_state *state)
{
    struct crtc_state *crtc_state = &state->crtc_state;
    struct connector_state *conn_state = &state->conn_state;
    struct logo_info *logo = &state->logo;
    int hdisplay, vdisplay, ret;
    printf("zjj.rk3399.uboot %s %s %d \n",__FILE__,__FUNCTION__,__LINE__);
​
    ret = display_init(state);
    if (!state->is_init || ret)
        return -ENODEV;
​
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
​
    crtc_state->dma_addr = (u32)(unsigned long)logo->mem + logo->offset;
​
    printf("dma_addr %x\n", crtc_state->dma_addr);
​
    crtc_state->xvir = ALIGN(crtc_state->src_w * logo->bpp, 32) >> 5;
​
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
​
        if (crtc_state->src_h >= vdisplay) {
            crtc_state->crtc_y = 0;
            crtc_state->crtc_h = vdisplay;
        } else {
            crtc_state->crtc_y = (vdisplay - crtc_state->src_h) / 2;
            crtc_state->crtc_h = crtc_state->src_h;
        }
    }
    display_set_plane(state);
    display_enable(state);
​
    while(1)
    {
        fb_data(WHITE);
        mdelay(2000);
        
        fb_data(YELLOW);
        mdelay(2000);
        
        fb_data(MAGENTA);
        mdelay(2000);
    
        fb_data(RED);
        mdelay(2000);
        
        fb_data(CYAN);
        mdelay(2000);
        
        fb_data(GREEN);
        mdelay(2000);
​
        fb_data(BLUE);
        mdelay(2000);
        
        fb_data(DGRAY);
        mdelay(2000);
        
        fb_data(LGRAY);
        mdelay(2000);
​
        fb_data(OLIVE);
        mdelay(2000);
        
        fb_data(PURPLE);
        mdelay(2000);
        
        fb_data(MAROON);
        mdelay(2000);
        
        fb_data(BLACK);
        mdelay(2000);
​
        fb_data(DCYAN);
        mdelay(2000);
​
        fb_data(DGREEN);
        mdelay(2000);
​
        fb_data(NAVY);
        mdelay(2000);
​
    }
​
    return 0;
}
```

## (六)、参考资料(特别感谢)：

[（1）【\[RK3399\]\[Android7.1\] Uboot display 加载过程小结】](https://blog.csdn.net/kris_fei/article/details/79003925)

[（2）【Connect to TS050 Touchscreen】](https://docs.khadas.com/zh-cn/edge/ConnectLcd.html#3-%EF%BC%88MIPI-HDMI%EF%BC%89%E5%B1%8F%E5%B9%95%E9%85%8D%E7%BD%AE)

[（3）【Firefly-RK3399 LCD使用】](http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/driver_lcd.html)

[（4）【RK3288——LCD裸机】](https://hceng.cn/2018/07/19/RK3288%E2%80%94%E2%80%94LCD%E8%A3%B8%E6%9C%BA/)