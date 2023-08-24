---
title: Android 10 Display System源码分析（5）：ModeTest分析（Android 10.0 && Kernel 4.15）
cover: https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.26.jpg
categories: 
  - Display
tags:
  - Android
  - Linux
  - Graphics
toc: true
abbrlink: 20210710
date: 2021-07-10 09:25:00
---

## （一）、ModeTest显示图像

>//modetest
>adb shell
>modetest -M rockchip
>Encoders:
>id      crtc    type    possible crtcs  possible clones
>88      56      DSI     0x00000001      0x00000000
>&&
>89      88      connected       DSI-1           0x0             1       88
>stop
>modetest -M rockchip -s 89@56:1088x1920 -v
>                        //connector@crtc = 89@56

![](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/Android10.Display.5/modetest.jpg)

## （二）、ModeTest显示流程图Flow

![](https://raw.githubusercontent.com/zhoujinjiankim/PicGo/master/Android10.Display.5/ModeTestFlow.png)

```
X:\external\libdrm\tests\modetest\modetest.c

int main(int argc, char **argv)
{
    struct device dev;

    int c;
    int encoders = 0, connectors = 0, crtcs = 0, planes = 0, framebuffers = 0;
    int drop_master = 0;
    int test_vsync = 0;
    int test_cursor = 0;
    char *device = NULL;
    char *module = NULL;
    unsigned int i;
    unsigned int count = 0, plane_count = 0;
    unsigned int prop_count = 0;
    struct pipe_arg *pipe_args = NULL;
    struct plane_arg *plane_args = NULL;
    struct property_arg *prop_args = NULL;
    unsigned int args = 0;
    int ret;

    memset(&dev, 0, sizeof dev);

    ......
    dev.fd = util_open(device, module);
#define dump_resource(dev, res) if (res) dump_##res(dev)

    dump_resource(&dev, encoders);
    dump_resource(&dev, connectors);
    dump_resource(&dev, crtcs);
    dump_resource(&dev, planes);
    dump_resource(&dev, framebuffers);

    for (i = 0; i < prop_count; ++i)
        set_property(&dev, &prop_args[i]);

    if (count || plane_count) {
        uint64_t cap = 0;

        ret = drmGetCap(dev.fd, DRM_CAP_DUMB_BUFFER, &cap);
        if (ret || cap == 0) {
            fprintf(stderr, "driver doesn't support the dumb buffer API\n");
            return 1;
        }

        if (count)
            set_mode(&dev, pipe_args, count);

        if (plane_count)
            set_planes(&dev, plane_args, plane_count);

        if (test_cursor)
            set_cursors(&dev, pipe_args, count);

        if (test_vsync)
            test_page_flip(&dev, pipe_args, count);

        if (drop_master)
            drmDropMaster(dev.fd);

        getchar();

        if (test_cursor)
            clear_cursors(&dev);

        if (plane_count)
            clear_planes(&dev, plane_args, plane_count);

        if (count)
            clear_mode(&dev);
    }

    free_resources(dev.resources);

    return 0;
}
```

## （三）、ModeTest代码分析



### （1）、drmOpen(module, device)

```
X:\external\libdrm\tests\util\kms.c
int util_open(const char *device, const char *module)
{
	int fd;

	if (module) {
		fd = drmOpen(module, device);
		if (fd < 0) {
			fprintf(stderr, "failed to open device '%s': %s\n",
				module, strerror(errno));
			return -errno;
		}
	} else {
		unsigned int i;

		for (i = 0; i < ARRAY_SIZE(modules); i++) {
			printf("trying to open device '%s'...", modules[i]);

			fd = drmOpen(modules[i], device);
			if (fd < 0) {
				printf("failed\n");
			} else {
				printf("done\n");
				break;
			}
		}
		......
	}

	return fd;
}

```

### （2）、分配内存空间bo_create && 填充数据

``` c
static void set_mode(struct device *dev, struct pipe_arg *pipes, unsigned int count)
{
	uint32_t handles[4] = {0}, pitches[4] = {0}, offsets[4] = {0};
	unsigned int fb_id;
	struct bo *bo;
	unsigned int i;
	unsigned int j;
	int ret, x;

	dev->mode.width = 0;
	dev->mode.height = 0;
	dev->mode.fb_id = 0;

	for (i = 0; i < count; i++) {
		struct pipe_arg *pipe = &pipes[i];

		ret = pipe_find_crtc_and_mode(dev, pipe);
		if (ret < 0)
			continue;

		dev->mode.width += pipe->mode->hdisplay;
		if (dev->mode.height < pipe->mode->vdisplay)
			dev->mode.height = pipe->mode->vdisplay;
	}

	bo = bo_create(dev->fd, pipes[0].fourcc, dev->mode.width,
		       dev->mode.height, handles, pitches, offsets,
		       UTIL_PATTERN_SMPTE);
	......
	dev->mode.bo = bo;

	ret = drmModeAddFB2(dev->fd, dev->mode.width, dev->mode.height,
			    pipes[0].fourcc, handles, pitches, offsets, &fb_id, 0);
	......
	dev->mode.fb_id = fb_id;

	x = 0;
	for (i = 0; i < count; i++) {
		struct pipe_arg *pipe = &pipes[i];
		......
		printf("setting mode %s-%dHz@%s on connectors ",
		       pipe->mode_str, pipe->mode->vrefresh, pipe->format_str);
		for (j = 0; j < pipe->num_cons; ++j)
			printf("%s, ", pipe->cons[j]);
		printf("crtc %d\n", pipe->crtc->crtc->crtc_id);

		ret = drmModeSetCrtc(dev->fd, pipe->crtc->crtc->crtc_id, fb_id,
				     x, 0, pipe->con_ids, pipe->num_cons,
				     pipe->mode);

		/* XXX: Actually check if this is needed */
		drmModeDirtyFB(dev->fd, fb_id, NULL, 0);

		x += pipe->mode->hdisplay;
		......
	}
}

```


#### 2.1、映射物理内存drm_mmap
```
struct bo *
bo_create(int fd, unsigned int format,
	  unsigned int width, unsigned int height,
	  unsigned int handles[4], unsigned int pitches[4],
	  unsigned int offsets[4], enum util_fill_pattern pattern)
{
	unsigned int virtual_height;
	struct bo *bo;
	unsigned int bpp;
	void *planes[3] = { 0, };
	void *virtual;
	int ret;
	......
	bo = bo_create_dumb(fd, width, virtual_height, bpp);
	......
	ret = bo_map(bo, &virtual);
	......
	switch (format) {
	......
	util_fill_pattern(format, pattern, planes, width, height, pitches[0]);
	bo_unmap(bo);

	return bo;
}

static struct bo *
bo_create_dumb(int fd, unsigned int width, unsigned int height, unsigned int bpp)
{
	struct drm_mode_create_dumb arg;
	struct bo *bo;
	int ret;

	bo = calloc(1, sizeof(*bo));
	if (bo == NULL) {
		fprintf(stderr, "failed to allocate buffer object\n");
		return NULL;
	}

	memset(&arg, 0, sizeof(arg));
	arg.bpp = bpp;
	arg.width = width;
	arg.height = height;

	ret = drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &arg);
	if (ret) {
		fprintf(stderr, "failed to create dumb buffer: %s\n",
			strerror(errno));
		free(bo);
		return NULL;
	}

	bo->fd = fd;
	bo->handle = arg.handle;
	bo->size = arg.size;
	bo->pitch = arg.pitch;

	return bo;
}

static int bo_map(struct bo *bo, void **out)
{
	struct drm_mode_map_dumb arg;
	void *map;
	int ret;

	memset(&arg, 0, sizeof(arg));
	arg.handle = bo->handle;

	ret = drmIoctl(bo->fd, DRM_IOCTL_MODE_MAP_DUMB, &arg);
	if (ret)
		return ret;

	map = drm_mmap(0, bo->size, PROT_READ | PROT_WRITE, MAP_SHARED,
		       bo->fd, arg.offset);
	if (map == MAP_FAILED)
		return -EINVAL;

	bo->ptr = map;
	*out = map;

	return 0;
}

```

#### 2.2、填充数据

``` c
X:\external\libdrm\tests\util\pattern.c
void util_fill_pattern(uint32_t format, enum util_fill_pattern pattern,
		       void *planes[3], unsigned int width,
		       unsigned int height, unsigned int stride)
{
	const struct util_format_info *info;

	info = util_format_info_find(format);
	if (info == NULL)
		return;

	switch (pattern) {
	case UTIL_PATTERN_TILES:
		return fill_tiles(info, planes, width, height, stride);

	case UTIL_PATTERN_SMPTE:
		return fill_smpte(info, planes, width, height, stride);

	case UTIL_PATTERN_PLAIN:
		return fill_plain(planes, height, stride);

	default:
		printf("Error: unsupported test pattern %u.\n", pattern);
		break;
	}
}

```

### （3）、set_planes()

``` c
static void set_planes(struct device *dev, struct plane_arg *p, unsigned int count)
{
	unsigned int i;

	/* set up planes/overlays */
	for (i = 0; i < count; i++)
		if (set_plane(dev, &p[i]))
			return;
}

static int set_plane(struct device *dev, struct plane_arg *p)
{
	drmModePlane *ovr;
	uint32_t handles[4] = {0}, pitches[4] = {0}, offsets[4] = {0};
	uint32_t plane_id;
	struct bo *plane_bo;
	uint32_t plane_flags = 0;
	int crtc_x, crtc_y, crtc_w, crtc_h;
	struct crtc *crtc = NULL;
	unsigned int pipe;
	unsigned int i;

	/* Find an unused plane which can be connected to our CRTC. Find the
	 * CRTC index first, then iterate over available planes.
	 */
	for (i = 0; i < (unsigned int)dev->resources->res->count_crtcs; i++) {
		if (p->crtc_id == dev->resources->res->crtcs[i]) {
			crtc = &dev->resources->crtcs[i];
			pipe = i;
			break;
		}
	}
	......
	plane_id = p->plane_id;
	......
	fprintf(stderr, "testing %dx%d@%s overlay plane %u\n",
		p->w, p->h, p->format_str, plane_id);

	plane_bo = bo_create(dev->fd, p->fourcc, p->w, p->h, handles,
			     pitches, offsets, UTIL_PATTERN_TILES);
	if (plane_bo == NULL)
		return -1;

	p->bo = plane_bo;

	/* just use single plane format for now.. */
	if (drmModeAddFB2(dev->fd, p->w, p->h, p->fourcc,
			handles, pitches, offsets, &p->fb_id, plane_flags)) {
		fprintf(stderr, "failed to add fb: %s\n", strerror(errno));
		return -1;
	}

	crtc_w = p->w * p->scale;
	crtc_h = p->h * p->scale;
	if (!p->has_position) {
		/* Default to the middle of the screen */
		crtc_x = (crtc->mode->hdisplay - crtc_w) / 2;
		crtc_y = (crtc->mode->vdisplay - crtc_h) / 2;
	} else {
		crtc_x = p->x;
		crtc_y = p->y;
	}

	/* note src coords (last 4 args) are in Q16 format */
	if (drmModeSetPlane(dev->fd, plane_id, crtc->crtc->crtc_id, p->fb_id,
			    plane_flags, crtc_x, crtc_y, crtc_w, crtc_h,
			    0, 0, p->w << 16, p->h << 16)) {
		fprintf(stderr, "failed to enable plane: %s\n",
			strerror(errno));
		return -1;
	}

	ovr->crtc_id = crtc->crtc->crtc_id;

	return 0;
}
```


### （4）、set_cursors()

``` c
static void set_cursors(struct device *dev, struct pipe_arg *pipes, unsigned int count)
{
	uint32_t handles[4] = {0}, pitches[4] = {0}, offsets[4] = {0};
	struct bo *bo;
	unsigned int i;
	int ret;

	/* maybe make cursor width/height configurable some day */
	uint32_t cw = 64;
	uint32_t ch = 64;

	/* create cursor bo.. just using PATTERN_PLAIN as it has
	 * translucent alpha
	 */
	bo = bo_create(dev->fd, DRM_FORMAT_ARGB8888, cw, ch, handles, pitches,
		       offsets, UTIL_PATTERN_PLAIN);
	if (bo == NULL)
		return;

	dev->mode.cursor_bo = bo;

	for (i = 0; i < count; i++) {
		struct pipe_arg *pipe = &pipes[i];
		ret = cursor_init(dev->fd, handles[0],
				pipe->crtc->crtc->crtc_id,
				pipe->mode->hdisplay, pipe->mode->vdisplay,
				cw, ch);
		if (ret) {
			fprintf(stderr, "failed to init cursor for CRTC[%u]\n",
					pipe->crtc_id);
			return;
		}
	}

	cursor_start();
}

```

### （5）、test_page_flip

``` c
static void test_page_flip(struct device *dev, struct pipe_arg *pipes, unsigned int count)
{
	uint32_t handles[4] = {0}, pitches[4] = {0}, offsets[4] = {0};
	unsigned int other_fb_id;
	struct bo *other_bo;
	drmEventContext evctx;
	unsigned int i;
	int ret;

	other_bo = bo_create(dev->fd, pipes[0].fourcc, dev->mode.width,
			     dev->mode.height, handles, pitches, offsets,
			     UTIL_PATTERN_PLAIN);
	if (other_bo == NULL)
		return;

	ret = drmModeAddFB2(dev->fd, dev->mode.width, dev->mode.height,
			    pipes[0].fourcc, handles, pitches, offsets,
			    &other_fb_id, 0);
	if (ret) {
		fprintf(stderr, "failed to add fb: %s\n", strerror(errno));
		goto err;
	}

	for (i = 0; i < count; i++) {
		struct pipe_arg *pipe = &pipes[i];

		if (pipe->mode == NULL)
			continue;

		ret = drmModePageFlip(dev->fd, pipe->crtc->crtc->crtc_id,
				      other_fb_id, DRM_MODE_PAGE_FLIP_EVENT,
				      pipe);
		if (ret) {
			fprintf(stderr, "failed to page flip: %s\n", strerror(errno));
			goto err_rmfb;
		}
		gettimeofday(&pipe->start, NULL);
		pipe->swap_count = 0;
		pipe->fb_id[0] = dev->mode.fb_id;
		pipe->fb_id[1] = other_fb_id;
		pipe->current_fb_id = other_fb_id;
	}

	memset(&evctx, 0, sizeof evctx);
	evctx.version = DRM_EVENT_CONTEXT_VERSION;
	evctx.vblank_handler = NULL;
	evctx.page_flip_handler = page_flip_handler;

	while (1) {
#if 0
		struct pollfd pfd[2];

		pfd[0].fd = 0;
		pfd[0].events = POLLIN;
		pfd[1].fd = fd;
		pfd[1].events = POLLIN;

		if (poll(pfd, 2, -1) < 0) {
			fprintf(stderr, "poll error\n");
			break;
		}

		if (pfd[0].revents)
			break;
#else
		struct timeval timeout = { .tv_sec = 3, .tv_usec = 0 };
		fd_set fds;

		FD_ZERO(&fds);
		FD_SET(0, &fds);
		FD_SET(dev->fd, &fds);
		ret = select(dev->fd + 1, &fds, NULL, NULL, &timeout);

		if (ret <= 0) {
			fprintf(stderr, "select timed out or error (ret %d)\n",
				ret);
			continue;
		} else if (FD_ISSET(0, &fds)) {
			break;
		}
#endif

		drmHandleEvent(dev->fd, &evctx);
	}

err_rmfb:
	drmModeRmFB(dev->fd, other_fb_id);
err:
	bo_destroy(other_bo);
}
```

### （6）、page_flip_handler

``` c
static void
page_flip_handler(int fd, unsigned int frame,
		  unsigned int sec, unsigned int usec, void *data)
{
	struct pipe_arg *pipe;
	unsigned int new_fb_id;
	struct timeval end;
	double t;

	pipe = data;
	if (pipe->current_fb_id == pipe->fb_id[0])
		new_fb_id = pipe->fb_id[1];
	else
		new_fb_id = pipe->fb_id[0];

	drmModePageFlip(fd, pipe->crtc->crtc->crtc_id, new_fb_id,
			DRM_MODE_PAGE_FLIP_EVENT, pipe);
	pipe->current_fb_id = new_fb_id;
	pipe->swap_count++;
	if (pipe->swap_count == 60) {
		gettimeofday(&end, NULL);
		t = end.tv_sec + end.tv_usec * 1e-6 -
			(pipe->start.tv_sec + pipe->start.tv_usec * 1e-6);
		fprintf(stderr, "freq: %.02fHz\n", pipe->swap_count / t);
		pipe->swap_count = 0;
		pipe->start = end;
	}
}
```

## （四）、Log

``` xml
[BEGIN] 2020/10/14 11:28:26
[2020/10/14 11:28:26] Connecting to COM3...
[2020/10/14 11:28:27] Connected.
[2020/10/14 11:28:27] 
[2020/10/14 11:30:42] [  243.668600] zjj.rk3399.kernel drivers/gpu/drm/drm_drv.c drm_stub_open 956 
[2020/10/14 11:30:42] [  243.668713] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_drv.c rockchip_drm_open 1654 
[2020/10/14 11:30:42] [  243.668789] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_VERSION
[2020/10/14 11:30:42] [  243.668828] zjj.rk3399.kernel drivers/gpu/drm/drm_ioctl.c drm_version 497 
[2020/10/14 11:30:42] [  243.668897] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_VERSION
[2020/10/14 11:30:42] [  243.668930] zjj.rk3399.kernel drivers/gpu/drm/drm_ioctl.c drm_version 497 
[2020/10/14 11:30:42] [  243.669027] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_GET_UNIQUE
[2020/10/14 11:30:42] [  243.669047] zjj.rk3399.kernel drivers/gpu/drm/drm_ioctl.c drm_getunique 116 
[2020/10/14 11:30:42] [  243.669110] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_GET_UNIQUE
[2020/10/14 11:30:42] [  243.669139] zjj.rk3399.kernel drivers/gpu/drm/drm_ioctl.c drm_getunique 116 
[2020/10/14 11:30:42] [  243.669178] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_SET_CLIENT_CAP
[2020/10/14 11:30:42] [  243.669205] zjj.rk3399.kernel drivers/gpu/drm/drm_ioctl.c drm_setclientcap 311 
[2020/10/14 11:30:42] [  243.669253] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_SET_CLIENT_CAP
[2020/10/14 11:30:42] [  243.669304] zjj.rk3399.kernel drivers/gpu/drm/drm_ioctl.c drm_setclientcap 311 
[2020/10/14 11:30:42] [  243.669347] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETRESOURCES
[2020/10/14 11:30:42] [  243.669391] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_config.c drm_mode_getresources 101 
[2020/10/14 11:30:42] [  243.669439] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETRESOURCES
[2020/10/14 11:30:42] [  243.669466] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_config.c drm_mode_getresources 101 
[2020/10/14 11:30:42] [  243.669565] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETCRTC
[2020/10/14 11:30:42] [  243.669638] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETCRTC
[2020/10/14 11:30:42] [  243.669681] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETENCODER
[2020/10/14 11:30:42] [  243.669710] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_mode_getencoder 226 
[2020/10/14 11:30:42] [  243.669735] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_get_crtc 194 
[2020/10/14 11:30:42] [  243.669773] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETENCODER
[2020/10/14 11:30:42] [  243.669792] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_mode_getencoder 226 
[2020/10/14 11:30:42] [  243.669808] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_get_crtc 194 
[2020/10/14 11:30:42] [  243.669851] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETENCODER
[2020/10/14 11:30:42] [  243.669878] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_mode_getencoder 226 
[2020/10/14 11:30:42] [  243.669902] zjj.rk3399.kernel drivers/gpu/drm/drm_encoder.c drm_encoder_get_crtc 194 
[2020/10/14 11:30:42] [  243.669982] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETCONNECTOR
[2020/10/14 11:30:42] [  243.670014] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_mode_getconnector 1783 
[2020/10/14 11:30:42] [  243.670130] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETCONNECTOR
[2020/10/14 11:30:42] [  243.670158] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_mode_getconnector 1783 
[2020/10/14 11:30:42] [  243.670240] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETCONNECTOR
[2020/10/14 11:30:42] [  243.670268] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_mode_getconnector 1783 
[2020/10/14 11:30:42] [  243.670362] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETCONNECTOR
[2020/10/14 11:30:42] [  243.670391] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_mode_getconnector 1783 
[2020/10/14 11:30:42] [  243.670499] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETCONNECTOR
[2020/10/14 11:30:42] [  243.670528] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_mode_getconnector 1783 
[2020/10/14 11:30:42] [  243.670573] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_get_modes 629 
[2020/10/14 11:30:42] [  243.670599] zjj.rk3399.kernel drivers/gpu/drm/panel/panel-simple.c panel_simple_get_fixed_modes 373 
[2020/10/14 11:30:42] [  243.670740] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETCONNECTOR
[2020/10/14 11:30:42] [  243.670771] zjj.rk3399.kernel drivers/gpu/drm/drm_connector.c drm_mode_getconnector 1783 
[2020/10/14 11:30:42] [  243.670904] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.670976] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.671034] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.671053] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.671122] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671151] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671185] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671212] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671250] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671277] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671308] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671326] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671359] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671385] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671417] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671443] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671478] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671504] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671536] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671554] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671581] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671607] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671638] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671664] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671700] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671726] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671758] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671784] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671819] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671840] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671872] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671898] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.671934] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.671995] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672030] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672056] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672085] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672111] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672143] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672169] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672209] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.672237] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.672279] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.672306] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.672369] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672396] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672428] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672454] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672489] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672515] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672546] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672565] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672588] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672614] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672646] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672671] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672708] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672734] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672765] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672791] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672826] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672843] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672875] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.672902] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.672969] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673005] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673039] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673066] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673095] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673121] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673153] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673179] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673216] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673242] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673274] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673300] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673336] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673353] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673386] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673412] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673450] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.673476] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.673533] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.673560] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.673637] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673665] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673696] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673721] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673754] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673779] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673813] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673830] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673882] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.673908] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.673972] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674001] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674045] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674064] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674086] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674113] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674148] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674175] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674206] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674233] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674267] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674293] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674325] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674343] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674380] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674406] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674437] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674463] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674499] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674526] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674557] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674574] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674601] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674627] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674658] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674684] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.674720] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.674746] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.674802] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.674824] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.674923] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.674979] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675015] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675041] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675075] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675093] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675126] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675151] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675203] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675229] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675261] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675287] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675330] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675350] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675382] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675408] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675444] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675470] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675501] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675527] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675560] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675578] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675611] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675637] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675684] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675710] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675787] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675809] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675863] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.675892] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.675924] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676106] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676162] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676190] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676222] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676248] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676285] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676311] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676334] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676360] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676399] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676426] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676459] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676485] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676532] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676558] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676582] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676607] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676654] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676681] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676712] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676738] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676770] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676796] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676819] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676846] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676883] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676910] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.676973] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.676992] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677019] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677053] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677078] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677095] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677122] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677139] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677162] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677179] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677207] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.677225] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.677275] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.677293] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.677366] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677398] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677431] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677459] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677494] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677522] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677557] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677575] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677627] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677655] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677688] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677716] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677758] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677787] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677810] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677838] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677876] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.677905] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.677977] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.678009] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.678049] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.678067] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.678102] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.678129] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.678166] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.678194] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.678227] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.678254] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.678292] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.678309] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.678344] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.678371] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.678409] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.678436] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.678469] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.678495] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.678535] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANERESOURCES
[2020/10/14 11:30:42] [  243.678552] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane_res 572 
[2020/10/14 11:30:42] [  243.678589] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANERESOURCES
[2020/10/14 11:30:42] [  243.678616] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane_res 572 
[2020/10/14 11:30:42] [  243.678661] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.678689] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.678724] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.678752] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.678793] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.678814] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.678851] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.678879] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.678916] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.678975] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.679015] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.679038] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.679067] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.679095] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.679128] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.679155] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.679192] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.679219] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.679253] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.679280] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.679308] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.679335] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.679369] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPLANE
[2020/10/14 11:30:42] [  243.679396] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_getplane 613 
[2020/10/14 11:30:42] [  243.679434] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.679461] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.679539] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.679558] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.679676] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.679704] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.679739] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.679767] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.679813] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.679841] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.679875] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.679902] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.679969] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680000] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680035] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680052] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680094] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680122] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680156] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680182] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680219] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680246] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680279] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680296] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680335] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680362] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680395] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680421] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680458] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680485] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680518] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680535] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680574] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680602] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680635] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680661] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680699] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680727] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680759] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680777] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680817] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680845] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680877] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.680905] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.680971] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681003] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681028] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681055] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681094] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681122] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681155] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681183] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681221] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681249] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681273] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681300] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681337] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681365] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681398] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681424] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681461] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681489] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681512] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681540] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681578] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681606] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681669] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681700] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681757] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681775] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681810] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681837] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681875] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.681903] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.681988] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.682010] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.682039] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.682070] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.682117] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.682169] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.682229] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.682272] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.682306] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.682347] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.682407] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.682451] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.682496] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.682529] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.682588] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.682617] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.682663] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.682690] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.682803] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.682832] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.682865] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.682893] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.682974] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683006] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683040] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683057] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683095] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683123] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683156] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683183] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683220] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683247] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683279] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683297] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683335] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683363] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683396] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683423] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683461] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683488] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683521] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683539] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683591] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683620] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683654] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683682] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683719] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683747] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683779] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683796] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683834] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683861] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683894] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.683922] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.683990] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684020] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684045] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684072] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684109] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684136] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684170] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684197] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684235] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684262] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684285] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684314] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684350] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684378] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684410] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684437] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684475] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684503] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684526] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684554] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684591] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684618] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684651] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684678] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684716] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684744] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684769] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684796] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684852] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684879] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684912] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.684964] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.684997] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685014] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685037] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685067] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685104] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685131] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685165] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685192] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685228] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685256] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685280] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685307] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685345] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685372] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685405] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685432] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685469] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685497] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685522] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685549] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685597] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.685625] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.685670] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.685698] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.685805] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685833] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.685868] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.685895] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686021] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686041] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686081] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686109] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686147] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686176] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686210] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686228] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686255] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686272] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686295] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686322] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686359] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686388] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686422] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686450] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686489] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686516] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686540] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686568] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686606] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686635] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686669] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686696] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686732] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686760] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686784] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686811] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686847] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686874] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.686908] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.686964] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687001] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687019] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687042] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687071] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687112] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687140] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687173] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687201] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687238] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687265] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687289] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687316] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687353] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687381] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687413] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687441] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687478] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687506] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687530] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687562] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687601] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687629] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687663] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687690] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687727] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687754] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687779] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687806] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687863] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687891] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.687924] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.687980] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688011] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688028] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688051] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688081] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688120] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688149] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688182] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688209] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688246] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688274] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688297] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688325] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688361] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688389] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688422] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688448] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688486] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.688514] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.688552] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.688581] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.688688] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688716] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688750] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688776] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688813] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688841] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688873] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688900] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.688965] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.688999] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689025] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689042] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689070] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689087] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689110] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689127] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689190] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689208] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689231] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689251] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689294] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689323] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689356] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689383] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689420] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689447] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689480] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689498] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689535] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689563] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689596] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689623] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689660] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689690] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689723] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689740] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689780] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689808] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689840] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689867] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689903] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.689931] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.689986] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690003] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690048] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690076] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690108] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690135] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690173] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690201] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690234] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690252] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690290] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690318] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690351] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690378] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690414] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690442] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690474] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690492] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690530] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690558] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690592] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690619] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690677] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690704] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690737] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690754] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690794] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690822] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690854] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690881] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690919] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.690973] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.690999] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691017] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691058] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691086] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691118] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691145] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691183] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691210] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691244] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691261] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691299] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.691327] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.691374] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_OBJ_GETPROPERTIES
[2020/10/14 11:30:42] [  243.691403] zjj.rk3399.kernel drivers/gpu/drm/drm_mode_object.c drm_mode_obj_get_properties_ioctl 385 
[2020/10/14 11:30:42] [  243.691512] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691541] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691576] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691604] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691652] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691680] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691713] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691740] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691768] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691795] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691828] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691855] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691894] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.691922] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.691986] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692004] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692030] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692059] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692092] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692119] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692155] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692183] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692215] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692243] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692273] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692301] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692334] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692361] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692397] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692425] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692458] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692484] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692514] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692541] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:42] [  243.692574] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:42] [  243.692601] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.692637] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.692665] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.692698] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.692725] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.692754] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.692782] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.692814] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.692841] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.692878] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.692906] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.692966] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.692985] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693014] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693044] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693077] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693104] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693141] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693168] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693201] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693227] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693256] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693283] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693316] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693343] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693379] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693407] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693440] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693468] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693513] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693541] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693573] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693600] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693638] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693666] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693699] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693725] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693753] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693781] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693813] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693840] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693876] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693904] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.693965] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.693984] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.695281] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.695314] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.695347] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.695374] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.695411] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.695438] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.695471] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.695498] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.695526] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.695553] zjj.rk3399.kernel drivers/gpu/drm/drm_property.c drm_mode_getproperty_ioctl 468 
[2020/10/14 11:30:43] [  243.695586] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_GETPROPERTY
[2020/10/14 11:30:43] [  243.721667] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_MAP_DUMB
[2020/10/14 11:30:43] [  243.721713] zjj.rk3399.kernel drivers/gpu/drm/drm_dumb_buffers.c drm_mode_mmap_dumb_ioctl 120 
[2020/10/14 11:30:43] [  243.721741] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_gem.c rockchip_gem_dumb_map_offset 920 
[2020/10/14 11:30:43] [  243.721826] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_gem.c rockchip_gem_mmap 607 
[2020/10/14 11:30:43] [  243.762609] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_ADDFB2
[2020/10/14 11:30:43] [  243.762651] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_mode_addfb2 337 
[2020/10/14 11:30:43] [  243.762677] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_internal_framebuffer_create 280 
[2020/10/14 11:30:43] [  243.762707] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_user_fb_create 187 
[2020/10/14 11:30:43] [  243.762736] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_alloc 125 
[2020/10/14 11:30:43] [  243.762765] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/14 11:30:43] [  243.762797] [FB:90]
[2020/10/14 11:30:43] [  243.763078] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_SETCRTC
[2020/10/14 11:30:43] [  243.763101] zjj.rk3399.kernel drivers/gpu/drm/drm_crtc.c drm_mode_setcrtc 582 
[2020/10/14 11:30:43] [  243.763203] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_set_config 3010 
[2020/10/14 11:30:43] [  243.763224] drm_atomic.c Allocated atomic state 00000000d17907c1
[2020/10/14 11:30:43] [  243.763268] drm_atomic.c Added [CRTC:56:crtc-0] 00000000e13b5b93 state to 00000000d17907c1
[2020/10/14 11:30:43] [  243.763291] drm_atomic.c Added [PLANE:55:plane-0] 000000003f76ad83 state to 00000000d17907c1
[2020/10/14 11:30:43] [  243.763326] drm_atomic.c Set [MODE:1088x1920] for [CRTC:56:crtc-0] state 00000000e13b5b93
[2020/10/14 11:30:43] [  243.763344] Link [PLANE:55:plane-0] state 000000003f76ad83 to [CRTC:56:crtc-0]
[2020/10/14 11:30:43] [  243.763361] Set [FB:90] for [PLANE:55:plane-0] state 000000003f76ad83
[2020/10/14 11:30:43] [  243.763389] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_add_affected_connectors 1848 
[2020/10/14 11:30:43] [  243.763406] Adding all current connectors for [CRTC:56:crtc-0] to 00000000d17907c1
[2020/10/14 11:30:43] [  243.763426] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/14 11:30:43] [  243.763467] drm_atomic.c Added [CONNECTOR:89:DSI-1] 000000001ad6009d state to 00000000d17907c1
[2020/10/14 11:30:43] [  243.763513] Link [CONNECTOR:89:DSI-1] state 000000001ad6009d to [NOCRTC]
[2020/10/14 11:30:43] [  243.763540] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_get_connector_state 1277 
[2020/10/14 11:30:43] [  243.763579] Link [CONNECTOR:89:DSI-1] state 000000001ad6009d to [CRTC:56:crtc-0]
[2020/10/14 11:30:43] [  243.763613] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_commit 2019 
[2020/10/14 11:30:43] [  243.763639] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_check_only 1946 
[2020/10/14 11:30:43] [  243.763662] checking 00000000d17907c1
[2020/10/14 11:30:43] [  243.763680] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check 916 
[2020/10/14 11:30:43] [  243.763696] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check_modeset 597 
[2020/10/14 11:30:43] [  243.763727] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c update_connector_routing 287 
[2020/10/14 11:30:43] [  243.763752] Updating routing for [CONNECTOR:89:DSI-1]
[2020/10/14 11:30:43] [  243.763780] [CONNECTOR:89:DSI-1] keeps [ENCODER:88:DSI-88], now on [CRTC:56:crtc-0]
[2020/10/14 11:30:43] [  243.763809] zjj.rk3399.kernel drivers/gpu/drm/rockchip/dw-mipi-dsi.c dw_mipi_dsi_encoder_atomic_check 1340 
[2020/10/14 11:30:43] [  243.763837] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/14 11:30:43] [  243.763868] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_is_logo 34 
[2020/10/14 11:30:43] [  243.763893] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_dma_addr 43 
[2020/10/14 11:30:43] [  243.763918] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_kvaddr 54 
[2020/10/14 11:30:43] [  243.763972] zjj.rk3399.kernel addr: 0x00000000009db000 yrgb_mst: 0x00000000009db000  yrgb_kvaddr: 0x0000000000000000
[2020/10/14 11:30:43] [  243.763981] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/14 11:30:43] [  243.764031] committing 00000000d17907c1
[2020/10/14 11:30:43] [  243.764057] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_atomic_commit 470 
[2020/10/14 11:30:43] [  243.764100] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_atomic_commit_complete 400 
[2020/10/14 11:30:43] [  243.764227] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_update 1758 
[2020/10/14 11:30:43] [  243.764264] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_flush 3558 
[2020/10/14 11:30:43] [  243.764294] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_cfg_update 3485 
[2020/10/14 11:30:43] [  243.764329] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_tv_config_update 3397 
[2020/10/14 11:30:43] [  243.770706] drm_atomic.c Clearing atomic state 00000000d17907c1
[2020/10/14 11:30:43] [  243.770798] drm_atomic.c Freeing atomic state 00000000d17907c1
[2020/10/14 11:30:43] [  243.770886] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_DIRTYFB
[2020/10/14 11:30:43] [  243.770921] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_mode_dirtyfb_ioctl 547 
[2020/10/14 11:30:43] [  243.770993] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_fb_dirty 102 
[2020/10/14 11:30:43] [  243.771058] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_CREATE_DUMB
[2020/10/14 11:30:43] [  243.771090] zjj.rk3399.kernel drivers/gpu/drm/drm_dumb_buffers.c drm_mode_create_dumb 61 
[2020/10/14 11:30:43] [  243.771117] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_gem.c rockchip_gem_dumb_create 762 
[2020/10/14 11:30:43] [  243.795173] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_MAP_DUMB
[2020/10/14 11:30:43] [  243.795223] zjj.rk3399.kernel drivers/gpu/drm/drm_dumb_buffers.c drm_mode_mmap_dumb_ioctl 120 
[2020/10/14 11:30:43] [  243.795250] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_gem.c rockchip_gem_dumb_map_offset 920 
[2020/10/14 11:30:43] [  243.795335] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_gem.c rockchip_gem_mmap 607 
[2020/10/14 11:30:43] [  243.813500] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_ADDFB2
[2020/10/14 11:30:43] [  243.813545] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_mode_addfb2 337 
[2020/10/14 11:30:43] [  243.813572] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_internal_framebuffer_create 280 
[2020/10/14 11:30:43] [  243.813591] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_user_fb_create 187 
[2020/10/14 11:30:43] [  243.813626] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_alloc 125 
[2020/10/14 11:30:43] [  243.813658] zjj.rk3399.kernel drivers/gpu/drm/drm_framebuffer.c drm_framebuffer_init 691 
[2020/10/14 11:30:43] [  243.813690] [FB:92]
[2020/10/14 11:30:43] [  243.813728] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_PAGE_FLIP
[2020/10/14 11:30:43] [  243.813757] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_page_flip_ioctl 1125 
[2020/10/14 11:30:43] [  243.813798] drm_atomic.c Allocated atomic state 000000003eb8c734
[2020/10/14 11:30:43] [  243.813857] drm_atomic.c Added [CRTC:56:crtc-0] 00000000c2492712 state to 000000003eb8c734
[2020/10/14 11:30:43] [  243.813901] drm_atomic.c Added [PLANE:55:plane-0] 0000000099d2e0cd state to 000000003eb8c734
[2020/10/14 11:30:43] [  243.813930] Set [FB:92] for [PLANE:55:plane-0] state 0000000099d2e0cd
[2020/10/14 11:30:43] [  243.814024] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_check_only 1946 
[2020/10/14 11:30:43] [  243.814050] checking 000000003eb8c734
[2020/10/14 11:30:43] [  243.814078] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check 916 
[2020/10/14 11:30:43] [  243.814095] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check_modeset 597 
[2020/10/14 11:30:43] [  243.814131] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/14 11:30:43] [  243.814163] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_is_logo 34 
[2020/10/14 11:30:43] [  243.814188] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_dma_addr 43 
[2020/10/14 11:30:43] [  243.814212] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_kvaddr 54 
[2020/10/14 11:30:43] [  243.814240] zjj.rk3399.kernel addr: 0x00000000011d3000 yrgb_mst: 0x00000000011d3000  yrgb_kvaddr: 0x0000000000000000
[2020/10/14 11:30:43] [  243.814248] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/14 11:30:43] [  243.814299] committing 000000003eb8c734 nonblocking
[2020/10/14 11:30:43] [  243.814325] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_atomic_commit 470 
[2020/10/14 11:30:43] [  243.814396] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_drm_atomic_work 455 
[2020/10/14 11:30:43] [  243.814423] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_atomic_commit_complete 400 
[2020/10/14 11:30:43] [  243.814496] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_update 1758 
[2020/10/14 11:30:43] [  243.814542] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_flush 3558 
[2020/10/14 11:30:43] [  243.814568] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_cfg_update 3485 
[2020/10/14 11:30:43] [  243.814592] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_tv_config_update 3397 
[2020/10/14 11:30:43] [  243.834400] drm_atomic.c Clearing atomic state 000000003eb8c734
[2020/10/14 11:30:43] [  243.834461] zjj.rk3399.kernel pid=1726, dev=0xe200, auth=1, DRM_IOCTL_MODE_PAGE_FLIP
[2020/10/14 11:30:43] [  243.834496] drm_atomic.c Freeing atomic state 000000003eb8c734
[2020/10/14 11:30:43] [  243.834503] zjj.rk3399.kernel drivers/gpu/drm/drm_plane.c drm_mode_page_flip_ioctl 1125 
[2020/10/14 11:30:43] [  243.834529] drm_atomic.c Allocated atomic state 00000000b873c6f7
[2020/10/14 11:30:43] [  243.834556] drm_atomic.c Added [CRTC:56:crtc-0] 000000009d2b5db7 state to 00000000b873c6f7
[2020/10/14 11:30:43] [  243.834604] drm_atomic.c Added [PLANE:55:plane-0] 00000000789926ee state to 00000000b873c6f7
[2020/10/14 11:30:43] [  243.834623] Set [FB:90] for [PLANE:55:plane-0] state 00000000789926ee
[2020/10/14 11:30:43] [  243.834714] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic.c drm_atomic_check_only 1946 
[2020/10/14 11:30:43] [  243.834750] checking 00000000b873c6f7
[2020/10/14 11:30:43] [  243.834770] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check 916 
[2020/10/14 11:30:43] [  243.834803] zjj.rk3399.kernel drivers/gpu/drm/drm_atomic_helper.c drm_atomic_helper_check_modeset 597 
[2020/10/14 11:30:43] [  243.834835] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_plane_atomic_check 1577 
[2020/10/14 11:30:43] [  243.834871] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_is_logo 34 
[2020/10/14 11:30:43] [  243.834888] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_dma_addr 43 
[2020/10/14 11:30:43] [  243.834914] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_fb.c rockchip_fb_get_kvaddr 54 
[2020/10/14 11:30:43] [  243.834981] zjj.rk3399.kernel addr: 0x00000000009db000 yrgb_mst: 0x00000000009db000  yrgb_kvaddr: 0x0000000000000000
[2020/10/14 11:30:43] [  243.834989] zjj.rk3399.kernel drivers/gpu/drm/rockchip/rockchip_drm_vop.c vop_crtc_atomic_check 3212 
[2020/10/14 11:30:43] [  243.835053] committing 00000000b873c6f7 nonblocking
```