---
title:  Android 10 Camera源码分析5：rkisp_demo分析
cover: https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.35.jpg
categories: 
  - Camera
tags:
  - Camera
toc: true
abbrlink: 20220301
date: 2022-03-01 03:01:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）
 [【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjiann) 
 [【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)
[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

 [V4L2框架-videobuf2](https://deepinout.com/v4l2-tutorials/linux-v4l2-architecture.html) 

--------------------------------------------------------------------------------
==源码（部分）==：
**F:\Khadas_Edge_Android_Q\hardware\rockchip\camera_engine_rkisp\tests\rkisp_demo\rkisp_demo.cpp**

-------------------------------------------------------------------------------

## （一）、V4L2 App流程简介
#### 1、打开video设备

在需要进行视频数据流的操作之前，首先要通过标准的字符设备操作接口open方法来打开一个video设备，并且将返回的字符句柄存在本地，之后的一系列操作都是基于该句柄，而在打开的过程中，会去给每一个子设备的上电，并完成各自的一系列初始化操作。

#### 2、查看并设置设备

在打开设备获取其文件句柄之后，就需要查询设备的属性，该动作主要通过ioctl传入VIDIOC_QUERYCAP参数来完成，其中该系列属性通过v4l2_capability结构体来表达，除此之外，还可以通过传入VIDIOC_ENUM_FMT来枚举支持的数据格式，通过传入VIDIOC_G_FMT/VIDIOC_S_FMT来分别获取和获取当前的数据格式，通过传入VIDIOC_G_PARM/VIDIOC_S_PARM来分别获取和设置参数。

#### 3、申请帧缓冲区

完成设备的配置之后，便可以开始向设备申请多个用于**盛装**图像数据的帧缓冲区，该动作通过调用ioctl并且传入VIDIOC_REQBUFS命令来完成，最后将缓冲区通过mmap方式映射到用户空间。

#### 4、将帧缓冲区入队

申请好帧缓冲区之后，通过调用ioctl方法传入VIDIOC_QBUF命令来将帧缓冲区加入到v4l2 框架中的缓冲区队列中，静等硬件模块将图像数据填充到缓冲区中。

#### 5、开启数据流

将所有的缓冲区都加入队列中之后便可以调用ioctl并且传入VIDIOC_STREAMON命令，来通知整个框架开始进行数据传输，其中大致包括了通知各个子设备开始进行工作，最终将数据填充到V4L2框架中的缓冲区队列中。

#### 6、将帧缓冲区出队

一旦数据流开始进行流转了，我们就可以通过调用ioctl下发VIDIOC_DQBUF命令来获取帧缓冲区，并且将缓冲区的图像数据取出，进行预览、拍照或者录像的处理，处理完成之后，需要将此次缓冲区再次放入V4L2框架中的队列中等待下次的图像数据的填充。

整个采集图像数据的流程现在看来还是比较简单的，接口的控制逻辑很清晰，主要原因是为了提供给用户的接口简单而且抽象，这样方便用户进行集成开发，其中的大部分复杂的业务处理都被V4L2很好的封装了，接下来我们来详细了解下V4L2框架内部是如何表达以及如何运转的。

![](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android.10.Camera.05/rkisp_demo_flow.png)


## （二）、rkisp_demo分析

#### 1、流程图
流程图根据rkisp_demo源码分析得出：
![](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android.10.Camera.05/V4L2-IOCTL.png)


#### 2、源码

``` c
F:\Khadas_Edge_Android_Q\hardware\rockchip\camera_engine_rkisp\tests\rkisp_demo\rkisp_demo.cpp
//VIDIOC_REQBUFS && mmap
static void init_mmap(void)
{
        struct v4l2_requestbuffers req;
        req.count = BUFFER_COUNT;
        req.type = buf_type;
        req.memory = V4L2_MEMORY_MMAP;

        if (-1 == xioctl(fd, VIDIOC_REQBUFS, &req)) {
                ......
        }
        ......
        buffers = (struct buffer*)calloc(req.count, sizeof(*buffers));
        ......
        for (n_buffers = 0; n_buffers < req.count; ++n_buffers) {
                struct v4l2_buffer buf;
                struct v4l2_plane planes[FMT_NUM_PLANES];
                CLEAR(buf);
                CLEAR(planes);

                buf.type = buf_type;
                buf.memory = V4L2_MEMORY_MMAP;
                buf.index = n_buffers;

                if (V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE == buf_type) {
                    buf.m.planes = planes;
                    buf.length = FMT_NUM_PLANES;
                }

                if (-1 == xioctl(fd, VIDIOC_QUERYBUF, &buf))
                        errno_exit("VIDIOC_QUERYBUF");

                if (V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE == buf_type) {
                    buffers[n_buffers].length = buf.m.planes[0].length;
                    buffers[n_buffers].start =
                        mmap(NULL /* start anywhere */,
                              buf.m.planes[0].length,
                              PROT_READ | PROT_WRITE /* required */,
                              MAP_SHARED /* recommended */,
                              fd, buf.m.planes[0].m.mem_offset);
                } else {
                    buffers[n_buffers].length = buf.length;
                    buffers[n_buffers].start =
                        mmap(NULL /* start anywhere */,
                              buf.length,
                              PROT_READ | PROT_WRITE /* required */,
                              MAP_SHARED /* recommended */,
                              fd, buf.m.offset);
                }

                if (MAP_FAILED == buffers[n_buffers].start)
                        errno_exit("mmap");
        }
}
//open device
static void open_device(void)
{
        fd = open(dev_name, O_RDWR /* required */ /*| O_NONBLOCK*/, 0);
......
}


static void init_device(void)
{
        struct v4l2_capability cap;
        struct v4l2_format fmt;
//VIDIOC_QUERYCAP
        if (-1 == xioctl(fd, VIDIOC_QUERYCAP, &cap)) {
               ........
        }
......
        if (cap.capabilities & V4L2_CAP_VIDEO_CAPTURE)
            buf_type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        else if (cap.capabilities & V4L2_CAP_VIDEO_CAPTURE_MPLANE)
            buf_type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;

        CLEAR(fmt);
        fmt.type = buf_type;
        fmt.fmt.pix.width = width;
        fmt.fmt.pix.height = height;
        fmt.fmt.pix.pixelformat = format;
        fmt.fmt.pix.field = V4L2_FIELD_INTERLACED;
        //VIDIOC_S_FMT
        if (-1 == xioctl(fd, VIDIOC_S_FMT, &fmt))
                errno_exit("VIDIOC_S_FMT");

        if (io == IO_METHOD_MMAP)
            init_mmap();
        else if (io == IO_METHOD_USERPTR)
            init_userp(fmt.fmt.pix.sizeimage, width, height);
        else if (io == IO_METHOD_DMABUF)
            init_dmabuf(fmt.fmt.pix.sizeimage, width, height);
    	......
}

static void start_capturing(void)
{
        unsigned int i;
        struct RKisp_media_ctl rkisp;
        enum v4l2_buf_type type;

    	.......
        for (i = 0; i < n_buffers; ++i) {
                struct v4l2_buffer buf;

                CLEAR(buf);
                buf.type = buf_type;
                if (io == IO_METHOD_MMAP)
                    buf.memory = V4L2_MEMORY_MMAP;
                else if (io == IO_METHOD_DMABUF)
                    buf.memory = V4L2_MEMORY_DMABUF;
                else
                    buf.memory = V4L2_MEMORY_USERPTR;
                buf.index = i;

                if (io == IO_METHOD_USERPTR) {
                    buf.m.userptr = (unsigned long)buffers[i].start;
                    buf.length = buffers[i].length;
                } else if (io == IO_METHOD_DMABUF) {
                    buf.m.fd = buffers[i].v4l2_buf.m.fd;
                    buf.length = buffers[i].length;
                }

                if (V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE == buf_type) {
                    struct v4l2_plane planes[FMT_NUM_PLANES];

                    if (io == IO_METHOD_USERPTR) {
                        planes[0].m.userptr = (unsigned long)buffers[i].start;
                        planes[0].length = buffers[i].length;
                    } else if (io == IO_METHOD_DMABUF) {
                        planes[0].m.fd = buffers[i].v4l2_buf.m.fd;
                        planes[0].length = buffers[i].length;
                    }
                    buf.m.planes = planes;
                    buf.length = FMT_NUM_PLANES;
                }
                //VIDIOC_QBUF
                if (-1 == xioctl(fd, VIDIOC_QBUF, &buf))
                        errno_exit("VIDIOC_QBUF");
        }
        type = buf_type;
        //VIDIOC_STREAMON
        if (-1 == xioctl(fd, VIDIOC_STREAMON, &type))
                errno_exit("VIDIOC_STREAMON");
}

static void mainloop(void)
{
        unsigned int count = frame_count;
        float exptime, expgain;
        int64_t frame_id, frame_sof;
......
        while (count-- > 0) {
            DBG("No.%d\n",frame_count - count);        //显示当前帧数目
            ......
            //循环读取camera 数据
            read_frame(fp);
        }
        DBG("\nREAD AND SAVE DONE!\n");
}
static void stop_capturing(void)
{
        enum v4l2_buf_type type;

    	if (_RKIspFunc.stop_func != NULL) {
    	    DBG("stop rkisp engine\n");
    	    _RKIspFunc.stop_func(_rkisp_engine);
    	}
        type = buf_type;
        if (-1 == xioctl(fd, VIDIOC_STREAMOFF, &type))
            errno_exit("VIDIOC_STREAMOFF");
}
static void uninit_device(void)
{
        unsigned int i;
        struct drm_mode_destroy_dumb destory_arg;

        for (i = 0; i < n_buffers; ++i) {
                if (-1 == munmap(buffers[i].start, buffers[i].length))
                        errno_exit("munmap");

                if (io == IO_METHOD_DMABUF) {
                        close(buffers[i].v4l2_buf.m.fd);
                        CLEAR(destory_arg);
                        destory_arg.handle = drm_handle;
                        drmIoctl(drm_fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destory_arg);
                }
        }

        free(buffers);

        if (_RKIspFunc.deinit_func != NULL) {
            _RKIspFunc.deinit_func(_rkisp_engine);
            deinit_3A_control_params();
        }

        dlclose(_RKIspFunc.rkisp_handle);
        if (drm_fd != -1)
            deinit_drm(drm_fd);
}

static void close_device(void)
{
        if (-1 == close(fd))
                errno_exit("close");
        fd = -1;
}

int main(int argc, char **argv)
{
        parse_args(argc, argv);
......
        open_device();
        init_device();
        start_capturing();
        mainloop();
        fclose(fp);
        stop_capturing();
        uninit_device();
        close_device();
        return 0;
}

```


## （三）、rkisp_demo运行效果

``` c
cd F:\Khadas_Edge_Android_Q\hardware\rockchip\camera_engine_rkisp\ 
mm 编译得到bin文件：rkisp_demo 
adb push out\target\product\rk3399_Android10\commit_id.xml /etc/
adb push out\target\product\rk3399_Android10\vendor\bin\rkisp_demo /vendor/bin/
运行：
adb shell
rkisp_demo -d /dev/video0 -w 1280 -h 960 -f NV21 --count 16 -o /data/misc/cameraserver/rk3399_1280_960_16.nv21
然后pull文件：
adb pull /data/misc/cameraserver/rk3399_1280_960_16.nv21
```
使用RawView打开(没经过算法优化感觉画质不好)：
![](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android.10.Camera.05/rkisp-demo-camera-data.png)
