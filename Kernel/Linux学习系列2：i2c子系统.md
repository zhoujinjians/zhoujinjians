---
title: Linux学习系列2：i2c子系统
cover: https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.42.jpg
categories:
 - Linux
tags:
 - Linux
toc: true
abbrlink: 20220715
date: 2022-07-15 00:00:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Kernel-4.15**==）&&（==**文章基于 Android 10.0**==）

[【me.zhoujinjian.com博客原图链接】](https://github.com/zhoujinjianmax) 

[【开发板 RockPi4bPlusV1.6】](https://shop.allnetchina.cn/collections/frontpage/products/rock-pi-4-model-b-board-only-2-4-5ghz-wlan-bluetooth-5-0)

[【开发板 RockPi4bPlusV1.6 Android 10.0 && Linux（Kernel 4.15）源码链接】](https://github.com/radxa/manifests/tree/rockchip-android-10)



正是由于前人（各位大神）的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

 [【从零开始系列】](https://blog.csdn.net/qq_16777851/category_7923311.html)    

 [【Linux CLOCK子系统】](http://www.mysixue.com/?p=129)  




# 1、I2C协议

参考资料：

* i2c_spec.pdf

## 1.1. 硬件连接

I2C在硬件上的接法如下所示，主控芯片引出两条线SCL,SDA线，在一条I2C总线上可以接很多I2C设备，我们还会放一个上拉电阻（放一个上拉电阻的原因以后我们再说）。

![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/005_i2c_hardware_connect.png)

## 1.2. 传输数据类比

怎么通过I2C传输数据，我们需要把数据从主设备发送到从设备上去，也需要把数据从从设备传送到主设备上去，数据涉及到双向传输。

举个例子：

![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/006_teacher_and_student.png)

体育老师：可以把球发给学生，也可以把球从学生中接过来。

* 发球：
  * 老师：开始了(start)
  * 老师：A！我要发球给你！(地址/方向)
  * 学生A：到！(回应)
  * 老师把球发出去（传输）
  * A收到球之后，应该告诉老师一声（回应）
  * 老师：结束（停止）

* 接球：
  * 老师：开始了(start)
  * 老师：B！把球发给我！(地址/方向)
  * 学生B：到！
  * B把球发给老师（传输）
  * 老师收到球之后，给B说一声，表示收到球了（回应）
  * 老师：结束（停止）

我们就使用这个简单的例子，来解释一下IIC的传输协议：

* 老师说开始了，表示开始信号(start)
* 老师提醒某个学生要发球，表示发送地址和方向(address/read/write)
* 老师发球/接球，表示数据的传输
* 收到球要回应：回应信号(ACK)
* 老师说结束，表示IIC传输结束(P)



## 1.3. IIC传输数据的格式

### 1.3.1 写操作

流程如下：

* 主芯片要发出一个start信号
* 然后发出一个设备地址(用来确定是往哪一个芯片写数据)，方向(读/写，0表示写，1表示读)
* 从设备回应(用来确定这个设备是否存在)，然后就可以传输数据
* 主设备发送一个字节数据给从设备，并等待回应
* 每传输一字节数据，接收方要有一个回应信号（确定数据是否接受完成)，然后再传输下一个数据。
* 数据发送完之后，主芯片就会发送一个停止信号。

下图：白色背景表示"主→从"，灰色背景表示"从→主"

![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/007_i2c_write.png)

### 1.3.2 读操作

流程如下：

* 主芯片要发出一个start信号
* 然后发出一个设备地址(用来确定是往哪一个芯片写数据)，方向(读/写，0表示写，1表示读)
* 从设备回应(用来确定这个设备是否存在)，然后就可以传输数据

* 从设备发送一个字节数据给主设备，并等待回应
* 每传输一字节数据，接收方要有一个回应信号（确定数据是否接受完成)，然后再传输下一个数据。
* 数据发送完之后，主芯片就会发送一个停止信号。

下图：白色背景表示"主→从"，灰色背景表示"从→主"



![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/008_i2c_read.png)

### 1.3.3 I2C信号

I2C协议中数据传输的单位是字节，也就是8位。但是要用到9个时钟：前面8个时钟用来传输8数据，第9个时钟用来传输回应信号。传输时，先传输最高位(MSB)。

* 开始信号（S）：SCL为高电平时，SDA山高电平向低电平跳变，开始传送数据。
* 结束信号（P）：SCL为高电平时，SDA由低电平向高电平跳变，结束传送数据。
* 响应信号(ACK)：接收器在接收到8位数据后，在第9个时钟周期，拉低SDA
* SDA上传输的数据必须在SCL为高电平期间保持稳定，SDA上的数据只能在SCL为低电平期间变化

I2C协议信号如下：

![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/009_i2c_signal.png)

### 1.3.4 协议细节

* 如何在SDA上实现双向传输？
  主芯片通过一根SDA线既可以把数据发给从设备，也可以从SDA上读取数据，连接SDA线的引脚里面必然有两个引脚（发送引脚/接受引脚）。

* 主、从设备都可以通过SDA发送数据，肯定不能同时发送数据，怎么错开时间？
  在9个时钟里，
  前8个时钟由主设备发送数据的话，第9个时钟就由从设备发送数据；
  前8个时钟由从设备发送数据的话，第9个时钟就由主设备发送数据。
* 双方设备中，某个设备发送数据时，另一方怎样才能不影响SDA上的数据？
  设备的SDA中有一个三极管，使用开极/开漏电路(三极管是开极，CMOS管是开漏，作用一样)，如下图：
  ![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/010_i2c_signal_internal.png)

真值表如下：
![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/011_true_value_table.png)

从真值表和电路图我们可以知道：

* 当某一个芯片不想影响SDA线时，那就不驱动这个三极管
* 想让SDA输出高电平，双方都不驱动三极管(SDA通过上拉电阻变为高电平)
* 想让SDA输出低电平，就驱动三极管



从下面的例子可以看看数据是怎么传的（实现双向传输）。
举例：主设备发送（8bit）给从设备

* 前8个clk
  * 从设备不要影响SDA，从设备不驱动三极管
  * 主设备决定数据，主设备要发送1时不驱动三极管，要发送0时驱动三极管

* 第9个clk，由从设备决定数据
  * 主设备不驱动三极管
  * 从设备决定数据，要发出回应信号的话，就驱动三极管让SDA变为0
  * 从这里也可以知道ACK信号是低电平



从上面的例子，就可以知道怎样在一条线上实现双向传输，这就是SDA上要使用上拉电阻的原因。



为何SCL也要使用上拉电阻？
在第9个时钟之后，如果有某一方需要更多的时间来处理数据，它可以一直驱动三极管把SCL拉低。
当SCL为低电平时候，大家都不应该使用IIC总线，只有当SCL从低电平变为高电平的时候，IIC总线才能被使用。
当它就绪后，就可以不再驱动三极管，这是上拉电阻把SCL变为高电平，其他设备就可以继续使用I2C总线了。



对于IIC协议它只能规定怎么传输数据，数据是什么含义由从设备决定。

 

# 2、SMBus协议

参考资料：

* Linux内核文档：`Documentation\i2c\smbus-protocol.rst`

* SMBus协议：

  * http://www.smbus.org/specs/
* `SMBus_3_0_20141220.pdf`
* I2CTools: `https://mirrors.edge.kernel.org/pub/software/utils/i2c-tools/`

## 2.1. SMBus是I2C协议的一个子集

SMBus: System Management Bus，系统管理总线。
SMBus最初的目的是为智能电池、充电电池、其他微控制器之间的通信链路而定义的。
SMBus也被用来连接各种设备，包括电源相关设备，系统传感器，EEPROM通讯设备等等。
SMBus 为系统和电源管理这样的任务提供了一条控制总线，使用 SMBus 的系统，设备之间发送和接收消息都是通过 SMBus，而不是使用单独的控制线，这样可以节省设备的管脚数。
SMBus是基于I2C协议的，SMBus要求更严格，SMBus是I2C协议的子集。

![image-20210224093827621](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/017_i2c_and_smbus.png)

SMBus有哪些更严格的要求？跟一般的I2C协议有哪些差别？

* VDD的极限值不一样

  * I2C协议：范围很广，甚至讨论了高达12V的情况
  * SMBus：1.8V~5V

* 最小时钟频率、最大的`Clock Stretching `

  * Clock Stretching含义：某个设备需要更多时间进行内部的处理时，它可以把SCL拉低占住I2C总线

  * I2C协议：时钟频率最小值无限制，Clock Stretching时长也没有限制
  * SMBus：时钟频率最小值是10KHz，Clock Stretching的最大时间值也有限制

* 地址回应(Address Acknowledge)

  * 一个I2C设备接收到它的设备地址后，是否必须发出回应信号？
  * I2C协议：没有强制要求必须发出回应信号
  * SMBus：强制要求必须发出回应信号，这样对方才知道该设备的状态：busy，failed，或是被移除了

* SMBus协议明确了数据的传输格式
  * I2C协议：它只定义了怎么传输数据，但是并没有定义数据的格式，这完全由设备来定义
  * SMBus：定义了几种数据格式(后面分析)

* REPEATED START Condition(重复发出S信号)
  * 比如读EEPROM时，涉及2个操作：
    * 把存储地址发给设备
    * 读数据
  * 在写、读之间，可以不发出P信号，而是直接发出S信号：这个S信号就是`REPEATED START`
  * 如下图所示
    ![image-20210224100056055](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/018_repeated_start.png)
* SMBus Low Power Version 
  * SMBus也有低功耗的版本



## 2.2. SMBus协议分析

对于I2C协议，它只定义了怎么传输数据，但是并没有定义数据的格式，这完全由设备来定义。
对于SMBus协议，它定义了几种数据格式。



**注意**：

* 下面文档中的`Functionality flag`是Linux的某个I2C控制器驱动所支持的功能。
* 比如`Functionality flag: I2C_FUNC_SMBUS_QUICK`，表示需要I2C控制器支持`SMBus Quick Command`

### 2.2.1 symbols(符号)

```shell
S     (1 bit) : Start bit(开始位)
Sr    (1 bit) : 重复的开始位
P     (1 bit) : Stop bit(停止位)
R/W#  (1 bit) : Read/Write bit. Rd equals 1, Wr equals 0.(读写位)
A, N  (1 bit) : Accept and reverse accept bit.(回应位)
Address(7 bits): I2C 7 bit address. Note that this can be expanded as usual to
                get a 10 bit I2C address.
                (地址位，7位地址)
Command Code  (8 bits): Command byte, a data byte which often selects a register on
                the device.
                (命令字节，一般用来选择芯片内部的寄存器)
Data Byte (8 bits): A plain data byte. Sometimes, I write DataLow, DataHigh
                for 16 bit data.
                (数据字节，8位；如果是16位数据的话，用2个字节来表示：DataLow、DataHigh)
Count (8 bits): A data byte containing the length of a block operation.
				(在block操作总，表示数据长度)
[..]:           Data sent by I2C device, as opposed to data sent by the host
                adapter.
                (中括号表示I2C设备发送的数据，没有中括号表示host adapter发送的数据)
```



### 2.2.2 SMBus Quick Command

![image-20210224105224903](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/019_smbus_quick_command.png)

只是用来发送一位数据：R/W#本意是用来表示读或写，但是在SMBus里可以用来表示其他含义。
比如某些开关设备，可以根据这一位来决定是打开还是关闭。

```shell
Functionality flag: I2C_FUNC_SMBUS_QUICK
```



### 2.2.3 SMBus Receive Byte

![image-20210224113511225](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/020_smbus_receive_byte.png)

I2C-tools中的函数：i2c_smbus_read_byte()。
读取一个字节，Host adapter接收到一个字节后不需要发出回应信号(上图中N表示不回应)。

```shell
Functionality flag: I2C_FUNC_SMBUS_READ_BYTE
```



### 2.2.4 SMBus Send Byte

![image-20210224110638143](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/021_smbus_send_byte.png)

I2C-tools中的函数：i2c_smbus_write_byte()。
发送一个字节。

```shell
Functionality flag: I2C_FUNC_SMBUS_WRITE_BYTE
```



### 2.2.5 SMBus Read Byte

![image-20210224110812872](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/022_smbus_read_byte.png)

I2C-tools中的函数：i2c_smbus_read_byte_data()。

先发出`Command Code`(它一般表示芯片内部的寄存器地址)，再读取一个字节的数据。
上面介绍的`SMBus Receive Byte`是不发送Comand，直接读取数据。

```shell
Functionality flag: I2C_FUNC_SMBUS_READ_BYTE_DATA
```



### 2.2.6 SMBus Read Word

![image-20210224111404096](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/023_smbus_read_word.png)

I2C-tools中的函数：i2c_smbus_read_word_data()。

先发出`Command Code`(它一般表示芯片内部的寄存器地址)，再读取2个字节的数据。

```shell
Functionality flag: I2C_FUNC_SMBUS_READ_WORD_DATA
```



### 2.2.7 SMBus Write Byte

![image-20210224111542576](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/024_smbus_write_byte.png)

I2C-tools中的函数：i2c_smbus_write_byte_data()。

先发出`Command Code`(它一般表示芯片内部的寄存器地址)，再发出1个字节的数据。

```shell
Functionality flag: I2C_FUNC_SMBUS_WRITE_BYTE_DATA
```



### 2.2.8 SMBus Write Word

![image-20210224111840257](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/025_smbus_write_word.png)

I2C-tools中的函数：i2c_smbus_write_word_data()。

先发出`Command Code`(它一般表示芯片内部的寄存器地址)，再发出1个字节的数据。

```shell
Functionality flag: I2C_FUNC_SMBUS_WRITE_WORD_DATA
```



### 2.2.9 SMBus Block Read

![image-20210224112524185](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/026_smbus_block_read.png)

I2C-tools中的函数：i2c_smbus_read_block_data()。

先发出`Command Code`(它一般表示芯片内部的寄存器地址)，再发起度操作：

* 先读到一个字节(Block Count)，表示后续要读的字节数
* 然后读取全部数据

```shell
Functionality flag: I2C_FUNC_SMBUS_READ_BLOCK_DATA
```



### 2.2.10 SMBus Block Write

![image-20210224112629201](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/027_smbus_block_write.png)

I2C-tools中的函数：i2c_smbus_write_block_data()。

先发出`Command Code`(它一般表示芯片内部的寄存器地址)，再发出1个字节的`Byte Conut`(表示后续要发出的数据字节数)，最后发出全部数据。

```shell
Functionality flag: I2C_FUNC_SMBUS_WRITE_BLOCK_DATA
```



### 2.2.11 I2C Block Read

在一般的I2C协议中，也可以连续读出多个字节。
它跟`SMBus Block Read`的差别在于设备发出的第1个数据不是长度N，如下图所示：

![image-20210225094024082](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/033_i2c_block_read.png)

I2C-tools中的函数：i2c_smbus_read_i2c_block_data()。

先发出`Command Code`(它一般表示芯片内部的寄存器地址)，再发出1个字节的`Byte Conut`(表示后续要发出的数据字节数)，最后发出全部数据。

```shell
Functionality flag: I2C_FUNC_SMBUS_READ_I2C_BLOCK
```



### 2.2.12 I2C Block Write

在一般的I2C协议中，也可以连续发出多个字节。
它跟`SMBus Block Write`的差别在于发出的第1个数据不是长度N，如下图所示：

![image-20210225094359443](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/034_i2c_block_write.png)

I2C-tools中的函数：i2c_smbus_write_i2c_block_data()。

先发出`Command Code`(它一般表示芯片内部的寄存器地址)，再发出1个字节的`Byte Conut`(表示后续要发出的数据字节数)，最后发出全部数据。

```shell
Functionality flag: I2C_FUNC_SMBUS_WRITE_I2C_BLOCK
```



### 2.2.13 SMBus Block Write - Block Read Process Call

![image-20210224112940865](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/028_smbus_block_write_block_read_process_call.png)先写一块数据，再读一块数据。

```shell
Functionality flag: I2C_FUNC_SMBUS_BLOCK_PROC_CALL
```



### 2.2.14 Packet Error Checking (PEC)

PEC是一种错误校验码，如果使用PEC，那么在P信号之前，数据发送方要发送一个字节的PEC码(它是CRC-8码)。

以`SMBus Send Byte`为例，下图中，一个未使用PEC，另一个使用PEC：

![image-20210224113416249](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/029_smbus_pec_example.png)



## 2.3. SMBus和I2C的建议

因为很多设备都实现了SMBus，而不是更宽泛的I2C协议，所以优先使用SMBus。
即使I2C控制器没有实现SMBus，软件方面也是可以使用I2C协议来模拟SMBus。
所以：Linux建议优先使用SMBus。

# 3、I2C系统的重要结构体

参考资料：

* Linux驱动程序: `drivers/i2c/i2c-dev.c`
* I2CTools: `https://mirrors.edge.kernel.org/pub/software/utils/i2c-tools/`



## 3.1. I2C硬件框架

![image-20210208125100022](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/001_i2c_hardware_block.png)



## 3.2. I2C传输协议

* 写操作

![image-20210220150757825](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/007_i2c_write-16299579619801.png)

* 读操作

![image-20210220150954993](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/008_i2c_read-16299579619802.png)



## 3.3. Linux软件框架

![image-20210219173436295](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/003_linux_i2c_software_block.png)



## 3.4. 重要结构体

使用一句话概括I2C传输：APP通过I2C Controller与I2C Device传输数据。

在Linux中：

* 怎么表示I2C Controller

  * 一个芯片里可能有多个I2C Controller，比如第0个、第1个、……

  * 对于使用者，只要确定是第几个I2C Controller即可

  * 使用i2c_adapter表示一个I2C BUS，或称为I2C Controller

  * 里面有2个重要的成员：

    * nr：第几个I2C BUS(I2C Controller)

    * i2c_algorithm，里面有该I2C BUS的传输函数，用来收发I2C数据

  * i2c_adapter

  ![image-20210223103217183](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/012_i2c_adapter.png)

  * i2c_algorithm
    ![image-20210223100333813](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/013_i2c_algorithm.png)

* 怎么表示I2C Device

  * 一个I2C Device，一定有**设备地址**
  * 它连接在哪个I2C Controller上，即对应的i2c_adapter是什么
  * 使用i2c_client来表示一个I2C Device
    ![image-20210223100602285](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/014_i2c_client.png)

* 怎么表示要传输的数据

  * 在上面的i2c_algorithm结构体中可以看到要传输的数据被称为：i2c_msg

  * i2c_msg
    ![image-20210223100924756](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/015_i2c_msg.png)

  * i2c_msg中的flags用来表示传输方向：bit 0等于I2C_M_RD表示读，bit 0等于0表示写

  * 一个i2c_msg要么是读，要么是写

  * 举例：设备地址为0x50的EEPROM，要读取它里面存储地址为0x10的一个字节，应该构造几个i2c_msg？

    * 要构造2个i2c_msg

    * 第一个i2c_msg表示写操作，把要访问的存储地址0x10发给设备

    * 第二个i2c_msg表示读操作

    * 代码如下

      ```c
      u8 data_addr = 0x10;
      i8 data;
      struct i2c_msg msgs[2];
      
      msgs[0].addr   = 0x50;
      msgs[0].flags  = 0;
      msgs[0].len    = 1;
      msgs[0].buf    = &data_addr;
      
      msgs[1].addr   = 0x50;
      msgs[1].flags  = I2C_M_RD;
      msgs[1].len    = 1;
      msgs[1].buf    = &data;
      ```

## 3.5. 内核里怎么传输数据

使用一句话概括I2C传输：

* APP通过I2C Controller与I2C Device传输数据

* APP通过i2c_adapter与i2c_client传输i2c_msg

* 内核函数i2c_transfer

  * i2c_msg里含有addr，所以这个函数里不需要i2c_client

  ![image-20210223102320133](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/016_i2c_transfer.png)

# 4、无需编写驱动直接访问设备\_I2C-Tools介绍

Rockpi4B+ Android 10自带I2C-Tools。

APP访问硬件肯定是需要驱动程序的，
对于I2C设备，内核提供了驱动程序`drivers/i2c/i2c-dev.c`，通过它可以直接使用下面的I2C控制器驱动程序来访问I2C设备。
框架如下：

![image-20210224172517485](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/030_i2ctools_and_i2c_dev.png)

i2c-tools是一套好用的工具，也是一套示例代码。

## 4.1 用法(i2c-tools直接访问EEPROM)

* **i2cdetect：I2C检测**

```dtd
// 列出当前的I2C Adapter(或称为I2C Bus、I2C Controller)
i2cdetect -l

// 打印某个I2C Adapter的Functionalities, I2CBUS为0、1、2等整数
i2cdetect -F I2CBUS

// 看看有哪些I2C设备, I2CBUS为0、1、2等整数
i2cdetect -y -a I2CBUS

// 效果如下
rk3399_Android10:/ # i2cdetect -l
i2c-1   i2c             rk3x-i2c                                I2C Adapter
i2c-4   i2c             rk3x-i2c                                I2C Adapter
i2c-2   i2c             rk3x-i2c                                I2C Adapter
i2c-0   i2c             rk3x-i2c                                I2C Adapter
i2c-9   i2c             DesignWare HDMI                         I2C Adapter
i2c-7   i2c             rk3x-i2c                                I2C Adapter

# i2cdetect -F 0
rk3399_Android10:/ # i2cdetect -F 0
Functionalities implemented by /dev/i2c-0:
I2C                              yes
SMBus Quick Command              yes
SMBus Send Byte                  yes
SMBus Receive Byte               yes
SMBus Write Byte                 yes
SMBus Read Byte                  yes
SMBus Write Word                 yes
SMBus Read Word                  yes
SMBus Process Call               yes
SMBus Write Block                yes
SMBus Read Block                 no
SMBus Block Process Call         no
SMBus PEC                        yes
I2C Write Block                  yes
I2C Read Block                   yes

// --表示没有该地址对应的设备, UU表示有该设备并且它已经有驱动程序,
// 数值表示有该设备但是没有对应的设备驱动
rk3399_Android10:/ # i2cdetect -y -a 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- UU -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: UU UU -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: 50 51 -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
```

**i2cget：I2C读**
使用说明如下：

```shell
# i2cget
rk3399_Android10:/ # i2cget --help
usage: i2cget [-fy] BUS CHIP ADDR

Read an i2c register.

-f      Force access to busy devices
-y      Answer "yes" to confirmation prompts (for script use)
```



使用示例：

```dtd
rk3399_Android10:/ # i2cget -y  0 0x50 240
 0x69
```

**i2cset：I2C写**
使用说明如下：

```shell
rk3399_Android10:/ #      i2cset --help
usage: i2cset [-fy] BUS CHIP ADDR VALUE... MODE

Write an i2c register. MODE is b for byte, w for 16-bit word, i for I2C block.

-f      Force access to busy devices
-y      Answer "yes" to confirmation prompts (for script use)

```

使用示例：

```dtd
rk3399_Android10:/ # i2cset -y  0 0x50 240 0x69 b
```

**i2cdump**

```shell
rk3399_Android10:/ # i2cdump --help
usage: i2cdump [-fy] BUS CHIP

Dump i2c registers.

-f      Force access to busy devices
-y      Answer "yes" to confirmation prompts (for script use)
```

使用示例：

```dtd
rk3399_Android10:/ # i2cdump -y 0 0x50
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f    0123456789abcdef
00: 69 6d 65 72 6d 69 74 69 6d 2e 77 77 77 2e 68 65    imermitim.www.he
10: 72 6d 69 74 69 6d 2e 63 6f 6d 2d 2d 2d 2d 69 6d    rmitim.com----im
20: 65 72 6d 69 74 69 6d 2e 77 77 77 2e 68 65 72 6d    ermitim.www.herm
30: 69 74 69 6d 2e 63 6f 6d 2d 2d 2d 2d 69 6d 65 72    itim.com----imer
40: 6d 69 74 69 6d 2e 77 77 77 2e 68 65 72 6d 69 74    mitim.www.hermit
50: 69 6d 2e 63 6f 6d 2d 2d 2d 2d 69 6d 65 72 6d 69    im.com----imermi
60: 74 69 6d 2e 77 77 77 2e 68 65 72 6d 69 74 69 6d    tim.www.hermitim
70: 2e 63 6f 6d 2d 2d 2d 2d 69 6d 65 72 6d 69 74 69    .com----imermiti
80: 6d 2e 77 77 77 2e 68 65 72 6d 69 74 69 6d 2e 63    m.www.hermitim.c
90: 6f 6d 2d 2d 2d 2d 69 6d 65 72 6d 69 74 69 6d 2e    om----imermitim.
a0: 77 77 77 2e 68 65 72 6d 69 74 69 6d 2e 63 6f 6d    www.hermitim.com
b0: 2d 2d 2d 2d 69 6d 65 72 6d 69 74 69 6d 2e 77 77    ----imermitim.ww
c0: 77 2e 68 65 72 6d 69 74 69 6d 2e 63 6f 6d 2d 2d    w.hermitim.com--
d0: 2d 2d 69 6d 65 72 6d 69 74 69 6d 2e 77 77 77 2e    --imermitim.www.
e0: 68 65 72 6d 69 74 69 6d 2e 63 6f 6d 2d 2d 2d 2d    hermitim.com----
f0: 69 6d 65 72 6d 69 74 69 6d 2e 77 77 77 2e 68 65    imermitim.www.he
```

## 4.2. 源码分析

### 4.2.1 使用SMBus方式

示例代码：i2cget.c、i2cset.c



![032_i2cget_i2cset_flow](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/032_i2cget_i2cset_flow.png)

```c
[2021/8/23 11:40:08] [  144.059712] Hardware name: ROCKPI 4B (DT)
[2021/8/23 11:40:08] [  144.059719] Call trace:
[2021/8/23 11:40:08] [  144.059737]  dump_backtrace+0x0/0x178
[2021/8/23 11:40:08] [  144.059744]  show_stack+0x14/0x20
[2021/8/23 11:40:08] [  144.059752]  dump_stack+0x94/0xb4
[2021/8/23 11:40:08] [  144.059762]  rk3x_i2c_fill_transmit_buf+0x138/0x150
[2021/8/23 11:40:08] [  144.059769]  rk3x_i2c_xfer+0x3a8/0x460
[2021/8/23 11:40:08] [  144.059776]  __i2c_transfer+0x144/0x5a0
[2021/8/23 11:40:08] [  144.059783]  i2c_smbus_xfer_emulated+0x14c/0x5a8
[2021/8/23 11:40:08] [  144.059789]  __i2c_smbus_xfer+0xfc/0x438
[2021/8/23 11:40:08] [  144.059796]  i2c_smbus_xfer+0x64/0x98
[2021/8/23 11:40:08] [  144.059802]  i2cdev_ioctl_smbus+0xf8/0x2e0
[2021/8/23 11:40:08] [  144.059809]  i2cdev_ioctl+0x124/0x418
[2021/8/23 11:40:08] [  144.059817]  do_vfs_ioctl+0xbc/0xda8
[2021/8/23 11:40:08] [  144.059823]  ksys_ioctl+0x84/0x98
[2021/8/23 11:40:08] [  144.059829]  __arm64_sys_ioctl+0x18/0x28
[2021/8/23 11:40:08] [  144.059838]  el0_svc_common.constprop.0+0xb8/0x178
[2021/8/23 11:40:08] [  144.059845]  el0_svc_handler+0x28/0x78
[2021/8/23 11:40:08] [  144.059852]  el0_svc+0x8/0xc
[2021/8/23 11:40:08] [  144.059857] rk3x-i2c write:
[2021/8/23 11:40:16] [  144.059858]  69
[2021/8/23 11:40:16] [  151.871251] CPU: 4 PID: 1869 Comm: i2cget Not tainted 4.19.111 #16
[2021/8/23 11:40:16] [  151.871280] Hardware name: ROCKPI 4B (DT)
[2021/8/23 11:40:16] [  151.871286] Call trace:
[2021/8/23 11:40:16] [  151.871305]  dump_backtrace+0x0/0x178
[2021/8/23 11:40:16] [  151.871312]  show_stack+0x14/0x20
[2021/8/23 11:40:16] [  151.871320]  dump_stack+0x94/0xb4
[2021/8/23 11:40:16] [  151.871330]  rk3x_i2c_prepare_read+0xb0/0xc0
[2021/8/23 11:40:16] [  151.871337]  rk3x_i2c_xfer+0x378/0x460
[2021/8/23 11:40:16] [  151.871343]  __i2c_transfer+0x144/0x5a0
[2021/8/23 11:40:16] [  151.871350]  i2c_smbus_xfer_emulated+0x14c/0x5a8
[2021/8/23 11:40:16] [  151.871356]  __i2c_smbus_xfer+0xfc/0x438
[2021/8/23 11:40:16] [  151.871362]  i2c_smbus_xfer+0x64/0x98
[2021/8/23 11:40:16] [  151.871369]  i2cdev_ioctl_smbus+0xf8/0x2e0
[2021/8/23 11:40:16] [  151.871375]  i2cdev_ioctl+0x124/0x418
[2021/8/23 11:40:16] [  151.871384]  do_vfs_ioctl+0xbc/0xda8
[2021/8/23 11:40:16] [  151.871390]  ksys_ioctl+0x84/0x98
[2021/8/23 11:40:16] [  151.871396]  __arm64_sys_ioctl+0x18/0x28
[2021/8/23 11:40:16] [  151.871406]  el0_svc_common.constprop.0+0xb8/0x178
[2021/8/23 11:40:16] [  151.871413]  el0_svc_handler+0x28/0x78
[2021/8/23 11:40:16] [  151.871419]  el0_svc+0x8/0xc
[2021/8/23 11:40:16] [  151.871424] rk3x-i2c prepare read:
[2021/8/23 11:40:16] [  151.871588] CPU: 0 PID: 0 Comm: swapper/0 Not tainted 4.19.111 #16
[2021/8/23 11:40:16] [  151.871628] Hardware name: ROCKPI 4B (DT)
[2021/8/23 11:40:16] [  151.871640] Call trace:
[2021/8/23 11:40:16] [  151.871658]  dump_backtrace+0x0/0x178
[2021/8/23 11:40:16] [  151.871671]  show_stack+0x14/0x20
[2021/8/23 11:40:16] [  151.871680]  dump_stack+0x94/0xb4
[2021/8/23 11:40:16] [  151.871689]  rk3x_i2c_irq+0x31c/0x330
[2021/8/23 11:40:16] [  151.871704]  __handle_irq_event_percpu+0x6c/0x2a0
[2021/8/23 11:40:16] [  151.871717]  handle_irq_event_percpu+0x34/0x88
[2021/8/23 11:40:16] [  151.871730]  handle_irq_event+0x48/0x78
[2021/8/23 11:40:16] [  151.871744]  handle_fasteoi_irq+0xac/0x188
[2021/8/23 11:40:16] [  151.871756]  generic_handle_irq+0x24/0x38
[2021/8/23 11:40:16] [  151.871769]  __handle_domain_irq+0x5c/0xb8
[2021/8/23 11:40:16] [  151.871781]  gic_handle_irq+0xc4/0x178
[2021/8/23 11:40:16] [  151.871794]  el1_irq+0xe8/0x198
[2021/8/23 11:40:16] [  151.871808]  cpuidle_enter_state+0xb4/0x3c8
[2021/8/23 11:40:16] [  151.871820]  cpuidle_enter+0x18/0x20
[2021/8/23 11:40:16] [  151.871834]  call_cpuidle+0x18/0x30
[2021/8/23 11:40:16] [  151.871847]  do_idle+0x214/0x278
[2021/8/23 11:40:16] [  151.871860]  cpu_startup_entry+0x20/0x28
[2021/8/23 11:40:16] [  151.871874]  rest_init+0xcc/0xd8
[2021/8/23 11:40:16] [  151.871889]  start_kernel+0x4c0/0x4ec
[2021/8/23 11:40:16] [  151.871901] rk3x-i2c read:
[2021/8/23 11:40:52] [  151.871907]  69
```

# 5、通用驱动i2c-dev分析

参考资料：

* Linux驱动程序: `drivers/i2c/i2c-dev.c`
* I2C-Tools-4.2: `https://mirrors.edge.kernel.org/pub/software/utils/i2c-tools/`
* AT24cxx.pdf

## 5.1. 回顾字符设备驱动程序

![image-20210226154141671](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/045_char_driver.png)

怎么编写字符设备驱动程序？

* 确定主设备号
* 创建file_operations结构体
  * 在里面填充drv_open/drv_read/drv_ioctl等函数
* 注册file_operations结构体
  * register_chrdev(major, &fops, name)
* 谁调用register_chrdev？在入口函数调用
* 有入口自然就有出口
  * 在出口函数unregister_chrdev
* 辅助函数(帮助系统自动创建设备节点)
  * class_create
  * device_create



## 5.2. i2c-dev.c注册过程分析

 ### 5.2.1 register_chrdev的内部实现

  ![image-20210226163844390](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/046_register_chrdev_internal.png)



### 5.2.2 i2c-dev驱动的注册过程

![image-20210226164128588](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/047_i2c-dev_register.png)



## 5.3. file_operations函数分析

i2c-dev.c的核心：

```c
static const struct file_operations i2cdev_fops = {
	.owner		= THIS_MODULE,
	.llseek		= no_llseek,
	.read		= i2cdev_read,
	.write		= i2cdev_write,
	.unlocked_ioctl	= i2cdev_ioctl,
	.compat_ioctl	= compat_i2cdev_ioctl,
	.open		= i2cdev_open,
	.release	= i2cdev_release,
};
```

主要的系统调用：open, ioctl：
![image-20210226165250492](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/048_i2c-dev_interface.png)



要理解这些接口，记住一句话：APP通过I2C Controller与I2C Device传输数据。

### 3.13.1 i2cdev_open

![image-20210226170350844](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/049_i2cdev_open.png)



### 5.3.2 i2cdev_ioctl: I2C_SLAVE/I2C_SLAVE_FORCE

![image-20210226172800990](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/050_ioctl_I2C_SLAVE_FORCE.png)



### 5.3.3 i2cdev_ioctl: I2C_RDWR

![image-20210226173625871](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/051_ioctl_I2C_RDWR.png)

### 5.3.4 i2cdev_ioctl: I2C_SMBUS

![image-20210226173952800](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/052_ioctl_I2C_SMBUS.png)



### 5.3.5 总结

![image-20210226175142066](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/053_app_to_i2c_driver.png)

# 6、I2C系统驱动程序模型

参考资料：

* Linux内核文档:
  * `Documentation\i2c\instantiating-devices.rst`
  * `Documentation\i2c\writing-clients.rst`
* Linux内核驱动程序示例:
  * `drivers/eeprom/at24.c`

## 6.1. I2C驱动程序的层次

![image-20210227143624667](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/054_i2c_driver_layers.png)

I2C Core就是I2C核心层，它的作用：

* 提供统一的访问函数，比如i2c_transfer、i2c_smbus_xfer等
* 实现`I2C总线-设备-驱动模型`，管理：I2C设备(i2c_client)、I2C设备驱动(i2c_driver)、I2C控制器(i2c_adapter)



## 6.2. I2C总线-设备-驱动模型

![image-20210227151413993](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/055_i2c_bus_dev_drv.png)



## 6.2.1 i2c_driver

i2c_driver表明能支持哪些设备：

* 使用of_match_table来判断
  * 设备树中，某个I2C控制器节点下可以创建I2C设备的节点
    * 如果I2C设备节点的compatible属性跟of_match_table的某项兼容，则匹配成功
  * i2c_client.name跟某个of_match_table[i].compatible值相同，则匹配成功
* 使用id_table来判断
  * i2c_client.name跟某个id_table[i].name值相同，则匹配成功

i2c_driver跟i2c_client匹配成功后，就调用i2c_driver.probe函数。



## 6.2.2 i2c_client

i2c_client表示一个I2C设备，创建i2c_client的方法有4种：

* 方法1

  * 通过I2C bus number来创建

    ```c
    int i2c_register_board_info(int busnum, struct i2c_board_info const *info, unsigned len);
    ```

  * 通过设备树来创建

    ```shell
    F:\Rock4Plus_Android10\u-boot\arch\arm\dts\rk3399.dtsi
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
    F:\Rock4Plus_Android10\kernel\arch\arm64\boot\dts\rockchip\rk3399-rockpi-4b.dts
    &i2c0 {
    	status = "okay";
    	i2c-scl-rising-time-ns = <168>;
    	i2c-scl-falling-time-ns = <4>;
    	clock-frequency = <400000>;
    
    	......
        at24c08@50 {
            status = "okay";
            compatible = "atmel,24c08";
            reg = <0x50>;
            pagesize = <8>;
        };
    };
    
    rk3399_Android10:/sys/bus/i2c/devices/i2c-0/0-0050 # cat name
    24c08
    ```

* 方法2
  有时候无法知道该设备挂载哪个I2C bus下，无法知道它对应的I2C bus number。
  但是可以通过其他方法知道对应的i2c_adapter结构体。
  可以使用下面两个函数来创建i2c_client：

  * i2c_new_device

    ```c
      static struct i2c_board_info sfe4001_hwmon_info = {
    	I2C_BOARD_INFO("max6647", 0x4e),
      };
    
      int sfe4001_init(struct efx_nic *efx)
      {
    	(...)
    	efx->board_info.hwmon_client =
    		i2c_new_device(&efx->i2c_adap, &sfe4001_hwmon_info);
    
    	(...)
      }
    ```

    

  * i2c_new_probed_device

    ```c
      static const unsigned short normal_i2c[] = { 0x2c, 0x2d, I2C_CLIENT_END };
    
      static int usb_hcd_nxp_probe(struct platform_device *pdev)
      {
    	(...)
    	struct i2c_adapter *i2c_adap;
    	struct i2c_board_info i2c_info;
    
    	(...)
    	i2c_adap = i2c_get_adapter(2);
    	memset(&i2c_info, 0, sizeof(struct i2c_board_info));
    	strscpy(i2c_info.type, "isp1301_nxp", sizeof(i2c_info.type));
    	isp1301_i2c_client = i2c_new_probed_device(i2c_adap, &i2c_info,
    						   normal_i2c, NULL);
    	i2c_put_adapter(i2c_adap);
    	(...)
      }
    ```

  * 差别：

    * i2c_new_device：会创建i2c_client，即使该设备并不存在

    * i2c_new_probed_device：

      * 它成功的话，会创建i2c_client，并且表示这个设备肯定存在

      * I2C设备的地址可能发生变化，比如AT24C02的引脚A2A1A0电平不一样时，设备地址就不一样

      * 可以罗列出可能的地址

      * i2c_new_probed_device使用这些地址判断设备是否存在

        

        

* 方法3(不推荐)：由i2c_driver.detect函数来判断是否有对应的I2C设备并生成i2c_client

* 方法4：通过用户空间(user-space)生成
  调试时、或者不方便通过代码明确地生成i2c_client时，可以通过用户空间来生成。

```c
  // 创建一个i2c_client, .name = "eeprom", .addr=0x50, .adapter是i2c-3
  # echo eeprom 0x50 > /sys/bus/i2c/devices/i2c-3/new_device
  
  // 删除一个i2c_client
  # echo 0x50 > /sys/bus/i2c/devices/i2c-3/delete_device
```

  

# 7、编写设备驱动之i2c_driver

  ​      

## 7.1. 套路

## 7.1.1 I2C总线-设备-驱动模型

![image-20210227151413993](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/055_i2c_bus_dev_drv-16299605261355.png)



## 7.1.2 示例

分配、设置、注册一个i2c_driver结构体，类似`drivers/eeprom/at24.c`：
![image-20210301095132959](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/056_i2c_driver_at24.png)

在probe_new函数中，分配、设置、注册file_operations结构体。
在file_operations的函数中，使用i2c_transfer等函数发起I2C传输。

# 8. I2C_Adapter驱动框架

## 8.1 核心的结构体

### 8.1. i2c_adapter

  ![image-20210223103217183](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/012_i2c_adapter-16299607923736.png)

### 8.2. i2c_algorithm

![image-20210303121043020](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/058_i2c_algorithm.png)

* master_xfer：这是最重要的函数，它实现了一般的I2C传输，用来传输一个或多个i2c_msg

* master_xfer_atomic：

  * 可选的函数，功能跟master_xfer一样，在`atomic context`环境下使用
  * 比如在关机之前、所有中断都关闭的情况下，用来访问电源管理芯片

* smbus_xfer：实现SMBus传输，如果不提供这个函数，SMBus传输会使用master_xfer来模拟

* smbus_xfer_atomic：

  * 可选的函数，功能跟smbus_xfer一样，在`atomic context`环境下使用
  * 比如在关机之前、所有中断都关闭的情况下，用来访问电源管理芯片

* functionality：返回所支持的flags：各类I2C_FUNC_*

* reg_slave/unreg_slave：

  * 有些I2C Adapter也可工作与Slave模式，用来实现或模拟一个I2C设备

* reg_slave就是让把一个i2c_client注册到I2C Adapter，换句话说就是让这个I2C Adapter模拟该i2c_client

  * unreg_slave：反注册

  

  

### 8.3 驱动程序框架

分配、设置、注册一个i2c_adpater结构体：

* i2c_adpater的核心是i2c_algorithm
* i2c_algorithm的核心是master_xfer函数

#### 8.3.1. 所涉及的函数

* 分配

  ```c
  struct i2c_adpater *adap = kzalloc(sizeof(struct i2c_adpater), GFP_KERNEL);
  ```

* 设置

  ```c
  adap->owner = THIS_MODULE;
  adap->algo = &stm32f7_i2c_algo;
  ```

* 注册：i2c_add_adapter/i2c_add_numbered_adapter

  ```c
  ret = i2c_add_adapter(adap);          // 不管adap->nr原来是什么，都动态设置adap->nr
  ret = i2c_add_numbered_adapter(adap); // 如果adap->nr == -1 则动态分配nr; 否则使用该nr   
  ```

* 反注册

  ```c
  i2c_del_adapter(adap);
  ```

#### 8.3.2. i2c_algorithm示例

>  F:\Rock4Plus_Android10\kernel\drivers\i2c\busses\i2c-rk3x.c
>
> ![image-20210826145626299](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/image-20210826145626299.png)

# 9、具体芯片的I2C_Adapter驱动分析



### 1. I2C控制器内部结构

#### 通用的简化结构

![image-20210315101935654](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/069_i2c_block_general.png)



### 2. I2C控制器操作方法

* 使能时钟、设置时钟
* 发送数据：
  * 把数据写入**tx_register**，等待中断发生
  * 中断发生后，判断状态：是否发生错误、是否得到回应信号(ACK)
  * 把下一个数据写入**tx_register**，等待中断：如此循环
* 接收数据：
  * 设置**controller_register**，进入接收模式，启动接收，等待中断发生
  * 中断发生后，判断状态，读取**rx_register**得到数据
  * 如此循环

### 3. 分析代码

#### 3.1 设备树

* RK3399:  `F:\Rock4Plus_Android10\u-boot\arch\arm\dts\rk3399.dtsi`

  ```shell
  F:\Rock4Plus_Android10\u-boot\arch\arm\dts\rk3399.dtsi
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
  F:\Rock4Plus_Android10\kernel\arch\arm64\boot\dts\rockchip\rk3399-rockpi-4b.dts
  &i2c0 {
  	status = "okay";
  	i2c-scl-rising-time-ns = <168>;
  	i2c-scl-falling-time-ns = <4>;
  	clock-frequency = <400000>;
  
  	......
      at24c08@50 {
          status = "okay";
          compatible = "atmel,24c08";
          reg = <0x50>;
          pagesize = <8>;
      };
  };
  
  rk3399_Android10:/sys/bus/i2c/devices/i2c-0/0-0050 # cat name
  24c08
  ```

  


### 3.2 驱动程序分析

读I2C数据时，要先发出设备地址，这是写操作，然后再发起读操作，涉及写、读操作。所以以读I2C数据为例讲解核心代码。

* RK3399：函数`rk3x_i2c_xfer`分析：

  > F:\Rock4Plus_Android10\kernel\drivers\i2c\busses\i2c-rk3x.c

#### 3.2.1 读

![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/i2c-read.png)

```dtd
Log:
[2021/8/23 11:40:16] [  151.871286] Call trace:
[2021/8/23 11:40:16] [  151.871305]  dump_backtrace+0x0/0x178
[2021/8/23 11:40:16] [  151.871312]  show_stack+0x14/0x20
[2021/8/23 11:40:16] [  151.871320]  dump_stack+0x94/0xb4
[2021/8/23 11:40:16] [  151.871330]  rk3x_i2c_prepare_read+0xb0/0xc0
[2021/8/23 11:40:16] [  151.871337]  rk3x_i2c_xfer+0x378/0x460
[2021/8/23 11:40:16] [  151.871343]  __i2c_transfer+0x144/0x5a0
[2021/8/23 11:40:16] [  151.871350]  i2c_smbus_xfer_emulated+0x14c/0x5a8
[2021/8/23 11:40:16] [  151.871356]  __i2c_smbus_xfer+0xfc/0x438
[2021/8/23 11:40:16] [  151.871362]  i2c_smbus_xfer+0x64/0x98
[2021/8/23 11:40:16] [  151.871369]  i2cdev_ioctl_smbus+0xf8/0x2e0
[2021/8/23 11:40:16] [  151.871375]  i2cdev_ioctl+0x124/0x418
[2021/8/23 11:40:16] [  151.871384]  do_vfs_ioctl+0xbc/0xda8
[2021/8/23 11:40:16] [  151.871390]  ksys_ioctl+0x84/0x98
[2021/8/23 11:40:16] [  151.871396]  __arm64_sys_ioctl+0x18/0x28
[2021/8/23 11:40:16] [  151.871406]  el0_svc_common.constprop.0+0xb8/0x178
[2021/8/23 11:40:16] [  151.871413]  el0_svc_handler+0x28/0x78
[2021/8/23 11:40:16] [  151.871419]  el0_svc+0x8/0xc
[2021/8/23 11:40:16] [  151.871424] rk3x-i2c prepare read:

static void rk3x_i2c_handle_read(struct rk3x_i2c *i2c, unsigned int ipd)
{
	if(i2c->msg->addr == 0x50) {	
		dump_stack();
		printk("rk3x-i2c read:\n");
	}
}
//rk3x_i2c_handle_read：

[2021/8/23 11:40:16] [  151.871640] Call trace:
[2021/8/23 11:40:16] [  151.871658]  dump_backtrace+0x0/0x178
[2021/8/23 11:40:16] [  151.871671]  show_stack+0x14/0x20
[2021/8/23 11:40:16] [  151.871680]  dump_stack+0x94/0xb4
[2021/8/23 11:40:16] [  151.871689]  rk3x_i2c_irq+0x31c/0x330
[2021/8/23 11:40:16] [  151.871704]  __handle_irq_event_percpu+0x6c/0x2a0
[2021/8/23 11:40:16] [  151.871717]  handle_irq_event_percpu+0x34/0x88
[2021/8/23 11:40:16] [  151.871730]  handle_irq_event+0x48/0x78
[2021/8/23 11:40:16] [  151.871744]  handle_fasteoi_irq+0xac/0x188
[2021/8/23 11:40:16] [  151.871756]  generic_handle_irq+0x24/0x38
[2021/8/23 11:40:16] [  151.871769]  __handle_domain_irq+0x5c/0xb8
[2021/8/23 11:40:16] [  151.871781]  gic_handle_irq+0xc4/0x178
[2021/8/23 11:40:16] [  151.871794]  el1_irq+0xe8/0x198
[2021/8/23 11:40:16] [  151.871808]  cpuidle_enter_state+0xb4/0x3c8
[2021/8/23 11:40:16] [  151.871820]  cpuidle_enter+0x18/0x20
[2021/8/23 11:40:16] [  151.871834]  call_cpuidle+0x18/0x30
[2021/8/23 11:40:16] [  151.871847]  do_idle+0x214/0x278
[2021/8/23 11:40:16] [  151.871860]  cpu_startup_entry+0x20/0x28
[2021/8/23 11:40:16] [  151.871874]  rest_init+0xcc/0xd8
[2021/8/23 11:40:16] [  151.871889]  start_kernel+0x4c0/0x4ec
[2021/8/23 11:40:16] [  151.871901] rk3x-i2c read:
[2021/8/23 11:40:52] [  151.871907]  69
```



#### 3.2.2 写

![](https://raw.githubusercontent.com/zhoujinjianmax/PicGo/master/kernel.i2c/i2c-write.png)

```dtd
Log:
[2021/8/23 11:40:08] [  144.059737]  dump_backtrace+0x0/0x178
[2021/8/23 11:40:08] [  144.059744]  show_stack+0x14/0x20
[2021/8/23 11:40:08] [  144.059752]  dump_stack+0x94/0xb4
[2021/8/23 11:40:08] [  144.059762]  rk3x_i2c_fill_transmit_buf+0x138/0x150
[2021/8/23 11:40:08] [  144.059769]  rk3x_i2c_xfer+0x3a8/0x460
[2021/8/23 11:40:08] [  144.059776]  __i2c_transfer+0x144/0x5a0
[2021/8/23 11:40:08] [  144.059783]  i2c_smbus_xfer_emulated+0x14c/0x5a8
[2021/8/23 11:40:08] [  144.059789]  __i2c_smbus_xfer+0xfc/0x438
[2021/8/23 11:40:08] [  144.059796]  i2c_smbus_xfer+0x64/0x98
[2021/8/23 11:40:08] [  144.059802]  i2cdev_ioctl_smbus+0xf8/0x2e0
[2021/8/23 11:40:08] [  144.059809]  i2cdev_ioctl+0x124/0x418
[2021/8/23 11:40:08] [  144.059817]  do_vfs_ioctl+0xbc/0xda8
[2021/8/23 11:40:08] [  144.059823]  ksys_ioctl+0x84/0x98
[2021/8/23 11:40:08] [  144.059829]  __arm64_sys_ioctl+0x18/0x28
[2021/8/23 11:40:08] [  144.059838]  el0_svc_common.constprop.0+0xb8/0x178
[2021/8/23 11:40:08] [  144.059845]  el0_svc_handler+0x28/0x78
[2021/8/23 11:40:08] [  144.059852]  el0_svc+0x8/0xc
[2021/8/23 11:40:08] [  144.059857] rk3x-i2c write:
```

