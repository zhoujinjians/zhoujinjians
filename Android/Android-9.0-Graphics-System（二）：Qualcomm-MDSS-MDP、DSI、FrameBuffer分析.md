---
title: Android P Graphics System（二）：Qualcomm MDSS - MDP、DSI、FrameBuffer分析
cover: https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/hexo.themes/bing-wallpaper-2018.04.37.jpg
categories:
  - Graphics
tags:
  - Android
  - Graphics
toc: true
abbrlink: 20190608
date: 2019-06-08 09:25:00
---

--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 9.0 && Linux（Kernel 3.18）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Qualcomm ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-3.18**==）&&（==**文章基于 Android 9.0**==）

[【开发板 Intrinsyc Open-Q™ 820 µSOM Development Kit】](https://www.intrinsyc.com/snapdragon-embedded-development-kits/open-q-820-usom-development-kit/)
[【开发板 Android 9.0 && Linux（Kernel 3.18）源码链接】](https://gitlab.com/zhoujinjianin/apq8096_la.um.7.5.r1-03100-8x96.0_p_v5.0)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【[Android5.1][RK3288] LCD Mipi 调试方法及问题汇总】](https://blog.csdn.net/dearsq/article/details/52354593) 
[（2）【[Android6.0][RK3399] Mipi LCD 通用移植调试流程】](https://blog.csdn.net/dearsq/article/details/77341120) 
[（3）【MDP分析，DSI分析，FB分析，Display resume suspend分析】](https://github.com/wj123/github/tree/master/android/module/display/liuzl) 
[（4）【高通平台LCD之MDP code解析】](https://blog.csdn.net/u012452964/article/details/74841290) 



--------------------------------------------------------------------------------
==源码（部分）==：

>Qualcomm MDSS

- G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\

--------------------------------------------------------------------------------
依赖于高通的强大的显示框架，我们只需要加入很少的patch就可以让一块LCD 屏幕亮屏并显示画面，接下里来博主来粗略的分析下。

先看看开发板LCD的patch（TouchScreen部分省略）。

``` patch
From 5b78c501d54013ab4fd687d2d66c0e04de05b1d5 Mon Sep 17 00:00:00 2001
From: Jitendra Lanka <jlanka@xxx.com>
Date: Tue, 17 Jan 2017 12:39:47 -0800
Subject: Support OSD panel & Goodix touchscreen

Change-Id: I9e648b8957a185e55529562e481b286897ea3dfb
---
 .../boot/dts/qcom/apq8096-dragonboard.dtsi    |  71 ++++++++-
 .../qcom/dsi-panel-jd9161-fwvga-video.dtsi    | 144 ++++++++++++++++++
 .../boot/dts/qcom/msm8996-mdss-panels.dtsi    |   8 +
 arch/arm/boot/dts/qcom/msm8996-mdss.dtsi      |   1 +
 arch/arm/boot/dts/qcom/msm8996-pinctrl.dtsi   |  55 +++++++
 arch/arm64/configs/msm-perf_defconfig         |   3 +
 arch/arm64/configs/msm_defconfig              |   3 +
 7 files changed, 283 insertions(+), 2 deletions(-)
 create mode 100644 arch/arm/boot/dts/qcom/dsi-panel-jd9161-fwvga-video.dtsi

diff --git a/arch/arm/boot/dts/qcom/apq8096-dragonboard.dtsi b/arch/arm/boot/dts/qcom/apq8096-dragonboard.dtsi
index 1020e3ca2333..836226a48617 100644
--- a/arch/arm/boot/dts/qcom/apq8096-dragonboard.dtsi
+++ b/arch/arm/boot/dts/qcom/apq8096-dragonboard.dtsi
 -334,7 +334,17 @@
 	qcom,mdss-dsi-bl-pmic-pwm-frequency = <50>;
 	qcom,mdss-dsi-bl-pmic-bank-select = <0>;
 	qcom,mdss-dsi-pwm-gpio = <&pm8994_gpios 5 0>;
-	qcom,panel-supply-entries = <&dsi_panel_pwr_supply>;
+	qcom,panel-supply-entries = <&dsi_panel_pwr_supply_vdd_no_labibb>;
+};
+
+&dsi_jd9161_fwvga_vid {
+	qcom,mdss-dsi-bl-pmic-control-type = "bl_ctrl_pwm";
+	qcom,mdss-dsi-bl-min-level = <1>;
+	qcom,mdss-dsi-bl-max-level = <255>;
+	qcom,mdss-dsi-bl-pmic-pwm-frequency = <50>;
+	qcom,mdss-dsi-bl-pmic-bank-select = <0>;
+	qcom,mdss-dsi-pwm-gpio = <&pm8994_gpios 5 0>;
+	qcom,panel-supply-entries = <&dsi_panel_pwr_supply_vdd_no_labibb>;
 };
 
 &mdss_mdp {
 -343,7 +353,7 @@
 
 
 &mdss_dsi0 {
-	qcom,dsi-pref-prim-pan = <&dsi_hx8379a_fwvga_truly_vid>;
+	qcom,dsi-pref-prim-pan = <&dsi_jd9161_fwvga_vid>;
 	pinctrl-names = "mdss_default", "mdss_sleep";
 	pinctrl-0 = <&mdss_dsi_active &mdss_te_active &mdss_disp_bkl_active>;
 	pinctrl-1 = <&mdss_dsi_suspend &mdss_te_suspend &mdss_disp_bkl_suspend>;
 -550,6 +560,63 @@
 			clocks = <&clock_gcc clk_gcc_blsp2_ahb_clk>,
 				 <&clock_gcc clk_gcc_blsp2_qup6_i2c_apps_clk>;
 		};
//......省略
 	};
 
 	gpio_keys {
diff --git a/arch/arm/boot/dts/qcom/dsi-panel-jd9161-fwvga-video.dtsi b/arch/arm/boot/dts/qcom/dsi-panel-jd9161-fwvga-video.dtsi
new file mode 100644
index 000000000000..f56e6524e9e5
--- /dev/null
+++ b/arch/arm/boot/dts/qcom/dsi-panel-jd9161-fwvga-video.dtsi
 -0,0 +1,144 @@
+
+&mdss_mdp {
+	dsi_jd9161_fwvga_vid: qcom,mdss_dsi_jd9161_fwvga_vid {
+		qcom,mdss-dsi-panel-name = "JD9161 fwvga video mode dsi panel";
+		qcom,mdss-dsi-panel-type = "dsi_video_mode";
+		qcom,mdss-dsi-panel-framerate = <60>;
+		qcom,mdss-dsi-virtual-channel-id = <0>;
+		qcom,mdss-dsi-stream = <0>;
+		qcom,mdss-dsi-panel-width = <480>;
+		qcom,mdss-dsi-panel-height = <854>;
+		qcom,mdss-dsi-h-front-porch = <10>;
+		qcom,mdss-dsi-h-back-porch = <10>;
+		qcom,mdss-dsi-h-pulse-width = <10>;
+		qcom,mdss-dsi-h-sync-skew = <0>;
+		qcom,mdss-dsi-v-back-porch = <6>;
+		qcom,mdss-dsi-v-front-porch = <6>;
+		qcom,mdss-dsi-v-pulse-width = <4>;
+		qcom,mdss-dsi-h-left-border = <0>;
+		qcom,mdss-dsi-h-right-border = <0>;
+		qcom,mdss-dsi-v-top-border = <0>;
+		qcom,mdss-dsi-v-bottom-border = <0>;
+		qcom,mdss-dsi-bpp = <24>;
+		qcom,mdss-dsi-color-order = "rgb_swap_rgb";
+		qcom,mdss-dsi-underflow-color = <0xff>;
+		qcom,mdss-dsi-border-color = <0>;
+		qcom,mdss-dsi-on-command = [
+			39 01 00 00 01 00 04
+				BF 91 61 F2
+			39 01 00 00 01 00 03
+				B3 00 9B
+			39 01 00 00 01 00 03
+				B4 00 9B
+			39 01 00 00 01 00 02
+				C3 04
+			39 01 00 00 01 00 07
+				B8 00 6F 01
+				00 6F 01
+			39 01 00 00 01 00 04
+				BA 34 23 00
+			39 01 00 00 01 00 03
+				C4 30 6A
+			39 01 00 00 01 00 0A
+				C7 00 01 32
+				05 65 2A 12
+				A5 A5
+			39 01 00 00 01 00 27
+				C8 7F 6A 5A
+				4E 49 39 3B
+				23 37 32 2F
+				49 35 3B 31
+				2B 1E 0F 00
+				7F 6A 5A 4E
+				49 39 3B 23
+				37 32 2F 49
+				35 3B 31 2B
+				1E 0F 00
+			39 01 00 00 01 00 11
+				D4 1E 1F 1F
+				1F 06 04 0A
+				08 00 02 1F
+				1F 1F 1F 1F
+				1F
+			39 01 00 00 01 00 11
+				D5 1E 1F 1F
+				1F 07 05 0B
+				09 01 03 1F
+				1F 1F 1F 1F
+				1F
+			39 01 00 00 01 00 11
+				D6 1F 1E 1F
+				1F 07 09 0B
+				05 03 01 1F
+				1F 1F 1F 1F
+				1F
+			39 01 00 00 01 00 11
+				D7 1F 1E 1F
+				1F 06 08 0A
+				04 02 00 1F
+				1F 1F 1F 1F
+				1F
+			39 01 00 00 01 00 15
+				D8 20 00 00
+				30 08 20 01
+				02 00 01 02
+				06 7B 00 00
+				72 0A 0E 49
+				08
+			39 01 00 00 01 00 14
+				D9 00 0A 0A
+				89 00 00 06
+				7B 00 00 00
+				3B 33 1F 00
+				00 00 03 7B
+			05 01 00 00 01 00 01
+				35
+			39 01 00 00 01 00 02
+				BE 01
+			39 01 00 00 01 00 02
+				C1 10
+			39 01 00 00 01 00 0B
+				CC 34 20 38
+				60 11 91 00
+				40 00 00
+			39 01 00 00 00 00 02
+				BE 00
+			05 01 00 00 78 00 01
+				11
+			05 01 00 00 10 00 01
+				29
+			];
+		qcom,mdss-dsi-off-command = [
+			05 01 00 00 32 00 01
+				28
+			05 01 00 00 78 00 01
+				10
+			];
+		qcom,mdss-dsi-on-command-state = "dsi_lp_mode";
+		qcom,mdss-dsi-off-command-state = "dsi_hs_mode";
+		qcom,mdss-dsi-h-sync-pulse = <0>;
+		qcom,mdss-dsi-traffic-mode = "non_burst_sync_event";
+		qcom,mdss-dsi-bllp-eof-power-mode;
+		qcom,mdss-dsi-bllp-power-mode;
+		qcom,mdss-dsi-lane-0-state;
+		qcom,mdss-dsi-lane-1-state;
+		qcom,mdss-dsi-panel-timings =
+                                       [66 14 0C 00 34 38 10 18 0E 03 04 00];
+		qcom,mdss-dsi-t-clk-post = <0x04>;
+		qcom,mdss-dsi-t-clk-pre = <0x1D>;
+		qcom,mdss-dsi-dma-trigger = "trigger_sw";
+		qcom,mdss-dsi-mdp-trigger = "none";
+		qcom,mdss-dsi-reset-sequence = <1 20>, <0 2>, <1 20>;
+	};
+};
diff --git a/arch/arm/boot/dts/qcom/msm8996-mdss-panels.dtsi b/arch/arm/boot/dts/qcom/msm8996-mdss-panels.dtsi
index c1ec7ec4c495..7e8e01de0e6f 100644
--- a/arch/arm/boot/dts/qcom/msm8996-mdss-panels.dtsi
+++ b/arch/arm/boot/dts/qcom/msm8996-mdss-panels.dtsi
 -23,6 +23,7 @@
 #include "dsi-panel-sim-dualmipi-cmd.dtsi"
 #include "dsi-panel-nt35597-dsc-wqxga-cmd.dtsi"
 #include "dsi-panel-hx8379a-truly-fwvga-video.dtsi"
+#include "dsi-panel-jd9161-fwvga-video.dtsi"
 #include "dsi-panel-r69007-dualdsi-wqxga-cmd.dtsi"
 #include "dsi-adv7533-1024-600p.dtsi"
 #include "dsi-adv7533-720p.dtsi"
 -259,6 +260,13 @@
 		23 20 06 09 05 03 04 a0
 		23 2e 06 08 05 03 04 a0];
 };
+&dsi_jd9161_fwvga_vid {
+	qcom,mdss-dsi-panel-timings-phy-v2 = [1D 19 04 05 01 03 04 a0
+		1D 19 04 05 01 03 04 a0
+		1D 19 04 05 01 03 04 a0
+		1D 19 04 05 01 03 04 a0
+		1D 1A 03 04 01 03 04 a0];
+};
 &dsi_r69007_wqxga_cmd {
 	qcom,mdss-dsi-panel-timings-phy-v2 = [23 1f 07 09 05 03 04 a0
 		23 1f 07 09 05 03 04 a0
diff --git a/arch/arm/boot/dts/qcom/msm8996-mdss.dtsi b/arch/arm/boot/dts/qcom/msm8996-mdss.dtsi
index ebddd6d12793..85cd7d9b8b04 100644
--- a/arch/arm/boot/dts/qcom/msm8996-mdss.dtsi
+++ b/arch/arm/boot/dts/qcom/msm8996-mdss.dtsi
@@ -409,6 +409,7 @@
 
 			qcom,timing-db-mode;
 			oled-vdda-supply = <&pm8994_l19>;
+			vdd-supply = <&pm8994_l22>;
 			vddio-supply = <&pm8994_l14>;
 			lab-supply = <&lab_regulator>;
 			ibb-supply = <&ibb_regulator>;
diff --git a/arch/arm/boot/dts/qcom/msm8996-pinctrl.dtsi b/arch/arm/boot/dts/qcom/msm8996-pinctrl.dtsi
index 4919c2c496d9..123f5ed84c8f 100644
--- a/arch/arm/boot/dts/qcom/msm8996-pinctrl.dtsi
+++ b/arch/arm/boot/dts/qcom/msm8996-pinctrl.dtsi
@@ -761,6 +761,61 @@
 			};
 		};
//......省略
```


首先来看看打印log：

``` cpp
/kernel/msm-3.18/arch/arm/boot/dts/qcom/msm8996-mdss.dtsi
&soc {
    mdss_mdp: qcom,mdss_mdp@900000 {
        compatible = "qcom,mdss_mdp";
......
    mdss_dsi: qcom,mdss_dsi@0 {
        compatible = "qcom,mdss-dsi";
......
        mdss_dsi0: qcom,mdss_dsi_ctrl0@994000 {
            compatible = "qcom,mdss-dsi-ctrl";
            label = "MDSS DSI CTRL->0";
 
 
 
msm8996:/sys/devices/soc/900000.qcom,mdss_mdp # ls -l
total 0
drwxr-xr-x 3 root root    0 1970-01-01 01:12 900000.qcom,mdss_mdp:qcom,mdss_fb_hdmi
drwxr-xr-x 4 root root    0 1970-01-01 01:12 900000.qcom,mdss_mdp:qcom,mdss_fb_primary
drwxr-xr-x 3 root root    0 1970-01-01 01:12 900000.qcom,mdss_mdp:qcom,mdss_fb_wfd
drwxr-xr-x 3 root root    0 1970-01-01 01:12 900000.qcom,mdss_mdp:qcom,smmu_mdp_sec_cb
drwxr-xr-x 3 root root    0 1970-01-01 01:12 900000.qcom,mdss_mdp:qcom,smmu_mdp_unsec_cb
drwxr-xr-x 3 root root    0 1970-01-01 01:12 900000.qcom,mdss_mdp:qcom,smmu_rot_sec_cb
drwxr-xr-x 3 root root    0 1970-01-01 01:12 900000.qcom,mdss_mdp:qcom,smmu_rot_unsec_cb
-rw-rw-r-- 1 root root 4096 1970-01-01 01:12 bw_mode_bitmap
-r--r--r-- 1 root root 4096 1970-01-01 01:12 caps
lrwxrwxrwx 1 root root    0 1970-01-01 01:12 driver -> ../../../bus/platform/drivers/mdp
-rw-r--r-- 1 root root 4096 1970-01-01 01:12 driver_override
-r--r--r-- 1 root root 4096 1970-01-01 01:12 modalias
drwxr-xr-x 2 root root    0 1970-01-01 01:12 power
lrwxrwxrwx 1 root root    0 1970-01-01 01:12 subsystem -> ../../../bus/platform
-rw-r--r-- 1 root root 4096 1970-01-01 01:12 uevent
 
 
msm8996:/sys/devices/soc/soc:qcom,mdss_dsi@0 # ls -l
total 0
drwxr-xr-x 3 root root    0 1970-01-01 01:12 994000.qcom,mdss_dsi_ctrl0
lrwxrwxrwx 1 root root    0 2019-02-23 10:38 driver -> ../../../bus/platform/drivers/mdss_dsi
-rw-r--r-- 1 root root 4096 2019-02-23 10:38 driver_override
-r--r--r-- 1 root root 4096 2019-02-23 10:38 modalias
drwxr-xr-x 2 root root    0 1970-01-01 01:12 power
lrwxrwxrwx 1 root root    0 2019-02-23 10:38 subsystem -> ../../../bus/platform
-rw-r--r-- 1 root root 4096 1970-01-01 01:12 uevent
 
 
msm8996:/sys/devices/soc/900000.qcom,mdss_mdp/900000.qcom,mdss_mdp:qcom,mdss_fb_primary # ls -l
total 0
lrwxrwxrwx 1 root root    0 1970-01-01 01:12 driver -> ../../../../bus/platform/drivers/mdss_fb
-rw-r--r-- 1 root root 4096 1970-01-01 01:12 driver_override
drwxr-xr-x 3 root root    0 1970-01-01 01:12 leds
-r--r--r-- 1 root root 4096 1970-01-01 01:12 modalias
drwxr-xr-x 2 root root    0 1970-01-01 01:12 power
lrwxrwxrwx 1 root root    0 1970-01-01 01:12 subsystem -> ../../../../bus/platform
-rw-r--r-- 1 root root 4096 1970-01-01 01:12 uevent
msm8996:/sys/devices/soc/900000.qcom,mdss_mdp/900000.qcom,mdss_mdp:qcom,mdss_fb_primary #
 
 
--->>>Logs:
 
mdss_mdp.panel=1:dsi:0:qcom,mdss_dsi_jd9161_fwvga_vid:1:none:cfg:dual_dsi skip_initramfs rootwait ro init=/init root=/dev/sde18
 
1.0 mdss_mdp_probe()
 
 [ [    2.183106]  [ mdss_mdp_probe [ : mdss version = 0x10070002, bootloader display is on, num 1, intf_sel=0x00000100
 [ [    2.191112]  [ mdss_smmu_probe [ : iommu v2 domain[0] mapping and clk register successful!
 [ [    2.191547]  [ mdss_smmu_probe [ : iommu v2 domain[1] mapping and clk register successful!
 [ [    2.191935]  [ mdss_smmu_probe [ : iommu v2 domain[2] mapping and clk register successful!
 [ [    2.192318]  [ mdss_smmu_probe [ : iommu v2 domain[3] mapping and clk register successful!
 
2.1 mdss_dsi__probe()
-->mdss_dsi_res_init
---->mdss_dsi_get_dt_vreg_data
 
 [ [    2.194876]  [ mdss_dsi_get_dt_vreg_data [ : error reading ulp load. rc=-22
 [ [    2.194903]  [ mdss_dsi_get_dt_vreg_data [ : error reading ulp load. rc=-22
 [ [    2.194923]  [ mdss_dsi_get_dt_vreg_data [ : error reading ulp load. rc=-22
 [ [    2.197217]  [ mdss_dsi_ctrl_probe [ : DSI Ctrl name = MDSS DSI CTRL->0
 [ [    2.197830]  [ mdss_dsi_find_panel_of_node [ : cmdline:0:qcom,mdss_dsi_jd9161_fwvga_vid:1:none:cfg:dual_dsi panel_name:qcom,mdss_dsi_jd9161_fwvga_vid
 [ [    2.197853]  [ mdss_dsi_panel_init [ : Panel Name = JD9161 fwvga video mode dsi panel
 [ [    2.197953]  [ mdss_dsi_panel_timing_from_dt [ : found new timing "qcom,mdss_dsi_jd9161_fwvga_vid" (ffffffc0b5217288)
 [ [    2.197973]  [ mdss_dsi_parse_dcs_cmds [ : failed, key=qcom,mdss-dsi-post-panel-on-command
 [ [    2.197981]  [ mdss_dsi_parse_dcs_cmds [ : failed, key=qcom,mdss-dsi-timing-switch-command
 [ [    2.197987]  [ mdss_dsi_panel_get_dsc_cfg_np [ : cannot find dsc config node:
 [ [    2.198059]  [ mdss_dsi_parse_dcs_cmds [ : failed, key=qcom,mdss-dsi-idle-on-command
 [ [    2.198067]  [ mdss_dsi_parse_dcs_cmds [ : failed, key=qcom,mdss-dsi-idle-off-command
 [ [    2.198082]  [ mdss_dsi_parse_panel_features [ : ulps feature disabled
 [ [    2.198090]  [ mdss_dsi_parse_panel_features [ : ulps during suspend feature disabled
 [ [    2.198097]  [ mdss_dsi_parse_dms_config [ : dynamic switch feature enabled: 0
 [ [    2.198112]  [ mdss_dsi_parse_dcs_cmds [ : failed, key=qcom,mdss-dsi-lp-mode-on
 [ [    2.198119]  [ mdss_dsi_parse_dcs_cmds [ : failed, key=qcom,mdss-dsi-lp-mode-off
 [ [    2.198526]  [ mdss_dsi_get_dt_vreg_data [ : error reading ulp load. rc=-22
 [ [    2.198544]  [ mdss_dsi_get_dt_vreg_data [ : error reading ulp load. rc=-22
 [ [    2.198843]  [ mdss_dsi_parse_ctrl_params:4072 Unable to read qcom,display-id, data=0000000000000000,len=20
 [ [    2.198977]  [ msm_dss_get_res_byname [ : 'dsi_phy_regulator' resource not found
 [ [    2.199006]  [ mdss_dsi_retrieve_ctrl_resources+0x170/0x254->msm_dss_ioremap_byname [ : 'dsi_phy_regulator' msm_dss_get_res_byname failed
 [ [    2.199013]  [ mdss_dsi_retrieve_ctrl_resources [ : ctrl_base=ffffff800179c000 ctrl_size=400 phy_base=ffffff800179e400 phy_size=588
 [ [    2.199326]  [ dsi_panel_device_register [ : Continuous splash enabled
 [ [    2.200158]  [ mdss_register_panel [ : adding framebuffer device 994000.qcom,mdss_dsi_ctrl0
 
2.2 mdss_dsi_ctrl_probe()
 
 [ [    2.201997]  [ mdss_dsi_ctrl_probe [ : Dsi Ctrl->0 initialized, DSI rev:0x10040001, PHY rev:0x2
 [ [    2.203915]  [ mdss_dsi_status_init [ : DSI status check interval:5000
 [ [    2.205862]  [ hdmi_tx_get_dt_data:4387 Unable to read qcom,display-id, data=0000000000000000,len=0
 [ [    2.206973]  [ hdmi_tx_init_resource [ : 'core_physical': start = 0xffffff80017b4000, len=0x50c
 [ [    2.206987]  [ hdmi_tx_init_resource [ : 'qfprom_physical': start = 0xffffff80017b8000, len=0x6158
 [ [    2.207077]  [ hdmi_tx_init_resource [ : 'hdcp_physical': start = 0xffffff80017b6000, len=0xfff
 [ [    2.208770]  [ mdss_register_panel [ : adding framebuffer device 9a0000.qcom,hdmi_tx
 [ [    2.212112]  [ mdss_register_panel [ : adding framebuffer device soc:qcom,mdss_wb_panel
 
3.0 mdss_fb_probe()
 
 [ [    2.214819]  [ mdss_fb_probe [ : fb0: split_mode:0 left:0 right:0
 [ [    2.216288]  [ mdss_fb_register [ : FrameBuffer[0] 480x854 registered successfully!
 [ [    2.217832]  [ mdss_fb_probe [ : fb1: split_mode:0 left:0 right:0
 [ [    2.218084]  [ mdss_fb_register [ : FrameBuffer[1] 640x480 registered successfully!
 
 [ [    2.270242]  [ mdss_fb_probe [ : fb2: split_mode:0 left:0 right:0
 [ [    2.270614]  [ mdss_fb_register [ : FrameBuffer[2] 640x480 registered successfully!
 
 
 
msm8996:/ # ps -A | grep "mdss"                                                
root           130     2       0      0 dsi_event_thread    0 D [mdss_dsi_event]
root           131     2       0      0 rescuer_thread      0 S [mdss_dsi_dba]
msm8996:/ #
--->>>Power On
msm8996:/ # ps -A | grep "mdss"                                                
root           130     2       0      0 dsi_event_thread    0 D [mdss_dsi_event]
root           131     2       0      0 rescuer_thread      0 S [mdss_dsi_dba]
root          4039     2       0      0 __mdss_fb_display_thread 0 D [mdss_fb0]
```


#### (一)、MDP分析

LCD相关code所在目录：
        kernel/drvier/video/msm/mdss/ 
软件驱动主要分为三部分：
        MDP 驱动
        DSI 控制器驱动
        FrameBuffer驱动
执行probe 的先后顺序：
       **MDP probe →  DSI probe → FB probe**


根据设备树的compatible 可知，当检测到同名的，会调用对应的probe()函数。
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/display.system/Android.PG2.mdp_probe.png)

``` cpp
/kernel/msm-3.18/arch/arm/boot/dts/qcom/msm8996-mdss.dtsi
&soc {
    mdss_mdp: qcom,mdss_mdp@900000 {
        compatible = "qcom,mdss_mdp";
......
```

``` cpp
G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\mdss_mdp.c
static const struct of_device_id mdss_mdp_dt_match[] = {
	{ .compatible = "qcom,mdss_mdp",},
	{}
};
MODULE_DEVICE_TABLE(of, mdss_mdp_dt_match);
static struct platform_driver mdss_mdp_driver = {
	.probe = mdss_mdp_probe,
	.remove = mdss_mdp_remove,
	.suspend = mdss_mdp_suspend,
	.resume = mdss_mdp_resume,
	.shutdown = NULL,
	.driver = {
		/*
		 * Driver name must match the device name added in
		 * platform.c.
		 */
		.name = "mdp",
		.of_match_table = mdss_mdp_dt_match,
		.pm = &mdss_mdp_pm_ops,
	},
};
```
首先看看mdss_mdp_probe(struct platform_device *pdev)函数：

##### 1.1、为关键驱动数据结构 struct mdss_data_type *mdata 分配空间

``` cpp
G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\mdss_mdp.c
static int mdss_mdp_probe(struct platform_device *pdev)
{
	struct resource *res;
	int rc;
	struct mdss_data_type *mdata;
	uint32_t intf_sel = 0;
	uint32_t split_display = 0;
	int num_of_display_on = 0;
	int i = 0;
	......
	mdata = devm_kzalloc(&pdev->dev, sizeof(*mdata), GFP_KERNEL);
	.......
	//初始化中断函数
	mdss_res->mdss_util = mdss_get_util_intf();
}

```
获得资源，赋值给mdata结构体

``` cpp
G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\mdss_mdp.c
struct mdss_hw mdss_mdp_hw = {
	.hw_ndx = MDSS_HW_MDP,
	.ptr = NULL,
	.irq_handler = mdss_mdp_isr,
};


static int mdss_mdp_probe(struct platform_device *pdev)
{
	//获得资源，赋值给mdata结构体
    解释如下项：
   /kernel/msm-3.18/arch/arm/boot/dts/qcom/msm8996-mdss.dtsi
		reg = <0x00900000 0x90000>,
		      <0x009b0000 0x1040>,
		      <0x009b8000 0x1040>;
		reg-names = "mdp_phys", "vbif_phys", "vbif_nrt_phys";
   /////////////////////////////////////////////////////////////////////
   	rc = msm_dss_ioremap_byname(pdev, &mdata->mdss_io, "mdp_phys");
	rc = msm_dss_ioremap_byname(pdev, &mdata->vbif_io, "vbif_phys");
	rc = msm_dss_ioremap_byname(pdev, &mdata->vbif_nrt_io, "vbif_nrt_phys");
    res = platform_get_resource(pdev, IORESOURCE_IRQ, 0);
    
	mdss_mdp_hw.irq_info = kzalloc(sizeof(struct irq_info), GFP_KERNEL); 
	mdss_mdp_hw.irq_info->irq = res->start; 
	mdss_mdp_hw.ptr = mdata; 
}

```
##### 1.2、clk和irq设置、ionclient创建，iommu初始化 
 接下来看看mdss_mdp_res_init(mdata)函数

``` cpp
G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\mdss_mdp.c

static u32 mdss_mdp_res_init(struct mdss_data_type *mdata)
{
	u32 rc = 0;
    ......
	mdata->res_init = true;
	mdata->clk_ena = false;
	mdss_mdp_hw.irq_info->irq_mask = MDSS_MDP_DEFAULT_INTR_MASK;
	mdss_mdp_hw.irq_info->irq_ena = false;
    //clk和irq设置
	rc = mdss_mdp_irq_clk_setup(mdata);
	......
	mdata->hist_intr.req = 0;
	mdata->hist_intr.curr = 0;
	mdata->hist_intr.state = 0;
	spin_lock_init(&mdata->hist_intr.lock);
    //ionclient创建，iommu初始化 
	mdata->iclient = msm_ion_client_create(mdata->pdev->name);
	......
	return rc;
}

---------------------------------------------------------------------------------------
                解释如下项：
                ----------start---------
                G:\android9.0\kernel\msm-3.18\arch\arm\boot\dts\qcom\msm8996-mdss.dtsi
                qcom,max-clk-rate = <412500000>;;
                vdd-supply = <&gdsc_mdss>;
				clock-names = "iface_clk", "bus_clk", "core_clk_src",
						"core_clk", "vsync_clk";
                G:\android9.0\kernel\msm-3.18\arch\arm\boot\dts\qcom\msm-gdsc-8996.dtsi
				gdsc_mdss: qcom,gdsc@8c2304 {
					compatible = "qcom,gdsc";
					regulator-name = "gdsc_mdss";
					reg = <0x8c2304 0x4>;
					status = "disabled";
				};

---------------------------------------------------------------------------------------

static int mdss_mdp_irq_clk_setup(struct mdss_data_type *mdata)
{
	int ret;

	ret = of_property_read_u32(mdata->pdev->dev.of_node,
			"qcom,max-clk-rate", &mdata->max_mdp_clk_rate);
    //注册中断，中断总入口
	ret = devm_request_irq(&mdata->pdev->dev, mdss_mdp_hw.irq_info->irq,
				mdss_irq_handler, IRQF_DISABLED, "MDSS", mdata);

	disable_irq(mdss_mdp_hw.irq_info->irq);

	mdata->fs = devm_regulator_get(&mdata->pdev->dev, "vdd");

	mdata->venus = devm_regulator_get_optional(&mdata->pdev->dev,
		"gdsc-venus");


	mdata->fs_ena = false;

	mdata->gdsc_cb.notifier_call = mdss_mdp_gdsc_notifier_call;
	mdata->gdsc_cb.priority = 5;
	if (regulator_register_notifier(mdata->fs, &(mdata->gdsc_cb)))
		pr_warn("GDSC notification registration failed!\n");
	else
		mdata->regulator_notif_register = true;

	mdata->vdd_cx = devm_regulator_get_optional(&mdata->pdev->dev,
				"vdd-cx");

	mdata->reg_bus_clt = mdss_reg_bus_vote_client_create("mdp\0");


	if (mdss_mdp_irq_clk_register(mdata, "bus_clk", MDSS_CLK_AXI) ||
	    mdss_mdp_irq_clk_register(mdata, "iface_clk", MDSS_CLK_AHB) ||
	    mdss_mdp_irq_clk_register(mdata, "core_clk",
				      MDSS_CLK_MDP_CORE))
		return -EINVAL;

	/* lut_clk is not present on all MDSS revisions */
	mdss_mdp_irq_clk_register(mdata, "lut_clk", MDSS_CLK_MDP_LUT);

	/* vsync_clk is optional for non-smart panels */
	mdss_mdp_irq_clk_register(mdata, "vsync_clk", MDSS_CLK_MDP_VSYNC);

	/* Setting the default clock rate to the max supported.*/
	mdss_mdp_set_clk_rate(mdata->max_mdp_clk_rate);
	pr_debug("mdp clk rate=%ld\n",
		mdss_mdp_get_clk_rate(MDSS_CLK_MDP_CORE, false));

	return 0;
}

```

``` cpp
pm_runtime_set_autosuspend_delay(&pdev->dev, AUTOSUSPEND_TIMEOUT_MS);
if (mdata->idle_pc_enabled)
	pm_runtime_use_autosuspend(&pdev->dev);
pm_runtime_set_suspended(&pdev->dev);
pm_runtime_enable(&pdev->dev);

rc = mdss_mdp_bus_scale_register(mdata);
......
mdss_mdp_footswitch_ctrl_splash(true);
mdss_hw_rev_init(mdata);
```

##### 1.3、解释dts为 struct mdss_data_type *mdata 赋值 

##### 1.3.1、mdss_mdp_parse_dt_hw_settings(pdev)

``` cpp
解释如下两项： struct mdss_hw_settings *hws;
----------start---------
qcom,vbif-settings
qcom,vbif-nrt-settings
qcom,mdp-settings
----------end---------
    mdata->hw_settings = hws

int mdss_mdp_parse_dt_hw_settings(struct platform_device *pdev)
{
	struct mdss_data_type *mdata = platform_get_drvdata(pdev);
	struct mdss_hw_settings *hws;
	const u32 *vbif_arr, *mdp_arr, *vbif_nrt_arr;
	int vbif_len, mdp_len, vbif_nrt_len;

	vbif_arr = of_get_property(pdev->dev.of_node, "qcom,vbif-settings",
			&vbif_len);

	vbif_len /= 2 * sizeof(u32);

	vbif_nrt_arr = of_get_property(pdev->dev.of_node,
				"qcom,vbif-nrt-settings", &vbif_nrt_len);

	vbif_nrt_len /= 2 * sizeof(u32);

	mdp_arr = of_get_property(pdev->dev.of_node, "qcom,mdp-settings",
			&mdp_len);

	mdp_len /= 2 * sizeof(u32);


	hws = devm_kzalloc(&pdev->dev, sizeof(*hws) * (vbif_len + mdp_len +
			vbif_nrt_len + 1), GFP_KERNEL);

	mdss_mdp_parse_dt_regs_array(vbif_arr, &mdata->vbif_io,
			hws, vbif_len);
	mdss_mdp_parse_dt_regs_array(vbif_nrt_arr, &mdata->vbif_nrt_io,
			hws, vbif_nrt_len);
	mdss_mdp_parse_dt_regs_array(mdp_arr, &mdata->mdss_io,
		hws + vbif_len, mdp_len);

	mdata->hw_settings = hws;

	return 0;
}
```
##### 1.3.2、mdss_mdp_parse_dt_pipe(pdev)

``` cpp
        解释如下项:
            ----------start---------
		qcom,mdss-pipe-vig-off = <0x00005000 0x00007000
					  0x00009000 0x0000B000>;
		qcom,mdss-pipe-rgb-off = <0x00015000 0x00017000
					  0x00019000 0x0001B000>;
		qcom,mdss-pipe-dma-off = <0x00025000 0x00027000>;
		qcom,mdss-pipe-cursor-off = <0x00035000 0x00037000>;

		qcom,mdss-pipe-vig-xin-id = <0 4 8 12>;
		qcom,mdss-pipe-rgb-xin-id = <1 5 9 13>;
		qcom,mdss-pipe-dma-xin-id = <2 10>;
		qcom,mdss-pipe-cursor-xin-id = <7 7>;

            ----------end ---------

static int mdss_mdp_parse_dt_pipe(struct platform_device *pdev)
{
	int rc = 0;
	u32 nfids = 0, len, nxids = 0, npipes = 0;
	u32 sw_reset_offset = 0;
	u32 data[4];

	struct mdss_data_type *mdata = platform_get_drvdata(pdev);

	mdata->has_pixel_ram = !mdss_mdp_parse_dt_prop_len(pdev,
						"qcom,mdss-smp-data");

	mdata->nvig_pipes = mdss_mdp_parse_dt_prop_len(pdev,
				"qcom,mdss-pipe-vig-off");
	mdata->nrgb_pipes = mdss_mdp_parse_dt_prop_len(pdev,
				"qcom,mdss-pipe-rgb-off");
	mdata->ndma_pipes = mdss_mdp_parse_dt_prop_len(pdev,
				"qcom,mdss-pipe-dma-off");
	mdata->ncursor_pipes = mdss_mdp_parse_dt_prop_len(pdev,
				"qcom,mdss-pipe-cursor-off");

	npipes = mdata->nvig_pipes + mdata->nrgb_pipes + mdata->ndma_pipes;

	rc = mdss_mdp_parse_dt_pipe_helper(pdev, MDSS_MDP_PIPE_TYPE_VIG, "vig",
			&mdata->vig_pipes, mdata->nvig_pipes, 0);
	mdata->nvig_pipes = rc;

	rc = mdss_mdp_parse_dt_pipe_helper(pdev, MDSS_MDP_PIPE_TYPE_RGB, "rgb",
			&mdata->rgb_pipes, mdata->nrgb_pipes,
			mdata->nvig_pipes);
	mdata->nrgb_pipes = rc;

	rc = mdss_mdp_parse_dt_pipe_helper(pdev, MDSS_MDP_PIPE_TYPE_DMA, "dma",
			&mdata->dma_pipes, mdata->ndma_pipes,
			mdata->nvig_pipes + mdata->nrgb_pipes);
	mdata->ndma_pipes = rc;

	rc = mdss_mdp_parse_dt_pipe_helper(pdev, MDSS_MDP_PIPE_TYPE_CURSOR,
			"cursor", &mdata->cursor_pipes, mdata->ncursor_pipes,
			0);
	mdata->ncursor_pipes = rc;

	rc = 0;
	......
}

```

##### 1.3.3、mdss_mdp_parse_dt_mixer(pdev)

``` cpp
            解释如下项：
            ----------start---------
            qcom,mdss-mixer-intf-off
            qcom,mdss-mixer-wb-off
            qcom,mdss-dspp-off
            qcom,mdss-pingpong-off
            ----------end------------

static int mdss_mdp_parse_dt_mixer(struct platform_device *pdev)
{

	u32 nmixers, npingpong;
	int rc = 0;
	u32 *mixer_offsets = NULL, *dspp_offsets = NULL,
	    *pingpong_offsets = NULL;
	u32 is_virtual_mixer_req = false;

	struct mdss_data_type *mdata = platform_get_drvdata(pdev);

	mdata->nmixers_intf = mdss_mdp_parse_dt_prop_len(pdev,
				"qcom,mdss-mixer-intf-off");
	mdata->nmixers_wb = mdss_mdp_parse_dt_prop_len(pdev,
				"qcom,mdss-mixer-wb-off");
	mdata->ndspp = mdss_mdp_parse_dt_prop_len(pdev,
				"qcom,mdss-dspp-off");
	npingpong = mdss_mdp_parse_dt_prop_len(pdev,
				"qcom,mdss-pingpong-off");
	nmixers = mdata->nmixers_intf + mdata->nmixers_wb;

	rc = of_property_read_u32(pdev->dev.of_node,
			"qcom,max-mixer-width", &mdata->max_mixer_width);


	mixer_offsets = kzalloc(sizeof(u32) * nmixers, GFP_KERNEL);

	dspp_offsets = kzalloc(sizeof(u32) * mdata->ndspp, GFP_KERNEL);

	pingpong_offsets = kzalloc(sizeof(u32) * npingpong, GFP_KERNEL);

    ......
			rc = mdss_mdp_mixer_addr_setup(mdata, mixer_offsets +
				mdata->nmixers_intf - mdata->ndma_pipes,
				NULL, NULL, MDSS_MDP_MIXER_TYPE_WRITEBACK,
				mdata->nmixers_wb);
	.....
}
```

##### 1.3.4、mdss_mdp_parse_dt_mixer(pdev)

``` cpp
static int mdss_mdp_parse_dt_misc(struct platform_device *pdev)
{
	struct mdss_data_type *mdata = platform_get_drvdata(pdev);
	u32 data, slave_pingpong_off;
	const char *wfd_data;
	int rc;
	struct property *prop = NULL;

	rc = of_property_read_u32(pdev->dev.of_node, "qcom,mdss-rot-block-size",
		&data);
	mdata->rot_block_size = (!rc ? data : 128);

	rc = of_property_read_u32(pdev->dev.of_node,
		"qcom,mdss-default-ot-rd-limit", &data);
	mdata->default_ot_rd_limit = (!rc ? data : 0);

	rc = of_property_read_u32(pdev->dev.of_node,
		"qcom,mdss-default-ot-wr-limit", &data);
	mdata->default_ot_wr_limit = (!rc ? data : 0);

	mdata->has_non_scalar_rgb = of_property_read_bool(pdev->dev.of_node,
		"qcom,mdss-has-non-scalar-rgb");
	mdata->has_bwc = of_property_read_bool(pdev->dev.of_node,
					       "qcom,mdss-has-bwc");
	mdata->has_decimation = of_property_read_bool(pdev->dev.of_node,
		"qcom,mdss-has-decimation");
	mdata->has_no_lut_read = of_property_read_bool(pdev->dev.of_node,
		"qcom,mdss-no-lut-read");
	mdata->needs_hist_vote = !(of_property_read_bool(pdev->dev.of_node,
		"qcom,mdss-no-hist-vote"));
	wfd_data = of_get_property(pdev->dev.of_node,
					"qcom,mdss-wfd-mode", NULL);
    ......
}
```
##### 1.3.5、mdss_mdp_parse_dt_wb(pdev)

``` cpp
static int mdss_mdp_parse_dt_wb(struct platform_device *pdev)
{
	int rc = 0;
	u32 *wb_offsets = NULL;
	u32 num_wb_mixer, nwb_offsets, num_intf_wb = 0;
	const char *wfd_data;
	struct mdss_data_type *mdata;

	mdata = platform_get_drvdata(pdev);

	num_wb_mixer = mdata->nmixers_wb;

	wfd_data = of_get_property(pdev->dev.of_node,
					"qcom,mdss-wfd-mode", NULL);
	nwb_offsets =  mdss_mdp_parse_dt_prop_len(pdev,
			"qcom,mdss-wb-off");

	wb_offsets = kzalloc(sizeof(u32) * nwb_offsets, GFP_KERNEL);
	rc = mdss_mdp_parse_dt_handler(pdev, "qcom,mdss-wb-off",
		wb_offsets, nwb_offsets);

	rc = mdss_mdp_wb_addr_setup(mdata, num_wb_mixer, num_intf_wb);

	mdata->nwb_offsets = nwb_offsets;
	mdata->wb_offsets = wb_offsets;

	return 0;
}
```

##### 1.3.6、mdss_mdp_parse_dt_ctl(pdev)

``` cpp
static int mdss_mdp_parse_dt_ctl(struct platform_device *pdev)
{
	int rc = 0;
	u32 *ctl_offsets = NULL;
	struct mdss_data_type *mdata = platform_get_drvdata(pdev);
	mdata->nctl = mdss_mdp_parse_dt_prop_len(pdev,
			"qcom,mdss-ctl-off");
	ctl_offsets = kzalloc(sizeof(u32) * mdata->nctl, GFP_KERNEL);
	
	rc = mdss_mdp_parse_dt_handler(pdev, "qcom,mdss-ctl-off",
		ctl_offsets, mdata->nctl);

	rc = mdss_mdp_ctl_addr_setup(mdata, ctl_offsets, mdata->nctl);

	return rc;
}
```
##### 1.3.7、mdss_mdp_parse_dt_video_intf(pdev)

``` cpp
static int mdss_mdp_parse_dt_video_intf(struct platform_device *pdev)
{
	struct mdss_data_type *mdata = platform_get_drvdata(pdev);
	u32 count;
	u32 *offsets;
	int rc;
	count = mdss_mdp_parse_dt_prop_len(pdev, "qcom,mdss-intf-off");

	offsets = kzalloc(sizeof(u32) * count, GFP_KERNEL);

	rc = mdss_mdp_parse_dt_handler(pdev, "qcom,mdss-intf-off",
			offsets, count);

	rc = mdss_mdp_video_addr_setup(mdata, offsets, count);

	return rc;
}
```
##### 1.3.8、mdss_mdp_parse_dt_smp(pdev)

``` cpp
static int mdss_mdp_parse_dt_smp(struct platform_device *pdev)
{
	struct mdss_data_type *mdata = platform_get_drvdata(pdev);
	u32 num;
	u32 data[2];
	int rc, len;
	const u32 *arr;

	num = mdss_mdp_parse_dt_prop_len(pdev, "qcom,mdss-smp-data");
	/*
	 * This property is optional for targets with fix pixel ram. Rest
	 * must provide no. of smp and size of each block.
	 */
	rc = mdss_mdp_parse_dt_handler(pdev, "qcom,mdss-smp-data", data, num);

	rc = mdss_mdp_smp_setup(mdata, data[0], data[1]);

	rc = of_property_read_u32(pdev->dev.of_node,
		"qcom,mdss-smp-mb-per-pipe", data);
	mdata->smp_mb_per_pipe = (!rc ? data[0] : 0);

	rc = 0;
	arr = of_get_property(pdev->dev.of_node,
			"qcom,mdss-pipe-rgb-fixed-mmb", &len);
	if (arr) {
		rc = mdss_mdp_update_smp_map(pdev, arr, len,
				mdata->nrgb_pipes, mdata->rgb_pipes);
	}

	arr = of_get_property(pdev->dev.of_node,
			"qcom,mdss-pipe-vig-fixed-mmb", &len);
	if (arr) {
		rc = mdss_mdp_update_smp_map(pdev, arr, len,
				mdata->nvig_pipes, mdata->vig_pipes);
	}
	return rc;
}
```

##### 1.3.9、mdss_mdp_parse_dt_xxx(pdev)

``` cpp
	rc = mdss_mdp_parse_dt_prefill(pdev);

	rc = mdss_mdp_parse_dt_ad_cfg(pdev);

	rc = mdss_mdp_parse_dt_cdm(pdev);

	rc = mdss_mdp_parse_dt_dsc(pdev);
```
##### 1.4、创建属性文件、挂接中断处理函数

``` cpp
	rc = mdss_mdp_get_cmdline_config(pdev);

	rc = mdss_mdp_debug_init(pdev, mdata);//创建debug

	rc = mdss_mdp_scaler_init(mdata, &pdev->dev);

	rc = mdss_mdp_register_sysfs(mdata);

	rc = mdss_fb_register_mdp_instance(&mdp5);

	rc = mdss_res->mdss_util->register_irq(&mdss_mdp_hw);

	rc = mdss_smmu_init(mdata, &pdev->dev);

	mdss_mdp_set_supported_formats(mdata);

	mdss_res->mdss_util->mdp_probe_done = true;

	mdss_hw_init(mdata);

	rc = mdss_mdp_pp_init(&pdev->dev);

```

##### 1.4.1、mdss_mdp_get_cmdline_config(pdev)

``` cpp
static int mdss_mdp_get_cmdline_config(struct platform_device *pdev)
{
	int rc, len = 0;
	int *intf_type;
	char *panel_name;
	struct mdss_panel_cfg *pan_cfg;
	struct mdss_data_type *mdata = platform_get_drvdata(pdev);

	mdata->pan_cfg.arg_cfg[MDSS_MAX_PANEL_LEN] = 0;
	pan_cfg = &mdata->pan_cfg;
	panel_name = &pan_cfg->arg_cfg[0];
	intf_type = &pan_cfg->pan_intf;

	/* reads from dt by default */
	pan_cfg->lk_cfg = true;

	len = strlen(mdss_mdp_panel);

	if (len > 0) {
		rc = mdss_mdp_get_pan_cfg(pan_cfg);
	}
    //+	qcom,dsi-pref-prim-pan = <&dsi_jd9161_fwvga_vid>;
	rc = mdss_mdp_parse_dt_pan_intf(pdev);

	pan_cfg->init_done = true;

	return 0;
}
```
##### 1.4.2、mdss_mdp_register_sysfs(mdata)

``` cpp
//在mdp目录下，创建一个caps属性文件
static int mdss_mdp_register_sysfs(struct mdss_data_type *mdata)
{
	struct device *dev = &mdata->pdev->dev;
	int rc;
	rc = sysfs_create_group(&dev->kobj, &mdp_fs_attr_group);

	return rc;
}
```
##### 1.4.3、mdss_fb_register_mdp_instance(&mdp5)

``` cpp
## 在mdss_fb_probe()的probe函数中
##    mfd->mdp = *mdp_instance; //将mdp_instance的所有内容拷贝一份给 mfd->mdp
## 所以mfd->mdp等于mdp5，mdp_instance指向mdp5


struct msm_mdp_interface mdp5 = {
	.init_fnc = mdss_mdp_overlay_init,
	.fb_mem_get_iommu_domain = mdss_fb_mem_get_iommu_domain,
	.fb_stride = mdss_mdp_fb_stride,
	.check_dsi_status = mdss_check_dsi_ctrl_status,
	.get_format_params = mdss_mdp_get_format_params,
};

int mdss_fb_register_mdp_instance(struct msm_mdp_interface *mdp)
{
	mdp_instance = mdp;
	return 0;
}
```
##### 1.4.4、register_irq(&mdss_mdp_hw)

``` cpp
	rc = mdss_res->mdss_util->register_irq(&mdss_mdp_hw);
```

##### 1.4.5、mdss_smmu_init(mdata, &pdev->dev)

``` cpp
int mdss_smmu_init(struct mdss_data_type *mdata, struct device *dev)
{
	mdss_smmu_device_create(dev);
	mdss_smmu_ops_init(mdata);
	mdata->mdss_util->iommu_lock = mdss_iommu_lock;
	mdata->mdss_util->iommu_unlock = mdss_iommu_unlock;
	return 0;
}
```
由于devicetree涉及太多字段，博主也没有完全搞明白每个字段的意义，只搞清楚一小部分，请参考：

> G:\android9.0\kernel\msm-3.18\Documentation\devicetree\bindings\fb\mdss-dsi.txt
> G:\android9.0\kernel\msm-3.18\Documentation\devicetree\bindings\fb\mdss-mdp.txt
> G:\android9.0\kernel\msm-3.18\Documentation\devicetree\bindings\fb\mdss-dsi-panel.txt

#### (二)、DSI分析
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/display.system/Android.PG2.dsi_probe.png)

``` cpp
-------------------------------------------------------------
        mdss_dsi0: qcom,mdss_dsi_ctrl0@994000 {
            compatible = "qcom,mdss-dsi-ctrl";
            label = "MDSS DSI CTRL->0";
-------------------------------------------------------------
static const struct of_device_id mdss_dsi_ctrl_dt_match[] = {
	{.compatible = "qcom,mdss-dsi-ctrl"},
	{}
};
MODULE_DEVICE_TABLE(of, mdss_dsi_ctrl_dt_match);

static struct platform_driver mdss_dsi_ctrl_driver = {
	.probe = mdss_dsi_ctrl_probe,
	.remove = mdss_dsi_ctrl_remove,
	.shutdown = NULL,
	.driver = {
		.name = "mdss_dsi_ctrl",
		.of_match_table = mdss_dsi_ctrl_dt_match,
	},
};

static int mdss_dsi_ctrl_register_driver(void)
{
	return platform_driver_register(&mdss_dsi_ctrl_driver);
}

static int __init mdss_dsi_ctrl_driver_init(void)
{
	int ret;

	ret = mdss_dsi_ctrl_register_driver();
	if (ret) {
		pr_err("mdss_dsi_ctrl_register_driver() failed!\n");
		return ret;
	}

	return ret;
}
module_init(mdss_dsi_ctrl_driver_init);

------------------  mdss-dsi  ------------------
static const struct of_device_id mdss_dsi_dt_match[] = {
	{.compatible = "qcom,mdss-dsi"},
	{}
};
MODULE_DEVICE_TABLE(of, mdss_dsi_dt_match);
static struct platform_driver mdss_dsi_driver = {
	.probe = mdss_dsi_probe,
	.remove = mdss_dsi_remove,
	.shutdown = NULL,
	.driver = {
		.name = "mdss_dsi",
		.of_match_table = mdss_dsi_dt_match,
	},
};

static int mdss_dsi_register_driver(void)
{
	return platform_driver_register(&mdss_dsi_driver);
}

static int __init mdss_dsi_driver_init(void)
{
	int ret;

	ret = mdss_dsi_register_driver();
	if (ret) {
		pr_err("mdss_dsi_register_driver() failed!\n");
		return ret;
	}

	return ret;
}
module_init(mdss_dsi_driver_init);
```
再看mdss_dsi_ctrl_probe(struct platform_device *pdev) 函数之前先看看mdss_dsi_probe(struct platform_device *pdev)
函数


##### 2.1、mdss_dsi_probe(struct platform_device *pdev)
为关键驱动数据结构 struct mdss_dsi_ctrl_pdata *ctrl_pdata 分配空间
``` cpp
G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\mdss_dsi.c
static int mdss_dsi_probe(struct platform_device *pdev)
{
	struct mdss_panel_cfg *pan_cfg = NULL;
	struct mdss_util_intf *util;
	char *panel_cfg;
	int rc = 0;

	util = mdss_get_util_intf();

	pan_cfg = util->panel_intf_type(MDSS_PANEL_INTF_HDMI);
	
	pan_cfg = util->panel_intf_type(MDSS_PANEL_INTF_DSI);

	rc = mdss_dsi_res_init(pdev);

	rc = mdss_dsi_parse_hw_cfg(pdev, panel_cfg);

	mdss_dsi_parse_pll_src_cfg(pdev, panel_cfg);

	of_platform_populate(pdev->dev.of_node, mdss_dsi_ctrl_dt_match,
				NULL, &pdev->dev);

	rc = mdss_dsi_validate_config(pdev);

	mdss_dsi_config_clk_src(pdev);

error:
	return rc;
}


static int mdss_dsi_res_init(struct platform_device *pdev)
{
	int rc = 0, i;
	struct dsi_shared_data *sdata;

	mdss_dsi_res = platform_get_drvdata(pdev);
	if (!mdss_dsi_res) {
		mdss_dsi_res = devm_kzalloc(&pdev->dev,
					  sizeof(struct mdss_dsi_data),
					  GFP_KERNEL);

		mdss_dsi_res->shared_data = devm_kzalloc(&pdev->dev,
				sizeof(struct dsi_shared_data),
				GFP_KERNEL);

		sdata = mdss_dsi_res->shared_data;

		rc = mdss_dsi_parse_dt_params(pdev, sdata);

		rc = mdss_dsi_core_clk_init(pdev, sdata);

		/* Parse the regulator information */
		for (i = DSI_CORE_PM; i < DSI_MAX_PM; i++) {
			rc = mdss_dsi_get_dt_vreg_data(&pdev->dev,
				pdev->dev.of_node, &sdata->power_data[i], i);
			if (rc) {
				i--;
				for (; i >= DSI_CORE_PM; i--)
					mdss_dsi_put_dt_vreg_data(&pdev->dev,
						&sdata->power_data[i]);
				goto mem_fail;
			}
		}
		rc = mdss_dsi_regulator_init(pdev, sdata);

		rc = mdss_dsi_bus_scale_init(pdev, sdata);

		mutex_init(&sdata->phy_reg_lock);
		mutex_init(&sdata->pm_qos_lock);

		for (i = 0; i < DSI_CTRL_MAX; i++) {
			mdss_dsi_res->ctrl_pdata[i] = devm_kzalloc(&pdev->dev,
					sizeof(struct mdss_dsi_ctrl_pdata),
					GFP_KERNEL);
				
				rc = -ENOMEM;
				goto mem_fail;
			}

			mdss_dsi_res->ctrl_pdata[i]->shared_data =
				mdss_dsi_res->shared_data;
		}

		platform_set_drvdata(pdev, mdss_dsi_res);
	}

	mdss_dsi_res->pdev = pdev;

	return 0;
}
```

mdss_dsi_ctrl_probe(struct platform_device *pdev) 函数

##### 2.2、mdss_dsi_ctrl_probe(struct platform_device *pdev)

``` cpp
static int mdss_dsi_ctrl_probe(struct platform_device *pdev)
{
	int rc = 0;
	u32 index;
	struct mdss_dsi_ctrl_pdata *ctrl_pdata = NULL;
	struct mdss_panel_info *pinfo = NULL;
	struct device_node *dsi_pan_node = NULL;
	const char *ctrl_name;
	struct mdss_util_intf *util;
	static int te_irq_registered;
	struct mdss_panel_data *pdata;
	struct mdss_panel_cfg *pan_cfg = NULL;

    //cell-index = <0>;
	rc = of_property_read_u32(pdev->dev.of_node,
				  "cell-index", &index);

	ctrl_pdata = mdss_dsi_get_ctrl(index);

	platform_set_drvdata(pdev, ctrl_pdata);

	util = mdss_get_util_intf();

	pan_cfg = util->panel_intf_type(MDSS_PANEL_INTF_SPI);

	ctrl_pdata->mdss_util = util;
	atomic_set(&ctrl_pdata->te_irq_ready, 0);
    //label = "MDSS DSI CTRL->0";
	ctrl_name = of_get_property(pdev->dev.of_node, "label", NULL); 
    //pin init
	rc = mdss_dsi_pinctrl_init(pdev);

	if (index == 0) {
		ctrl_pdata->panel_data.panel_info.pdest = DISPLAY_1;
		ctrl_pdata->ndx = DSI_CTRL_0;
	} else {
		ctrl_pdata->panel_data.panel_info.pdest = DISPLAY_2;
		ctrl_pdata->ndx = DSI_CTRL_1;
	}

	if (mdss_dsi_ctrl_clock_init(pdev, ctrl_pdata)) {
	}

	dsi_pan_node = mdss_dsi_config_panel(pdev, index);

	if (!mdss_dsi_is_hw_config_split(ctrl_pdata->shared_data) ||
		(mdss_dsi_is_hw_config_split(ctrl_pdata->shared_data) &&
		(ctrl_pdata->panel_data.panel_info.pdest == DISPLAY_1))) {
		rc = mdss_panel_parse_bl_settings(dsi_pan_node, ctrl_pdata);
		if (rc) {
			ctrl_pdata->bklt_ctrl = UNKNOWN_CTRL;
		}
	} else {
		ctrl_pdata->bklt_ctrl = UNKNOWN_CTRL;
	}
    ......

```
##### 2.2.1、mdss_dsi_config_panel()

解释panel的dts，为ctrl_pdata->ctrl_pdata赋值，并挂接如下三个重要的函数
	ctrl_pdata->on = mdss_dsi_panel_on;
	ctrl_pdata->post_panel_on = mdss_dsi_post_panel_on;
	ctrl_pdata->off = mdss_dsi_panel_off;

``` cpp
static struct device_node *mdss_dsi_config_panel(struct platform_device *pdev,
	int ndx)
{
	struct mdss_dsi_ctrl_pdata *ctrl_pdata = platform_get_drvdata(pdev);
	char panel_cfg[MDSS_MAX_PANEL_LEN];
	struct device_node *dsi_pan_node = NULL;
	int rc = 0;

	/* DSI panels can be different between controllers */
	rc = mdss_dsi_get_panel_cfg(panel_cfg, ctrl_pdata);

	/* find panel device node */
	dsi_pan_node = mdss_dsi_find_panel_of_node(pdev, panel_cfg);

	rc = mdss_dsi_panel_init(dsi_pan_node, ctrl_pdata, ndx);

	return dsi_pan_node;
}

int mdss_dsi_panel_init(struct device_node *node,
	struct mdss_dsi_ctrl_pdata *ctrl_pdata,
	int ndx)
{
	int rc = 0;
	static const char *panel_name;
	struct mdss_panel_info *pinfo;

	pinfo = &ctrl_pdata->panel_data.panel_info;

	pinfo->panel_name[0] = '\0';
	panel_name = of_get_property(node, "qcom,mdss-dsi-panel-name", NULL);

	rc = mdss_panel_parse_dt(node, ctrl_pdata);

	pinfo->dynamic_switch_pending = false;
	pinfo->is_lpm_mode = false;
	pinfo->esd_rdy = false;
	pinfo->persist_mode = false;

	ctrl_pdata->on = mdss_dsi_panel_on;
	ctrl_pdata->post_panel_on = mdss_dsi_post_panel_on;
	ctrl_pdata->off = mdss_dsi_panel_off;
	ctrl_pdata->low_power_config = mdss_dsi_panel_low_power_config;
	ctrl_pdata->panel_data.set_backlight = mdss_dsi_panel_bl_ctrl;
	ctrl_pdata->panel_data.apply_display_setting =
			mdss_dsi_panel_apply_display_setting;
	ctrl_pdata->switch_mode = mdss_dsi_panel_switch_mode;
	ctrl_pdata->panel_data.get_idle = mdss_dsi_panel_get_idle_mode;
	return 0;
}

```
##### 2.2.1.1、mdss_panel_parse_dt(node, ctrl_pdata)

``` cpp
G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\mdss_dsi_panel.c

static int mdss_panel_parse_dt(struct device_node *np,
			struct mdss_dsi_ctrl_pdata *ctrl_pdata)
{
	u32 tmp;
	u8 lanes = 0;
	int rc = 0;
	const char *data;
	static const char *pdest;
	struct mdss_panel_info *pinfo = &(ctrl_pdata->panel_data.panel_info);

	if (mdss_dsi_is_hw_config_split(ctrl_pdata->shared_data))
		pinfo->is_split_display = true;

	rc = of_property_read_u32(np,
		"qcom,mdss-pan-physical-width-dimension", &tmp);
	pinfo->physical_width = (!rc ? tmp : 0);
	rc = of_property_read_u32(np,
		"qcom,mdss-pan-physical-height-dimension", &tmp);
	pinfo->physical_height = (!rc ? tmp : 0);

	rc = of_property_read_u32(np, "qcom,mdss-dsi-bpp", &tmp);

	pinfo->bpp = (!rc ? tmp : 24);
	pinfo->mipi.mode = DSI_VIDEO_MODE;
	data = of_get_property(np, "qcom,mdss-dsi-panel-type", NULL);
	if (data && !strncmp(data, "dsi_cmd_mode", 12))
		pinfo->mipi.mode = DSI_CMD_MODE;
	pinfo->mipi.boot_mode = pinfo->mipi.mode;
	tmp = 0;
	data = of_get_property(np, "qcom,mdss-dsi-pixel-packing", NULL);
	if (data && !strcmp(data, "loose"))
		pinfo->mipi.pixel_packing = 1;
	else
		pinfo->mipi.pixel_packing = 0;
	rc = mdss_panel_get_dst_fmt(pinfo->bpp,
		pinfo->mipi.mode, pinfo->mipi.pixel_packing,
		&(pinfo->mipi.dst_format));

	pdest = of_get_property(np,
		"qcom,mdss-dsi-panel-destination", NULL);

	rc = of_property_read_u32(np,
		"qcom,mdss-dsi-underflow-color", &tmp);
	pinfo->lcdc.underflow_clr = (!rc ? tmp : 0xff);
	rc = of_property_read_u32(np,
		"qcom,mdss-dsi-border-color", &tmp);
	pinfo->lcdc.border_clr = (!rc ? tmp : 0);
	data = of_get_property(np, "qcom,mdss-dsi-panel-orientation", NULL);
	......

	rc = of_property_read_u32(np, "qcom,mdss-brightness-max-level", &tmp);
	pinfo->brightness_max = (!rc ? tmp : MDSS_MAX_BL_BRIGHTNESS);
	rc = of_property_read_u32(np, "qcom,mdss-dsi-bl-min-level", &tmp);
	pinfo->bl_min = (!rc ? tmp : 0);
	rc = of_property_read_u32(np, "qcom,mdss-dsi-bl-max-level", &tmp);
	pinfo->bl_max = (!rc ? tmp : 255);
	ctrl_pdata->bklt_max = pinfo->bl_max;

	rc = of_property_read_u32(np, "qcom,mdss-dsi-interleave-mode", &tmp);
	pinfo->mipi.interleave_mode = (!rc ? tmp : 0);

	pinfo->mipi.vsync_enable = of_property_read_bool(np,
		"qcom,mdss-dsi-te-check-enable");

	if (pinfo->sim_panel_mode == SIM_SW_TE_MODE)
		pinfo->mipi.hw_vsync_mode = false;
	else
		pinfo->mipi.hw_vsync_mode = of_property_read_bool(np,
			"qcom,mdss-dsi-te-using-te-pin");

	rc = of_property_read_u32(np,
		"qcom,mdss-dsi-h-sync-pulse", &tmp);
	pinfo->mipi.pulse_mode_hsa_he = (!rc ? tmp : false);

	pinfo->mipi.hfp_power_stop = of_property_read_bool(np,
		"qcom,mdss-dsi-hfp-power-mode");
	pinfo->mipi.hsa_power_stop = of_property_read_bool(np,
		"qcom,mdss-dsi-hsa-power-mode");
	pinfo->mipi.hbp_power_stop = of_property_read_bool(np,
		"qcom,mdss-dsi-hbp-power-mode");
	pinfo->mipi.last_line_interleave_en = of_property_read_bool(np,
		"qcom,mdss-dsi-last-line-interleave");
	pinfo->mipi.bllp_power_stop = of_property_read_bool(np,
		"qcom,mdss-dsi-bllp-power-mode");
	pinfo->mipi.eof_bllp_power_stop = of_property_read_bool(
		np, "qcom,mdss-dsi-bllp-eof-power-mode");
	pinfo->mipi.traffic_mode = DSI_NON_BURST_SYNCH_PULSE;
	data = of_get_property(np, "qcom,mdss-dsi-traffic-mode", NULL);
	if (data) {
		if (!strcmp(data, "non_burst_sync_event"))
			pinfo->mipi.traffic_mode = DSI_NON_BURST_SYNCH_EVENT;
		else if (!strcmp(data, "burst_mode"))
			pinfo->mipi.traffic_mode = DSI_BURST_MODE;
	}
	rc = of_property_read_u32(np,
		"qcom,mdss-dsi-te-dcs-command", &tmp);
	pinfo->mipi.insert_dcs_cmd =
			(!rc ? tmp : 1);
	rc = of_property_read_u32(np,
		"qcom,mdss-dsi-wr-mem-continue", &tmp);
	pinfo->mipi.wr_mem_continue =
			(!rc ? tmp : 0x3c);
	rc = of_property_read_u32(np,
		"qcom,mdss-dsi-wr-mem-start", &tmp);
	pinfo->mipi.wr_mem_start =
			(!rc ? tmp : 0x2c);
	rc = of_property_read_u32(np,
		"qcom,mdss-dsi-te-pin-select", &tmp);
	pinfo->mipi.te_sel =
			(!rc ? tmp : 1);
	rc = of_property_read_u32(np, "qcom,mdss-dsi-virtual-channel-id", &tmp);
	pinfo->mipi.vc = (!rc ? tmp : 0);
	pinfo->mipi.rgb_swap = DSI_RGB_SWAP_RGB;
	data = of_get_property(np, "qcom,mdss-dsi-color-order", NULL);
	if (data) {
		if (!strcmp(data, "rgb_swap_rbg"))
			pinfo->mipi.rgb_swap = DSI_RGB_SWAP_RBG;
		else if (!strcmp(data, "rgb_swap_bgr"))
			pinfo->mipi.rgb_swap = DSI_RGB_SWAP_BGR;
		else if (!strcmp(data, "rgb_swap_brg"))
			pinfo->mipi.rgb_swap = DSI_RGB_SWAP_BRG;
		else if (!strcmp(data, "rgb_swap_grb"))
			pinfo->mipi.rgb_swap = DSI_RGB_SWAP_GRB;
		else if (!strcmp(data, "rgb_swap_gbr"))
			pinfo->mipi.rgb_swap = DSI_RGB_SWAP_GBR;
	}
	pinfo->mipi.data_lane0 = of_property_read_bool(np,
		"qcom,mdss-dsi-lane-0-state");
	pinfo->mipi.data_lane1 = of_property_read_bool(np,
		"qcom,mdss-dsi-lane-1-state");
	pinfo->mipi.data_lane2 = of_property_read_bool(np,
		"qcom,mdss-dsi-lane-2-state");
	pinfo->mipi.data_lane3 = of_property_read_bool(np,
		"qcom,mdss-dsi-lane-3-state");

	if (pinfo->mipi.data_lane0)
		lanes++;
	if (pinfo->mipi.data_lane1)
		lanes++;
	if (pinfo->mipi.data_lane2)
		lanes++;
	if (pinfo->mipi.data_lane3)
		lanes++;
	/*
	 * needed to set default lanes during
	 * resolution switch for bridge chips
	 */
	pinfo->mipi.default_lanes = lanes;

	rc = mdss_panel_parse_display_timings(np, &ctrl_pdata->panel_data);

	rc = mdss_dsi_parse_hdr_settings(np, pinfo);

	pinfo->mipi.rx_eot_ignore = of_property_read_bool(np,
		"qcom,mdss-dsi-rx-eot-ignore");
	pinfo->mipi.tx_eot_append = of_property_read_bool(np,
		"qcom,mdss-dsi-tx-eot-append");

	rc = of_property_read_u32(np, "qcom,mdss-dsi-stream", &tmp);
	pinfo->mipi.stream = (!rc ? tmp : 0);

	data = of_get_property(np, "qcom,mdss-dsi-panel-mode-gpio-state", NULL);
	if (data) {
		if (!strcmp(data, "high"))
			pinfo->mode_gpio_state = MODE_GPIO_HIGH;
		else if (!strcmp(data, "low"))
			pinfo->mode_gpio_state = MODE_GPIO_LOW;
	} else {
		pinfo->mode_gpio_state = MODE_GPIO_NOT_VALID;
	}

	rc = of_property_read_u32(np, "qcom,mdss-mdp-transfer-time-us", &tmp);
	pinfo->mdp_transfer_time_us = (!rc ? tmp : DEFAULT_MDP_TRANSFER_TIME);

	pinfo->mipi.lp11_init = of_property_read_bool(np,
					"qcom,mdss-dsi-lp11-init");
	rc = of_property_read_u32(np, "qcom,mdss-dsi-init-delay-us", &tmp);
	pinfo->mipi.init_delay = (!rc ? tmp : 0);

	rc = of_property_read_u32(np, "qcom,mdss-dsi-post-init-delay", &tmp);
	pinfo->mipi.post_init_delay = (!rc ? tmp : 0);

	mdss_dsi_parse_trigger(np, &(pinfo->mipi.mdp_trigger),
		"qcom,mdss-dsi-mdp-trigger");

	mdss_dsi_parse_trigger(np, &(pinfo->mipi.dma_trigger),
		"qcom,mdss-dsi-dma-trigger");

	mdss_dsi_parse_reset_seq(np, pinfo->rst_seq, &(pinfo->rst_seq_len),
		"qcom,mdss-dsi-reset-sequence");

	mdss_dsi_parse_dcs_cmds(np, &ctrl_pdata->off_cmds,
		"qcom,mdss-dsi-off-command", "qcom,mdss-dsi-off-command-state");

	mdss_dsi_parse_dcs_cmds(np, &ctrl_pdata->idle_on_cmds,
		"qcom,mdss-dsi-idle-on-command",
		"qcom,mdss-dsi-idle-on-command-state");

	mdss_dsi_parse_dcs_cmds(np, &ctrl_pdata->idle_off_cmds,
		"qcom,mdss-dsi-idle-off-command",
		"qcom,mdss-dsi-idle-off-command-state");

	rc = of_property_read_u32(np, "qcom,mdss-dsi-idle-fps", &tmp);
	pinfo->mipi.frame_rate_idle = (!rc ? tmp : 60);

	rc = of_property_read_u32(np, "qcom,adjust-timer-wakeup-ms", &tmp);
	pinfo->adjust_timer_delay_ms = (!rc ? tmp : 0);

	pinfo->mipi.force_clk_lane_hs = of_property_read_bool(np,
		"qcom,mdss-dsi-force-clock-lane-hs");

	rc = mdss_dsi_parse_panel_features(np, ctrl_pdata);

	mdss_dsi_parse_panel_horizintal_line_idle(np, ctrl_pdata);

	mdss_dsi_parse_dfps_config(np, ctrl_pdata);

	rc = mdss_panel_parse_dt_hdmi(np, ctrl_pdata);

	return 0;

}

```


``` cpp
        解释如下项：
----------start---------    

		qcom,mdss-dsi-panel-width = <480>;
		qcom,mdss-dsi-panel-height = <854>;
		qcom,mdss-dsi-h-front-porch = <10>;
		qcom,mdss-dsi-h-back-porch = <10>;
		qcom,mdss-dsi-h-pulse-width = <10>;
		qcom,mdss-dsi-h-sync-skew = <0>;
		qcom,mdss-dsi-v-back-porch = <6>;
		qcom,mdss-dsi-v-front-porch = <6>;
		qcom,mdss-dsi-v-pulse-width = <4>;
		qcom,mdss-dsi-h-left-border = <0>;
		qcom,mdss-dsi-h-right-border = <0>;
		qcom,mdss-dsi-v-top-border = <0>;
		qcom,mdss-dsi-v-bottom-border = <0>;
		qcom,mdss-dsi-bpp = <24>;
		qcom,mdss-dsi-color-order = "rgb_swap_rgb";
		qcom,mdss-dsi-underflow-color = <0xff>;
		qcom,mdss-dsi-border-color = <0>;
		qcom,mdss-dsi-on-command = [
			......
			];
		qcom,mdss-dsi-off-command = [
			05 01 00 00 32 00 01
				28
			05 01 00 00 78 00 01
				10
			];
		qcom,mdss-dsi-on-command-state = "dsi_lp_mode";
		qcom,mdss-dsi-off-command-state = "dsi_hs_mode";
		qcom,mdss-dsi-h-sync-pulse = <0>;
		qcom,mdss-dsi-traffic-mode = "non_burst_sync_event";
		qcom,mdss-dsi-bllp-eof-power-mode;
		qcom,mdss-dsi-bllp-power-mode;
		qcom,mdss-dsi-lane-0-state;
		qcom,mdss-dsi-lane-1-state;
		qcom,mdss-dsi-panel-timings =
                                       [66 14 0C 00 34 38 10 18 0E 03 04 00];
		qcom,mdss-dsi-t-clk-post = <0x04>;
		qcom,mdss-dsi-t-clk-pre = <0x1D>;
		qcom,mdss-dsi-dma-trigger = "trigger_sw";
		qcom,mdss-dsi-mdp-trigger = "none";
		qcom,mdss-dsi-reset-sequence = <1 20>, <0 2>, <1 20>;
----------end---------    
```


##### 2.2.2、dsi_panel_device_register(pdev, dsi_pan_node, ctrl_pdata)

``` cpp
int dsi_panel_device_register(struct platform_device *ctrl_pdev,
	struct device_node *pan_node, struct mdss_dsi_ctrl_pdata *ctrl_pdata)
{
	struct mipi_panel_info *mipi;
	int rc;
	struct dsi_shared_data *sdata;
	struct mdss_panel_info *pinfo = &(ctrl_pdata->panel_data.panel_info);
	struct resource *res;
	u64 clk_rate;

	mipi  = &(pinfo->mipi);

	rc = mdss_dsi_clk_div_config(pinfo, mipi->frame_rate);

	ctrl_pdata->pclk_rate = mipi->dsi_pclk_rate;
	clk_rate = pinfo->clk_rate;
	do_div(clk_rate, 8U);
	ctrl_pdata->byte_clk_rate = (u32)clk_rate;

	rc = mdss_dsi_get_dt_vreg_data(&ctrl_pdev->dev, pan_node,
		&ctrl_pdata->panel_power_data, DSI_PANEL_PM);

	rc = msm_dss_config_vreg(&ctrl_pdev->dev,
		ctrl_pdata->panel_power_data.vreg_config,
		ctrl_pdata->panel_power_data.num_vreg, 1);

	rc = mdss_dsi_parse_ctrl_params(ctrl_pdev, pan_node, ctrl_pdata);

	pinfo->panel_max_fps = mdss_panel_get_framerate(pinfo,
				FPS_RESOLUTION_HZ);
	pinfo->panel_max_vtotal = mdss_panel_get_vtotal(pinfo);

	rc = mdss_dsi_parse_gpio_params(ctrl_pdev, ctrl_pdata);

	if (mdss_dsi_retrieve_ctrl_resources(ctrl_pdev,
					     pinfo->pdest,
					     ctrl_pdata)) {
	}

	ctrl_pdata->panel_data.event_handler = mdss_dsi_event_handler;
	ctrl_pdata->panel_data.get_fb_node = mdss_dsi_get_fb_node_cb;

	if (ctrl_pdata->status_mode == ESD_REG ||
			ctrl_pdata->status_mode == ESD_REG_NT35596)
		ctrl_pdata->check_status = mdss_dsi_reg_status_check;
	else if (ctrl_pdata->status_mode == ESD_BTA)
		ctrl_pdata->check_status = mdss_dsi_bta_status_check;

	if (ctrl_pdata->status_mode == ESD_MAX) {
		pr_err("%s: Using default BTA for ESD check\n", __func__);
		ctrl_pdata->check_status = mdss_dsi_bta_status_check;
	}
	if (ctrl_pdata->bklt_ctrl == BL_PWM)
		mdss_dsi_panel_pwm_cfg(ctrl_pdata);

	mdss_dsi_ctrl_init(&ctrl_pdev->dev, ctrl_pdata);
	mdss_dsi_set_prim_panel(ctrl_pdata);

	ctrl_pdata->dsi_irq_line = of_property_read_bool(
				ctrl_pdev->dev.of_node, "qcom,dsi-irq-line");

	if (ctrl_pdata->dsi_irq_line) {
		/* DSI has it's own irq line */
		res = platform_get_resource(ctrl_pdev, IORESOURCE_IRQ, 0);
		rc = mdss_dsi_irq_init(&ctrl_pdev->dev, res->start, ctrl_pdata);
	}
	ctrl_pdata->ctrl_state = CTRL_STATE_UNKNOWN;

	/*
	 * If ULPS during suspend is enabled, add an extra vote for the
	 * DSI CTRL power module. This keeps the regulator always enabled.
	 * This is needed for the DSI PHY to maintain ULPS state during
	 * suspend also.
	 */
	sdata = ctrl_pdata->shared_data;

	if (pinfo->ulps_suspend_enabled) {
		rc = msm_dss_enable_vreg(
			sdata->power_data[DSI_PHY_PM].vreg_config,
			sdata->power_data[DSI_PHY_PM].num_vreg, 1);
		if (rc) {
			return rc;
		}
	}

	pinfo->cont_splash_enabled =
		ctrl_pdata->mdss_util->panel_intf_status(pinfo->pdest,
		MDSS_PANEL_INTF_DSI) ? true : false;

	rc = mdss_register_panel(ctrl_pdev, &(ctrl_pdata->panel_data));

	if (pinfo->pdest == DISPLAY_1) {
		mdss_debug_register_io("dsi0_ctrl", &ctrl_pdata->ctrl_io, NULL);
		mdss_debug_register_io("dsi0_phy", &ctrl_pdata->phy_io, NULL);
		if (ctrl_pdata->phy_regulator_io.len)
			mdss_debug_register_io("dsi0_phy_regulator",
				&ctrl_pdata->phy_regulator_io, NULL);
	} else {
		mdss_debug_register_io("dsi1_ctrl", &ctrl_pdata->ctrl_io, NULL);
		mdss_debug_register_io("dsi1_phy", &ctrl_pdata->phy_io, NULL);
		if (ctrl_pdata->phy_regulator_io.len)
			mdss_debug_register_io("dsi1_phy_regulator",
				&ctrl_pdata->phy_regulator_io, NULL);
	}
	return 0;
}
```
##### 2.2.2.1、mdss_register_panel(ctrl_pdev, &(ctrl_pdata->panel_data))

``` cpp
int mdss_register_panel(struct platform_device *pdev,
	struct mdss_panel_data *pdata)
{
	struct platform_device *fb_pdev, *mdss_pdev;
	struct device_node *node = NULL;
	int rc = 0;
	bool master_panel = true;

	if (pdata->get_fb_node)
		node = pdata->get_fb_node(pdev);


	mdss_pdev = of_find_device_by_node(node->parent);

	pdata->active = true;
	fb_pdev = of_find_device_by_node(node); //mdss_mdp: qcom,mdss_mdp@900000
	if (fb_pdev) {
		rc = mdss_fb_register_extra_panel(fb_pdev, pdata); //如果是双屏，这个是第二块屏使用
		if (rc == 0)
			master_panel = false;
	} else {
		pr_info("adding framebuffer device %s\n", dev_name(&pdev->dev));
		//注册qcom,mdss_fb_primary设备，设备注册完成后将与驱动的match,match后调用 mdss_fb_probe(),这个将在fb分析
		fb_pdev = of_platform_device_create(node, NULL,
				&mdss_pdev->dev);
		if (fb_pdev)
			fb_pdev->dev.platform_data = pdata;
	}

	if (master_panel && mdp_instance->panel_register_done)
		mdp_instance->panel_register_done(pdata);

mdss_notfound:
	of_node_put(node);
	return rc;
}
```

#### (三)、FrameBuffer分析

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/display.system/Android.PG2.fb_probe.png)

``` cpp
=======================================================================================
【 fbmem.c (kernel\drivers\video) 】
static int __init  fbmem_init(void)
{
    1.1、在/proc/下创建fb节点
    proc_create("fb", 0, NULL, &fb_proc_fops);
    1.2、创建字符设备fb
    //#define FB_MAJOR      29   /* /dev/fb* framebuffers */  android下在/dev/graphics/fb*
    register_chrdev(FB_MAJOR,"fb",&fb_fops) //file_operations----|
    1.3、在/sys/class/目录下创建graphics                           |
    fb_class = class_create(THIS_MODULE, "graphics");            |
    return 0;                                                    |
}                                                                |
static const struct file_operations fb_fops = {  <---------------|
    .owner =    THIS_MODULE,
    .read =     fb_read,
    .write =    fb_write,
    .unlocked_ioctl = fb_ioctl,
    .mmap =     fb_mmap,
    .open =     fb_open,
    .release =  fb_release,
    .llseek =   default_llseek,
};
---------------------------------------------------------------------------
```
接下来看看mdss_fb_probe(struct platform_device *pdev) 函数。

``` cpp
G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\mdss_fb.c

static const struct of_device_id mdss_fb_dt_match[] = {
	{ .compatible = "qcom,mdss-fb",},
	{}
};
EXPORT_COMPAT("qcom,mdss-fb");

static struct platform_driver mdss_fb_driver = {
	.probe = mdss_fb_probe,
	.remove = mdss_fb_remove,
	.suspend = mdss_fb_suspend,
	.resume = mdss_fb_resume,
	.shutdown = mdss_fb_shutdown,
	.driver = {
		.name = "mdss_fb",
		.of_match_table = mdss_fb_dt_match,
		.pm = &mdss_fb_pm_ops,
	},
};

static int mdss_fb_probe(struct platform_device *pdev)
{
	struct msm_fb_data_type *mfd = NULL;
	struct mdss_panel_data *pdata;
	struct fb_info *fbi;
	int rc;
	const char *data;
	
	pdata = dev_get_platdata(&pdev->dev);
    //为关键结构体struct msm_fb_data_type *mfd和 struct fb_info *fbi 分配空间
	/*
	 * alloc framebuffer info + par data
	 */
	fbi = framebuffer_alloc(sizeof(struct msm_fb_data_type), NULL);
	if (fbi == NULL) {
		pr_err("can't allocate framebuffer info data!\n");
		return -ENOMEM;
	}
    //为mfd的部分成员赋初值
    
	mfd = (struct msm_fb_data_type *)fbi->par;
	mfd->key = MFD_KEY;
	mfd->fbi = fbi;
	mfd->panel_info = &pdata->panel_info;
	mfd->panel.type = pdata->panel_info.type;
	mfd->panel.id = mfd->index;
	mfd->fb_page = MDSS_FB_NUM;
	mfd->index = fbi_list_index;
	mfd->mdp_fb_page_protection = MDP_FB_PAGE_PROTECTION_WRITECOMBINE;

	mfd->ext_ad_ctrl = -1;
	if (mfd->panel_info && mfd->panel_info->brightness_max > 0)
		MDSS_BRIGHT_TO_BL(mfd->bl_level, backlight_led.brightness,
		mfd->panel_info->bl_max, mfd->panel_info->brightness_max);
	else
		mfd->bl_level = 0;

	mfd->bl_scale = 1024;
	mfd->bl_min_lvl = 30;
	mfd->ad_bl_level = 0;
	mfd->fb_imgType = MDP_RGBA_8888;
	mfd->calib_mode_bl = 0;
	mfd->unset_bl_level = U32_MAX;

	mfd->pdev = pdev;
	......
	mfd->split_fb_left = mfd->split_fb_right = 0;

	mdss_fb_set_split_mode(mfd, pdata);

   // mdp的接口函数拷贝一份到mfd->mdp
	mfd->mdp = *mdp_instance;

	rc = of_property_read_bool(pdev->dev.of_node,
		"qcom,boot-indication-enabled");

	if (rc) {
		led_trigger_register_simple("boot-indication",
			&(mfd->boot_notification_led));
	}

	INIT_LIST_HEAD(&mfd->file_list);

	mutex_init(&mfd->bl_lock);
	mutex_init(&mfd->mdss_sysfs_lock);
	mutex_init(&mfd->switch_lock);

	fbi_list[fbi_list_index++] = fbi;

	platform_set_drvdata(pdev, mfd);

	rc = mdss_fb_register(mfd);

	mdss_fb_create_sysfs(mfd);
	mdss_fb_send_panel_event(mfd, MDSS_EVENT_FB_REGISTERED, fbi);


    //mdss_mdp_overlay_init(),初始化mfd->mdp结构体的成员
	if (mfd->mdp.init_fnc) {
		rc = mfd->mdp.init_fnc(mfd);
		if (rc) {
			pr_err("init_fnc failed\n");
			return rc;
		}
	}
	mdss_fb_init_fps_info(mfd);

	rc = pm_runtime_set_active(mfd->fbi->dev);

	pm_runtime_enable(mfd->fbi->dev);

	/* android supports only one lcd-backlight/lcd for now */
	if (!lcd_backlight_registered) {
		backlight_led.brightness = mfd->panel_info->brightness_max;
		backlight_led.max_brightness = mfd->panel_info->brightness_max;
		if (led_classdev_register(&pdev->dev, &backlight_led))
			pr_err("led_classdev_register failed\n");
		else
			lcd_backlight_registered = 1;
	}

	mdss_fb_init_panel_modes(mfd, pdata);
    ......
	return rc;
}

```


##### 3.1、mdss_fb_register(mfd)

``` cpp
static int mdss_fb_register(struct msm_fb_data_type *mfd)
{
	int ret = -ENODEV;
	int bpp;
	char panel_name[20];
	struct mdss_panel_info *panel_info = mfd->panel_info;
	struct fb_info *fbi = mfd->fbi;
	struct fb_fix_screeninfo *fix;
	struct fb_var_screeninfo *var;
	int *id;

	/*
	 * fb info initialization
	 */
	fix = &fbi->fix;
	var = &fbi->var;

	fix->type_aux = 0;	/* if type == FB_TYPE_INTERLEAVED_PLANES */
	fix->visual = FB_VISUAL_TRUECOLOR;	/* True Color */
	fix->ywrapstep = 0;	/* No support */
	fix->mmio_start = 0;	/* No MMIO Address */
	fix->mmio_len = 0;	/* No MMIO Address */
	fix->accel = FB_ACCEL_NONE;/* FB_ACCEL_MSM needes to be added in fb.h */

	var->xoffset = 0,	/* Offset from virtual to visible */
	var->yoffset = 0,	/* resolution */
	var->grayscale = 0,	/* No graylevels */
	var->nonstd = 0,	/* standard pixel format */
	var->activate = FB_ACTIVATE_VBL,	/* activate it at vsync */
	var->height = -1,	/* height of picture in mm */
	var->width = -1,	/* width of picture in mm */
	var->accel_flags = 0,	/* acceleration flags */
	var->sync = 0,	/* see FB_SYNC_* */
	var->rotate = 0,	/* angle we rotate counter clockwise */
	mfd->op_enable = false;

	switch (mfd->fb_imgType) {
	case MDP_RGB_565:
		fix->type = FB_TYPE_PACKED_PIXELS;
		fix->xpanstep = 1;
		fix->ypanstep = 1;
		var->vmode = FB_VMODE_NONINTERLACED;
		var->blue.offset = 0;
		var->green.offset = 5;
		var->red.offset = 11;
		var->blue.length = 5;
		var->green.length = 6;
		var->red.length = 5;
		var->blue.msb_right = 0;
		var->green.msb_right = 0;
		var->red.msb_right = 0;
		var->transp.offset = 0;
		var->transp.length = 0;
		bpp = 2;
		break;

	//......

	default:
		pr_err("msm_fb_init: fb %d unkown image type!\n",
			    mfd->index);
		return ret;
	}

	mdss_panelinfo_to_fb_var(panel_info, var);

	fix->type = panel_info->is_3d_panel;
	if (mfd->mdp.fb_stride)
		fix->line_length = mfd->mdp.fb_stride(mfd->index, var->xres,
							bpp);
	else
		fix->line_length = var->xres * bpp;

	var->xres_virtual = var->xres;
	var->yres_virtual = panel_info->yres * mfd->fb_page;
	var->bits_per_pixel = bpp * 8;	/* FrameBuffer color depth */

	/*
	 * Populate smem length here for uspace to get the
	 * Framebuffer size when FBIO_FSCREENINFO ioctl is called.
	 */
	fix->smem_len = PAGE_ALIGN(fix->line_length * var->yres) * mfd->fb_page;

	/* id field for fb app  */
	id = (int *)&mfd->panel;

	snprintf(fix->id, sizeof(fix->id), "mdssfb_%x", (u32) *id);

	fbi->fbops = &mdss_fb_ops;

|--------------------------------------------------------------|
static struct fb_ops mdss_fb_ops = {
	.owner = THIS_MODULE,
	.fb_open = mdss_fb_open,
	.fb_release = mdss_fb_release,
	.fb_check_var = mdss_fb_check_var,	/* vinfo check */
	.fb_set_par = mdss_fb_set_par,	/* set the video mode */
	.fb_blank = mdss_fb_blank,	/* blank display */
	.fb_pan_display = mdss_fb_pan_display,	/* pan display */
	.fb_ioctl_v2 = mdss_fb_ioctl,	/* perform fb specific ioctl */
#ifdef CONFIG_COMPAT
	.fb_compat_ioctl_v2 = mdss_fb_compat_ioctl,
#endif
	.fb_mmap = mdss_fb_mmap,
};
|-------------------------------------------------------------|

	fbi->flags = FBINFO_FLAG_DEFAULT;
	fbi->pseudo_palette = mdss_fb_pseudo_palette;

	mfd->ref_cnt = 0;
	mfd->panel_power_state = MDSS_PANEL_POWER_OFF;
	mfd->dcm_state = DCM_UNINIT;


    //为fb设备准备空间
	if (mdss_fb_alloc_fbmem(mfd))
		pr_warn("unable to allocate fb memory in fb register\n");

	mfd->op_enable = true;

	mutex_init(&mfd->update.lock);
	mutex_init(&mfd->no_update.lock);
	mutex_init(&mfd->mdp_sync_pt_data.sync_mutex);
	atomic_set(&mfd->mdp_sync_pt_data.commit_cnt, 0);
	atomic_set(&mfd->commits_pending, 0);
	atomic_set(&mfd->ioctl_ref_cnt, 0);
	atomic_set(&mfd->kickoff_pending, 0);

	init_timer(&mfd->no_update.timer);
	mfd->no_update.timer.function = mdss_fb_no_update_notify_timer_cb;
	mfd->no_update.timer.data = (unsigned long)mfd;
	mfd->update.ref_count = 0;
	mfd->no_update.ref_count = 0;
	mfd->update.init_done = false;
	init_completion(&mfd->update.comp);
	init_completion(&mfd->no_update.comp);
	init_completion(&mfd->power_off_comp);
	init_completion(&mfd->power_set_comp);
	init_waitqueue_head(&mfd->commit_wait_q);
	init_waitqueue_head(&mfd->idle_wait_q);
	init_waitqueue_head(&mfd->ioctl_q);
	init_waitqueue_head(&mfd->kickoff_wait_q);

	ret = fb_alloc_cmap(&fbi->cmap, 256, 0);
	
	if (register_framebuffer(fbi) < 0) {
		fb_dealloc_cmap(&fbi->cmap);

		mfd->op_enable = false;
		return -EPERM;
	}

	snprintf(panel_name, ARRAY_SIZE(panel_name), "mdss_panel_fb%d",
		mfd->index);
	mdss_panel_debugfs_init(panel_info, panel_name);
    //log:mdss_fb_register [ : FrameBuffer[1] 640x480 registered successfully!
	pr_info("FrameBuffer[%d] %dx%d registered successfully!\n", mfd->index,
					fbi->var.xres, fbi->var.yres);
	return 0;
}
```
> mdss_fb_register(mfd);  //关键函数
> 		1、为fbi的fix和var成员赋值
> 		2、为fb0分配空间		  //mdss_fb_alloc_fbmem(mfd)
> 	    3、初始化fbi->cmap		//fb_alloc_cmap(&fbi->cmap, 256, 0);
> 		4、register_framebuffer(fbi) //注册fbi，在/sys/class/graphics/目录下创建fb0，并创建相关属性
> 

##### 3.2、mdss_mdp_overlay_init(struct msm_fb_data_type *mfd)

> mfd->mdp的接口函数
> 在mdss_fb_probe()的probe函数中
>     mfd->mdp = *mdp_instance; //将mdp_instance的所有内容拷贝一份给 mfd->mdp
> 所以mfd->mdp等于mdp5，mdp_instance指向mdp5   

``` cpp
G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\mdss_mdp.c
struct msm_mdp_interface mdp5 = {
	.init_fnc = mdss_mdp_overlay_init,
	.fb_mem_get_iommu_domain = mdss_fb_mem_get_iommu_domain,
	.fb_stride = mdss_mdp_fb_stride,
	.check_dsi_status = mdss_check_dsi_ctrl_status,
	.get_format_params = mdss_mdp_get_format_params,
};
```

``` cpp
G:\android9.0\kernel\msm-3.18\drivers\video\msm\mdss\mdss_mdp_overlay.c
int mdss_mdp_overlay_init(struct msm_fb_data_type *mfd)
{
	struct device *dev = mfd->fbi->dev;
	struct msm_mdp_interface *mdp5_interface = &mfd->mdp;
	struct mdss_overlay_private *mdp5_data = NULL;
	struct irq_info *mdss_irq;
	int rc;

	mdp5_data = kzalloc(sizeof(struct mdss_overlay_private), GFP_KERNEL);

	mdp5_data->frc_fsm
		= kzalloc(sizeof(struct mdss_mdp_frc_fsm), GFP_KERNEL);

	mdp5_data->mdata = dev_get_drvdata(mfd->pdev->dev.parent);

	mdp5_interface->on_fnc = mdss_mdp_overlay_on;
	mdp5_interface->off_fnc = mdss_mdp_overlay_off;
	mdp5_interface->release_fnc = __mdss_mdp_overlay_release_all;
	mdp5_interface->do_histogram = NULL;
	if (mdp5_data->mdata->ncursor_pipes)
		mdp5_interface->cursor_update = mdss_mdp_hw_cursor_pipe_update;
	else
		mdp5_interface->cursor_update = mdss_mdp_hw_cursor_update;
	mdp5_interface->async_position_update =
		mdss_mdp_async_position_update;
	mdp5_interface->dma_fnc = mdss_mdp_overlay_pan_display;
	mdp5_interface->ioctl_handler = mdss_mdp_overlay_ioctl_handler;
	mdp5_interface->kickoff_fnc = mdss_mdp_overlay_kickoff;
	mdp5_interface->mode_switch = mdss_mode_switch;
	mdp5_interface->mode_switch_post = mdss_mode_switch_post;
	mdp5_interface->pre_commit_fnc = mdss_mdp_overlay_precommit;
	mdp5_interface->splash_init_fnc = mdss_mdp_splash_init;
	mdp5_interface->configure_panel = mdss_mdp_update_panel_info;
	mdp5_interface->input_event_handler = mdss_mdp_input_event_handler;
	mdp5_interface->signal_retire_fence = mdss_mdp_signal_retire_fence;
	mdp5_interface->is_twm_en = NULL;
    ......
	rc = mdss_mdp_overlay_fb_parse_dt(mfd);

	/*
	 * disable BWC if primary panel is video mode on specific
	 * chipsets to workaround HW problem.
	 */
	if (mdss_has_quirk(mdp5_data->mdata, MDSS_QUIRK_BWCPANIC) &&
	    mfd->panel_info->type == MIPI_VIDEO_PANEL && (0 == mfd->index))
		mdp5_data->mdata->has_bwc = false;

	mfd->panel_orientation = mfd->panel_info->panel_orientation;

	if ((mfd->panel_info->panel_orientation & MDP_FLIP_LR) &&
	    (mfd->split_mode == MDP_DUAL_LM_DUAL_DISPLAY))
		mdp5_data->mixer_swap = true;
     //在/sys/class/graphics/fb0/创建 vsync_event和ad属性文件
	rc = sysfs_create_group(&dev->kobj, &mdp_overlay_sysfs_group);

	......

	rc = sysfs_create_link_nowarn(&dev->kobj,
			&mdp5_data->mdata->pdev->dev.kobj, "mdp");

	rc = sysfs_create_link_nowarn(&dev->kobj,
			&mfd->pdev->dev.kobj, "mdss_fb");

	......

	return rc;
}
```

#### (四)、/dev/graphics/fb0---ioctl重要流程分析

##### 4.1、device_fd_ = open( "/dev/graphics/fb0", O_RDWR)
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/display.system/Android.PG2.fb_open.png)

##### 4.2、ioctl(device_fd_, INT(MSMFB_ATOMIC_COMMIT), &mdp_disp_commit_)
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/display.system/Android.PG2.fb_MSMFB_ATOMIC_COMMIT.png)

#### (五)、FrameBufferTest测试app分析

> commit ---  ioctl(fd, FBIOPAN_DISPLAY, &info) 
>|------>mdss_fb_ioctl:FBIOPAN_DISPLAY
>|----------->mdss_fb_pan_display();
>|--------------->mdss_fb_pan_display_ex();
唤醒__mdss_fb_display_thread后流程大概一致，就不分析了。
##### 小结：
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianin/PicGo/master/display.system/Android.PG2.mdp.dsi.fb_probe.png)

#### (五)、参考资料(特别感谢)：

[（1）【[Android5.1][RK3288] LCD Mipi 调试方法及问题汇总】](https://blog.csdn.net/dearsq/article/details/52354593) 
[（2）【[Android6.0][RK3399] Mipi LCD 通用移植调试流程】](https://blog.csdn.net/dearsq/article/details/77341120) 
[（3）【MDP分析，DSI分析，FB分析，Display resume suspend分析】](https://github.com/wj123/github/tree/master/android/module/display/liuzl) 
[（4）【高通平台LCD之MDP code解析】](https://blog.csdn.net/u012452964/article/details/74841290) 



