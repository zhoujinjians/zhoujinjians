---
title:  Android 10 Camera源码分析3：videobuf2 申请与map、入队与出队
cover: https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.33.jpg
categories: 
  - Camera
tags:
  - Camera
toc: true
abbrlink: 20220201
date: 2022-02-01 02:01:00
---

注：文章都是通过阅读各位前辈总结的资料 Android 10.0 && Linux（Kernel 4.19）Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。
（==**文章基于 Kernel-4.19**==）&&（==**文章基于 Android 10.0**==）
 [【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjianm) 
 [【开发板 Khadas Edge V】](https://www.khadas.cn/edge-v)
[【开发板 Khadas Edge V Android 10.0 && Linux（Kernel 4.19）源码链接】](https://github.com/khadas/linux/tree/khadas-edge-Qt)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

 [V4L2框架-videobuf2](https://yellowmax.blog.csdn.net/article/details/81054611) 
 [camera linux v4l2相关接口](https://blog.csdn.net/weixin_43503508/category_10257454.html) 

--------------------------------------------------------------------------------
==源码（部分）==：
**F:\Khadas_Edge_Android_Q\kernel\include\uapi\linux\**
**F:\Khadas_Edge_Android_Q\kernel\drivers\phy\rockchip\**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\v4l2-core\**
**F:\Khadas_Edge_Android_Q\kernel\drivers\media\common\videobuf2\**

-------------------------------------------------------------------------------

videobuf2是嵌入到v4l2子系统，以供驱动与用户空间提供数据申请与交互的接口集合，它实现了包括buffer分配，根据状态可以入队出队的控制流。既然是buffer相关。一般分配有几种可能。

> 1、 buffer物理地址不连续，硬件上支持scatter/gather DMA，rkisp内部有iommu，可以实现分段dma。

---
> 2、 buffer 虚拟地址连续，物理不连续，也就是vmalloc操作，也很难使用dma

---
> 3、 物理地址连续，一般可以从cma内存申请(预留)。  
    分别对应文件videobuf2-dma-sg.c,videobuf2-vmalloc.c,videobuf2-dma-contig.c  
    核心文件是videobuf2-core.c，videobuf2-v4l2.c

---
其实我真正关心的是buffer的申请与返回。数据的流转，dma怎么设置。对于架构都是一样的，有关于VB2_buffer 的主要是数据的获取过程，步骤分析如下：

> 1、首先是==创建并初始化一个vb2_queue==结构体 ，

``` c
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\common.h
/* One structure per video node */
struct rkisp1_vdev_node {
	struct vb2_queue buf_queue;
	struct video_device vdev;
	struct media_pad pad;
};
F:\Khadas_Edge_Android_Q\kernel\drivers\media\platform\rockchip\isp1\capture.c
static int rkisp_init_vb2_queue(struct vb2_queue *q,
				struct rkisp1_stream *stream,
				enum v4l2_buf_type buf_type)
{
	q->type = buf_type;  // 类型
	q->io_modes = VB2_MMAP | VB2_USERPTR | VB2_DMABUF; //该队列支持的模式
	q->drv_priv = stream;  // 自定义模式
	q->ops = &rkisp1_vb2_ops;
	q->mem_ops = &vb2_dma_contig_memops;
	q->buf_struct_size = sizeof(struct rkisp1_buffer);// 将vb2_buffer结构体封装到我们自己的buffer中，此为我们自己的buffer的size
	q->min_buffers_needed = CIF_ISP_REQ_BUFS_MIN;
	q->timestamp_flags = V4L2_BUF_FLAG_TIMESTAMP_MONOTONIC;
	q->lock = &stream->ispdev->apilock;
	q->dev = stream->ispdev->dev;
	printk("zjj.android10.kernel.camera || %s %s %d \n",
		__FUNCTION__, __FILE__, __LINE__); //zhoujinjian

	return vb2_queue_init(q);
}
//其中rkisp1_vb2_ops 是队列的操作函数，以后的REQBUFS 等请求会调用到此中的函数，其结构如下
static struct vb2_ops rkisp1_vb2_ops = {
	.queue_setup = rkisp1_queue_setup,// 必须有，vb2_queue_init中会判断
	.buf_queue = rkisp1_buf_queue, // 必须有，vb2_queue_init中会判断
	.wait_prepare = vb2_ops_wait_prepare,
	.wait_finish = vb2_ops_wait_finish,
	.stop_streaming = rkisp1_stop_streaming,
	.start_streaming = rkisp1_start_streaming,
};

//其中vb2_vmalloc_memops用于有关此队列中mem分配问题，其中的函数一般不需要我们自己写，使用默认
F:\Khadas_Edge_Android_Q\kernel\drivers\media\common\videobuf2\videobuf2-dma-contig.c
const struct vb2_mem_ops vb2_dma_contig_memops = {
	.alloc		= vb2_dc_alloc,
	.put		= vb2_dc_put,
	.get_dmabuf	= vb2_dc_get_dmabuf,
	.cookie		= vb2_dc_cookie,
	.vaddr		= vb2_dc_vaddr,
	.mmap		= vb2_dc_mmap,
	.get_userptr	= vb2_dc_get_userptr,
	.put_userptr	= vb2_dc_put_userptr,
	.prepare	= vb2_dc_prepare,
	.finish		= vb2_dc_finish,
	.map_dmabuf	= vb2_dc_map_dmabuf,
	.unmap_dmabuf	= vb2_dc_unmap_dmabuf,
	.attach_dmabuf	= vb2_dc_attach_dmabuf,
	.detach_dmabuf	= vb2_dc_detach_dmabuf,
	.num_users	= vb2_dc_num_users,
};
 
```

----

> 2、初始化完成后，调用==VIDIOC_REQBUFS== 请求系统分配缓冲区，我们使用的是V4L2_MEMORY_MMAP类型,函数的调用过程如下.
``` c
v4l_reqbufs
    vb2_ioctl_reqbufs
		vb2_core_reqbufs
			__vb2_queue_alloc
				vb2_dc_alloc
					__iommu_alloc_attrs
```

----

> 3、查询映射缓冲区==VIDIOC_QUERYBUF==返回实际上分配到的buffer，
      查询分配好的缓存区，返回v4l2_buffer结构，设置vb->state 

----

> 4、使用==mmap==
    vb2_mmap ==》q->mem_ops->mmap   即  vb2_vmalloc_mmap  用于映射，将上面分配好的vb2_buffer->planes[0].mem_priv指向的空间重映射到mmap参数中的用户空间

----

> 5、把缓冲区放入队列==VIDIOC_QBUF==   
        vb2_qbuf    将 list_add_tail(&vb->queued_entry, &q->queued_list);    将vb2_buffer 放入队列q的queued_list中
        设置vb->state = VB2_BUF_STATE_PREPARED;

----

> 6、启动摄像头==VIDIOC_STREAMON==
    vb2_streamon 
        q->streaming = 1;   
// 如果 q->queued_list 中部位空，即有qbuf没有被处理 调用__enqueue_in_driver （）

----

> 7、 用select查询是否有数据  会调用==poll==函数   
       vb2_poll   等待 q->done_list 中有数据，   

----

> 8、怎样往 q->done_list 中添加数据呢 ？
每次调用qbuf 和 vidioc_streamon 时候都会查询，如果这两个条件都成立，则调用q->ops->buf_queue 将 核心中的vb2_buffer调如我们写的驱动中，放入一个列表， 在RKISP1中 周期性的调用函数向这个列表中的vb缓冲区中添加数据 即 向vb2_buffer->planes中添加数据 ，然后后调用  ==vb2_buffer_done==(&vb, VB2_BUF_STATE_DONE);  ==》 list_add_tail(&vb->done_entry, &q->done_list);  将RKISP1驱动中的vb2 放入 q->done_list中 ，然后设置vb->state = VB2_BUF_STATE_DONE;最后wake_up(&q->done_wq);  唤醒poll中休眠的进程。

----

> 9、调用==VIDIOC_DQBUF==从队列里取出缓冲区
 vb2_dqbuf  ==> __vb2_get_done_vb  将q->done_list 中的vb2_buffer中提出来，然后 将vb2_buffer中的v4l2_buffer信息返回，并将其从q->done_list 中删除

----

> 10，应用程序将数据取出来（mmap的空间）

### （一）、v4l2 ioctl堆栈信息（以RKISP1为例）
#### 1、fd = open(/dev/video0, O_RDWR, 0)

``` c
[2021/6/17 9:36:30] [   10.216308] Call trace:
[2021/6/17 9:36:30] [   10.216330]  dump_backtrace+0x0/0x178
[2021/6/17 9:36:30] [   10.216334]  show_stack+0x14/0x20
[2021/6/17 9:36:30] [   10.216341]  dump_stack+0x94/0xb4
[2021/6/17 9:36:30] [   10.216347]  rkisp1_fh_open+0x48/0xa0
[2021/6/17 9:36:30] [   10.216352]  v4l2_open+0xd0/0x150
[2021/6/17 9:36:30] [   10.216357]  chrdev_open+0xb8/0x1f8
[2021/6/17 9:36:30] [   10.216361]  do_dentry_open+0x228/0x368
[2021/6/17 9:36:30] [   10.216364]  vfs_open+0x28/0x30
[2021/6/17 9:36:30] [   10.216368]  path_openat+0x5c4/0xe80
[2021/6/17 9:36:30] [   10.216370]  do_filp_open+0x74/0xf8
[2021/6/17 9:36:30] [   10.216374]  do_sys_open+0x154/0x260
[2021/6/17 9:36:30] [   10.216378]  __arm64_compat_sys_openat+0x1c/0x28
[2021/6/17 9:36:30] [   10.216384]  el0_svc_common.constprop.0+0xb8/0x178
[2021/6/17 9:36:30] [   10.216387]  el0_svc_compat_handler+0x18/0x20
[2021/6/17 9:36:30] [   10.216390]  el0_svc_compat+0x8/0x34

```

####  2、ioctl(fd, VIDIOC_QUERYCAP, &cap)

``` c
[2021/6/17 9:36:30] [   10.328515] Call trace:
[2021/6/17 9:36:30] [   10.328535]  dump_backtrace+0x0/0x178
[2021/6/17 9:36:30] [   10.328543]  show_stack+0x14/0x20
[2021/6/17 9:36:30] [   10.328552]  dump_stack+0x94/0xb4
[2021/6/17 9:36:30] [   10.328566]  rkisp1_querycap+0x5c/0xd0
[2021/6/17 9:36:30] [   10.328576]  v4l_querycap+0x60/0xc8
[2021/6/17 9:36:30] [   10.328585]  __video_do_ioctl+0x1a0/0x348
[2021/6/17 9:36:30] [   10.328595]  video_usercopy+0x390/0x730
[2021/6/17 9:36:30] [   10.328600]  video_ioctl2+0x14/0x20
[2021/6/17 9:36:30] [   10.328607]  v4l2_ioctl+0x80/0xb0
[2021/6/17 9:36:30] [   10.328615]  v4l2_compat_ioctl32+0x504/0x3a48
[2021/6/17 9:36:30] [   10.328623]  __arm64_compat_sys_ioctl+0xbc/0x15b0
[2021/6/17 9:36:30] [   10.328635]  el0_svc_common.constprop.0+0xb8/0x178
[2021/6/17 9:36:30] [   10.328644]  el0_svc_compat_handler+0x18/0x20
[2021/6/17 9:36:30] [   10.328653]  el0_svc_compat+0x8/0x34
```


####  3、ioctl(fd, VIDIOC_S_FMT, &fmt)

``` c
[2021/6/16 13:39:33] [   82.481336] Call trace:
[2021/6/16 13:39:33] [   82.481348]  dump_backtrace+0x0/0x178
[2021/6/16 13:39:33] [   82.481357]  show_stack+0x14/0x20
[2021/6/16 13:39:33] [   82.481367]  dump_stack+0x94/0xb4
[2021/6/16 13:39:33] [   82.481378]  rkisp1_set_fmt+0x70/0x380
[2021/6/16 13:39:33] [   82.481388]  rkisp1_s_fmt_vid_cap_mplane+0x5c/0x88
[2021/6/16 13:39:33] [   82.481397]  v4l_s_fmt+0x2a4/0x518
[2021/6/16 13:39:33] [   82.481407]  __video_do_ioctl+0x1a0/0x348
[2021/6/16 13:39:33] [   82.481416]  video_usercopy+0x390/0x730
[2021/6/16 13:39:33] [   82.481425]  video_ioctl2+0x14/0x20
[2021/6/16 13:39:33] [   82.481435]  v4l2_ioctl+0x44/0x68
[2021/6/16 13:39:33] [   82.481445]  v4l2_compat_ioctl32+0x1d0/0x3a48
[2021/6/16 13:39:33] [   82.481455]  __arm64_compat_sys_ioctl+0xbc/0x15b0
[2021/6/16 13:39:33] [   82.481466]  el0_svc_common.constprop.0+0xb8/0x178
[2021/6/16 13:39:33] [   82.481476]  el0_svc_compat_handler+0x18/0x20
[2021/6/16 13:39:33] [   82.481485]  el0_svc_compat+0x8/0x34
```


####  4、ioctl(fd, VIDIOC_REQBUFS, &req)

``` c
（1）：
[2021/6/16 13:39:33] [   82.483800] Call trace:
[2021/6/16 13:39:33] [   82.483811]  dump_backtrace+0x0/0x178
[2021/6/16 13:39:33] [   82.483818]  show_stack+0x14/0x20
[2021/6/16 13:39:33] [   82.483825]  dump_stack+0x94/0xb4
[2021/6/16 13:39:33] [   82.483833]  rkisp1_queue_setup+0x54/0xe0
[2021/6/16 13:39:33] [   82.483841]  vb2_core_reqbufs+0x104/0x388
[2021/6/16 13:39:33] [   82.483849]  vb2_ioctl_reqbufs+0x70/0xa0
[2021/6/16 13:39:33] [   82.483855]  v4l_reqbufs+0x48/0x58
[2021/6/16 13:39:33] [   82.483863]  __video_do_ioctl+0x1a0/0x348
[2021/6/16 13:39:33] [   82.483869]  video_usercopy+0x390/0x730
[2021/6/16 13:39:33] [   82.483875]  video_ioctl2+0x14/0x20
[2021/6/16 13:39:33] [   82.483882]  v4l2_ioctl+0x44/0x68
[2021/6/16 13:39:33] [   82.483890]  v4l2_compat_ioctl32+0x504/0x3a48
[2021/6/16 13:39:33] [   82.483898]  __arm64_compat_sys_ioctl+0xbc/0x15b0
[2021/6/16 13:39:33] [   82.483907]  el0_svc_common.constprop.0+0xb8/0x178
[2021/6/16 13:39:33] [   82.483913]  el0_svc_compat_handler+0x18/0x20
[2021/6/16 13:39:33] [   82.483920]  el0_svc_compat+0x8/0x34
（2）：
[2021/6/16 13:39:33] [   82.485800] Call trace:
[2021/6/16 13:39:33] [   82.485814]  dump_backtrace+0x0/0x178
[2021/6/16 13:39:33] [   82.485828]  show_stack+0x14/0x20
[2021/6/16 13:39:33] [   82.485843]  dump_stack+0x94/0xb4
[2021/6/16 13:39:33] [   82.485860]  rk_iommu_zap_iova+0x58/0x168
[2021/6/16 13:39:33] [   82.485873]  rk_iommu_zap_iova_first_last+0x24/0x50
[2021/6/16 13:39:33] [   82.485887]  rk_iommu_map+0x3d0/0x4f0
[2021/6/16 13:39:33] [   82.485899]  iommu_map+0x210/0x290
[2021/6/16 13:39:33] [   82.485911]  iommu_map_sg+0xfc/0x180
[2021/6/16 13:39:33] [   82.485923]  iommu_dma_alloc+0x214/0x388
[2021/6/16 13:39:33] [   82.485937]  __iommu_alloc_attrs+0x270/0x3e8
[2021/6/16 13:39:33] [   82.485952]  vb2_dc_alloc+0x190/0x1e0
[2021/6/16 13:39:33] [   82.485966]  __vb2_queue_alloc+0x30c/0x3e0
[2021/6/16 13:39:33] [   82.485983]  vb2_core_reqbufs+0x204/0x388
[2021/6/16 13:39:33] [   82.485996]  vb2_ioctl_reqbufs+0x70/0xa0
[2021/6/16 13:39:33] [   82.486009]  v4l_reqbufs+0x48/0x58
[2021/6/16 13:39:33] [   82.486024]  __video_do_ioctl+0x1a0/0x348
[2021/6/16 13:39:33] [   82.486036]  video_usercopy+0x390/0x730
[2021/6/16 13:39:33] [   82.486045]  video_ioctl2+0x14/0x20
[2021/6/16 13:39:33] [   82.486053]  v4l2_ioctl+0x44/0x68
[2021/6/16 13:39:33] [   82.486063]  v4l2_compat_ioctl32+0x504/0x3a48
[2021/6/16 13:39:33] [   82.486075]  __arm64_compat_sys_ioctl+0xbc/0x15b0
[2021/6/16 13:39:33] [   82.486087]  el0_svc_common.constprop.0+0xb8/0x178
[2021/6/16 13:39:33] [   82.486100]  el0_svc_compat_handler+0x18/0x20
[2021/6/16 13:39:33] [   82.486110]  el0_svc_compat+0x8/0x34
```

####  5、ioctl(fd, VIDIOC_QUERYBUF, &req)

``` c
[2021/6/17 11:13:49] [  126.705932] Call trace:
[2021/6/17 11:13:49] [  126.705973]  dump_backtrace+0x0/0x178
[2021/6/17 11:13:49] [  126.705996]  show_stack+0x14/0x20
[2021/6/17 11:13:49] [  126.706021]  dump_stack+0x94/0xb4
[2021/6/17 11:13:49] [  126.706058]  vb2_buffer_in_use+0x40/0xb8
[2021/6/17 11:13:49] [  126.706082]  __fill_v4l2_buffer+0x130/0x288
[2021/6/17 11:13:49] [  126.706115]  vb2_core_querybuf+0x30/0x40
[2021/6/17 11:13:49] [  126.706155]  vb2_querybuf+0x88/0xd0
[2021/6/17 11:13:49] [  126.706187]  vb2_ioctl_querybuf+0x20/0x30
[2021/6/17 11:13:49] [  126.706225]  v4l_querybuf+0x44/0x58
[2021/6/17 11:13:49] [  126.706245]  __video_do_ioctl+0x1a0/0x348
[2021/6/17 11:13:49] [  126.706283]  video_usercopy+0x224/0x730
[2021/6/17 11:13:49] [  126.706303]  video_ioctl2+0x14/0x20
[2021/6/17 11:13:49] [  126.706337]  v4l2_ioctl+0x80/0xb0
[2021/6/17 11:13:49] [  126.706368]  v4l2_compat_ioctl32+0x1d0/0x3a48
[2021/6/17 11:13:49] [  126.706407]  __arm64_compat_sys_ioctl+0xbc/0x15b0
[2021/6/17 11:13:49] [  126.706444]  el0_svc_common.constprop.0+0xb8/0x178
[2021/6/17 11:13:49] [  126.706479]  el0_svc_compat_handler+0x18/0x20
[2021/6/17 11:13:49] [  126.706501]  el0_svc_compat+0x8/0x34
```

####  6、mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_SHARED, fd, m.mem_offset)

``` c
[2021/6/17 11:13:45] [  122.373183] Call trace:
[2021/6/17 11:13:45] [  122.373215]  dump_backtrace+0x0/0x178
[2021/6/17 11:13:45] [  122.373231]  show_stack+0x14/0x20
[2021/6/17 11:13:45] [  122.373250]  dump_stack+0x94/0xb4
[2021/6/17 11:13:45] [  122.373282]  vb2_vmalloc_mmap+0x3c/0xc0
[2021/6/17 11:13:45] [  122.373298]  vb2_mmap+0x174/0x268
[2021/6/17 11:13:45] [  122.373323]  vb2_fop_mmap+0x20/0x30
[2021/6/17 11:13:45] [  122.373341]  v4l2_mmap+0x9c/0xd8
[2021/6/17 11:13:45] [  122.373370]  mmap_region+0x364/0x500
[2021/6/17 11:13:45] [  122.373399]  do_mmap+0x2cc/0x418
[2021/6/17 11:13:45] [  122.373420]  vm_mmap_pgoff+0xe4/0x110
[2021/6/17 11:13:45] [  122.373449]  ksys_mmap_pgoff+0xbc/0xf0
[2021/6/17 11:13:45] [  122.373484]  __arm64_compat_sys_aarch32_mmap2+0x1c/0x28
[2021/6/17 11:13:45] [  122.373505]  el0_svc_common.constprop.0+0xb8/0x178
[2021/6/17 11:13:45] [  122.373524]  el0_svc_compat_handler+0x18/0x20
[2021/6/17 11:13:45] [  122.373547]  el0_svc_compat+0x8/0x34
```

####  7、ioctl(fd, VIDIOC_QBUF, &buf)

``` c
[2021/6/16 13:39:34] [   83.502204] Call trace:
[2021/6/16 13:39:34] [   83.502236]  dump_backtrace+0x0/0x178
[2021/6/16 13:39:34] [   83.502248]  show_stack+0x14/0x20
[2021/6/16 13:39:34] [   83.502267]  dump_stack+0x94/0xb4
[2021/6/16 13:39:34] [   83.502285]  rkisp1_buf_queue+0x54/0x250
[2021/6/16 13:39:34] [   83.502300]  __enqueue_in_driver+0x48/0xf0
[2021/6/16 13:39:34] [   83.502312]  vb2_core_qbuf+0x258/0x280
[2021/6/16 13:39:34] [   83.502323]  vb2_qbuf+0x3c/0x68
[2021/6/16 13:39:34] [   83.502334]  vb2_ioctl_qbuf+0x48/0x58
[2021/6/16 13:39:34] [   83.502347]  v4l_qbuf+0x44/0x58
[2021/6/16 13:39:34] [   83.502360]  __video_do_ioctl+0x1a0/0x348
[2021/6/16 13:39:34] [   83.502370]  video_usercopy+0x224/0x730
[2021/6/16 13:39:34] [   83.502379]  video_ioctl2+0x14/0x20
[2021/6/16 13:39:34] [   83.502394]  v4l2_ioctl+0x44/0x68
[2021/6/16 13:39:34] [   83.502409]  v4l2_compat_ioctl32+0x1d0/0x3a48
[2021/6/16 13:39:34] [   83.502427]  __arm64_compat_sys_ioctl+0xbc/0x15b0
[2021/6/16 13:39:34] [   83.502463]  el0_svc_compat+0x8/0x34
```



####  8、ioctl(fd, VIDIOC_STREAMON, &type)

``` c
[2021/6/11 10:37:23] [  317.817415]  dump_backtrace+0x0/0x178
[2021/6/11 10:37:23] [  317.817420]  show_stack+0x14/0x20
[2021/6/11 10:37:23] [  317.817432]  dump_stack+0x94/0xb4
[2021/6/11 10:37:23] [  317.817439]  rkisp1_start_streaming+0x64/0x7d0
[2021/6/11 10:37:23] [  317.817447]  vb2_start_streaming+0x68/0x148
[2021/6/11 10:37:23] [  317.817452]  vb2_core_streamon+0x60/0x110
[2021/6/11 10:37:23] [  317.817459]  vb2_streamon+0x14/0x40
[2021/6/11 10:37:23] [  317.817464]  vb2_ioctl_streamon+0x48/0x58
[2021/6/11 10:37:23] [  317.817473]  v4l_streamon+0x20/0x28
[2021/6/11 10:37:23] [  317.817479]  __video_do_ioctl+0x1a0/0x348
[2021/6/11 10:37:23] [  317.817486]  video_usercopy+0x390/0x730
[2021/6/11 10:37:23] [  317.817503]  video_ioctl2+0x14/0x20
[2021/6/11 10:37:23] [  317.817517]  v4l2_ioctl+0x44/0x68
[2021/6/11 10:37:23] [  317.817533]  v4l2_compat_ioctl32+0x1d0/0x3a48
[2021/6/11 10:37:23] [  317.817549]  __arm64_compat_sys_ioctl+0xbc/0x15b0
[2021/6/11 10:37:23] [  317.817564]  el0_svc_common.constprop.0+0xb8/0x178
[2021/6/11 10:37:23] [  317.817579]  el0_svc_compat_handler+0x18/0x20
[2021/6/11 10:37:23] [  317.817593]  el0_svc_compat+0x8/0x34
```



####  9、ioctl(fd, VIDIOC_DQBUF, &buf)

``` c
(1):
[2021/6/16 13:39:36] [   85.392500] Call trace:
[2021/6/16 13:39:36] [   85.392521]  dump_backtrace+0x0/0x178
[2021/6/16 13:39:36] [   85.392534]  show_stack+0x14/0x20
[2021/6/16 13:39:36] [   85.392549]  dump_stack+0x94/0xb4
[2021/6/16 13:39:36] [   85.392564]  vb2_buffer_done+0x25c/0x2c0
[2021/6/16 13:39:36] [   85.392578]  rkisp1_params_isr+0x11c/0x150
[2021/6/16 13:39:36] [   85.392591]  rkisp1_isp_isr+0x2f4/0x4e8
[2021/6/16 13:39:36] [   85.392601]  rkisp1_irq_handler+0xd0/0xd8
[2021/6/16 13:39:36] [   85.392617]  __handle_irq_event_percpu+0x6c/0x2a0
[2021/6/16 13:39:36] [   85.392627]  handle_irq_event_percpu+0x34/0x88
[2021/6/16 13:39:36] [   85.392637]  handle_irq_event+0x48/0x78
[2021/6/16 13:39:36] [   85.392651]  handle_fasteoi_irq+0xac/0x188
[2021/6/16 13:39:36] [   85.392661]  generic_handle_irq+0x24/0x38
[2021/6/16 13:39:36] [   85.392670]  __handle_domain_irq+0x5c/0xb8
[2021/6/16 13:39:36] [   85.392682]  gic_handle_irq+0xc4/0x178
[2021/6/16 13:39:36] [   85.392691]  el1_irq+0xe8/0x198
[2021/6/16 13:39:36] [   85.392706]  cpuidle_enter_state+0xb4/0x3c8
[2021/6/16 13:39:36] [   85.392716]  cpuidle_enter+0x18/0x20
[2021/6/16 13:39:36] [   85.392731]  call_cpuidle+0x18/0x30
[2021/6/16 13:39:36] [   85.392740]  do_idle+0x214/0x278
[2021/6/16 13:39:36] [   85.392748]  cpu_startup_entry+0x20/0x28
[2021/6/16 13:39:36] [   85.392757]  rest_init+0xcc/0xd8
[2021/6/16 13:39:36] [   85.392772]  start_kernel+0x4c0/0x4ec
(2):
[2021/6/17 11:13:45] [  122.548273] Call trace:
[2021/6/17 11:13:45] [  122.548296]  dump_backtrace+0x0/0x178
[2021/6/17 11:13:45] [  122.548311]  show_stack+0x14/0x20
[2021/6/17 11:13:45] [  122.548330]  dump_stack+0x94/0xb4
[2021/6/17 11:13:45] [  122.548348]  vb2_buffer_in_use+0x40/0xb8
[2021/6/17 11:13:45] [  122.548367]  __fill_v4l2_buffer+0x130/0x288
[2021/6/17 11:13:45] [  122.548383]  vb2_core_dqbuf+0x268/0x4f8
[2021/6/17 11:13:45] [  122.548401]  vb2_dqbuf+0x6c/0xb8
[2021/6/17 11:13:45] [  122.548418]  vb2_ioctl_dqbuf+0x50/0x60
[2021/6/17 11:13:45] [  122.548434]  v4l_dqbuf+0x44/0x58
[2021/6/17 11:13:45] [  122.548449]  __video_do_ioctl+0x1a0/0x348
[2021/6/17 11:13:45] [  122.548465]  video_usercopy+0x390/0x730
[2021/6/17 11:13:45] [  122.548481]  video_ioctl2+0x14/0x20
[2021/6/17 11:13:45] [  122.548499]  v4l2_ioctl+0x80/0xb0
[2021/6/17 11:13:45] [  122.548519]  v4l2_compat_ioctl32+0x1d0/0x3a48
[2021/6/17 11:13:45] [  122.548538]  __arm64_compat_sys_ioctl+0xbc/0x15b0
[2021/6/17 11:13:45] [  122.548559]  el0_svc_common.constprop.0+0xb8/0x178
[2021/6/17 11:13:45] [  122.548578]  el0_svc_compat_handler+0x18/0x20
[2021/6/17 11:13:45] [  122.548594]  el0_svc_compat+0x8/0x34
```

####  10、ioctl(fd, VIDIOC_STREAMOFF, &buf)

``` c
06-15 04:48:22.666     0     0 W Call trace:  
06-15 04:48:22.666     0     0 W         : dump_backtrace+0x0/0x178
06-15 04:48:22.666     0     0 W         : show_stack+0x14/0x20
06-15 04:48:22.666     0     0 W         : dump_stack+0x94/0xb4
06-15 04:48:22.666     0     0 W         : imx214_s_stream+0xd0/0x5d8
06-15 04:48:22.666     0     0 W         : rkisp1_pipeline_set_stream+0x174/0x248
06-15 04:48:22.666     0     0 W         : rkisp1_stop_streaming+0x74/0x200
06-15 04:48:22.666     0     0 W         : __vb2_queue_cancel+0x5c/0x1e0
06-15 04:48:22.666     0     0 W         : vb2_core_streamoff+0x70/0xa8
06-15 04:48:22.666     0     0 W         : vb2_streamoff+0x4c/0x80
06-15 04:48:22.666     0     0 W         : vb2_ioctl_streamoff+0x48/0x58
06-15 04:48:22.666     0     0 W         : v4l_streamoff+0x20/0x28
06-15 04:48:22.666     0     0 W         : __video_do_ioctl+0x1a0/0x348
06-15 04:48:22.666     0     0 W         : video_usercopy+0x390/0x730
06-15 04:48:22.666     0     0 W         : video_ioctl2+0x14/0x20
06-15 04:48:22.666     0     0 W         : v4l2_ioctl+0x80/0xb0
06-15 04:48:22.666     0     0 W         : v4l2_compat_ioctl32+0x1d0/0x3a48
06-15 04:48:22.666     0     0 W         : __arm64_compat_sys_ioctl+0xbc/0x15b0
06-15 04:48:22.666     0     0 W         : el0_svc_common.constprop.0+0xb8/0x178
06-15 04:48:22.666     0     0 W         : el0_svc_compat_handler+0x18/0x20
06-15 04:48:22.666     0     0 W         : el0_svc_compat+0x8/0x34
```

### （二）、流程图

![](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/Android.10.Camera.03/V4L2-IOCTL.png)
