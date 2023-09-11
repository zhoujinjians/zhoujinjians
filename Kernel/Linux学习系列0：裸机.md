---
title: Linux学习系列0：裸机
cover: https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.41.jpg
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

[【me.zhoujinjian.com博客原图链接】](https://github.com/zhoujinjiani) 

[【开发板 RockPi4bPlusV1.6】](https://shop.allnetchina.cn/collections/frontpage/products/rock-pi-4-model-b-board-only-2-4-5ghz-wlan-bluetooth-5-0)

[【开发板 RockPi4bPlusV1.6 Android 10.0 && Linux（Kernel 4.15）源码链接】](https://github.com/radxa/manifests/tree/rockchip-android-10)



正是由于前人（各位大神）的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

 [【RK3399—裸机大全】](https://hceng.cn/2018/08/16/RK3399%E2%80%94%E2%80%94%E8%A3%B8%E6%9C%BA%E5%A4%A7%E5%85%A8/)    

以64位的RK3399为例，实现裸机的启动：LED、中断、串口(printf移植)、定时器、ADC、I2C、SPI；  



1.ARMv8基础
===================================

1.1 基本概念
--------------------------------

**1.架构和内核型号**

*   **架构(Architecture)**:就是常说的ARMv5(32bits)、ARMv6(32bits)、ARMv7(32bits)、ARMv8(32/64bits)；
*   **内核型号**:就是常说的ARM7、ARM9、Cortex-A系列(Aplication)、Cortex-R系列(Runtime)、Cortex-M系列(MCU)；
*   **举例**:单片机STM32F103C8T6采用Cortex-M3内核，采用ARMv7-M架构；  
    　　　　瑞芯微RK3288采用4个Cortex-A17，采用ARMv7-A架构；  
    　　　　瑞芯微RK3399采用2个Cortex-A72和4个Cortex-A53组成，Cortex-A72和Cortex-A53都是ARMv8-A架构。  
    　　　　高通骁龙845处理器由4个Cortex-A75和4个Cortex-A55组成，Cortex-A75和Cortex-A55都是ARMv8-A架构。
*   **发展迭代**：
    
    ![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/1.jpg)
    

**2.AArch64/AArch32/A64/A32/T32**

| 名字 | 类型 | 说明 |
| --- | --- | --- |
| AArch64 | 架构 | 指基于64bits运作的ARMv8架构（通用寄存器X0-X30） |
| AArch32 | 架构 | 指基于32bits运作的ARMv8架构，并且兼容之前的ARMv7架构（通用寄存器R0-R15） |
| A64 | 指令集 | 指在AArch64模式下支持的ARM 64bits指令集 |
| A32 | 指令集 | 指ARMv7架构下支持的ARM 32bits指令集，在ARMv8中也有新加入的A32指令集 |
| T32 | 指令集 | 指ARMv7架构下支持的Thumb2 16/32bits指定集，在ARMv8中也有新加入的T32指令集。 |

1.2 AArch64/32寄存器
-----------------------------------------------------------

| AArch64 | Special | Role in the procedure call standard |
| --- | --- | --- |
| x0…x7 |  | Parameter/result registers(参数传入/返回结果） |
| x8 |  | Indirect result location register |
| x9…x15 |  | Temporary registers(临时寄存器) |
| x16 | IP0 | The first intra-procedure-call scratch register (can be used by call veneers and PLT code); at other times may be used as a temporary register. |
| x17 | IP1 | The second intra-procedure-call temporary register (can be used by call veneers and PLT code); at other times may be used as a temporary register. |
| x18 |  | The Platform Register, if needed; otherwise a temporary register. |
| x19…x28 |  | Callee-saved registers(由被调用者保存的寄存器) |
| x29 | FP | The Frame Pointer(栈帧指针) |
| x30 | LR | The Link Register(链接寄存器) |
| SP |  | The Stack Pointer(栈指针) |
| AArch32 | Special | Role in the procedure call standard |
| --- | --- | --- |
| r0…r3 |  | Parameter/result registers |
| r4…r11 |  | Temporary registers (r9 also as platform register) |
| r12 | IP | The Intra-Procedure-call scratch register. |
| r13 | SP | The second intra-procedure-call temporary register (can be used by call veneers and PLT code); at other times may be used as a temporary register. |
| r14 | LR | The Platform Register, if needed; otherwise a temporary register. |
| r15 | PC | Callee-saved registers |

两者区别：

| Execution State | Note |
| --- | --- |
| AArch64 | 1\. 提供31个64bits的通用寄存器(x0~x30，其中x30可作为LR)  |
|  |2\. 提供64bits程序计数器(PC)、栈指针(SP)、异常链接寄存器(ELR)|
|  |3\. 提供32个128bits 的SIMD Vector与Scalar Floating-Point寄存器|
|  |4\. 定义ARMv8 EL0~EL3共4个执行权限(Execution Privilege)|
|  |5\. 支持64bits Virtual-Addressing|
| |6\. 定义一组PSTATE用以保存PE(Processing Element)状态|
| AArch32 | 1\. 提供16个32bits的通用寄存器(r0~r12，其中r13=SP、r14=LR、r15=PC，且r14需要同时供ELR与LR之用）  |
|  |2\. 提供一个ELR，用以作为从Hyp-Mode的Exception返回之用|
|  |3\. 提供32个64bits的Advanced SIMD Vector与Scalar Floating-Point寄存器|
|  |4\. 提供A32与T32两种指令集的组合|
|  |5\. 使用32bits Virtual-Addressing|
| |6\. 只使用CPSR(当前程序状态寄存器)保存PE(Processing Element)状态。|

1.3 ARMv8 Exception Level
-----------------------------------------------------------------------------------

针对Security的需求，ARMv8的系统软件设计可以提供安全模式与非安全模式的状态。  
ARMv8规定了CPU有4种运行级别。每种运行级别下标的数字越大，其权力级别越高。其中EL0为非特权等级，即平时应用程序运行时的级别；EL1为特权等级，即操作系统运行时的级别；EL2为虚拟机监视器运行级别，即虚拟机的控制层运行的级别；EL3为切换EL1和EL2级别时需要进入的一个级别，为CPU的最高级别。

若底层EL(Exception Level)为32bits，则上层EL的软件就只能是32位。

![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/2-16297118529022.png)

若底层的EL为64bits，则上层EL就可以依据需求选择为32bits或是64bits。

![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/3-16297118317991.png)

2.RK3399启动
======================================

先看一下RK3399的启动流程图\[1\]：

![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/4.png)

从图中可以得到以下几个结论：

*   1.RK3399上电后，会从`0xffff0000`获取`romcode`并运行；
*   2.然后依次从Nor Flash、Nand Flash、eMMC、SD/MMC获取`ID BLOCK`，`ID BLOCK`正确则启动，都不正确则从USB端口下载；
*   3.如果emmc启动，则先读取SDRAM(DDR)初始化代码到内部SRAM，然后初始化DDR，再将emmc上的代码(剩下的用户代码)复制到DDR运行；
*   4.如果从USB下载，则先获取DDR初始化代码，下载到内部SRAM中，然后运行代码初始化DDR，再获取loader代码(用户代码)，放到DDR中并运行；
*   5.无论是何种方式，都需要DDR的初始化代码。

2.1 官方启动分析
--------------------------------------

如何分析一款芯片的启动方式?大致就是先用厂家提供的资料，配置相关环境、编译、烧写，运行起来。然后就有了U-boot源码，从U-boot就可以几乎提取出所有的裸机代码，本文也是这样做的。

分析U-Boot的编译流程，可以看到如下内容：  

```c
load addr is 0x200000!
pack input u-boot.bin 
pack file size: 1034744(1010 KB)
crc = 0x2434c863
uboot version: U-Boot 2017.09-g745ea52 #xwj (Aug 03 2021 - 12:20:53)
pack uboot.img success! 
pack uboot okay! Input: u-boot.bin

out:trust.img
merge success(trust.img)
pack trust okay! Input: /home/zhoujinjian/Rock4Plus_Android10/Sync_Rock_PI4_Plus_Android10/rkbin/RKTRUST/RK3399TRUST.ini

/home/zhoujinjian/Rock4Plus_Android10/Sync_Rock_PI4_Plus_Android10/u-boot
out:rk3399_loader_v1.20.126.bin
fix opt:rk3399_loader_v1.20.126.bin
merge success(rk3399_loader_v1.20.126.bin)
pack loader okay! Input: /home/zhoujinjian/Rock4Plus_Android10/Sync_Rock_PI4_Plus_Android10/rkbin/RKBOOT/RK3399MINIALL.ini
/home/zhoujinjian/Rock4Plus_Android10/Sync_Rock_PI4_Plus_Android10/u-boot
Image Type:   Rockchip RK33 (SD/MMC) boot image
Init Data Size: 73728 bytes
Boot Data Size: 86016 bytes
Input:
    /home/zhoujinjian/Rock4Plus_Android10/Sync_Rock_PI4_Plus_Android10/rkbin/RKBOOT/RK3399MINIALL.ini
    /home/xwj/zhoujinjian/Rock4Plus_Android10/Sync_Rock_PI4_Plus_Android10/rkbin/bin/rk33/rk3399_ddr_800MHz_v1.20.bin
    /home/zhoujinjian/Rock4Plus_Android10/Sync_Rock_PI4_Plus_Android10/rkbin/bin/rk33/rk3399_miniloader_v1.26.bin

Pack rk3399 idbloader.img okay!
```

可以看出这里使用了三个工具，产生了三个文件：  
①:使用`boot_merger`，参数为`RK3399MINIALL.ini`，得到loader文件`rk3399_loader_v1.20.126.bin`，打开`RK3399MINIALL.ini`内容为：  

```c
[CHIP_NAME]
NAME=RK330C
[VERSION]
MAJOR=1
MINOR=26
[CODE471_OPTION]
NUM=1
Path1=bin/rk33/rk3399_ddr_800MHz_v1.20.bin
Sleep=1
[CODE472_OPTION]
NUM=1
Path1=bin/rk33/rk3399_usbplug_v1.26.bin
[LOADER_OPTION]
NUM=2
LOADER1=FlashData
LOADER2=FlashBoot
FlashData=bin/rk33/rk3399_ddr_800MHz_v1.20.bin
FlashBoot=bin/rk33/rk3399_miniloader_v1.26.bin
[OUTPUT]
PATH=rk3399_loader_v1.20.126.bin
```

得知依赖的文件有:DDR相关的`rk3399_ddr_800MHz_v1.20.bin`、、USB相关的`rk3399_usbplug_v1.26.bin`、miniloader(瑞芯微修改的一个bootloader)相关的`rk3399_miniloader_v1.26.bin`。  
`boot_merger`将这三个bin文件最后合并成rk3399_loader_v1.20.126.bin。

②:使用`trust_merger`，参数为`RK3399TRUST.ini`，生成`trust.img`；

③:使用`loaderimage`将`u-boot.bin`变成`uboot.img`；

2.2 制作裸机启动文件
--------------------------------------------

经过分析和测试，现实现了emmc裸机程序，并把整个过程整理了一个工程模板。  
工程模板见[GitHub](https://github.com/zhoujinjiani/u_boot_demo)，替换官方u-boot，然后编译update.img

```c
# u_boot_demo for rockpi4b_plus_v1.6

# 0、Env(NO) && Download Source Code

$ curl https://storage.googleapis.com/git-repo-downloads/repo-1 > ~/bin/repo

$ chmod a+x ~/bin/repo

$ python3 ~/bin/repo init -u https://github.com/radxa/manifests.git -b rockchip-android-10 -m rockchip-q-release.xml

$ cd u-boot 

$ rm -rf *

$ cp u-boot-xxx-ok/* to u-boot 


# 1、ROCKPI 4A/B（编译u-boot）

$ cd u-boot

$ ./make.sh rockpi4b

$ cd -


# 2、ROCKPI 4A/B（编译kernel）

$ cd kernel

$ make ARCH=arm64 rockchip_defconfig android-10.config rockpi_4b.config

$ make ARCH=arm64 rk3399-rockpi-4b.img -j8

$ cd -


# 3、ROCKPI 4A/B（编译Android）

$ source build/envsetup.sh

$ lunch RockPi4B-userdebug

$ make -j8


# 4、ROCKPI 4A/B（mkimage）

$ ln -s RKTools/linux/Linux_Pack_Firmware/rockdev/ rockdev

$ ./mkimage.sh


# 5、ROCKPI 4A/B（编译update.img）

$ cd rockdev

$ ln -s Image-RockPi4B Image

$ ./mkupdate_rk3399.sh


Please burn rockdev/update.img

https://wiki.radxa.com/Android/android_tool

```



烧录update.img（每次都烧录update.img好像有点~.~） ：

![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/6.png)

3.Uboot启动部分分析
===============================================

为了方便后面从U-boot提取所需裸机代码，有必要先对U-boot进行分析，本节只分析启动部分的，后续具体某个模块，如LED将在后面对应的章节分析。  
另外，本次分析是的RK3399，64位的ARMv8架构，与市面上较多的32位ARMv7架构SOC略有区别，注意不要混淆。  
U-boot执行的第一个文件是start.S，下面开始对其进行分析。

3.1 start.S
-----------------------------------------

所在文件路径：[`u-boot/arch/arm/cpu/armv8/start.S`](https://github.com/radxa/u-boot/blob/stable-4.4-rockpi4/arch/arm/cpu/armv8/start.S)

```c

/*************************************************************************
 *
 * Startup Code (reset vector)
 *
 *************************************************************************/

.globl	_start
_start:
	b	reset
	.align 3
......
reset:
	/* Allow the board to save important registers */
	b	save_boot_params
.globl	save_boot_params_ret
save_boot_params_ret:
	/*
	 * Could be EL3/EL2/EL1, Initial State:
	 * Little Endian, MMU Disabled, i/dCache Disabled
	 */
	adr	x0, vectors
	switch_el x1, 3f, 2f, 1f
3:	msr	vbar_el3, x0
	mrs	x0, scr_el3
	orr	x0, x0, #0xf			/* SCR_EL3.NS|IRQ|FIQ|EA */
	msr	scr_el3, x0
	msr	cptr_el3, xzr			/* Enable FP/SIMD */
	/* COUNTER_FREQUENCY 24000000 \u-boot\include\configs\rockchip-common.h */
	ldr	x0, =24000000
	msr	cntfrq_el0, x0			/* Initialize CNTFRQ */
	b	0f
2:	msr	vbar_el2, x0
	mov	x0, #0x33ff
	msr	cptr_el2, x0			/* Enable FP/SIMD */
	b	0f
1:	msr	vbar_el1, x0
	mov	x0, #3 << 20
	msr	cpacr_el1, x0			/* Enable FP/SIMD */
0:

	/*
	 * Enable SMPEN bit for coherency.
	 * This register is not architectural but at the moment
	 * this bit should be set for A53/A57/A72.
	 */
	switch_el x1, 3f, 1f, 1f
3:
	mrs     x0, S3_1_c15_c2_1               /* cpuectlr_el1 */
	orr     x0, x0, #0x40
	msr     S3_1_c15_c2_1, x0
1:

	/* Apply ARM core specific erratas */
	bl	apply_core_errata

	/*
	 * Cache/BPB/TLB Invalidate
	 * i-cache is invalidated before enabled in icache_enable()
	 * tlb is invalidated before mmu is enabled in dcache_enable()
	 * d-cache is invalidated before enabled in dcache_enable()
	 */

	/* Processor specific initialization */
	bl	lowlevel_init 
```

*   **1.设置中断向量等 \[important\]**
    
    ```c
    /*F:\Rock4Plus_Android10\u-boot\arch\arm\cpu\armv8\exceptions.S*/
    
    /*
     * Exception vectors.
     */
    	.align	11
    	.globl	vectors
    vectors:
    	.align	7		/* Current EL Synchronous Thread */
    	stp	x29, x30, [sp, #-16]!
    	bl	_exception_entry
    	/*bl	do_bad_sync*/
    	b	exception_exit
    
    	.align	7		/* Current EL IRQ Thread */
    	stp	x29, x30, [sp, #-16]!
    	bl	_exception_entry
    	/*bl	do_bad_irq*/
    	b	exception_exit
    
    	.align	7		/* Current EL FIQ Thread */
    	stp	x29, x30, [sp, #-16]!
    	bl	_exception_entry
    	/*bl	do_bad_fiq*/
    	b	exception_exit
    
    	.align	7		/* Current EL Error Thread */
    	stp	x29, x30, [sp, #-16]!
    	bl	_exception_entry
    	/*bl	do_bad_error*/
    	b	exception_exit
    
    	.align	7		 /* Current EL Synchronous Handler */
    	stp	x29, x30, [sp, #-16]!
    	bl	_exception_entry
    	/*bl	do_sync*/
    	b	exception_exit
    
    	.align	7		 /* Current EL IRQ Handler */
    	stp	x29, x30, [sp, #-16]!
    	bl	_exception_entry
    	//bl	do_irq
    	//bl	board_init_f_init_reserve
    	b	exception_exit
    
    	.align	7		 /* Current EL FIQ Handler */
    	stp	x29, x30, [sp, #-16]!
    	bl	_exception_entry
    	/*bl	do_fiq*/
    	b	exception_exit
    
    	.align	7		 /* Current EL Error Handler */
    	stp	x29, x30, [sp, #-16]!
    	bl	_exception_entry
    	/*bl	do_error*/
    	b	exception_exit
    
    /*
     * Enter Exception.
     * This will save the processor state that is ELR/X0~X30
     * to the stack frame.
     */
    _exception_entry:
    	stp	x27, x28, [sp, #-16]!
    	stp	x25, x26, [sp, #-16]!
    	stp	x23, x24, [sp, #-16]!
    	stp	x21, x22, [sp, #-16]!
    	stp	x19, x20, [sp, #-16]!
    	stp	x17, x18, [sp, #-16]!
    	stp	x15, x16, [sp, #-16]!
    	stp	x13, x14, [sp, #-16]!
    	stp	x11, x12, [sp, #-16]!
    	stp	x9, x10, [sp, #-16]!
    	stp	x7, x8, [sp, #-16]!
    	stp	x5, x6, [sp, #-16]!
    	stp	x3, x4, [sp, #-16]!
    	stp	x1, x2, [sp, #-16]!
    
    	/* Could be running at EL3/EL2/EL1 */
    	switch_el x11, 3f, 2f, 1f
    3:	mrs	x1, esr_el3
    	mrs	x2, elr_el3
    	mrs	x3, daif
    	mrs	x4, vbar_el3
    	mrs	x5, spsr_el3
    	sub	x6, sp, #(8*30)
    	mrs	x7, sctlr_el3
    	mrs	x8, scr_el3
    	mrs	x9, ttbr0_el3
    	b	0f
    2:	mrs	x1, esr_el2
    	mrs	x2, elr_el2
    	mrs	x3, daif
    	mrs	x4, vbar_el2
    	mrs	x5, spsr_el2
    	sub	x6, sp, #(8*30)
    	mrs	x7, sctlr_el2
    	mrs	x8, hcr_el2
    	mrs	x9, ttbr0_el2
    	b	0f
    
    1:	mrs	x1, esr_el1
    	mrs	x2, elr_el1
    	mrs	x3, daif
    	mrs	x4, vbar_el1
    	mrs	x5, spsr_el1
    	sub	x6, sp, #(8*30)
    	mrs	x7, sctlr_el1
    	mov	x8, #0
    	mrs	x9, ttbr0_el1
    0:
    	stp     x2, x0, [sp, #-16]!
    	stp	x3, x1, [sp, #-16]!
    	stp	x5, x4, [sp, #-16]!
    	stp	x7, x6, [sp, #-16]!
    	stp	x9, x8, [sp, #-16]!
    	mov	x0, sp
    	ret
    
    
    exception_exit:
    	add	sp, sp, #(8*8)
    	ldp	x2, x0, [sp],#16
    	switch_el x11, 3f, 2f, 1f
    3:	msr	elr_el3, x2
    	b	0f
    2:	msr	elr_el2, x2
    	b	0f
    1:	msr	elr_el1, x2
    0:
    	ldp	x1, x2, [sp],#16
    	ldp	x3, x4, [sp],#16
    	ldp	x5, x6, [sp],#16
    	ldp	x7, x8, [sp],#16
    	ldp	x9, x10, [sp],#16
    	ldp	x11, x12, [sp],#16
    	ldp	x13, x14, [sp],#16
    	ldp	x15, x16, [sp],#16
    	ldp	x17, x18, [sp],#16
    	ldp	x19, x20, [sp],#16
    	ldp	x21, x22, [sp],#16
    	ldp	x23, x24, [sp],#16
    	ldp	x25, x26, [sp],#16
    	ldp	x27, x28, [sp],#16
    	ldp	x29, x30, [sp],#16
    	eret
    ```
    
    > 注：  
    > 1.`switch_el`这一宏定义伪指令在`u-boot/arch/arm/include/asm/macro.h`定义；  
    > 2.`vbar_el3`等寄存器定义在文档`ARMv8-A_Architecture_Reference_Manual_(Issue_A.a).pdf`\[2\]中；  
    > 3.`XZR/WZR`(word zero rigiser)分别代表64/32位，`zero register`的作用就是0，写进去代表丢弃结果，拿出来是0；
    

中断向量的定义在文件`u-boot/arch/arm/cpu/armv8/exceptions.S`中。


这一部分功能就是根据当前的EL级别，配置中断向量、MMU、Endian、i/d Cache等，比较重要。

*   **2.配置ARM核心特定勘误表 \[unimportant\]**
    
    ```c
    /* Apply ARM core specific erratas */
    bl	apply_core_errata
    ```
    
    看样子是对ARM做一些勘误，实测没有用到，不重要。
    
*   **3.lowlevel\_init \[important\]**
    
    ```c
    	/* Processor specific initialization */
    	bl	lowlevel_init
    
    	……
    /*-----------------------------------------------------------------------*/
    
    WEAK(lowlevel_init)
    	mov	x29, lr			/* Save LR */
     	branch_if_slave x0, 1f
    	ldr	x0, =GICD_BASE
    	bl	gic_init_secure
    1:
     	ldr	x0, =GICR_BASE
    	bl	gic_init_secure_percpu
     
     	/*
    	 * Setting HCR_EL2.TGE AMO IMO FMO for exception rounting to EL2
    	 */
    	mrs	x0, CurrentEL		/* check currentEL */
    	cmp	x0, 0x8
    	b.ne	end			/* currentEL != EL2 */
    
    	mrs	x9, hcr_el2
    	orr	x9, x9, #(7 << 3)	/* HCR_EL2.AMO IMO FMO set */
    	orr	x9, x9, #(1 << 27)	/* HCR_EL2.TGE set */
    	msr	hcr_el2, x9
    
    end:
    	nop
    
    2:
    	mov	lr, x29			/* Restore LR */
    	ret
    ENDPROC(lowlevel_init)
    ```
    

> 注：  
> 1.`branch_if_slave`在`u-boot/arch/arm/include/asm/macro.h`定义；  
> 2.`gic_init_secure`和`gic_init_secure_percpu`这两个中断初始化的关键函数在`u-boot/arch/arm/lib/gic_64.S`定义；  

`lowlevel_init`的主要功能就是中断的初始化，后面写中断服务程序的使用会用到。

*   **4.跳转到\_main \[important\]**  
    到此`start.S`的工作就基本完成了，接下来就交给ARM公共的`_main`。

3.2 crt0\_64.S
------------------------------------------------

所在文件路径：[`u-boot/arch/arm/cpu/armv8/start.S`](https://github.com/radxa/u-boot/blob/stable-4.4-rockpi4/arch/arm/lib/crt0_64.S)  
`_main`在`crt0_64.S`里，`crt0`是`C-runtime Startup Code`的简称，意思就是运行C代码之前的准备工作，包括设置栈、重定位、清理BSS段等；

*   **1.设置栈 \[important\]**
    
    ```c
    
    ENTRY(_main)
    	.equ SCTLR_A_BIT,		(1 << 1)
    	.equ SCTLR_SA_BIT,		(1 << 3)
    	.equ SCTLR_I_BIT,		(1 << 12)
    
    /*
     * Set up initial C runtime environment and call board_init_f(0).
     */
    
    	ldr	x0, =(CONFIG_SYS_INIT_SP_ADDR)
    	bic	sp, x0, #0xf	/* 16-byte alignment for ABI compliance */
    	//mov	x0, sp
    	//bl	board_init_f_alloc_reserve
    	//mov	sp, x0
    	/* set up gd here, outside any C code */
    	//mov	x18, x0
    	//bl	board_init_f_init_reserve
    	//bl	board_init_f_init_serial
    
    	//mov	x0, #0
    	bl	main //bl board_init_f

这里栈需要16字节对齐，即要求地址为16的倍数，只需要二进制位最后四位为0(2的4次方)，与前面中断向量地址需要2K对齐，实现原理类似。  另外U-Boot在SP上面分配了一块GD(global data)，后面写裸机用不到。

*   **2.board\_init\_f \[important\]**
    
    ```plain
    mov	x0, #0          //将0作为参数传入board_init_f
    bl	board_init_f
    ```
    
    `board_init_f`所在文件路径：[`u-boot/common/board_f.c`](https://github.com/radxa/u-boot/blob/stable-4.4-rockpi4/common/board_f.c)。  
    `board_init_f`中调用`initcall_run_list(init_sequence_f)`，`init_sequence_f`是个数组，里面是将要进行初始化的函数列表，完成一些前期的初始化工作，比如board相关的early的初始化`board_early_init_f`、环境变量初始化`env_init`、串口初始化的`serial_init`、I2C初始化`init_func_i2c`、设备树相关准备工作`fdtdec_prepare_fdt`、打印CPU信息`print_cpuinfo`、SDRAM初始化`dram_init`、计算重定位信息`setup_reloc`等；
    
*   **3.重定位 \[important\]**
    
    ```c
    /*F:\Rock4Plus_Android10\u-boot\arch\arm\lib\relocate_64.S*/
    #define R_AARCH64_RELATIVE	1027
    
    /*
     * void relocate_code (addr_moni)
     *
     * This function relocates the monitor code.
     * x0 holds the destination address.
     */
    ENTRY(relocate_code)
    	stp	x29, x30, [sp, #-32]!	/* create a stack frame */
    	mov	x29, sp
    	str	x0, [sp, #16]
    	/*
    	 * Copy u-boot from flash to RAM
    	 */
    	adr	x1, __image_copy_start	/* x1 <- Run &__image_copy_start */
    	subs	x9, x0, x1		/* x8 <- Run to copy offset */
    	b.eq	relocate_done		/* skip relocation */
    	/*
    	 * Dont ldr x1, __image_copy_start here, since if the code is already
    	 * running at an address other than it was linked to, that instruction
    	 * will load the relocated value of __image_copy_start. To
    	 * correctly apply relocations, we need to know the linked value.
    	 *
    	 * Linked &__image_copy_start, which we know was at
    	 * CONFIG_SYS_TEXT_BASE, which is stored in _TEXT_BASE, as a non-
    	 * relocated value, since it isnt a symbol reference.
    	 */
    	ldr	x1, _TEXT_BASE		/* x1 <- Linked &__image_copy_start */
    	subs	x9, x0, x1		/* x9 <- Link to copy offset */
    
    	adr	x1, __image_copy_start	/* x1 <- Run &__image_copy_start */
    	adr	x2, __image_copy_end	/* x2 <- Run &__image_copy_end */
    copy_loop:
    	ldp	x10, x11, [x1], #16	/* copy from source address [x1] */
    	stp	x10, x11, [x0], #16	/* copy to   target address [x0] */
    	cmp	x1, x2			/* until source end address [x2] */
    	b.lo	copy_loop
    	str	x0, [sp, #24]
    
    	/*
    	 * Fix .rela.dyn relocations
    	 */
    	adr	x2, __rel_dyn_start	/* x2 <- Run &__rel_dyn_start */
    	adr	x3, __rel_dyn_end	/* x3 <- Run &__rel_dyn_end */
    fixloop:
    	ldp	x0, x1, [x2], #16	/* (x0,x1) <- (SRC location, fixup) */
    	ldr	x4, [x2], #8		/* x4 <- addend */
    	and	x1, x1, #0xffffffff
    	cmp	x1, #R_AARCH64_RELATIVE
    	bne	fixnext
    
    	/* relative fix: store addend plus offset at dest location */
    	add	x0, x0, x9
    	add	x4, x4, x9
    	str	x4, [x0]
    fixnext:
    	cmp	x2, x3
    	b.lo	fixloop
    
    relocate_done:
    	switch_el x1, 3f, 2f, 1f
    	bl	hang
    3:	mrs	x0, sctlr_el3
    	b	0f
    2:	mrs	x0, sctlr_el2
    	b	0f
    1:	mrs	x0, sctlr_el1
    0:	tbz	w0, #2, 5f	/* skip flushing cache if disabled */
    	tbz	w0, #12, 4f	/* skip invalidating i-cache if disabled */
    	ic	iallu		/* i-cache invalidate all */
    	isb	sy
    4:	ldp	x0, x1, [sp, #16]
    	bl	__asm_flush_dcache_range
    5:	ldp	x29, x30, [sp],#32
    	ret
    ENDPROC(relocate_code)
    
    ```
    
    先是更新了gd结构体，然后根据宏`CONFIG_SKIP_RELOCATE_UBOOT`决定是否要重定位。  
    这里是不需要重定位的，因为链接脚本[`u-boot.lds`](https://github.com/radxa/u-boot/blob/stable-4.4-rockpi4/arch/arm/cpu/armv8/u-boot.lds)里面的链接地址是`0x00000000`，而RK3399上电后，加头的`boot code`会自动将代码复制到DDR(0x00000000)，两者地址相同，不需要重定位。  
    重定位的代码在`u-boot/arch/arm/lib/relocate_64.S`里面。
    
*   **4.重新设置异常向量表 \[important\]**
    
    ```c
    /*
     * Set up final (full) environment
     */
    	bl	c_runtime_cpu_setup		/* still call old routine */
    ```
    
    如果发生了重定位，需要重新设置异常向量表。`c_runtime_cpu_setup`定义在`start.S`里面。
    
    ```c
    ENTRY(c_runtime_cpu_setup)
    	/* Relocate vBAR */
    	adr	x0, vectors
    	switch_el x1, 3f, 2f, 1f
    3:	msr	vbar_el3, x0
    	b	0f
    2:	msr	vbar_el2, x0
    	b	0f
    1:	msr	vbar_el1, x0
    0:
    
    	ret
    ENDPROC(c_runtime_cpu_setup)
    ```
    
*   **5.清理BSS段 \[important\]**  
    接下来就是清除BSS段，将未定义的全局变量设置为0。在以前使用Keil单片机编程时，未初始化的全局变量默认为0，那是因为集成开发环境为我们做了清理BSS段的操作。
    
    ```c
    /*
     * Clear BSS section
     */
    	ldr	x0, =__bss_start		/* this is auto-relocated! */
    	ldr	x1, =__bss_end			/* this is auto-relocated! */
    clear_loop:
    	str	xzr, [x0], #8
    	cmp	x0, x1
    	b.lo	clear_loop
    
    	/* call board_init_r(gd_t *id, ulong dest_addr) */
    	//mov	x0, x18				/* gd_t */
    	//ldr	x1, [x18, #GD_RELOCADDR]	/* dest_addr */
    	//b	board_init_r			/* PC relative jump */
    
    	/* NOTREACHED - board_init_r() does not return */
    
*   **6.board\_init\_r \[important\]**  
    接下来就是板子的后半部分的初始化：
    
    ```c
    	/* call board_init_r(gd_t *id, ulong dest_addr) */
    	//mov	x0, x18				/* gd_t */
    	//ldr	x1, [x18, #GD_RELOCADDR]	/* dest_addr */
    	//b	board_init_r			/* PC relative jump */
    
    	/* NOTREACHED - board_init_r() does not return */

`crt0_64.S`主要就是为C语言运行设置栈和进行了重定位，以及两个阶段的初始化:`board_init_f`(front)和`board_init_r`(rear)，最后进入主循环。

- `board_init_r`所在文件路径：[`u-boot/common/board_f.c`](https://github.com/radxa/u-boot/blob/stable-4.4-rockpi4/common/board_r.c)。
  与前面的`board_init_f`类似，`board_init_r`中调用`initcall_run_list(init_sequence_r)`，`init_sequence_r`是个数组，里面是将要进行初始化的函数列表，又是一系列的初始化操作。之前遇到的LCD初始化就是在这里。
  初始化数组列表最后一个成员是`run_main_loop`，将最终跳到主循环[`main_loop`](https://github.com/radxa/u-boot/blob/stable-4.4-rockpi4/common/main.c)。

`crt0_64.S`主要就是为C语言运行设置栈和进行了重定位，以及两个阶段的初始化:`board_init_f`(front)和`board_init_r`(rear)，最后进入主循环。

3.3 总结
--------------------------

U-Boot启动流程示意图：

![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/5.png)

4.中断
====================

4.1 分析
--------------------------

在U-Boot中找到如下几个文件：

> `u-boot/arch/arm/cpu/armv8/rk33xx/irqs.c`:包含中断的基本操作，如：初始化、注册、使能等；  
> `u-boot/arch/arm/cpu/armv8/rk33xx/irqs-gic.c`:包含非GPIO类型中断的使能、去能；  
> `u-boot/arch/arm/cpu/armv8/rk33xx/irqs-gpio.c`:包含GPIO类型中断的使能、去能、触发类型；  
> `u-boot/board/rockchip/rk33xx/demo.c`:包含一些测试代码，如：定时器中断测试、GPIO中断测试；

*   `irqs.c`里的函数:  
    首先是**`irq_init()`**里面包含gic中断初始化和gpio中断初始化，函数里注释`gic has been init in Start.S`和之前的猜测一样，在`start.S`里面已经gic初始化了；  
    然后是**`irq_install_handler()`**里面实现了中断的注册，即把对应中断号放在`g_irq_handler[]`数组里；  
    再是**`irq_handler_enable()`**，将对应的中断处理函数使能，具体实现的函数在`irqs-gic.c`和`irqs-gpio.c`里面。此外还有使能总中断`enable_interrupts()`；  
    最后就是**`do_irq()`**中断处理函数。
    
*   `irqs-gic.c`里的函数:  
    包含`gic_handler_enable()`和`gic_handler_disable()`，在前面`irq_handler_enable()`调用；
    
*   `irqs-gpio.c`里的函数:  
    包含`gic_handler_enable()`、`gpio_irq_enable`和`gpio_irq_set_type()`，在前面`irq_handler_enable()`调用；
    
*   `demo.c`里的函数:  
    包含定时器中断测试`board_gic_test()`和GPIO中断测试`board_gpio_irq_test()`；
    

因此，除了`start.S`里的初始化，还需移植`irq_install_handler()`、`irq_handler_enable`、`do_irq()`三个函数，此外还有定时器中断测试和GPIO测试函数。

4.2 启动和中断代码
-----------------------------------------

因为`start.S`里面包含了中断初始化代码，即`gic_init_secure`和`gic_init_secure_percpu`，移植的时候直接复制过来的，因此也把`start.S`贴出来。  
`start.S`是对U-boot的`start.S`进行了裁剪和修改，思路和前面U-Boot的流程差不多，几个重定位、绝对跳转、代码对齐的坑，都踩完了，下面的`start.S`有时间的话可以好好看下。  

```c
/*
 * (C) Copyright 2013
 * David Feng <fenghua@phytium.com.cn>
 * F:\Rock4Plus_Android10\u-boot\arch\arm\cpu\armv8\start.S
 * SPDX-License-Identifier:	GPL-2.0+
 */

//#include <asm-offsets.h>
#define GENERATED_GBL_DATA_SIZE 464 /* (sizeof(struct global_data) + 15) & ~15	// */
#define GENERATED_BD_INFO_SIZE 336 /* (sizeof(struct bd_info) + 15) & ~15	// */
#define GD_SIZE 456 /* sizeof(struct global_data)	// */
#define GD_BD 0 /* offsetof(struct global_data, bd)	// */
#define GD_MALLOC_BASE 280 /* offsetof(struct global_data, malloc_base)	// */
#define GD_RELOCADDR 96 /* offsetof(struct global_data, relocaddr)	// */
#define GD_RELOC_OFF 136 /* offsetof(struct global_data, reloc_off)	// */
#define GD_START_ADDR_SP 128 /* offsetof(struct global_data, start_addr_sp)	// */
#define PM_CTX_SIZE 136 /* sizeof(struct pm_ctx)	// */
#define PM_CTX_PHYS 408 /* offsetof(struct global_data, pm_ctx_phys)	// */
#define GD_NEW_GD 144 /* offsetof(struct global_data, new_gd)	// */


//#include <config.h>
/*F:\Rock4Plus_Android10\u-boot\include\configs\rk3399_common.h*/
#define CONFIG_SYS_TEXT_BASE		0x00200000
#define CONFIG_SYS_INIT_SP_ADDR		0x00400000

#define GICD_BASE			0xFEE00000
#define GICR_BASE			0xFEF00000



//#include <asm/macro.h>

/*F:\Rock4Plus_Android10\u-boot\arch\arm\include\asm\macro.h start*/

/*
 * These macros provide a convenient way to write 8, 16 and 32 bit data
 * to any address.
 * Registers r4 and r5 are used, any data in these registers are
 * overwritten by the macros.
 * The macros are valid for any ARM architecture, they do not implement
 * any memory barriers so caution is recommended when using these when the
 * caches are enabled or on a multi-core system.
 */

.macro	write32, addr, data
	ldr	r4, =\addr
	ldr	r5, =\data
	str	r5, [r4]
.endm

.macro	write16, addr, data
	ldr	r4, =\addr
	ldrh	r5, =\data
	strh	r5, [r4]
.endm

.macro	write8, addr, data
	ldr	r4, =\addr
	ldrb	r5, =\data
	strb	r5, [r4]
.endm

/*
 * This macro generates a loop that can be used for delays in the code.
 * Register r4 is used, any data in this register is overwritten by the
 * macro.
 * The macro is valid for any ARM architeture. The actual time spent in the
 * loop will vary from CPU to CPU though.
 */

.macro	wait_timer, time
	ldr	r4, =\time
1:
	nop
	subs	r4, r4, #1
	bcs	1b
.endm

/*
 * Register aliases.
 */
lr	.req	x30

/*
 * Branch according to exception level
 */
.macro	switch_el, xreg, el3_label, el2_label, el1_label
	mrs	\xreg, CurrentEL
	cmp	\xreg, 0xc
	b.eq	\el3_label
	cmp	\xreg, 0x8
	b.eq	\el2_label
	cmp	\xreg, 0x4
	b.eq	\el1_label
.endm

/*
 * Branch if current processor is a Cortex-A35 core.
 */
.macro	branch_if_a35_core, xreg, a35_label
	mrs	\xreg, midr_el1
	lsr	\xreg, \xreg, #4
	and	\xreg, \xreg, #0x00000FFF
	cmp	\xreg, #0xD04		/* Cortex-A35 MPCore processor. */
	b.eq	\a35_label
.endm

/*
 * Branch if current processor is a Cortex-A57 core.
 */
.macro	branch_if_a57_core, xreg, a57_label
	mrs	\xreg, midr_el1
	lsr	\xreg, \xreg, #4
	and	\xreg, \xreg, #0x00000FFF
	cmp	\xreg, #0xD07		/* Cortex-A57 MPCore processor. */
	b.eq	\a57_label
.endm

/*
 * Branch if current processor is a Cortex-A53 core.
 */
.macro	branch_if_a53_core, xreg, a53_label
	mrs	\xreg, midr_el1
	lsr	\xreg, \xreg, #4
	and	\xreg, \xreg, #0x00000FFF
	cmp	\xreg, #0xD03		/* Cortex-A53 MPCore processor. */
	b.eq	\a53_label
.endm

/*
 * Branch if current processor is a slave,
 * choose processor with all zero affinity value as the master.
 */
.macro	branch_if_slave, xreg, slave_label
.endm

/*
 * Branch if current processor is a master,
 * choose processor with all zero affinity value as the master.
 */
.macro	branch_if_master, xreg1, xreg2, master_label
	b 	\master_label
.endm

/*
 * Switch from EL3 to EL2 for ARMv8
 * @ep:     kernel entry point
 * @flag:   The execution state flag for lower exception
 *          level, ES_TO_AARCH64 or ES_TO_AARCH32
 * @tmp:    temporary register
 *
 * For loading 32-bit OS, x1 is machine nr and x2 is ftaddr.
 * For loading 64-bit OS, x0 is physical address to the FDT blob.
 * They will be passed to the guest.
 */
.macro armv8_switch_to_el2_m, ep, flag, tmp
	msr	cptr_el3, xzr		/* Disable coprocessor traps to EL3 */
	mov	\tmp, #CPTR_EL2_RES1
	msr	cptr_el2, \tmp		/* Disable coprocessor traps to EL2 */

	/* Initialize Generic Timers */
	msr	cntvoff_el2, xzr

	/* Initialize SCTLR_EL2
	 *
	 * setting RES1 bits (29,28,23,22,18,16,11,5,4) to 1
	 * and RES0 bits (31,30,27,26,24,21,20,17,15-13,10-6) +
	 * EE,WXN,I,SA,C,A,M to 0
	 */
	ldr	\tmp, =(SCTLR_EL2_RES1 | SCTLR_EL2_EE_LE |\
			SCTLR_EL2_WXN_DIS | SCTLR_EL2_ICACHE_DIS |\
			SCTLR_EL2_SA_DIS | SCTLR_EL2_DCACHE_DIS |\
			SCTLR_EL2_ALIGN_DIS | SCTLR_EL2_MMU_DIS)
	msr	sctlr_el2, \tmp

	mov	\tmp, sp
	msr	sp_el2, \tmp		/* Migrate SP */
	mrs	\tmp, vbar_el3
	msr	vbar_el2, \tmp		/* Migrate VBAR */

	/* Check switch to AArch64 EL2 or AArch32 Hypervisor mode */
	cmp	\flag, #ES_TO_AARCH32
	b.eq	1f

	/*
	 * The next lower exception level is AArch64, 64bit EL2 | HCE |
	 * RES1 (Bits[5:4]) | Non-secure EL0/EL1.
	 * and the SMD depends on requirements.
	 */
	ldr	\tmp, =(SCR_EL3_RW_AARCH64 | SCR_EL3_HCE_EN |\
			SCR_EL3_SMD_DIS | SCR_EL3_RES1 |\
			SCR_EL3_NS_EN)
	msr	scr_el3, \tmp

	/* Return to the EL2_SP2 mode from EL3 */
	ldr	\tmp, =(SPSR_EL_DEBUG_MASK | SPSR_EL_SERR_MASK |\
			SPSR_EL_IRQ_MASK | SPSR_EL_FIQ_MASK |\
			SPSR_EL_M_AARCH64 | SPSR_EL_M_EL2H)
	msr	spsr_el3, \tmp
	msr	elr_el3, \ep
	eret

1:
	/*
	 * The next lower exception level is AArch32, 32bit EL2 | HCE |
	 * SMD | RES1 (Bits[5:4]) | Non-secure EL0/EL1.
	 */
	ldr	\tmp, =(SCR_EL3_RW_AARCH32 | SCR_EL3_HCE_EN |\
			SCR_EL3_SMD_DIS | SCR_EL3_RES1 |\
			SCR_EL3_NS_EN)
	msr	scr_el3, \tmp

	/* Return to AArch32 Hypervisor mode */
	ldr     \tmp, =(SPSR_EL_END_LE | SPSR_EL_ASYN_MASK |\
			SPSR_EL_IRQ_MASK | SPSR_EL_FIQ_MASK |\
			SPSR_EL_T_A32 | SPSR_EL_M_AARCH32 |\
			SPSR_EL_M_HYP)
	msr	spsr_el3, \tmp
	msr     elr_el3, \ep
	eret
.endm

/*
 * Switch from EL2 to EL1 for ARMv8
 * @ep:     kernel entry point
 * @flag:   The execution state flag for lower exception
 *          level, ES_TO_AARCH64 or ES_TO_AARCH32
 * @tmp:    temporary register
 *
 * For loading 32-bit OS, x1 is machine nr and x2 is ftaddr.
 * For loading 64-bit OS, x0 is physical address to the FDT blob.
 * They will be passed to the guest.
 */
.macro armv8_switch_to_el1_m, ep, flag, tmp
	/* Initialize Generic Timers */
	mrs	\tmp, cnthctl_el2
	/* Enable EL1 access to timers */
	orr	\tmp, \tmp, #(CNTHCTL_EL2_EL1PCEN_EN |\
		CNTHCTL_EL2_EL1PCTEN_EN)
	msr	cnthctl_el2, \tmp
	msr	cntvoff_el2, xzr

	/* Initilize MPID/MPIDR registers */
	mrs	\tmp, midr_el1
	msr	vpidr_el2, \tmp
	mrs	\tmp, mpidr_el1
	msr	vmpidr_el2, \tmp

	/* Disable coprocessor traps */
	mov	\tmp, #CPTR_EL2_RES1
	msr	cptr_el2, \tmp		/* Disable coprocessor traps to EL2 */
	msr	hstr_el2, xzr		/* Disable coprocessor traps to EL2 */
	mov	\tmp, #CPACR_EL1_FPEN_EN
	msr	cpacr_el1, \tmp		/* Enable FP/SIMD at EL1 */

	/* SCTLR_EL1 initialization
	 *
	 * setting RES1 bits (29,28,23,22,20,11) to 1
	 * and RES0 bits (31,30,27,21,17,13,10,6) +
	 * UCI,EE,EOE,WXN,nTWE,nTWI,UCT,DZE,I,UMA,SED,ITD,
	 * CP15BEN,SA0,SA,C,A,M to 0
	 */
	ldr	\tmp, =(SCTLR_EL1_RES1 | SCTLR_EL1_UCI_DIS |\
			SCTLR_EL1_EE_LE | SCTLR_EL1_WXN_DIS |\
			SCTLR_EL1_NTWE_DIS | SCTLR_EL1_NTWI_DIS |\
			SCTLR_EL1_UCT_DIS | SCTLR_EL1_DZE_DIS |\
			SCTLR_EL1_ICACHE_DIS | SCTLR_EL1_UMA_DIS |\
			SCTLR_EL1_SED_EN | SCTLR_EL1_ITD_EN |\
			SCTLR_EL1_CP15BEN_DIS | SCTLR_EL1_SA0_DIS |\
			SCTLR_EL1_SA_DIS | SCTLR_EL1_DCACHE_DIS |\
			SCTLR_EL1_ALIGN_DIS | SCTLR_EL1_MMU_DIS)
	msr	sctlr_el1, \tmp

	mov	\tmp, sp
	msr	sp_el1, \tmp		/* Migrate SP */
	mrs	\tmp, vbar_el2
	msr	vbar_el1, \tmp		/* Migrate VBAR */

	/* Check switch to AArch64 EL1 or AArch32 Supervisor mode */
	cmp	\flag, #ES_TO_AARCH32
	b.eq	1f

	/* Initialize HCR_EL2 */
	ldr	\tmp, =(HCR_EL2_RW_AARCH64 | HCR_EL2_HCD_DIS)
	msr	hcr_el2, \tmp

	/* Return to the EL1_SP1 mode from EL2 */
	ldr	\tmp, =(SPSR_EL_DEBUG_MASK | SPSR_EL_SERR_MASK |\
			SPSR_EL_IRQ_MASK | SPSR_EL_FIQ_MASK |\
			SPSR_EL_M_AARCH64 | SPSR_EL_M_EL1H)
	msr	spsr_el2, \tmp
	msr     elr_el2, \ep
	eret

1:
	/* Initialize HCR_EL2 */
	ldr	\tmp, =(HCR_EL2_RW_AARCH32 | HCR_EL2_HCD_DIS)
	msr	hcr_el2, \tmp

	/* Return to AArch32 Supervisor mode from EL2 */
	ldr	\tmp, =(SPSR_EL_END_LE | SPSR_EL_ASYN_MASK |\
			SPSR_EL_IRQ_MASK | SPSR_EL_FIQ_MASK |\
			SPSR_EL_T_A32 | SPSR_EL_M_AARCH32 |\
			SPSR_EL_M_SVC)
	msr     spsr_el2, \tmp
	msr     elr_el2, \ep
	eret
.endm

.macro gic_wait_for_interrupt_m xreg1
0 :	wfi
	mrs     \xreg1, ICC_IAR1_EL1
	msr     ICC_EOIR1_EL1, \xreg1
	cbnz    \xreg1, 0b
.endm


/*F:\Rock4Plus_Android10\u-boot\arch\arm\include\asm\macro.h end*/

//#include <asm/armv8/mmu.h>

/* #include <linux/linkage.h> start*/

#define STT_FUNC 2 /* elf.h */

#define ASM_NL ;

#define SYMBOL_NAME(X)		X
#define SYMBOL_NAME_LABEL(X)	X:
#define __ALIGN .align 0
#define __ALIGN_STR ".align 0"
#define ALIGN			__ALIGN
#define ALIGN_STR		__ALIGN_STR

#define LENTRY(name) \
	ALIGN ASM_NL \
	SYMBOL_NAME_LABEL(name)

#define ENTRY(name) \
	.globl SYMBOL_NAME(name) ASM_NL \
	LENTRY(name)

#define WEAK(name) \
	.weak SYMBOL_NAME(name) ASM_NL \
	LENTRY(name)

#define END(name) \
	.size name, .-name

#define ENDPROC(name) \
	.type name STT_FUNC ASM_NL \
	END(name)

/* #include <linux/linkage.h> end*/


/*************************************************************************
 *
 * Startup Code (reset vector)
 *
 *************************************************************************/

.globl	_start
_start:
	b	reset
	.align 3

.globl	_TEXT_BASE
_TEXT_BASE:
	.quad	CONFIG_SYS_TEXT_BASE

/*
 * These are defined in the linker script.
 */
.globl	_end_ofs
_end_ofs:
	.quad	_end - _start

.globl	_bss_start_ofs
_bss_start_ofs:
	.quad	__bss_start - _start

.globl	_bss_end_ofs
_bss_end_ofs:
	.quad	__bss_end - _start

reset:
	/* Allow the board to save important registers */
	b	save_boot_params
.globl	save_boot_params_ret
save_boot_params_ret:
	/*
	 * Could be EL3/EL2/EL1, Initial State:
	 * Little Endian, MMU Disabled, i/dCache Disabled
	 */
	adr	x0, vectors
	switch_el x1, 3f, 2f, 1f
3:	msr	vbar_el3, x0
	mrs	x0, scr_el3
	orr	x0, x0, #0xf			/* SCR_EL3.NS|IRQ|FIQ|EA */
	msr	scr_el3, x0
	msr	cptr_el3, xzr			/* Enable FP/SIMD */
	/* COUNTER_FREQUENCY 24000000 \u-boot\include\configs\rockchip-common.h */
	ldr	x0, =24000000
	msr	cntfrq_el0, x0			/* Initialize CNTFRQ */
	b	0f
2:	msr	vbar_el2, x0
	mov	x0, #0x33ff
	msr	cptr_el2, x0			/* Enable FP/SIMD */
	b	0f
1:	msr	vbar_el1, x0
	mov	x0, #3 << 20
	msr	cpacr_el1, x0			/* Enable FP/SIMD */
0:

	/*
	 * Enable SMPEN bit for coherency.
	 * This register is not architectural but at the moment
	 * this bit should be set for A53/A57/A72.
	 */
	switch_el x1, 3f, 1f, 1f
3:
	mrs     x0, S3_1_c15_c2_1               /* cpuectlr_el1 */
	orr     x0, x0, #0x40
	msr     S3_1_c15_c2_1, x0
1:

	/* Apply ARM core specific erratas */
	bl	apply_core_errata

	/*
	 * Cache/BPB/TLB Invalidate
	 * i-cache is invalidated before enabled in icache_enable()
	 * tlb is invalidated before mmu is enabled in dcache_enable()
	 * d-cache is invalidated before enabled in dcache_enable()
	 */

	/* Processor specific initialization */
	bl	lowlevel_init 
master_cpu:
	bl	_main
 
/*-----------------------------------------------------------------------*/

WEAK(apply_core_errata)

	mov	x29, lr			/* Save LR */
	/* For now, we support Cortex-A57 specific errata only */

	/* Check if we are running on a Cortex-A57 core */
	branch_if_a57_core x0, apply_a57_core_errata
0:
	mov	lr, x29			/* Restore LR */
	ret

apply_a57_core_errata:
	b 0b
ENDPROC(apply_core_errata)

/*-----------------------------------------------------------------------*/

WEAK(lowlevel_init)
	mov	x29, lr			/* Save LR */
 	branch_if_slave x0, 1f
	ldr	x0, =GICD_BASE
	bl	gic_init_secure
1:
 	ldr	x0, =GICR_BASE
	bl	gic_init_secure_percpu
 
 	/*
	 * Setting HCR_EL2.TGE AMO IMO FMO for exception rounting to EL2
	 */
	mrs	x0, CurrentEL		/* check currentEL */
	cmp	x0, 0x8
	b.ne	end			/* currentEL != EL2 */

	mrs	x9, hcr_el2
	orr	x9, x9, #(7 << 3)	/* HCR_EL2.AMO IMO FMO set */
	orr	x9, x9, #(1 << 27)	/* HCR_EL2.TGE set */
	msr	hcr_el2, x9

end:
	nop

2:
	mov	lr, x29			/* Restore LR */
	ret
ENDPROC(lowlevel_init)

WEAK(smp_kick_all_cpus)
	/* Kick secondary cpus up by SGI 0 interrupt */
 	ldr	x0, =GICD_BASE
	b	gic_kick_secondary_cpus
 	ret
ENDPROC(smp_kick_all_cpus)

/*-----------------------------------------------------------------------*/

ENTRY(c_runtime_cpu_setup)
	/* Relocate vBAR */
	adr	x0, vectors
	switch_el x1, 3f, 2f, 1f
3:	msr	vbar_el3, x0
	b	0f
2:	msr	vbar_el2, x0
	b	0f
1:	msr	vbar_el1, x0
0:

	ret
ENDPROC(c_runtime_cpu_setup)

WEAK(save_boot_params)
	b	save_boot_params_ret	/* back to my caller */
ENDPROC(save_boot_params) 

/*F:\Rock4Plus_Android10\u-boot\arch\arm\lib\gic_64.S*/
/*F:\Rock4Plus_Android10\u-boot\arch\arm\include\asm\gic.h*/
/* Register offsets for the ARM generic interrupt controller (GIC) */

#define GIC_DIST_OFFSET		0x1000
#define GIC_CPU_OFFSET_A9	0x0100
#define GIC_CPU_OFFSET_A15	0x2000

/* Distributor Registers */
#define GICD_CTLR		0x0000
#define GICD_TYPER		0x0004
#define GICD_IIDR		0x0008
#define GICD_STATUSR		0x0010
#define GICD_SETSPI_NSR		0x0040
#define GICD_CLRSPI_NSR		0x0048
#define GICD_SETSPI_SR		0x0050
#define GICD_CLRSPI_SR		0x0058
#define GICD_SEIR		0x0068
#define GICD_IGROUPRn		0x0080
#define GICD_ISENABLERn		0x0100
#define GICD_ICENABLERn		0x0180
#define GICD_ISPENDRn		0x0200
#define GICD_ICPENDRn		0x0280
#define GICD_ISACTIVERn		0x0300
#define GICD_ICACTIVERn		0x0380
#define GICD_IPRIORITYRn	0x0400
#define GICD_ITARGETSRn		0x0800
#define GICD_ICFGR		0x0c00
#define GICD_IGROUPMODRn	0x0d00
#define GICD_NSACRn		0x0e00
#define GICD_SGIR		0x0f00
#define GICD_CPENDSGIRn		0x0f10
#define GICD_SPENDSGIRn		0x0f20
#define GICD_IROUTERn		0x6000

/* Cpu Interface Memory Mapped Registers */
#define GICC_CTLR		0x0000
#define GICC_PMR		0x0004
#define GICC_BPR		0x0008
#define GICC_IAR		0x000C
#define GICC_EOIR		0x0010
#define GICC_RPR		0x0014
#define GICC_HPPIR		0x0018
#define GICC_ABPR		0x001c
#define GICC_AIAR		0x0020
#define GICC_AEOIR		0x0024
#define GICC_AHPPIR		0x0028
#define GICC_APRn		0x00d0
#define GICC_NSAPRn		0x00e0
#define GICC_IIDR		0x00fc
#define GICC_DIR		0x1000

/* ReDistributor Registers for Control and Physical LPIs */
#define GICR_CTLR		0x0000
#define GICR_IIDR		0x0004
#define GICR_TYPER		0x0008
#define GICR_STATUSR		0x0010
#define GICR_WAKER		0x0014
#define GICR_SETLPIR		0x0040
#define GICR_CLRLPIR		0x0048
#define GICR_SEIR		0x0068
#define GICR_PROPBASER		0x0070
#define GICR_PENDBASER		0x0078
#define GICR_INVLPIR		0x00a0
#define GICR_INVALLR		0x00b0
#define GICR_SYNCR		0x00c0
#define GICR_MOVLPIR		0x0100
#define GICR_MOVALLR		0x0110

/* ReDistributor Registers for SGIs and PPIs */
#define GICR_IGROUPRn		0x0080
#define GICR_ISENABLERn		0x0100
#define GICR_ICENABLERn		0x0180
#define GICR_ISPENDRn		0x0200
#define GICR_ICPENDRn		0x0280
#define GICR_ISACTIVERn		0x0300
#define GICR_ICACTIVERn		0x0380
#define GICR_IPRIORITYRn	0x0400
#define GICR_ICFGR0		0x0c00
#define GICR_ICFGR1		0x0c04
#define GICR_IGROUPMODRn	0x0d00
#define GICR_NSACRn		0x0e00

/* Cpu Interface System Registers */
#define ICC_IAR0_EL1		S3_0_C12_C8_0
#define ICC_IAR1_EL1		S3_0_C12_C12_0
#define ICC_EOIR0_EL1		S3_0_C12_C8_1
#define ICC_EOIR1_EL1		S3_0_C12_C12_1
#define ICC_HPPIR0_EL1		S3_0_C12_C8_2
#define ICC_HPPIR1_EL1		S3_0_C12_C12_2
#define ICC_BPR0_EL1		S3_0_C12_C8_3
#define ICC_BPR1_EL1		S3_0_C12_C12_3
#define ICC_DIR_EL1		S3_0_C12_C11_1
#define ICC_PMR_EL1		S3_0_C4_C6_0
#define ICC_RPR_EL1		S3_0_C12_C11_3
#define ICC_CTLR_EL1		S3_0_C12_C12_4
#define ICC_CTLR_EL3		S3_6_C12_C12_4
#define ICC_SRE_EL1		S3_0_C12_C12_5
#define ICC_SRE_EL2		S3_4_C12_C9_5
#define ICC_SRE_EL3		S3_6_C12_C12_5
#define ICC_IGRPEN0_EL1		S3_0_C12_C12_6
#define ICC_IGRPEN1_EL1		S3_0_C12_C12_7
#define ICC_IGRPEN1_EL3		S3_6_C12_C12_7
#define ICC_SEIEN_EL1		S3_0_C12_C13_0
#define ICC_SGI0R_EL1		S3_0_C12_C11_7
#define ICC_SGI1R_EL1		S3_0_C12_C11_5
#define ICC_ASGI1R_EL1		S3_0_C12_C11_6


/*************************************************************************
 *
 * void gic_init_secure(DistributorBase);
 *
 * Initialize secure copy of GIC at EL3.
 *
 *************************************************************************/
ENTRY(gic_init_secure)
	/*
	 * Initialize Distributor
	 * x0: Distributor Base
	 */
 	mov	w9, #0x37		/* EnableGrp0 | EnableGrp1NS */
					/* EnableGrp1S | ARE_S | ARE_NS */
	str	w9, [x0, GICD_CTLR]	/* Secure GICD_CTLR */
	ldr	w9, [x0, GICD_TYPER]
	and	w10, w9, #0x1f		/* ITLinesNumber */
	cbz	w10, 1f			/* No SPIs */
	add	x11, x0, (GICD_IGROUPRn + 4)
	add	x12, x0, (GICD_IGROUPMODRn + 4)
	mov	w9, #~0
0:	str	w9, [x11], #0x4
	str	wzr, [x12], #0x4	/* Config SPIs as Group1NS */
	sub	w10, w10, #0x1
	cbnz	w10, 0b
 
1:
	ret
ENDPROC(gic_init_secure)


/************************************************************************* 
 * For Gicv3:
 * void gic_init_secure_percpu(ReDistributorBase);
 *
 * Initialize secure copy of GIC at EL3.
 *
 *************************************************************************/
ENTRY(gic_init_secure_percpu)
 	/*
	 * Initialize ReDistributor
	 * x0: ReDistributor Base
	 */
	mrs	x10, mpidr_el1
	lsr	x9, x10, #32
	bfi	x10, x9, #24, #8	/* w10 is aff3:aff2:aff1:aff0 */
	mov	x9, x0
1:	ldr	x11, [x9, GICR_TYPER]
	lsr	x11, x11, #32		/* w11 is aff3:aff2:aff1:aff0 */
	cmp	w10, w11
	b.eq	2f
	add	x9, x9, #(2 << 16)
	b	1b

	/* x9: ReDistributor Base Address of Current CPU */
2:	mov	w10, #~0x2
	ldr	w11, [x9, GICR_WAKER]
	and	w11, w11, w10		/* Clear ProcessorSleep */
	str	w11, [x9, GICR_WAKER]
	dsb	st
	isb
3:	ldr	w10, [x9, GICR_WAKER]
	tbnz	w10, #2, 3b		/* Wait Children be Alive */

	add	x10, x9, #(1 << 16)	/* SGI_Base */
	mov	w11, #~0
	str	w11, [x10, GICR_IGROUPRn]
	str	wzr, [x10, GICR_IGROUPMODRn]	/* SGIs|PPIs Group1NS */
	mov	w11, #0x1		/* Enable SGI 0 */
	str	w11, [x10, GICR_ISENABLERn]

 	/* Rockchip: check elx */
	switch_el x0, el3_sre, el2_sre, el1_sre

	/* Initialize Cpu Interface */
el3_sre:
	mrs	x10, ICC_SRE_EL3
	orr	x10, x10, #0xf		/* SRE & Disable IRQ/FIQ Bypass & */
					/* Allow EL2 access to ICC_SRE_EL2 */
	msr	ICC_SRE_EL3, x10
	isb

el2_sre:
	mrs	x10, ICC_SRE_EL2
	orr	x10, x10, #0xf		/* SRE & Disable IRQ/FIQ Bypass & */
					/* Allow EL1 access to ICC_SRE_EL1 */
	msr	ICC_SRE_EL2, x10
	isb

el1_sre:
	mrs	x0, CurrentEL		/* check currentEL */
	cmp	x0, 0xC
	b.ne	el1_ctlr		/* currentEL != EL3 */

el3_ctlr:
	mov	x10, #0x3		/* EnableGrp1NS | EnableGrp1S */
	msr	ICC_IGRPEN1_EL3, x10
	isb

	msr	ICC_CTLR_EL3, xzr
	isb

el1_ctlr:
	mov	x10, #0x3		/* EnableGrp1NS | EnableGrp1S */
	msr	ICC_IGRPEN1_EL1, x10
	isb

	msr	ICC_CTLR_EL1, xzr	/* NonSecure ICC_CTLR_EL1 */
	isb

	mov	x10, #0xf0		/* Non-Secure access to ICC_PMR_EL1 */
	msr	ICC_PMR_EL1, x10
	isb
 
	ret
ENDPROC(gic_init_secure_percpu)


/************************************************************************* 
 * For Gicv3:
 * void gic_kick_secondary_cpus(void);
 *
 *************************************************************************/
ENTRY(gic_kick_secondary_cpus)
 	mov	x9, #(1 << 40)
	msr	ICC_ASGI1R_EL1, x9
	isb
	ret
ENDPROC(gic_kick_secondary_cpus)

 
/*F:\Rock4Plus_Android10\u-boot\arch\arm\cpu\armv8\exceptions.S*/

/*
 * Exception vectors.
 */
	.align	11
	.globl	vectors
vectors:
	.align	7		/* Current EL Synchronous Thread */
	stp	x29, x30, [sp, #-16]!
	bl	_exception_entry
	/*bl	do_bad_sync*/
	b	exception_exit

	.align	7		/* Current EL IRQ Thread */
	stp	x29, x30, [sp, #-16]!
	bl	_exception_entry
	/*bl	do_bad_irq*/
	b	exception_exit

	.align	7		/* Current EL FIQ Thread */
	stp	x29, x30, [sp, #-16]!
	bl	_exception_entry
	/*bl	do_bad_fiq*/
	b	exception_exit

	.align	7		/* Current EL Error Thread */
	stp	x29, x30, [sp, #-16]!
	bl	_exception_entry
	/*bl	do_bad_error*/
	b	exception_exit

	.align	7		 /* Current EL Synchronous Handler */
	stp	x29, x30, [sp, #-16]!
	bl	_exception_entry
	/*bl	do_sync*/
	b	exception_exit

	.align	7		 /* Current EL IRQ Handler */
	stp	x29, x30, [sp, #-16]!
	bl	_exception_entry
	//bl	do_irq
	//bl	board_init_f_init_reserve
	b	exception_exit

	.align	7		 /* Current EL FIQ Handler */
	stp	x29, x30, [sp, #-16]!
	bl	_exception_entry
	/*bl	do_fiq*/
	b	exception_exit

	.align	7		 /* Current EL Error Handler */
	stp	x29, x30, [sp, #-16]!
	bl	_exception_entry
	/*bl	do_error*/
	b	exception_exit

/*
 * Enter Exception.
 * This will save the processor state that is ELR/X0~X30
 * to the stack frame.
 */
_exception_entry:
	stp	x27, x28, [sp, #-16]!
	stp	x25, x26, [sp, #-16]!
	stp	x23, x24, [sp, #-16]!
	stp	x21, x22, [sp, #-16]!
	stp	x19, x20, [sp, #-16]!
	stp	x17, x18, [sp, #-16]!
	stp	x15, x16, [sp, #-16]!
	stp	x13, x14, [sp, #-16]!
	stp	x11, x12, [sp, #-16]!
	stp	x9, x10, [sp, #-16]!
	stp	x7, x8, [sp, #-16]!
	stp	x5, x6, [sp, #-16]!
	stp	x3, x4, [sp, #-16]!
	stp	x1, x2, [sp, #-16]!

	/* Could be running at EL3/EL2/EL1 */
	switch_el x11, 3f, 2f, 1f
3:	mrs	x1, esr_el3
	mrs	x2, elr_el3
	mrs	x3, daif
	mrs	x4, vbar_el3
	mrs	x5, spsr_el3
	sub	x6, sp, #(8*30)
	mrs	x7, sctlr_el3
	mrs	x8, scr_el3
	mrs	x9, ttbr0_el3
	b	0f
2:	mrs	x1, esr_el2
	mrs	x2, elr_el2
	mrs	x3, daif
	mrs	x4, vbar_el2
	mrs	x5, spsr_el2
	sub	x6, sp, #(8*30)
	mrs	x7, sctlr_el2
	mrs	x8, hcr_el2
	mrs	x9, ttbr0_el2
	b	0f

1:	mrs	x1, esr_el1
	mrs	x2, elr_el1
	mrs	x3, daif
	mrs	x4, vbar_el1
	mrs	x5, spsr_el1
	sub	x6, sp, #(8*30)
	mrs	x7, sctlr_el1
	mov	x8, #0
	mrs	x9, ttbr0_el1
0:
	stp     x2, x0, [sp, #-16]!
	stp	x3, x1, [sp, #-16]!
	stp	x5, x4, [sp, #-16]!
	stp	x7, x6, [sp, #-16]!
	stp	x9, x8, [sp, #-16]!
	mov	x0, sp
	ret


exception_exit:
	add	sp, sp, #(8*8)
	ldp	x2, x0, [sp],#16
	switch_el x11, 3f, 2f, 1f
3:	msr	elr_el3, x2
	b	0f
2:	msr	elr_el2, x2
	b	0f
1:	msr	elr_el1, x2
0:
	ldp	x1, x2, [sp],#16
	ldp	x3, x4, [sp],#16
	ldp	x5, x6, [sp],#16
	ldp	x7, x8, [sp],#16
	ldp	x9, x10, [sp],#16
	ldp	x11, x12, [sp],#16
	ldp	x13, x14, [sp],#16
	ldp	x15, x16, [sp],#16
	ldp	x17, x18, [sp],#16
	ldp	x19, x20, [sp],#16
	ldp	x21, x22, [sp],#16
	ldp	x23, x24, [sp],#16
	ldp	x25, x26, [sp],#16
	ldp	x27, x28, [sp],#16
	ldp	x29, x30, [sp],#16
	eret

/*F:\Rock4Plus_Android10\u-boot\arch\arm\lib\crt0_64.S*/

/*F:\Rock4Plus_Android10\u-boot\include\generated\generic-asm-offsets.h*/


ENTRY(_main)
	.equ SCTLR_A_BIT,		(1 << 1)
	.equ SCTLR_SA_BIT,		(1 << 3)
	.equ SCTLR_I_BIT,		(1 << 12)

/*
 * Set up initial C runtime environment and call board_init_f(0).
 */

	ldr	x0, =(CONFIG_SYS_INIT_SP_ADDR)
	bic	sp, x0, #0xf	/* 16-byte alignment for ABI compliance */
	//mov	x0, sp
	//bl	board_init_f_alloc_reserve
	//mov	sp, x0
	/* set up gd here, outside any C code */
	//mov	x18, x0
	//bl	board_init_f_init_reserve
	//bl	board_init_f_init_serial

	//mov	x0, #0
	bl	main

/*
 * Set up intermediate environment (new sp and gd) and call
 * relocate_code(addr_moni). Trick here is that we.ll return
 * here but relocated.
 */
	ldr	x0, [x18, #GD_START_ADDR_SP]	/* x0 <- gd->start_addr_sp */
	bic	sp, x0, #0xf	/* 16-byte alignment for ABI compliance */
	ldr	x18, [x18, #GD_NEW_GD]		/* x18 <- gd->new_gd */

	adr	lr, relocation_return

	/* Add in link-vs-relocation offset */
	ldr	x9, [x18, #GD_RELOC_OFF]	/* x9 <- gd->reloc_off */
	add	lr, lr, x9	/* new return address after relocation */
	ldr	x0, [x18, #GD_RELOCADDR]	/* x0 <- gd->relocaddr */
	b	relocate_code

relocation_return:

/*
 * Set up final (full) environment
 */
	bl	c_runtime_cpu_setup		/* still call old routine */


/*
 * Clear BSS section
 */
	ldr	x0, =__bss_start		/* this is auto-relocated! */
	ldr	x1, =__bss_end			/* this is auto-relocated! */
clear_loop:
	str	xzr, [x0], #8
	cmp	x0, x1
	b.lo	clear_loop

	/* call board_init_r(gd_t *id, ulong dest_addr) */
	//mov	x0, x18				/* gd_t */
	//ldr	x1, [x18, #GD_RELOCADDR]	/* dest_addr */
	//b	board_init_r			/* PC relative jump */

	/* NOTREACHED - board_init_r() does not return */

ENDPROC(_main)

/*F:\Rock4Plus_Android10\u-boot\arch\arm\lib\relocate_64.S*/
#define R_AARCH64_RELATIVE	1027

/*
 * void relocate_code (addr_moni)
 *
 * This function relocates the monitor code.
 * x0 holds the destination address.
 */
ENTRY(relocate_code)
	stp	x29, x30, [sp, #-32]!	/* create a stack frame */
	mov	x29, sp
	str	x0, [sp, #16]
	/*
	 * Copy u-boot from flash to RAM
	 */
	adr	x1, __image_copy_start	/* x1 <- Run &__image_copy_start */
	subs	x9, x0, x1		/* x8 <- Run to copy offset */
	b.eq	relocate_done		/* skip relocation */
	/*
	 * Dont ldr x1, __image_copy_start here, since if the code is already
	 * running at an address other than it was linked to, that instruction
	 * will load the relocated value of __image_copy_start. To
	 * correctly apply relocations, we need to know the linked value.
	 *
	 * Linked &__image_copy_start, which we know was at
	 * CONFIG_SYS_TEXT_BASE, which is stored in _TEXT_BASE, as a non-
	 * relocated value, since it isnt a symbol reference.
	 */
	ldr	x1, _TEXT_BASE		/* x1 <- Linked &__image_copy_start */
	subs	x9, x0, x1		/* x9 <- Link to copy offset */

	adr	x1, __image_copy_start	/* x1 <- Run &__image_copy_start */
	adr	x2, __image_copy_end	/* x2 <- Run &__image_copy_end */
copy_loop:
	ldp	x10, x11, [x1], #16	/* copy from source address [x1] */
	stp	x10, x11, [x0], #16	/* copy to   target address [x0] */
	cmp	x1, x2			/* until source end address [x2] */
	b.lo	copy_loop
	str	x0, [sp, #24]

	/*
	 * Fix .rela.dyn relocations
	 */
	adr	x2, __rel_dyn_start	/* x2 <- Run &__rel_dyn_start */
	adr	x3, __rel_dyn_end	/* x3 <- Run &__rel_dyn_end */
fixloop:
	ldp	x0, x1, [x2], #16	/* (x0,x1) <- (SRC location, fixup) */
	ldr	x4, [x2], #8		/* x4 <- addend */
	and	x1, x1, #0xffffffff
	cmp	x1, #R_AARCH64_RELATIVE
	bne	fixnext

	/* relative fix: store addend plus offset at dest location */
	add	x0, x0, x9
	add	x4, x4, x9
	str	x4, [x0]
fixnext:
	cmp	x2, x3
	b.lo	fixloop

relocate_done:
	switch_el x1, 3f, 2f, 1f
	bl	hang
3:	mrs	x0, sctlr_el3
	b	0f
2:	mrs	x0, sctlr_el2
	b	0f
1:	mrs	x0, sctlr_el1
0:	tbz	w0, #2, 5f	/* skip flushing cache if disabled */
	tbz	w0, #12, 4f	/* skip invalidating i-cache if disabled */
	ic	iallu		/* i-cache invalidate all */
	isb	sy
4:	ldp	x0, x1, [sp, #16]
	bl	__asm_flush_dcache_range
5:	ldp	x29, x30, [sp],#32
	ret
ENDPROC(relocate_code)

//.popsection

/*
 * void __asm_flush_dcache_range(start, end)
 *
 * clean & invalidate data cache in the range
 *
 * x0: start address
 * x1: end address
 */
//.pushsection .text.__asm_flush_dcache_range, "ax"
ENTRY(__asm_flush_dcache_range)
	mrs	x3, ctr_el0
	lsr	x3, x3, #16
	and	x3, x3, #0xf
	mov	x2, #4
	lsl	x2, x2, x3		/* cache line size */

	/* x2 <- minimal cache line size in cache system */
	sub	x3, x2, #1
	bic	x0, x0, x3
1:	dc	civac, x0	/* clean & invalidate data or unified cache */
	add	x0, x0, x2
	cmp	x0, x1
	b.lo	1b
	dsb	sy
	ret
ENDPROC(__asm_flush_dcache_range)

ENTRY(hang)
	bl hang
ENDPROC(hang)

```

前面的中断初始化完成了，接下来就是注册、使能、执行中断、中断测试几个函数：  

```c
#include "irq.h"
#include "led.h"
#include "timer.h"
#include "printf.h"
#include "uart.h"


/* Cpu Interface System Registers */
#define ICC_IAR0_EL1		S3_0_C12_C8_0
#define ICC_IAR1_EL1		S3_0_C12_C12_0
#define ICC_EOIR0_EL1		S3_0_C12_C8_1
#define ICC_EOIR1_EL1		S3_0_C12_C12_1
#define ICC_HPPIR0_EL1		S3_0_C12_C8_2
#define ICC_HPPIR1_EL1		S3_0_C12_C12_2
#define ICC_BPR0_EL1		S3_0_C12_C8_3
#define ICC_BPR1_EL1		S3_0_C12_C12_3
#define ICC_DIR_EL1			S3_0_C12_C11_1
#define ICC_PMR_EL1			S3_0_C4_C6_0
#define ICC_RPR_EL1			S3_0_C12_C11_3
#define ICC_CTLR_EL1		S3_0_C12_C12_4
#define ICC_CTLR_EL3		S3_6_C12_C12_4
#define ICC_SRE_EL1			S3_0_C12_C12_5
#define ICC_SRE_EL2			S3_4_C12_C9_5
#define ICC_SRE_EL3			S3_6_C12_C12_5
#define ICC_IGRPEN0_EL1		S3_0_C12_C12_6
#define ICC_IGRPEN1_EL1		S3_0_C12_C12_7
#define ICC_IGRPEN1_EL3		S3_6_C12_C12_7
#define ICC_SEIEN_EL1		S3_0_C12_C13_0
#define ICC_SGI0R_EL1		S3_0_C12_C11_7
#define ICC_SGI1R_EL1		S3_0_C12_C11_5
#define ICC_ASGI1R_EL1		S3_0_C12_C11_6


static struct s_irq_handler g_irq_handler[NR_IRQS];

void irq_init(void)
{
    /* gic has been init in Start.S */
}

void enable_interrupts(void)
{
    my_printf("enable_interrupts \n\r");

    asm volatile("msr	daifclr, #0x03");
}

/* irq interrupt install handle */
void irq_install_handler(int irq, interrupt_handler_t *handler, void *data)
{
    my_printf("irq_install_handler \n\r");

    if (g_irq_handler[irq].m_func != handler)
        g_irq_handler[irq].m_func = handler;
}

/* enable irq handler */
int irq_handler_enable(int irq)
{
    my_printf("irq_handler_enable start \n\r");

    unsigned long M, N;

    if (irq >= NR_GIC_IRQS)
        return -1;

    M = irq / 32;
    N = irq % 32;

    GICD->ISENABLER[M]  = (0x1 << N);
    my_printf("irq_handler_enable end \n\r");

    return 0;
}
/*do_irq*/
void my_do_irq(void)
{
    my_printf("do_irq start \n\r");

    unsigned long nintid;
    unsigned long long irqstat;

    asm volatile("mrs %0, " __stringify(ICC_IAR1_EL1) : "=r" (irqstat));

    nintid = (unsigned long)irqstat & 0x3FF;

    /* here we use gic id checking, not include gpio pin irq */
    if (nintid < NR_GIC_IRQS)
        g_irq_handler[nintid].m_func((void *)(unsigned long)nintid);

    asm volatile("msr " __stringify(ICC_EOIR1_EL1) ", %0" : : "r" ((unsigned long long)nintid));
    asm volatile("msr " __stringify(ICC_DIR_EL1) ", %0" : : "r" ((unsigned long long)nintid));
    isb();
    my_printf("do_irq end \n\r");
	
}

static void board_timer_isr(void)
{
    my_printf("board_timer_isr start \n\r");

    static unsigned char led_flag = 0;
	/* Rockchip RK3399TRM V1.3 Part1.pdf P959
	 * TIMER_n_INTSTATUS 
	 * bit[0] int_pd 
	 * This register contains the interrupt status for timer n. 
	 * Write 1 to this register will clear the interrupt. 
	 */

    TIMER3->INTSTATUS = 0x01;  //clrear interrupt

    if(led_flag == 0)
        led_ctl(0);
    else
        led_ctl(1);

    led_flag = !led_flag;
    my_printf("board_timer_isr end \n\r");
	
}

/* Rockchip RK3399TRM V1.3 Part1.pdf P960
 * TIMER_n_CONTROLREG 
 * Address: Operational Base + offset (0x1c) 
 * Timer n Control Register 
 * bit[2] int_en Timer interrupt mask, 0: mask 1: not mask
 * bit[1] timer_mode Timer mode. 0: free-running mode 1: user-defined count mode 
 * bit[0] timer_en Timer enable. 0: disable 1: enable 
 * (1<<0) + (1<<1) + (1<<2) = 5
 */
 
void test_timer_irq(void)
{
    my_printf("test_timer_irq start \n\r");

    /* enable exceptions */
    enable_interrupts();

    /* timer set */
	
    TIMER3->CURRENT_VALUE0 = 0x0FFFFFF;
    TIMER3->LOAD_COUNT0    = 0x0FFFFFF;
    TIMER3->CONTROL_REG    = 0x05; //auto reload & enable the timer

    /* register and enable */
    irq_install_handler(TIMER_INTR3, (interrupt_handler_t *)board_timer_isr, (void *)(0));
    irq_handler_enable(TIMER_INTR3);
    my_printf("test_timer_irq end \n\r");

}
```

在主函数里对重定位的验证可以尝试定义一个全局变量检查是否正常，对清BSS段的验证可以尝试定义一个未初始化的全局变量检查是否正常，对中断的验证可以测试定时器中断是否正常，GPIO中断通过外接按键检测是否正常：  

```c

#include "led.h"
#include "uart.h"
#include "printf.h"
#include "timer.h"
#include "irq.h"

void uart_test(void) {
    uart_init();
	printf_test();
    my_printf("uart_init ok. \n\r");
}

int main(void)
{
    led_init();
	led_ctl(0);

	uart_test();

    test_timer_irq();	

    while(1)
    {
	    led_ctl(1);
	    led_delay();

	    led_ctl(0);
	    led_delay();
    }

    return 0;
}

```

*   测试效果：  
    上电后，蓝色LED1间隔闪烁。
    
    ![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/u-boot-timer-irq-ok.png)

5.串口
====================

5.1 uart代码
--------------------------------------

可以看到`rk_uart_init()`是串口初始化，里面依次调用了引脚复用`rk_uart_iomux()`、串口复位`rk_uart_reset()`、设置IRDA SIR功能`rk_uart_set_iop()`、设置串口属性`rk_uart_set_lcr()`、设置波特率`rk_uart_set_baudrate()`、设置串口FIFO`rk_uart_set_fifo()`。  
初始化完成后，就可以收发数据了，这里只实现了发生数据`rk_uart_sendbyte()`。  
移植的过程还是比较简单，寄存器比较少，注意在设置波特率函数里，需要用到除法，为了简便，可先算出来直接赋值。这里的波特率为最大的1.5M。

```c

#include "led.h"
#include "uart.h"

/* Rockchip RK3399TRM V1.3 Part1.pdf
 * GRF(P29)
 * FF76_0000 && FF77_0000
 * Chapter 2 System Overview (Address Mapping)
 */

//#define    CRU_BASE                  0xFF760000
//#define    GRF_BASE                  0xFF770000

/* Rockchip RK3399TRM V1.3 Part1.pdf
 * GRF_GPIO4C_IOMUX
 * Address: Operational Base + offset (0x0e028) 
 * GPIO4C iomux control
 */

static volatile unsigned int *GRF_GPIO4C_IOMUX;

static void rk_uart_iomux(void)
{
	GRF_GPIO4C_IOMUX  = (volatile unsigned int *)(0xFF760000 + 0x0e028);
    *GRF_GPIO4C_IOMUX = (3 << (8 + 16)) | (3 << (6 + 16)) | (2 << 8) | (2 << 6);
}

static void rk_uart_reset(void)
{
    /* UART reset, rx fifo & tx fifo reset */
    UART2_SRR =  (0x01 << 1) | (0x01 << 2);
    led_ctl(0);
    /* interrupt disable */
    UART2_IER = 0x00;

}

static void rk_uart_set_iop(void)
{
    UART2_MCR = 0x00;
}

static  void rk_uart_set_lcr(void)
{
    UART2_LCR &= ~(0x03 << 0);
    UART2_LCR |=  (0x03 << 0); //8bits

    UART2_LCR &= ~(0x01 << 3); //parity disabled

    UART2_LCR &= ~(0x01 << 2); //1 stop bit
}

static void rk_uart_set_baudrate(void)
{
    volatile unsigned long rate;
    //unsigned long baudrate = 1500000;

    /* uart rate is div for 24M input clock */
    //rate = 24000000 / 16 / baudrate;
    rate = 1;

    UART2_LCR |= (0x01 << 7);

    UART2_DLL = (rate & 0xFF);
    UART2_DLH = ((rate >> 8) & 0xFF);

    UART2_LCR &= ~(0x01 << 7);
}

static void rk_uart_set_fifo(void)
{
    /* shadow FIFO enable */
    UART2_SFE = 0x01;
    /* fifo 2 less than */
    UART2_SRT = 0x03;
    /* 2 char in tx fifo */
    UART2_STET = 0x01;
}

void uart_init(void)
{

    rk_uart_iomux();
    rk_uart_reset();

    rk_uart_set_iop();
    rk_uart_set_lcr();

    rk_uart_set_baudrate();

    rk_uart_set_fifo();

}

void rk_uart_sendbyte(unsigned char byte)
{

    while((UART2_USR & (0x01 << 1)) == 0);

    UART2_THR = byte;
}


void rk_uart_sendstring(char *ptr)
{
    while(*ptr)
        rk_uart_sendbyte(*ptr++);
}


/* 0xABCDEF12 */
void rk_uart_sendhex(unsigned int val)
{
    int i;
    unsigned int arr[8];

    for (i = 0; i < 8; i++)
    {
        arr[i] = val & 0xf;
        val >>= 4;   /* arr[0] = 2, arr[1] = 1, arr[2] = 0xF */
    }

    /* printf */
    rk_uart_sendstring("0x");
    for (i = 7; i >= 0; i--)
    {
        if (arr[i] >= 0 && arr[i] <= 9)
            rk_uart_sendbyte(arr[i] + '0');
        else if(arr[i] >= 0xA && arr[i] <= 0xF)
            rk_uart_sendbyte(arr[i] - 0xA + 'A');
    }
}

```

5.2 printf移植
--------------------------------------------

printf库移植的方法：  
1.先在`printf.h`里，用`__out_putchar`替换成自己实现的字节发送函数`rk_uart_sendbyte`；  
2.然后在`printf.c`里，为其提供宏`va_start`、`va_arg`、`va_end`、`_INTSIZEOF`和`va_list`；

在第二步里，之前ARMv7的可以直接使用，现在使用ARMv8，实测发现打印有问题，找到交叉编译工具里对应宏的位置，直接加入头文件`stdarg.h`即可。  

```c

#include  "printf.h"
#include  <stdarg.h>

/************************************************************************************************/
#if 0
typedef char   *va_list;
#define _INTSIZEOF(n)   ( (sizeof(n) + sizeof(int) - 1) & ~(sizeof(int) - 1) )

#define va_start(ap,v)  ( ap = (va_list)&v + _INTSIZEOF(v) )
#define va_arg(ap,t)    ( *(t *)((ap += _INTSIZEOF(t)) - _INTSIZEOF(t)) )
#define va_arg(ap,t)    ( *(t *)( ap=ap + _INTSIZEOF(t), ap- _INTSIZEOF(t)) )
#define va_end(ap)      ( ap = (va_list)0 )
#endif
/************************************************************************************************/

unsigned char hex_tab[] = {'0', '1', '2', '3', '4', '5', '6', '7', \
                           '8', '9', 'a', 'b', 'c', 'd', 'e', 'f'
                          };

static int outc(int c)
{
    __out_putchar(c);
    return 0;
}

static int outs (const char *s)
{
    while (*s != '\0')
        __out_putchar(*s++);
    return 0;
}

static int out_num(long n, int base, char lead, int maxwidth)
{
    unsigned long m = 0;
    char buf[MAX_NUMBER_BYTES], *s = buf + sizeof(buf);
    int count = 0, i = 0;

    *--s = '\0';

    if (n < 0)
        m = -n;
    else
        m = n;

    do
    {
        *--s = hex_tab[m % base];
        count++;
    }
    while ((m /= base) != 0);

    if( maxwidth && count < maxwidth)
    {
        for (i = maxwidth - count; i; i--)
            *--s = lead;
    }

    if (n < 0)
        *--s = '-';

    return outs(s);
}


/*ref: int vprintf(const char *format, va_list ap); */
static int my_vprintf(const char *fmt, va_list ap)
{
    char lead = ' ';
    int  maxwidth = 0;

    for(; *fmt != '\0'; fmt++)
    {
        if (*fmt != '%')
        {
            outc(*fmt);
            continue;
        }
		
		lead=' ';
		maxwidth=0;

        //format : %08d, %8d,%d,%u,%x,%f,%c,%s
        fmt++;
        if(*fmt == '0')
        {
            lead = '0';
            fmt++;
        }

        while(*fmt >= '0' && *fmt <= '9')
        {
            maxwidth *= 10;
            maxwidth += (*fmt - '0');
            fmt++;
        }

        switch (*fmt)
        {
        case 'd':
            out_num(va_arg(ap, int),          10, lead, maxwidth);
            break;
        case 'o':
            out_num(va_arg(ap, unsigned int),  8, lead, maxwidth);
            break;
        case 'u':
            out_num(va_arg(ap, unsigned int), 10, lead, maxwidth);
            break;
        case 'x':
            out_num(va_arg(ap, unsigned int), 16, lead, maxwidth);
            break;
        case 'c':
            outc(va_arg(ap, int   ));
            break;
        case 's':
            outs(va_arg(ap, char *));
            break;

        default:
            outc(*fmt);
            break;
        }
    }
    return 0;
}


//ref: int printf(const char *format, ...);
int my_printf(const char *fmt, ...)
{
    va_list ap;

    va_start(ap, fmt);
    my_vprintf(fmt, ap);
    va_end(ap);
    return 0;
}

int printf_test(void)
{
    my_printf("=========This is printf test=========\n\r");
    my_printf("test char            = %c,%c,%c\n\r", 'z', 'j', 'j');
    my_printf("test decimal1 number = %d\n\r",     123456);
    my_printf("test decimal2 number = %d\n\r",     -123456);
    my_printf("test hex1    number  = 0x%x\n\r",   0x123456);
    my_printf("test hex2    number  = 0x%08x\n\r", 0x123456);
    my_printf("test string          = %s\n\r",    "www.hermitme.com");

    return 0;
}

void puts(char *ptr)
{
    while(*ptr)
        rk_uart_sendbyte(*ptr++);
}

```

*   测试效果：

![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/u-boot-uart-ok.png)

6.定时器
=======================

RK3399有12个通用定时器(timer0~timer11)、12个安全定时器(stimer0~stimer11)、2个PMU定时器(pmutimer0~pmutimer1)。  
定时器部分比较简单，很多东西都是固定的，比如定时器的时钟来源都是24MHz的晶振，也就是定时器周期为1/24us。此外定时器的计数只能由小向大增加。  
定时器支持两种模式:自由运行模式和用户自定义模式，其实就是前者计数达到设定值后，自动装载计数循环，后者需要手动重新装载，实现循环。

6.1 编程思路
--------------------------------

这里希望通过用定时器实现一个比较准确的延时函数，包括us、ms、s的延时。  
1.首先设置`CONTROLREG`，关闭定时器、设置为用户定义计数模式(用户确定循环次数)、中断屏蔽(不需要中断处理函数)；  
2.向`LOAD_COUNT0`、`LOAD_COUNT1`放入计数结束值，向`LOAD_COUNT2`、`LOAD_COUNT3`放入计数初始值，默认为0；  
3.设置`CONTROLREG`，开启定时器，计数器开始运行；  
4.读取中断状态`INTSTATUS`判断时候完成计数，清中断，本次计数完成；

6.2 实现代码
--------------------------------

```c

#include "timer.h"

//timer4 is used for delay.
void delay_us(volatile unsigned long int  i)
{
    unsigned long int count_value = 24 * i;  //24MHz; period=(1/24000000)*1000000=1/24us

    TIMER4->CONTROL_REG &= ~(0x01 << 0);     //Timer disable
    TIMER4->CONTROL_REG |=  (0x01 << 1);     //Timer mode:user-defined count mode
    TIMER4->CONTROL_REG &= ~(0x01 << 2);     //Timer interrupt mask

    TIMER4->LOAD_COUNT0 = count_value & 0xFFFFFFFF; //load_count_low bits
    TIMER4->LOAD_COUNT1 = (count_value >> 32);      //load_count_high bits
	
    TIMER4->CONTROL_REG |=  (0x01 << 0);     //Timer enable

    while(!(TIMER4->INTSTATUS & (0x01 << 0)));
    TIMER4->INTSTATUS |= (0x01 << 0);        //Write 1 clear the interrupt

    TIMER4->CONTROL_REG &= ~(0x01 << 0);     //Timer enable disable
}

void delay_ms(volatile unsigned long int i)
{
    for(; i > 0; i--)
        delay_us(1000);
}

void delay_s(volatile unsigned long int i)
{
    for(; i > 0; i--)
        delay_ms(1000);
}
```

7.ADC
=======================

RK3399有两类ADC：

*   TS-ADC(Temperature Sensor):  
    　　内嵌的两路ADC，一路检测CPU温度，一路检测GPU温度；  
    　　ADC精度10bit，时钟频率必须低于800KHZ;  
    　　测量范围为-40℃~125℃，精度只有5℃；  
    　　支持用户自定义和自动模式(前者用户自己控制，后者控制器自动查询)；
*   SAR-ADC(Successive Approximation Register):  
    　　六路ADC，精度10bit；  
    　　时钟频率必须小于13MHZ;

7.1 编程思路
--------------------------------

这里希望通过用SAR-ADC获取外部ADC值，通过TS-ADC获取内部CPU/GPU温度。

*   SAR-ADC  
    1.首先设置`SARADC_CTRL[3]`，关闭ADC；  
    2.设置`SARADC_CTRL[2:0]`，选择ADC通道；  
    3.设置`SARADC_CTRL[3]`，启动ADC转换；  
    4.读取ADC状态`SARADC_STAS`判断是否转换完成；  
    5.读取ADC数据`SARADC_DATA`；
    
*   TS-ADC(User-Define Mode)  
    1.首先设置`TSADC_AUTO_CON`为用户定义模式、ADC值与温度值负关系；  
    2.设置`TSADC_USER_CON`选择通道、复位、转换开始；  
    3.设置`TSADC_INT_EN`，使能ADC完成中断；  
    4.读取ADC中断状态`TSADC_INT_PD`判断是否转换完成，并清理；  
    5.根据选择的通道，从对应的`TSADC_DATA0`或`TSADC_DATA1`读取ADC数据；
    

这里的TS-ADC得到的数值和温度并不是完全的线性关系，根据提供的表格，可以计算出一个大致的线性关系：`y = 0.5823x - 273.62`

7.2 实现代码
--------------------------------

```c

#include "uart.h"
#include "printf.h"
#include "timer.h"
#include "int.h"
#include "adc.h"

unsigned int get_saradc_val(unsigned char channel)
{
    unsigned int val;

    //delay between power up and start command
    //SARADC_DLY_PU_SOC = 8; //DLY_PU_SOC + 2

    SARADC_CTRL &= ~(0x01 << 3); //ADC power down control bit

    SARADC_CTRL |= (channel << 0); //ADC input source selection

    //SARADC_CTRL |= (0x01<<3); //Interrupt enable.

    SARADC_CTRL |=  (0x01 << 3); //ADC power up and reset
    delay_us(100); //不能立即就判断状态

    while(SARADC_STAS & 0x01); //The status register of A/D Converter 1’b0: ADC stop

    val = SARADC_DATA & 0x3FF; //A/D value of the last conversion (DOUT[9:0]).

    return val;
}

//channel0: CPU temperature
//channel1: GPU temperature
int get_tsadc_temp(unsigned char channel)
{
    int val;

    if ((channel != 0) && (channel != 1))
    {
        printf("get_tsadc_temp set channel error.\n");
        return -255;
    }

    //User-Define Mode
    TSADC_AUTO_CON &= ~(0x01 << 0);     //TSADC controller works at user-define mode
    TSADC_AUTO_CON |=  (0x01 << 1);     //RK3399 is negative temprature coefficient

    TSADC_USER_CON &= ~(0x07 << 0);     //clear
    TSADC_USER_CON |=  (channel << 0);  //PD_DVDD and ADC input source selection
    TSADC_USER_CON |=  (0x01 << 3);     //CHSEL_DVDD and ADC power up and reset

    TSADC_USER_CON |=  (0x01 << 4);     //the start_of_conversion will be controlled by TSADC_USER_CON[5].
    TSADC_USER_CON |=  (0x01 << 5);     //start conversion

    TSADC_INT_EN   |=  (0x01 << 16);    //eoc_interrupt enable in user defined mode
    while(!(TSADC_INT_PD & (0x01 << 16))); //wait ADC conversion stop
    TSADC_INT_PD &= ~(0x01 << 16);

    if (0 == channel)
        val = (int)(0.5823 * (float)(TSADC_DATA0) - 273.62); //y = 0.5823x - 273.62
    else
        val = (int)(0.5823 * (float)(TSADC_DATA1) - 273.62); //y = 0.5823x - 273.62

    printf("get_tsadc_temp = %d \n", val);

    return val;
}
```

*   测试效果：

![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/u-boot-adc-ok.png)

8.I2C
=======================

RK3399拥有8个I2C，其功能和其它SOC的I2C差不多，这里通过I2C读写EEPROM。

8.1 编程思路
--------------------------------

0.首先是I2C引脚复用、设置SCK时钟、注册/使能中断(非必须)等；

*   写EEPROM  
    1.清空控制寄存器`CON`并使能；  
    2.设置I2C模式`transmit only`；  
    3.设置`CON`启动开始信号，并读取`IPD`等待开始信号发送完成；  
    4.设置`TXDATA0`实现从机地址、写地址、数据的设定，设置传输数据个数，等待传输完成；  
    5.设置`CON`发送结束信号，并读取`IPD`等待结束信号发送完成；
    
*   读EEPROM  
    1.清空控制寄存器`CON`并使能；  
    2.设置I2C模式`transmit only + restart + transmit address + receive only`；  
    3.设置`MRXADDR`设定从机地址，设置`MRXRADDR`设定从机寄存器地址；  
    4.设置`TXDATA0`实现从机地址、写地址、数据的设定，设置传输数据个数，等待传输完成；  
    5.设置`MRXCNT`值接收一个数据，读取`IPD`等待接收数据完成；  
    6.设置`CON`发送结束信号，并读取`IPD`等待结束信号发送完成；  
    7.读取`RXDATA0`获得数据；
    

8.2 实现代码
--------------------------------

```c
#include "i2c.h"
#include "uart.h"
#include "printf.h"
#include "timer.h"


//GPIO1_B3/I2C4_SDA
//GPIO1_B4/I2C4_SCL

void i2c_init(void)
{
    //1.GPIO1_B3/I2C4_SDA、GPIO1_B4/I2C4_SCL设置为功能引脚,注意高位要先置为1才能写;
    PMUGRF_GPIO1B_IOMUX |= ((0xFFFF0000 << 0) | (0x01 << 6) | (0x01 << 8));

    //2.设置SCK时钟
    //3.注册/使能中断
}


void eeprom_write(unsigned char addr, unsigned char data)
{
    //0.清空控制寄存器并使能
    I2C4->CON &= ~(0x7F << 0);
    I2C4->IPD &= ~(0x7F << 0);
    I2C4->CON |= 0x01 << 0; //使能

    //1.设置模式:transmit only
    I2C4->CON &= ~(0x03 << 1);

    //2.开始信号
    I2C4->CON |= 0x01 << 3; //开始信号
    while(!(I2C4->IPD & (0x01 << 4))); //等待开始信号发完
    I2C4->IPD |=  (0x01 << 4); //清开始信号标志

    //3.I2C从机地址+写地址+数据 (3个字节)
    I2C4->TXDATA0 = 0x46 | (addr << 8) | (data << 16);
    I2C4->MTXCNT = 3;
    while(!(I2C4->IPD & (0x01 << 2))); //MTXCNT data transmit finished interrupt pending bit
    I2C4->IPD |=  (0x01 << 2);

    //4.结束信号
    I2C4->CON &= ~(0x01 << 3); //手动清除start(注意:前面的开始信号控制位理论会自动清0,实测没有,这里必须手动清,否则是开始信号)
    I2C4->CON |= (0x01 << 4);
    while(!(I2C4->IPD & (0x01 << 5)));
    I2C4->IPD |=  (0x01 << 5);
}

//自动发送从机地址和从机寄存器地址
unsigned char eeprom_read(unsigned char addr)
{
    unsigned char data = 0;

    //0.清空控制寄存器并使能
    I2C4->CON &= ~(0x7F << 0);
    I2C4->IPD &= ~(0x7F << 0);
    I2C4->CON |= 0x01 << 0; //使能

    //必须收到ack,否则停止传输(非必需)
    //I2C4->CON |=  (0x01<<6); //stop transaction when NAK handshake is received

    //1.设置模式:transmit address (device + register address) --> restart --> transmit address –> receive only
    I2C4->CON |=  (0x01 << 1); //自动发送从机地址和从机寄存器地址

    //2.从机地址
    I2C4->MRXADDR = (0x46 | (1 << 24));

    //3.从机寄存器地址
    I2C4->MRXRADDR = (addr | (1 << 24)); //地址只有6位,超过6位怎么办?

    //4.开始信号
    I2C4->CON |=  (0x01 << 3);
    while(!(I2C4->IPD & (0x01 << 4)));
    I2C4->IPD |=  (0x01 << 4);

    //5.接收一个数据且不响应
    I2C4->CON |= (0x01 << 5);
    I2C4->MRXCNT = 1;
    while(!(I2C4->IPD & (0x01 << 3)));
    I2C4->IPD |=  (0x01 << 3);

    //6.结束信号
    I2C4->CON &= ~(0x01 << 3); //手动清除start
    I2C4->CON |= (0x01 << 4);
    while(!(I2C4->IPD & (0x01 << 5)));
    I2C4->IPD |=  (0x01 << 5);

    return (I2C4->RXDATA0 & 0xFF);
}
```

主函数里先向EEPROM(0x46地址不是连的EEPROM)写数据，再读出数据并打印出来，是否是预期的值。  

```c
void i2c_test(void){
	int i;
	i2c_init();
	
	//write eeprom.
	for(i=0; i<5; i++)
    {
        //eeprom_write(i,2*i);
        delay_ms(4);//Must be delayed more than 4ms.
    }
	
	my_printf("write eeprom ok\n\r");
	delay_ms(10);
	
	//read eeprom.
    for(i=0; i<10; i++)
    {
        my_printf("read_data%d = %d\n\r", i, eeprom_read(i));
        delay_ms(4);
    }
}

```

*   测试效果(0x46地址不是连的EEPROM，i2c测试失败，以后填坑)：

![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/u-boot-i2c-read-ok.png)

9.SPI
=======================

RK3399有6组SPI，协议也是标准的，没什么好说的。通过SPI读取Flash，也实现了两个版本：GPIO模拟和寄存器控制，这里主要介绍寄存器控制版本。

9.1 编程思路
--------------------------------

0.首先是SPI引脚复用、设置时钟、SPI模式(SCPH=1，SCPOL=1)等；  
1.实现发送一字节函数：使能SPI、向`TXDR[0]`写入待发送的数据、根据`SR`等待发送完成及空闲、关闭SPI；  
2.实现接收一字节函数：使能SPI、向`TXDR[0]`写入空数据、根据`SR`等待接收完成及空闲、读出`RXDR[0]`数据、关闭SPI；  
3.实现片选函数；  
4.剩下的就是SPI Flash(W25Q16DV)相关的操作，比如发送哪个指令读取ID，发送哪个指令擦除数据等，参考具体的Flash芯片手册；

值得注意的几点有：  
1.SPI Flash(W25Q16DV)每次写操作某个分区前都得先擦除该分区；  
2.注意片选的连续性，比如写使能指令(0x06)和写状态寄存器指令(0x01)之间的片选不能中断；

9.2 实现代码
--------------------------------

```c
#include "spi.h"
#include "uart.h"
#include "printf.h"
#include "timer.h"
#include "gpio.h"
#include "grf.h"

void spi_init(void)
{
    //SPI1_CSn0/GPIO1_B2_U
    //SPI1_CLK/GPIO1_B1_U
    //SPI1_TXD/GPIO1_B0_U
    //SPI1_RXD/GPIO1_A7_U

    //1.IOMUX
    PMUGRF_GPIO1A_IOMUX = 0xFFFF8000;
    PMUGRF_GPIO1B_IOMUX = 0xFFFF002A;

    SPI1->ENR &=  ~(0x01 << 0); //关闭SPI

    //2.Clock Ratios   master mode:Fspi_clk>= 2 × (maximum Fsclk_out)
    //CRU_CLKGATE2_CON &= ~(0x01<<9);  //默认SPI1 source clock开启
    //CRU_CLKGATE6_CON &= ~(0x01<<4);  //默认SPI1 APB clock开启
    SPI1->BAUDR = 24; //Fsclk_out = 48/24= 2M   48 >= 2x2

    //3.注册/使能中断(本程序未使用,用的查询)
    //register_irq(IRQ_SPI1, spi_irq_isr);
    //irq_handler_enable(IRQ_SPI1);
    //SPI1->IPR &= ~(0x01<<4); //Active Interrupt Polarity Level is HIGH(default)
    //SPI1->IMR |=  ((0x01<<4) | (0x01<<3) | (0x01<<2) | (0x01<<1) | (0x01<<0)); //Interrupt Mask

    //4.DMA(可以不用)
    //SPI1->DMACR |= ((0x01<<1) | (0x01<<0)); // Transmit/Receive DMA enabled
    //SPI1->DMATDLR = 1; //?
    //SPI1->DMARDLR = 1; //?

    //5.SPI模式
    //[1:0]Data Frame Size:8bit data
    //[5:2]Control Frame Size:8-bit serial data transfer
    //[6]SCPH:Serial clock toggles at start of first data bit
    //[7]SCPOL:Inactive state of serial clock is high
    //[13]BHT:apb 8bit write/read, spi 8bit write/read
    //[19:18]XFM(Transfer Mode):Transmit & Receive(default)
    //[20]OPM(Operation Mode):Master Mode(default)

    SPI1->CTRLR0 &= ~(0x03 << 0) ;
    SPI1->CTRLR0 |= ((0x01 << 0) | (0x07 << 2) | (0x01 << 6) | (0x01 << 7) | (0x01 << 13)); //设置SPI模式
}

void spi_send_byte(unsigned char val)
{
    SPI1->ENR |=  (0x01 << 0);      //SPI Enable

    SPI1->TXDR[0] = val & 0xFFFF;
    while(!(SPI1->SR & (0x01 << 2))); //Transmit FIFO is empty
    while(SPI1->SR & (0x01 << 0));  //SPI is idle or disabled

    SPI1->ENR &=  ~(0x01 << 0);     //SPI Disable
}

static unsigned char spi_recv_byte(void)
{
    unsigned char val = 0;

    SPI1->ENR |=  (0x01 << 0);    //SPI Enable

    SPI1->TXDR[0] = 0;            //因为是发送接收模式,FIFO在发送时也会接收数据,这里发送空数据,就可读取数据

    while(SPI1->SR & (0x01 << 3)); //SReceive FIFO is not empty
    while(SPI1->SR & (0x01 << 0)); //SPI is idle or disabled

    val = SPI1->RXDR[0] & 0xFF;  //读数据

    SPI1->ENR &=  ~(0x01 << 0);  //SPI Disable,为了清空FIFO

    return val;
}


void spi_flash_set_cs(unsigned char flag)
{
    if(!flag)
        SPI1->SER |=  (0x01 << 0);
    else
        SPI1->SER &= ~(0x01 << 0);
}




/* 通用部分 */
static void spi_flash_send_addr(unsigned int addr)
{
    spi_send_byte(addr >> 16);
    spi_send_byte(addr >> 8);
    spi_send_byte(addr & 0xff);
}

static void spi_flash_write_enable(int enable)
{
    if (enable)
    {
        spi_flash_set_cs(0);
        spi_send_byte(0x06);
        spi_flash_set_cs(1);
    }
    else
    {
        spi_flash_set_cs(0);
        spi_send_byte(0x04);
        spi_flash_set_cs(1);
    }

}

static unsigned char spi_flash_read_status_reg1(void)
{
    unsigned char val;

    spi_flash_set_cs(0);

    spi_send_byte(0x05);
    val = spi_recv_byte();

    spi_flash_set_cs(1);

    return val;
}

static unsigned char spi_flash_read_status_reg2(void)
{
    unsigned char val;

    spi_flash_set_cs(0);

    spi_send_byte(0x35);
    val = spi_recv_byte();

    spi_flash_set_cs(1);

    return val;
}

static void spi_flash_wait_when_busy(void)
{
    while (spi_flash_read_status_reg1() & 1);
}

static void spi_flash_write_status_reg(unsigned char reg1, unsigned char reg2)
{
    spi_flash_write_enable(1);

    spi_flash_set_cs(0);

    spi_send_byte(0x01);
    spi_send_byte(reg1);
    spi_send_byte(reg2);

    spi_flash_set_cs(1);

    spi_flash_wait_when_busy();
}

static void spi_flash_clear_protect_for_status_reg(void)
{
    unsigned char reg1, reg2;

    reg1 = spi_flash_read_status_reg1();
    reg2 = spi_flash_read_status_reg2();

    reg1 &= ~(1 << 7);
    reg2 &= ~(1 << 0);

    spi_flash_write_status_reg(reg1, reg2);
}


static void spi_flash_clear_protect_for_data(void)
{
    /* cmp=0,bp2,1,0=0b000 */
    unsigned char reg1, reg2;

    reg1 = spi_flash_read_status_reg1();
    reg2 = spi_flash_read_status_reg2();

    reg1 &= ~(7 << 2);
    reg2 &= ~(1 << 6);

    spi_flash_write_status_reg(reg1, reg2);
}

/* erase 4K */
void spi_flash_erase_sector(unsigned int addr)
{
    spi_flash_write_enable(1);

    spi_flash_set_cs(0);

    spi_send_byte(0x20);
    spi_flash_send_addr(addr);

    spi_flash_set_cs(1);

    spi_flash_wait_when_busy();
}

/* program */
void spi_flash_program(unsigned int addr, unsigned char *buf, int len)
{
    int i;

    spi_flash_write_enable(1);

    spi_flash_set_cs(0);

    spi_send_byte(0x02);
    spi_flash_send_addr(addr);

    for (i = 0; i < len; i++)
        spi_send_byte(buf[i]);

    spi_flash_set_cs(1);

    spi_flash_wait_when_busy();

}

void spi_flash_read(unsigned int addr, unsigned char *buf, int len)
{
    int i;

    spi_flash_set_cs(0);

    spi_send_byte(0x03);
    spi_flash_send_addr(addr);
    for (i = 0; i < len; i++)
        buf[i] = spi_recv_byte();

    spi_flash_set_cs(1);
}

void spi_flash_init(void)
{
    spi_flash_clear_protect_for_status_reg();
    spi_flash_clear_protect_for_data();
}

void spi_flash_read_ID(unsigned int *pMID, unsigned int *pDID)
{

    spi_flash_set_cs(0);

    spi_send_byte(0x90);

    spi_flash_send_addr(0);

    *pMID = spi_recv_byte();
    *pDID = spi_recv_byte();

    spi_flash_set_cs(1);
}
```

在主函数里先读取Flash的MID和PID，然后初始化Flash(去除写状态寄存器保护和写数据保护)，再写入数据，读出数据检测是否一致。  

```c
int main(void)
{
    led_init();
	led_ctl(0);

	unsigned int i, j;
	char str[20];
	unsigned int mid, pid; 
	
	uart_init();
	my_printf("uart_init ok.\n\r");

    spi_init();
	
	spi_flash_read_ID(&mid, &pid);
	my_printf("SPI Flash : MID = 0x%02x, PID = 0x%02x\n\r", mid, pid);	

    spi_flash_init();
    while(1)
	{
        spi_flash_erase_sector(4096);
        spi_flash_program(4096, "zjj", 7);
        spi_flash_read(4096, str, 7);
        my_printf("SPI Flash read from 4096: %s\n\r", str);
        
        delay_s(2);
    }

	return 0;

}

```

*   测试效果：

![](https://raw.githubusercontent.com/zhoujinjiani/PicGo/master/kernel.embedded/u-boot-spi-ok.png)

