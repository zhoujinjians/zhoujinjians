---
title: Android 8.1 Display System源码分析（2）：LCD显示原理（RK3399）
cover: https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/personal.website/post.cover.pictures.00011.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20200608
date: 2020-06-08 09:25:00
---



注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux @Rockchip版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-4.4**==）&&（==**文章基于 Android-8.1**==）
[【开发板 - Firefly-RK3399 7.85寸1024x768显示屏模组（MIPI接口）】](
http://wiki.t-firefly.com/zh_CN/Firefly-RK3399/compile_android8.1_firmware.html#)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

[（1）【How a Flat Screen Display works】](https://animagraffs.com/flat-screen-displays/)
[（2）【Adding a custom display】](https://www.digi.com/resources/documentation/digidocs/90001945-13/reference/yocto/r_an_adding_custom_display.htm) 
[（3）【Liquid Crystal Display Macro Example 】](https://commons.wikimedia.org/wiki/File:Liquid_Crystal_Display_Macro_Example_zoom_2.jpg)
[（4）【How-it-works: LCD screens explained】](https://pid.samsungdisplay.com/en/learning-center/blog/lcd-structure)
[（5）【显示面板技术说明 - LCD模式】](https://pid.samsungdisplay.cn/learning-center/white-papers/va-vs-ips)
[（6）【了解当今LCD屏技术】](https://pid.samsungdisplay.cn/learning-center/blog/lcd-structure)


--------------------------------------------------------------------------------
==源码（部分）==：

--------------------------------------------------------------------------------


####  （一）、了解当今LCD屏技术
##### （1）、LCD panels

An LCD panel is a matrix of pixels that are divided into rows and columns. These pixels are individually painted according to different signals and timing parameters, and you can control each pixel's color individually. The panel is continuously refreshed, typically at around 60 Hz, from the contents of the frame buffer memory. Each memory location on the frame buffer corresponds to a pixel on the LCD panel.

**工作原理：LCD屏说明**
液晶显示器（LCD）是一种利用液晶的电光特性将电刺激转换为视觉信号的装置。它助你将想象力与创意变为现实，并将其显示在屏幕上。

创造这种“神奇”体验的四个基本原则是：
> 光可以极化。
> 液晶取向可以通过电流改变。
> 液晶可以操纵（透射或阻挡）偏振光。
> 有一些可以导电的透明物质。
液晶分子排列可以通过外部电场操纵，这使得液晶显示器可以滴答作响。液晶（LC）是一种状态下的物质，具有常规液体和固体晶体的性质。液晶可能像液体一样流动，但其分子可以以晶体的方式取向。 

虽然LCD构成了显示面板的核心，但是还有其他光电和电路元件在显示效果上发挥重要作用。

想了解更多各种液晶模式吗？[那就阅读这份有关各种LCD模式的白皮书](https://pid.samsungdisplay.cn/learning-center/white-papers/va-vs-ips)。

##### （2）、您的LCD屏幕内是怎样运作的？

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.LCD_Function.gif)

当一束光通过LCD屏幕的各个组件时，我们通过它来了解：

1. 当屏幕通电时，背光LED会发出白光。
2. 光进入导光板（LGP），在内部反射，并均匀地分布在面板的上表面上。
3. 扩散片进一步分散光，所以LGP外面没有观察到热点。 
4. DBEF回收散射光，棱镜片确保光线聚焦并指向观看者。
5. 底部偏光片允许垂直波长光通过，同时阻挡其他取向。
6. 然后垂直偏振光通过液晶层。
7. 然后通过施加适当的电压通过TFT和公用电极来操纵液晶。液晶能在不同程度上阻挡白光。每个子像素前面的滤光片仅允许8通过适合其颜色的一系列波长。为了控制每个子像素的亮度，将液晶盒通电或断电，以便阻挡或透射光。
8. 光通过液晶和滤光片，产生原红、原绿及原蓝三色。
9. 然后通过顶部偏光片对偏振光进行滤光，仅透射水平偏振光。
10. 最后，观看者便能在数字显示器上享受鲜艳的色彩、高对比度和清晰的图像。
##### （3）、LCD组件解构

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.LCD_Component_Layers.gif)

> **底部底盘：**底部底盘保护LCD显示组件，并作为底座将组件放在一起。


> **背光：**液晶本身并不发光，所以需要另外的方式的供应光才能观看屏幕。光源可以是环境光或是位于屏幕背面或侧面的人造光源。LCD是一种透射显示器，因此需要外部光源。

LED背光式LCD显示器是一种使用LED背光的平板显示器。使用LED背光可使面板更薄、功耗更低、散热更好、显示更亮、对比度更好。发光二极管（LED）为显示器提供光源。当今最常见的背光布置是边缘式LED（更薄型材）或直下式LED（用于高亮度显示器或窄边框视频壁挂式显示器）。背光设计对于确保良好的色彩再现和广泛的色域十分重要。背光看起来是白色的，但并不意味着它具有广泛而均匀的光谱 — 它的“高峰”光谱可能极不规则（CCFL灯通常是这样）。

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.Edge_vs_Direct_backlight.png)


> **反射片：**反射片提供LCD背光回收。它通常称为DBEF（双亮度增强胶片）。DBEF增加轴上亮度，因此可以通过LCD透射更多的光。通常，一个BEF可将亮度提高40％ - 60％。在一些应用中，使用两个BEF来提高亮度透射率。


>**导光板（LGP）：**LGP是一种丙烯酸板，通常由纯PMMA（聚甲基丙烯酸甲酯）树脂制成。PMMA非常透明，耐候性好。导光板是刻有凸起图案的塑料板，可以反射特定方向的光。LGP将线形光源转换成均匀的平面形光源。LPG面板的底部刻有线阵，将光线引导到前面，这称为V切割。从侧面进入导光层的光将从前面射出。

>**扩散片：**扩散片设计用于均匀分解和分布光，以提供柔和的光线。扩散片将光线均匀地铺展在屏幕尺寸上，形成一个坚实的、均匀照明的方形，并减少LED热点。

>**棱镜片：**在液晶显示器导光板的上表面上提供棱镜片。棱镜片前表面上有小型角状脊，可再循环离轴光，直到以最佳视角透射。光波以对观看者来说最亮的角度射出，或者再次通过背光照明层发送，直至正确射出。

>**底部偏光片：**偏光片的制造如下：通过拉伸塑料状材料来延长其纤维，然后将材料浸入碘中进一步延长，并将材料的纤维组织成为人眼不可见的黑色平行线网格。这就像一个滤光片，仅允许垂直方向的光波通过，而阻挡其余的光波。

>**底玻璃基板（背板）：**特殊玻璃用作薄膜晶体管（TFT）制造工艺的起始基板。液晶通常“夹在”两个偏振滤光片之间，彼此成90度角。偏振光从背光式LED进入液晶背面。当向列晶体未通电时，便将偏振光“扭转”90度，使其通过第二偏振滤光片。当电场应用于液晶时，光不会受到扭曲，从而被第二偏振滤光片阻挡。

>**薄膜晶体管：**一种晶体管，其主动载流层是一层薄膜（通常是硅薄膜 - Si），与MOSFET相反，其在硅晶片上制成并使用体硅作为有源层。在平板显示器中，光必须能够通过基板材料到达观看者。不透明硅片显然不适合这些透射显示器。玻璃是最常用的起始基板，因为它高度透明并与常规的半导体处理步骤兼容。由于玻璃不像硅一样属于半导体，所以在顶部沉积硅薄膜，并使用该薄膜制造晶体管。TFT帮助操纵每个子像素处的电压来编排显示图像。



三星显示器的PID面板是 有源矩阵TFT，有源矩阵LCD显示面板依靠薄膜晶体管（TFT）保持扫描之间的每个像素状态，同时改善响应时间。TFT是一种微型开关晶体管（及相关联的电容器），布置在玻璃基板上的矩阵中，以控制每个元素（或像素）。打开其中一个TFT将激活相关像素。使用嵌入显示面板本身的有源开关装置来控制每个图像元件，有助于减少相邻像素之间的串扰，同时显着地改善显示响应时间。通过以非常小的增量仔细调节施加的电压量，可以产生灰度效应。尽管LCD标牌中使用的高端LCD显示器可以支持高达1024个不同的亮度级别，而当今大多数液晶显示器每像素最少支持256级亮度。这导致灰度性能提升，因此提升主要全暗或全亮的图像区域中的图像细节。

>**液晶：**液晶在施加的电场下改变取向，从而阻挡或通过光。液晶是一种棒状分子，向其施加电流时会发生扭曲。每个晶体的作用就像一个快门，要么允许光线通过要么阻挡光线。透明和深色晶体的图案形成图像。LCD显示器中的液晶是自然扭曲的形式。

>**公用电极：**由用于将电压施加到液晶层的透明氧化铟锡（ITO）制成。对于在整个LCD屏幕上保持均匀像素电压，公用电极起着关键作用。在彩色屏幕中，ITO现在分为红色、绿色和蓝色（RGB）三种颜色。

> **滤光片（RGB）：**滤光片为LCD上的图像创建颜色。滤光片由红色、绿色和蓝色颜料组成，并与细胞内的特定子像素对准。该滤光片由薄玻璃基板和彩色光阻组成。在玻璃基板上形成三种彩色光阻（红、绿、蓝）图案。R、G、B图案称为子像素。可以显示颜色的LCD必须具有三个具有红色、绿色和蓝色滤光片的子像素，以创建每个颜色像素。通过仔细控制和变化施加的电压，每个子像素的强度可以超过256色。组合子像素可以生成1680万种颜色的可能调色板（256种红色x 256种绿色x 256种蓝色，8位色深）。这些彩色显示器占用了大量的晶体管（TFT）。典型的笔记本电脑支持高达1,920x180的分辨率。如果我们将1,920列乘以1,080行乘以3个子像素，则将有6,220,800个晶体管蚀刻到背板玻璃上。

>**顶部玻璃基板：**用于滤光片制造工艺的特殊玻璃。

>**顶部偏光片：**光在顶部偏光片处水平偏振。偏光片的功能是改善颜色和清晰度，使得可以看到LCD的屏幕。如果从LCD中移除偏光片，则无法识别字母或图形。当两个偏光膜平行放置在另一个的顶部时，LCD的屏幕最亮。然而，当放置在彼此的顶部并且垂直时，屏幕看起来就像是黑的。因此，如上所述，LCD的光学特性，如亮度和对比度，受到偏光膜性质的严重影响。

>**顶部机箱或机架：**顶部机箱位于顶部，并使用开放式框架。

##### （4）、液晶运行模式

**面板中的液晶运行模式是显示面板的关键特性，对于其性能和功能起决定作用。** 当前大型和商用显示面板中使用最多的是VA和IPS技术。

###### 4.1、**垂直取向(VA)**

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.samsungpid-va-technology.jpg)

在垂直取向(VA)模式中，液晶与玻璃基板自然垂直对齐，也被称为垂面排列。在没有外部电压的情况下，偏振光在无偏光变化的情况下穿过面板，然后被第二组偏光片完全阻挡（与第一组偏光片成90度角），形成完美的黑色。电场的应用会使液晶分子旋转到水平位置，让光线完全穿过并显示白色。

与TFT升级、雾面涂层、现代背光装置和像素设计相结合后，赋予了VA LCD超高的对比度，给人印象深刻的视觉体验。随着技术的进步，其响应时间和像素延迟大幅缩短，显示滞后也不复存在。帧率和动态画面响应时间也变得更加顺畅。VA抗残影的能力更强，因为垂面排列可以加宽散布面。VA还能实现最佳的对比度，很多显示专业人士认为它在商用显示面板的显示品质方面排名第一。
###### 4.2、**共面转换(IPS)**

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.samsungpid-ips-technology.jpg)

共面转换(IPS)模式最初是为克服扭曲向列LCD中出现的窄视角问题而开发的。该技术通过改变施加的电压使液晶在与玻璃基板平行的面上排列，从而可更改共面上液晶分子的方向。IPS也被称为水平排列。[3] 由于偏光片位于同一面上，转换效果由液晶分子围绕垂直于其长度方向的轴旋转而实现的。[2] VA模式中的液晶分子与偏光轴对齐，但IPS中的液晶分子与偏光轴偏离。

IPS矩阵不仅晶体结构不同，而且一个晶圆上电极的位置也不同，会导致占用更多空间，导致屏幕对比度和亮度更低。该技术的另一个主要缺点在于响应时间长，有时G到G长达60毫米。在新一代产品中，该技术缩短了响应时间，但仍高于VA面板的平均响应时间。[6] 这往往会导致残影，特别是在零售环境下，如长时间显示特惠或促销静态图像时。

除低对比度外，IPS LCD往往还具有黑色纯度低的问题。现代IPS技术的透过率相比最初的技术提升了约30%。虽然这有助于增加对比度，但仍比不上VA面板的对比度。最众所周知的一个缺点是，从侧面看深色图像时IPS面板会出现白光，也被称为IPS光。[6] 由于液晶的特性，从侧面看时IPS虽仍可保持色彩均匀性，但从这些角度看其亮度会降低。

##### （5）、LCD色彩生成原理
传统的LCD显示屏有一个由许多**发光二极管(LED)**组成的背光系统。这些LED是蓝色的，但被绿色和红色荧光体覆盖，以产生白光。通过改变荧光体的浓度，还可以改变和控制LED的色温。

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.color-displays-wp-color-expression-lcd-panel-red.gif)

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.color-displays-wp-color-expression-lcd-panel-green.gif)

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.color-displays-wp-color-expression-lcd-panel-blue.gif)

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.color-displays-wp-color-expression-lcd-panel-white.gif)


从LED发出的光通过偏光滤光片后射到**液晶(LC)**上，液晶挡住光线或让光线通过红、绿、蓝**彩色滤光片(CF)**。这称为**子像素**。因为红色、绿色和蓝色可混合产生任何颜色，由三个子像素形成的每个颜色像素可创造出不同的色彩，然后形成图像。通过控制和改变电压可调节各子像素的强度，从而使其更亮或更暗，进而确定在显示屏上产生什么颜色。通过子像素的不同组合可产生数百万种颜色。

这种使用白光LED的机制简单、廉价 - 因此它在显示行业中得到了广泛采用，包括商用显示屏、电视、显示器、笔记本电脑、平板电脑和智能手机。

Macro shot of an LCD monitor showing the red, green and blue elements., with zoom-in(LCD显示器放大):

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.Liquid_Crystal_Display_Macro_Example_zoom_2.jpg)


####  （二）、How a Flat Screen Display works

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.flat-panel.gif)

####  （三）、LCD displays signals and timing parameters

##### （1）、LCD displays signals and timing parameters
The LCD controller paints frames from left to right and from top to bottom. Signals used on a typical LCD display include:
- **HSYNC:** Horizontal synch (FPLINE or LP) indicates the end of a line and the beginning of the next line.
- **VSYNC:** Vertical synch (FPFRAME, FLM, SPS or TV) indicates the end of the current frame. The next line index should restart at zero in the upper-left corner.
- **DRDY:** Data ready or data enable (DE). The data in the bus is valid and must be latched using the PIXCLK signal. While it’s active, each pulse draws a pixel.
- **PIXCLK:** Specifies the placing of RGB data on the bus.
- **RGB data:** Red, green, and blue pixel data. Usually 24 data lines for RGB888, 16 data lines for RGB666.


##### （2）、Typical timing parameters include

- **Horizontal Back Porch (HBP):** Number of PIXCLK pulses between HSYNC signal and the first valid pixel data.
- **Horizontal Front porch (HFP):** Number of PIXCLK pulses between the last valid pixel data in the line and the next HSYNC pulse.
- **Vertical Back Porch (VBP):** Number of lines (HSYNC pulses) from a VSYNC signal to the first valid line.
- **Vertical Front Porch (VFP):** Number of lines (HSYNC pulses) between the last valid line of the frame and the next VSYNC pulse.
- **VSYNC length:** Number of HSYNC pulses when a VSYNC signal is active.
- **HSYNC length:** Number of PIXCLK pulses when a HSYNC signal is active.
- **Active frame width (hactive):** Horizontal resolution.
- **Active frame height (vactive):** Vertical resolution.

![](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/zjj.sys.display.8.1.lcd/zjj.display.sys.dwg_lcd_display_signals.jpg)

Every manufacturer provides display timings in a slightly different way and some provide more detail than others. Most LCD panels work with a range of timing parameters.

This application note provides a couple of examples to help you further understand these concepts.
####  （四）、参考文档：

[（1）【How a Flat Screen Display works】](https://animagraffs.com/flat-screen-displays/)
[（2）【Adding a custom display】](https://www.digi.com/resources/documentation/digidocs/90001945-13/reference/yocto/r_an_adding_custom_display.htm) 
[（3）【Liquid Crystal Display Macro Example 】](https://commons.wikimedia.org/wiki/File:Liquid_Crystal_Display_Macro_Example_zoom_2.jpg)
[（4）【How-it-works: LCD screens explained】](https://pid.samsungdisplay.com/en/learning-center/blog/lcd-structure)
[（5）【显示面板技术说明 - LCD模式】](https://pid.samsungdisplay.cn/learning-center/white-papers/va-vs-ips)
[（6）【了解当今LCD屏技术】](https://pid.samsungdisplay.cn/learning-center/blog/lcd-structure)

