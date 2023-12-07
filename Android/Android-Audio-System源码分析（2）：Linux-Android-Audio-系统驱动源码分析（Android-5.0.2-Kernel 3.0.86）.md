---
title: Android Audio System源码分析（2）：Linux && Android Audio 系统驱动源码分析（Android 5.0.2 && Kernel 3.0.86）
cover: https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/post.cover.pictures.00008.jpg
categories:
  - Audio
tags:
  - Android
  - Linux
  - Audio
toc: true
abbrlink: 20200308
date: 2020-03-08 09:25:00
---

注：文章都是通过阅读 Android  && Linux 平台源码、各位前辈总结的资料、加上自己的思考分析总结出来的，其中难免有理不对的地方，欢迎大家批评指正。**文章为个人学习、研究、欣赏之用。图文内容、源码整理自互联网，如有侵权，请联系删除（◕‿◕）**，转载请注明出处（ ©Android @Linux 版权所有），谢谢(๑乛◡乛๑) 、（ ͡° ͜ʖ ͡°）、（ಡωಡ）！！！。
（==**文章基于 Kernel-3.0.86**==）&&（==**文章基于 Android 5.0.2**==）
[【开发板 - 友善之臂 FriendlyARM Cortex-A9 Tiny4412 ADK Exynos4412 （ Android 5.0.2）HD702高清电容屏 扩展套餐】](https://item.taobao.com/item.htm?spm=0.0.0.0.3GsGdQ&id=20131438062)
[【开发板 Android 5.0.2 && Kernel 3.0.86 源码链接： https://pan.baidu.com/s/1jJHm74q 密码：yfih】](https://pan.baidu.com/s/1jJHm74q)

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢！！！

--------------------------------------------------------------------------------

--------------------------------------------------------------------------------

####  （一）、ALSA 音频系统框架

##### （1）、声卡节点
我们首先看看tiny4412的声卡节电：
``` cpp
root@tiny4412:/ # ls -l /dev/snd/
crw-rw---- system   audio    116,   0 2016-01-01 12:00 controlC0 //起控制作用
crw-rw---- system   audio    116,  24 2016-01-01 12:00 pcmC0D0c  //card0，device0，capture
crw-rw---- system   audio    116,  16 2016-01-01 12:00 pcmC0D0p  //card0，device0，playback
crw-rw---- system   audio    116,  25 2016-01-01 12:00 pcmC0D1c  //card0，device1，capture
crw-rw---- system   audio    116,  17 2016-01-01 12:00 pcmC0D1p  //card0，device1，playback
crw-rw---- system   audio    116,  33 2016-01-01 12:00 timer
```
可以推断一个声卡可以有多个device，一个device有播放和录音通道。用户空间通过声卡节点访问内核空间，每个设备节点在驱动程序都有一个file_ops，上面的主设备号相同（116），对应同一个file_ops结构体。

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\sound.c

static int major = CONFIG_SND_MAJOR; // CONFIG_SND_MAJOR 为 116

static const struct file_operations snd_fops =
{
	.owner =	THIS_MODULE,
	.open =		snd_open,
	.llseek =	noop_llseek,
};


static int __init alsa_sound_init(void)
{
	snd_major = major;
	snd_ecards_limit = cards_limit;
	if (register_chrdev(major, "alsa", &snd_fops)) {
		snd_printk(KERN_ERR "unable to register native major device number %d\n", major);
		return -EIO;
	}
	if (snd_info_init() < 0) {
		unregister_chrdev(major, "alsa");
		return -ENOMEM;
	}
	snd_info_minor_register();
#ifndef MODULE
	printk(KERN_INFO "Advanced Linux Sound Architecture Driver Version " CONFIG_SND_VERSION CONFIG_SND_DATE ".\n");
#endif
	return 0;
}

static int snd_open(struct inode *inode, struct file *file)
{
	unsigned int minor = iminor(inode); 
	struct snd_minor *mptr = NULL;
	const struct file_operations *old_fops;
	int err = 0;

	if (minor >= ARRAY_SIZE(snd_minors))
		return -ENODEV;
	mutex_lock(&sound_mutex);
	mptr = snd_minors[minor];    //根据次设备号找到一项
	if (mptr == NULL) {
		mptr = autoload_device(minor);
		if (!mptr) {
			mutex_unlock(&sound_mutex);
			return -ENODEV;
		}
	}
	old_fops = file->f_op; //这一项有f_op结构体
	file->f_op = fops_get(mptr->f_ops);
	if (file->f_op == NULL) {
		file->f_op = old_fops;
		err = -ENODEV;
	}
	mutex_unlock(&sound_mutex);
	if (err < 0)
		return err;

	if (file->f_op->open) {
		err = file->f_op->open(inode, file);
		if (err) {
			fops_put(file->f_op);
			file->f_op = fops_get(old_fops);
		}
	}
	fops_put(old_fops);
	return err;
}

```

snd_fops  为中转作用，会根据次设备号找到具体的file_ops结构体。playback，capture都有自己的file_ops。

根据ALSA规范，事先定义好顶层结构体。APP就可以使用确定的接口访问声卡。

**顶层：Sound.c      snd_ops**


**（1）下层：controlC0 节点     snd_ctl_f_ops** 

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\control.c
static const struct file_operations snd_ctl_f_ops =
{
	.owner =	THIS_MODULE,
	.read =		snd_ctl_read,
	.open =		snd_ctl_open,
	.release =	snd_ctl_release,
	.llseek =	no_llseek,
	.poll =		snd_ctl_poll,
	.unlocked_ioctl =	snd_ctl_ioctl,
	.compat_ioctl =	snd_ctl_ioctl_compat,
	.fasync =	snd_ctl_fasync,
};
```

**（2）下层：pcmC0D0c 节点   snd_pcm_f_ops** 
**（3）下层：pcmC0D0p 节点   snd_pcm_f_ops**
snd_pcm_f_ops是一个数组，一个对应playback，一个对应capture。
``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\pcm_native.c
const struct file_operations snd_pcm_f_ops[2] = {
	{
		.owner =		THIS_MODULE,
		.write =		snd_pcm_write,
		.aio_write =		snd_pcm_aio_write,
		.open =			snd_pcm_playback_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_playback_poll,
		.unlocked_ioctl =	snd_pcm_playback_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	},
	{
		.owner =		THIS_MODULE,
		.read =			snd_pcm_read,
		.aio_read =		snd_pcm_aio_read,
		.open =			snd_pcm_capture_open,
		.release =		snd_pcm_release,
		.llseek =		no_llseek,
		.poll =			snd_pcm_capture_poll,
		.unlocked_ioctl =	snd_pcm_capture_ioctl,
		.compat_ioctl = 	snd_pcm_ioctl_compat,
		.mmap =			snd_pcm_mmap,
		.fasync =		snd_pcm_fasync,
		.get_unmapped_area =	snd_pcm_get_unmapped_area,
	}
};

```


##### （2）、怎么写声卡驱动？

随便找一个声卡驱动，总结怎么写声卡驱动?

```cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\drivers\staging\tm6000\tm6000-alsa.c
/*
 * Alsa Constructor - Component probe
 */
int tm6000_audio_init(struct tm6000_core *dev)
{
	struct snd_card		*card;
	struct snd_tm6000_card	*chip;
	int			rc;
	static int		devnr;
	char			component[14];
	struct snd_pcm		*pcm;
	......
	rc = snd_card_create(index[devnr], "tm6000", THIS_MODULE, 0, &card);
	......
	rc = snd_pcm_new(card, "TM6000 Audio", 0, 0, 1, &pcm);
	......
	rc = snd_card_register(card);
	......
	return 0;
}
```


> sound/core/sound.c      实现了最顶层的file_operations，它起中转作用
> sound/core/control.c     实现了控制接口的file_operations
> sound/core/pcm_native.c 实现了playback, capture的file_operations
> 
> 这些file_operations规定了ALSA接口
> 
>  实现硬件相关的代码即可: 创建、设置、注册snd_card结构体: 
> a. 创建：snd_card_create //里面会创建控制接口 
> b. 设置：snd_pcm_new     // 里面会创建playback, capture接口 
> c. 注册：snd_card_register


sound.c中确定最顶层file_ops结构体
``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\sound.c
register_chrdev(major, "alsa", &snd_fops) //确定最顶层file_ops结构体
```


#####  2.1、snd_card_create
``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\init.c
int snd_card_create(int idx, const char *xid,
		    struct module *module, int extra_size,
		    struct snd_card **card_ret)
{
	struct snd_card *card;
	int err, idx2;

	if (snd_BUG_ON(!card_ret))
		return -EINVAL;
	*card_ret = NULL;

	if (extra_size < 0)
		extra_size = 0;
	//创建分配snd_card结构体
	card = kzalloc(sizeof(*card) + extra_size, GFP_KERNEL);
	......
	snd_cards_lock |= 1 << idx;		/* lock it */
	if (idx >= snd_ecards_limit)
		snd_ecards_limit = idx + 1; /* increase the limit */
	mutex_unlock(&snd_card_mutex);
	card->number = idx;
	card->module = module;
	INIT_LIST_HEAD(&card->devices);
	init_rwsem(&card->controls_rwsem);
	rwlock_init(&card->ctl_files_rwlock);
	INIT_LIST_HEAD(&card->controls);
	INIT_LIST_HEAD(&card->ctl_files);
	spin_lock_init(&card->files_lock);
	INIT_LIST_HEAD(&card->files_list);
	init_waitqueue_head(&card->shutdown_sleep);
	atomic_set(&card->refcount, 0);
	
	/* the control interface cannot be accessed from the user space until */
	/* snd_cards_bitmask and snd_cards are set with snd_card_register */
	err = snd_ctl_create(card);
	......
	err = snd_info_card_create(card);
	......
	if (extra_size > 0)
		card->private_data = (char *)card + sizeof(struct snd_card);
	*card_ret = card;
	return 0;

      __error_ctl:
	snd_device_free_all(card, SNDRV_DEV_CMD_PRE);
      __error:
	kfree(card);
  	return err;
}


```

######  2.1.1、snd_ctl_create 创建控制接口

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\control.c
/*
 * create control core:
 * called from init.c
 */
int snd_ctl_create(struct snd_card *card)
{
	static struct snd_device_ops ops = {
		.dev_free = snd_ctl_dev_free,
		.dev_register =	snd_ctl_dev_register,
		.dev_disconnect = snd_ctl_dev_disconnect,
	};

	if (snd_BUG_ON(!card))
		return -ENXIO;
	return snd_device_new(card, SNDRV_DEV_CONTROL, card, &ops);
}


G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\device.c

int snd_device_new(struct snd_card *card, snd_device_type_t type,
		   void *device_data, struct snd_device_ops *ops)
{
	struct snd_device *dev;

	if (snd_BUG_ON(!card || !device_data || !ops))
		return -ENXIO;
	dev = kzalloc(sizeof(*dev), GFP_KERNEL);
	if (dev == NULL) {
		snd_printk(KERN_ERR "Cannot allocate device\n");
		return -ENOMEM;
	}
	dev->card = card;
	dev->type = type;
	dev->state = SNDRV_DEV_BUILD;
	dev->device_data = device_data;
	dev->ops = ops;
	list_add(&dev->list, &card->devices);	/* add to the head of list */
	return 0;
}

G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\control.c
/*
 * registration of the control device
 */
static int snd_ctl_dev_register(struct snd_device *device)
{
	struct snd_card *card = device->device_data;
	int err, cardnum;
	char name[16];

	if (snd_BUG_ON(!card))
		return -ENXIO;
	cardnum = card->number;
	if (snd_BUG_ON(cardnum < 0 || cardnum >= SNDRV_CARDS))
		return -ENXIO;
	sprintf(name, "controlC%i", cardnum);
	if ((err = snd_register_device(SNDRV_DEVICE_TYPE_CONTROL, card, -1,
				       &snd_ctl_f_ops, card, name)) < 0)
		return err;
	return 0;
}
```

#####  2.2、snd_pcm_new

``` cpp
int snd_pcm_new(struct snd_card *card, const char *id, int device,
		int playback_count, int capture_count,
	        struct snd_pcm ** rpcm)
{
	struct snd_pcm *pcm;
	int err;
	static struct snd_device_ops ops = {
		.dev_free = snd_pcm_dev_free,
		.dev_register =	snd_pcm_dev_register,
		.dev_disconnect = snd_pcm_dev_disconnect,
	};

	if (snd_BUG_ON(!card))
		return -ENXIO;
	if (rpcm)
		*rpcm = NULL;
	pcm = kzalloc(sizeof(*pcm), GFP_KERNEL);
	if (pcm == NULL) {
		snd_printk(KERN_ERR "Cannot allocate PCM\n");
		return -ENOMEM;
	}
	pcm->card = card;
	pcm->device = device;
	if (id)
		strlcpy(pcm->id, id, sizeof(pcm->id));
	if ((err = snd_pcm_new_stream(pcm, SNDRV_PCM_STREAM_PLAYBACK, playback_count)) < 0) {
		snd_pcm_free(pcm);
		return err;
	}
	if ((err = snd_pcm_new_stream(pcm, SNDRV_PCM_STREAM_CAPTURE, capture_count)) < 0) {
		snd_pcm_free(pcm);
		return err;
	}
	mutex_init(&pcm->open_mutex);
	init_waitqueue_head(&pcm->open_wait);
	if ((err = snd_device_new(card, SNDRV_DEV_PCM, pcm, &ops)) < 0) {
		snd_pcm_free(pcm);
		return err;
	}
	if (rpcm)
		*rpcm = pcm;
	return 0;
}



static int snd_pcm_dev_register(struct snd_device *device)
{
	int cidx, err;
	struct snd_pcm_substream *substream;
	struct snd_pcm_notify *notify;
	char str[16];
	struct snd_pcm *pcm;
	struct device *dev;

	if (snd_BUG_ON(!device || !device->device_data))
		return -ENXIO;
	pcm = device->device_data;
	mutex_lock(&register_mutex);
	err = snd_pcm_add(pcm);
	if (err) {
		mutex_unlock(&register_mutex);
		return err;
	}
	for (cidx = 0; cidx < 2; cidx++) {
		int devtype = -1;
		if (pcm->streams[cidx].substream == NULL)
			continue;
		switch (cidx) {
		case SNDRV_PCM_STREAM_PLAYBACK:
			sprintf(str, "pcmC%iD%ip", pcm->card->number, pcm->device);
			devtype = SNDRV_DEVICE_TYPE_PCM_PLAYBACK;
			break;
		case SNDRV_PCM_STREAM_CAPTURE:
			sprintf(str, "pcmC%iD%ic", pcm->card->number, pcm->device);
			devtype = SNDRV_DEVICE_TYPE_PCM_CAPTURE;
			break;
		}
		/* device pointer to use, pcm->dev takes precedence if
		 * it is assigned, otherwise fall back to card's device
		 * if possible */
		dev = pcm->dev;
		if (!dev)
			dev = snd_card_get_device_link(pcm->card);
		/* register pcm */
		err = snd_register_device_for_dev(devtype, pcm->card,
						  pcm->device,
						  &snd_pcm_f_ops[cidx],
						  pcm, str, dev);
		if (err < 0) {
			list_del(&pcm->list);
			mutex_unlock(&register_mutex);
			return err;
		}
		snd_add_device_sysfs_file(devtype, pcm->card, pcm->device,
					  &pcm_attrs);
		for (substream = pcm->streams[cidx].substream; substream; substream = substream->next)
			snd_pcm_timer_init(substream);
	}

	list_for_each_entry(notify, &snd_pcm_notify_list, list)
		notify->n_register(pcm);

	mutex_unlock(&register_mutex);
	return 0;
}
```

一个pcm就是一个逻辑设备device
一个pcm两个stream，playback，capture

何时注册，使用结构体？
#####  2.3、snd_card_regsiter

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\init.c
int snd_card_register(struct snd_card *card)
{
	int err;

	if (snd_BUG_ON(!card))
		return -EINVAL;

	if (!card->card_dev) {
		card->card_dev = device_create(sound_class, card->dev,
					       MKDEV(0, 0), card,
					       "card%i", card->number);
		if (IS_ERR(card->card_dev))
			card->card_dev = NULL;
	}

	if ((err = snd_device_register_all(card)) < 0)
		return err;
	mutex_lock(&snd_card_mutex);
	if (snd_cards[card->number]) {
		/* already registered */
		mutex_unlock(&snd_card_mutex);
		return 0;
	}
	snd_card_set_id_no_lock(card, card->id[0] == '\0' ? NULL : card->id);
	snd_cards[card->number] = card;
	mutex_unlock(&snd_card_mutex);
	init_info_for_card(card);
#if defined(CONFIG_SND_MIXER_OSS) || defined(CONFIG_SND_MIXER_OSS_MODULE)
	if (snd_mixer_oss_notify_callback)
		snd_mixer_oss_notify_callback(card, SND_MIXER_OSS_NOTIFY_REGISTER);
#endif
	if (card->card_dev) {
		err = device_create_file(card->card_dev, &card_id_attrs);
		if (err < 0)
			return err;
		err = device_create_file(card->card_dev, &card_number_attrs);
		if (err < 0)
			return err;
	}

	return 0;
}

/*
 * register all the devices on the card.
 * called from init.c
 */
int snd_device_register_all(struct snd_card *card)
{
	struct snd_device *dev;
	int err;
	
	if (snd_BUG_ON(!card))
		return -ENXIO;
	list_for_each_entry(dev, &card->devices, list) {
		if (dev->state == SNDRV_DEV_BUILD && dev->ops->dev_register) {
			if ((err = dev->ops->dev_register(dev)) < 0)
				return err;
			dev->state = SNDRV_DEV_REGISTERED;
		}
	}
	return 0;
}

```
前面设置了snd_ctl_dev_register 和  snd_pcm_dev_register，dev->ops->dev_register(dev) 就会调用对应的
dev_register函数完成注册。

上面就是大概如何写一个声卡驱动程序。


####  （二）、ASoC音频驱动框架


#####  2.1、ASoC音频驱动框架概括
ASoC：ALSA System on Chip，是建立在标准 ALSA 驱动之上，为了更好支持嵌入式系统和应用于移动设备的音频 codec 的一套软件体系，它依赖于标准 ALSA 驱动框架。内核文档 Documentation/alsa/soc/overview.txt 中详细介绍了 ASoC 的设计初衷，这里不一一引用，简单陈述如下：

• 独立的 codec 驱动，标准的 ALSA 驱动框架里面 codec 驱动往往与 SoC/CPU 耦合过于紧密，不利于在多样化的平台/机器上移植复用
• 方便 codec 与 SoC 通过 PCM/I2S 总线建立链接
• 动态音频电源管理 DAPM，使得 codec 任何时候都工作在最低功耗状态，同时负责音频路由的创建
• POPs 和 click 音抑制弱化处理，在 ASoC 中通过正确的音频部件上下电次序来实现
• Machine 驱动的特定控制，比如耳机、麦克风的插拔检测，外放功放的开关
在概述中已经介绍了 ASoC 硬件设备驱动的三大构成：Codec、Platform 和 Machine，下面列举各驱动的功能构成：

> （1）SoC Codec Driver：

• Codec DAI 和 PCM 的配置信息
• Codec 的控制接口，如 I2C/SPI
• Mixer 和其他音频控件
• Codec 的音频接口函数，见 snd_soc_dai_ops 结构体定义
• DAPM 描述信息
• DAPM 事件处理句柄
• DAC 数字静音控制

> （2）ASoC Platform Driver： 包括 dma 和 cpu_dai 两部分：

• dma 驱动实现音频 dma 操作，具体见 snd_pcm_ops 结构体定义
• cpu_dai 驱动实现音频数字接口控制器的描述和配置

> （3）ASoC Machine Driver：

作为链结 Platform 和 Codec 的载体，它必须配置 dai_link 为音频数据链路指定 Platform 和 Codec
处理机器特有的音频控件和音频事件，例如回放时打开外放功放
硬件设备驱动相关结构体：

• snd_soc_codec_driver：音频编解码芯片描述及操作函数，如控件/微件/音频路由的描述信息、时钟配置、IO 控制等
• snd_soc_dai_driver：音频数据接口描述及操作函数，根据 codec 端和 soc 端，分为 codec_dai 和 cpu_dai
• snd_soc_platform_driver：音频 dma 设备描述及操作函数
• snd_soc_dai_link：音频链路描述及板级操作函数


#####  2.2、ASoC音频驱动框架图
![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/asoc_audio_arc.png)


#####  2.3、ASoC音频驱动代码分析
######  2.3.1、SoC Codec Driver

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\codecs\wm8960.c

static struct snd_soc_codec_driver soc_codec_dev_wm8960 = {
	.probe =	wm8960_probe,
	.remove =	wm8960_remove,
	.suspend =	wm8960_suspend,
	.resume =	wm8960_resume,
	.set_bias_level = wm8960_set_bias_level,
	.reg_cache_size = ARRAY_SIZE(wm8960_reg),
	.reg_word_size = sizeof(u16),
	.reg_cache_default = wm8960_reg,
};

#if defined(CONFIG_I2C) || defined(CONFIG_I2C_MODULE)
static __devinit int wm8960_i2c_probe(struct i2c_client *i2c,
				      const struct i2c_device_id *id)
{
	struct wm8960_priv *wm8960;
	int ret;

	wm8960 = kzalloc(sizeof(struct wm8960_priv), GFP_KERNEL);
	if (wm8960 == NULL)
		return -ENOMEM;

	i2c_set_clientdata(i2c, wm8960);
	wm8960->control_type = SND_SOC_I2C;
	wm8960->control_data = i2c;

	ret = snd_soc_register_codec(&i2c->dev,
			&soc_codec_dev_wm8960, &wm8960_dai, 1);
	if (ret < 0)
		kfree(wm8960);
	return ret;
}

static const struct i2c_device_id wm8960_i2c_id[] = {
	{ "wm8960", 0 },
	{ }
};
MODULE_DEVICE_TABLE(i2c, wm8960_i2c_id);

static struct i2c_driver wm8960_i2c_driver = {
	.driver = {
		.name = "wm8960-codec",
		.owner = THIS_MODULE,
	},
	.probe =    wm8960_i2c_probe,
	.remove =   __devexit_p(wm8960_i2c_remove),
	.id_table = wm8960_i2c_id,
};
#endif

static int __init wm8960_modinit(void)
{
	int ret = 0;
#if defined(CONFIG_I2C) || defined(CONFIG_I2C_MODULE)
	ret = i2c_add_driver(&wm8960_i2c_driver);
	if (ret != 0) {
		printk(KERN_ERR "Failed to register WM8960 I2C driver: %d\n",
		       ret);
	}
#endif
	return ret;
}
module_init(wm8960_modinit);
```

######  2.3.2、ASoC Platform Driver 
（1）dma部分：
``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\samsung\dma-wrapper.c
static struct snd_pcm_ops asoc_platform_ops = {
	.open		= asoc_platform_open,
	.close		= asoc_platform_close,
	.ioctl		= asoc_platform_ioctl,
	.hw_params	= asoc_platform_hw_params,
	.hw_free	= asoc_platform_hw_free,
	.prepare	= asoc_platform_prepare,
	.trigger	= asoc_platform_trigger,
	.pointer	= asoc_platform_pointer,
	.mmap		= asoc_platform_mmap,
};

static void asoc_platform_free_dma_buffers(struct snd_pcm *pcm)
{
	struct snd_soc_platform_driver *platform = asoc_get_platform(pcm);

	if (platform->pcm_free)
		platform->pcm_free(pcm);
}

static int asoc_platform_new(struct snd_card *card,
				struct snd_soc_dai *dai, struct snd_pcm *pcm)
{
	struct snd_soc_platform_driver *platform = asoc_get_platform(pcm);

	if (platform->pcm_new)
		platform->pcm_new(card, dai, pcm);

	return 0;
}

static struct snd_soc_platform_driver asoc_dma_platform = {
	.ops		= &asoc_platform_ops,
	.pcm_new	= asoc_platform_new,
	.pcm_free	= asoc_platform_free_dma_buffers,
};

static int __devinit samsung_asoc_platform_probe(struct platform_device *pdev)
{
	return snd_soc_register_platform(&pdev->dev, &asoc_dma_platform);
}

static int __devexit samsung_asoc_platform_remove(struct platform_device *pdev)
{
	snd_soc_unregister_platform(&pdev->dev);
	return 0;
}

static struct platform_driver asoc_platform_driver = {
	.driver = {
		.name = "samsung-audio",
		.owner = THIS_MODULE,
	},

	.probe = samsung_asoc_platform_probe,
	.remove = __devexit_p(samsung_asoc_platform_remove),
};

static int __init samsung_asoc_init(void)
{
	return platform_driver_register(&asoc_platform_driver);
}
module_init(samsung_asoc_init);
```
（2）cpu_dai部分：

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\samsung\i2s.c

static __devinit int samsung_i2s_probe(struct platform_device *pdev)
{
	u32 dma_pl_chan, dma_cp_chan, dma_pl_sec_chan;
	struct i2s_dai *pri_dai, *sec_dai = NULL;
	struct s3c_audio_pdata *i2s_pdata;
	struct samsung_i2s *i2s_cfg;
	struct resource *res;
	u32 regs_base, quirks;
	int ret = 0;

	/* Call during Seconday interface registration */
	if (pdev->id >= SAMSUNG_I2S_SECOFF) {
		sec_dai = dev_get_drvdata(&pdev->dev);
		snd_soc_register_dai(&sec_dai->pdev->dev,
			&sec_dai->i2s_dai_drv);
		return 0;
	}

	i2s_pdata = pdev->dev.platform_data;

	res = platform_get_resource(pdev, IORESOURCE_DMA, 0);

	dma_pl_chan = res->start;

	res = platform_get_resource(pdev, IORESOURCE_DMA, 1);
	
	dma_cp_chan = res->start;

	res = platform_get_resource(pdev, IORESOURCE_DMA, 2);
	if (res)
		dma_pl_sec_chan = res->start;
	else
		dma_pl_sec_chan = 0;

	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    ......
	regs_base = res->start;

	i2s_cfg = &i2s_pdata->type.i2s;
	quirks = i2s_cfg->quirks;

	pri_dai = i2s_alloc_dai(pdev, false);
	

	pri_dai->dma_playback.dma_addr = regs_base + I2STXD;
	pri_dai->dma_capture.dma_addr = regs_base + I2SRXD;
	pri_dai->dma_playback.client =
		(struct s3c2410_dma_client *)&pri_dai->dma_playback;
	pri_dai->dma_capture.client =
		(struct s3c2410_dma_client *)&pri_dai->dma_capture;
	pri_dai->dma_playback.channel = dma_pl_chan;
	pri_dai->dma_capture.channel = dma_cp_chan;
	pri_dai->src_clk = i2s_cfg->src_clk;
	pri_dai->dma_playback.dma_size = 4;
	pri_dai->dma_capture.dma_size = 4;
	pri_dai->base = regs_base;
	pri_dai->quirks = quirks;
	if (pdev->id == 0) {
		pri_dai->audss_clk_enable = audss_clk_enable;
		pri_dai->audss_suspend = audss_suspend;
		pri_dai->audss_resume = audss_resume;
	}

	if (quirks & QUIRK_PRI_6CHAN)
		pri_dai->i2s_dai_drv.playback.channels_max = 6;

	if (quirks & QUIRK_SEC_DAI) {
		sec_dai = i2s_alloc_dai(pdev, true);
		if (!sec_dai) {
			dev_err(&pdev->dev, "Unable to alloc I2S_sec\n");
			ret = -ENOMEM;
			goto err2;
		}
		sec_dai->dma_playback.dma_addr = regs_base + I2STXDS;
		sec_dai->dma_playback.client =
			(struct s3c2410_dma_client *)&sec_dai->dma_playback;
		/* Use iDMA always if SysDMA not provided */
		sec_dai->dma_playback.channel = dma_pl_sec_chan ? : -1;
		sec_dai->src_clk = i2s_cfg->src_clk;
		sec_dai->dma_playback.dma_size = 4;
		sec_dai->base = regs_base;
		sec_dai->quirks = quirks;

		sec_dai->pri_dai = pri_dai;
		pri_dai->sec_dai = sec_dai;
		if (pdev->id == 0) {
			sec_dai->audss_clk_enable = audss_clk_enable;
			sec_dai->audss_suspend = audss_suspend;
			sec_dai->audss_resume = audss_resume;
		}
	}

    ......

	snd_soc_register_dai(&pri_dai->pdev->dev, &pri_dai->i2s_dai_drv);

	return 0;
    ......

	return ret;
}

static struct platform_driver samsung_i2s_driver = {
	.probe  = samsung_i2s_probe,
	.remove = samsung_i2s_remove,
	.driver = {
		.name = "samsung-i2s",
		.owner = THIS_MODULE,
	},
};

static int __init samsung_i2s_init(void)
{
	return platform_driver_register(&samsung_i2s_driver);
}
module_init(samsung_i2s_init);

```

######  2.3.3、ASoC Machine Driver

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\samsung\tiny4412_wm8960.c

static struct snd_soc_dai_link s3c2440_uda1341_dai_link = {
	.name = "100ask_UDA1341",
	.stream_name = "100ask_UDA1341",
	.codec_name = "wm8960-codec.0-001a",
	.codec_dai_name = "wm8960-hifi",
	.cpu_dai_name = "samsung-i2s.0",
	.ops = &s3c2440_uda1341_ops,
	.platform_name	= "samsung-audio",
	.init = tiny4412_wm8960_machine_init,
};


static struct snd_soc_card myalsa_card = {
	.name = "S3C2440_UDA1341",
	.owner = THIS_MODULE,
	.dai_link = &s3c2440_uda1341_dai_link,
	.num_links = 1,
};

static void asoc_release(struct device * dev)
{
}

static struct platform_device asoc_dev = {
    .name         = "soc-audio",
    .id       = -1,
    .dev = { 
    	.release = asoc_release, 
	},
};

static int s3c2440_uda1341_init(void)
{
	platform_set_drvdata(&asoc_dev, &myalsa_card);
    platform_device_register(&asoc_dev);    
    return 0;
}

```

#####  2.4、ASoC音频驱动代码分析
latform_device 还有个好伙伴 platform_driver 跟它配对。而 .name="soc-audio" 的 platform_driver 定义在 soc-core.c 中：
两者匹配后，soc_probe() 会被调用，继而调用 snd_soc_register_card() 注册声卡。由于该过程很冗长，这里不一一贴代码分析了，但整个流程是比较简单的，流程图如下：

![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/snd_soc_register_card.png)


- 取出 platform_device 的私有数据，该私有数据就是 snd_soc_card ；
- snd_soc_register_card() 为每个 dai_link 分配一个 snd_soc_pcm_runtime 实例，别忘了之前提过 snd_soc_pcm_runtime 是 ASoC 的桥梁，保存着 codec、codec_dai、cpu_dai、platform 等硬件设备实例。
- 随后的工作都在 snd_soc_instantiate_card() 进行：
- 遍历 dai_list、codec_list、platform_list 链表，为每个音频链路找到对应的 cpu_dai、codec_dai、codec、platform；找到的 cpu_dai、codec_dai、codec、platform 保存到 snd_soc_pcm_runtime ，完成音频链路的设备绑定；
- 调用 snd_card_create() 创建声卡；
- soc_probe_dai_link() 依次回调 cpu_dai、codec、platform、codec_dai 的 probe() 函数，完成各音频设备的初始化，随后调用 soc_new_pcm() 创建 pcm 逻辑设备（因为涉及到本系列的重点内容，后面具体分析这个函数）；
- 最后调用 snd_card_register() 注册声卡。
soc_new_pcm 源码分析：

``` cpp
/* create a new pcm */
int soc_new_pcm(struct snd_soc_pcm_runtime *rtd, int num)
{
    struct snd_soc_codec *codec = rtd->codec;
    struct snd_soc_platform *platform = rtd->platform;
    struct snd_soc_dai *codec_dai = rtd->codec_dai;
    struct snd_soc_dai *cpu_dai = rtd->cpu_dai;
    struct snd_pcm_ops *soc_pcm_ops = &rtd->ops;
    struct snd_pcm *pcm;
    char new_name[64];
    int ret = 0, playback = 0, capture = 0;

    // 初始化 snd_soc_pcm_runtime 的 ops 字段，成员函数其实依次调用 machine、codec_dai、cpu_dai、platform 的回调；如 soc_pcm_hw_params：
    // |-> rtd->dai_link->ops->hw_params() 
    // |-> codec_dai->driver->ops->hw_params() 
    // |-> cpu_dai->driver->ops->hw_params()
    // |-> platform->driver->ops->hw_params()
    // 在这里把底层硬件的操作接口抽象起来，pcm native 不用知道底层硬件细节
    soc_pcm_ops->open   = soc_pcm_open;
    soc_pcm_ops->close  = soc_pcm_close;
    soc_pcm_ops->hw_params  = soc_pcm_hw_params;
    soc_pcm_ops->hw_free    = soc_pcm_hw_free;
    soc_pcm_ops->prepare    = soc_pcm_prepare;
    soc_pcm_ops->trigger    = soc_pcm_trigger;
    soc_pcm_ops->pointer    = soc_pcm_pointer;

    /* check client and interface hw capabilities */
    snprintf(new_name, sizeof(new_name), "%s %s-%d",
            rtd->dai_link->stream_name, codec_dai->name, num);

    if (codec_dai->driver->playback.channels_min)
        playback = 1;
    if (codec_dai->driver->capture.channels_min)
        capture = 1;

    // 创建 pcm 逻辑设备
    ret = snd_pcm_new(rtd->card->snd_card, new_name,
            num, playback, capture, &pcm);
    if (ret < 0) {
        printk(KERN_ERR "asoc: can't create pcm for codec %s\n", codec->name);
        return ret;
    }

    /* DAPM dai link stream work */
    INIT_DELAYED_WORK(&rtd->delayed_work, close_delayed_work);

    rtd->pcm = pcm;
    pcm->private_data = rtd; // pcm 的私有数据指向 snd_soc_pcm_runtime
    if (platform->driver->ops) {
        // 初始化 snd_soc_pcm_runtime 的 ops 字段，这些与 pcm_dma 操作相关，一般我们只用留意 pointer 回调
        soc_pcm_ops->mmap = platform->driver->ops->mmap;
        soc_pcm_ops->pointer = platform->driver->ops->pointer;
        soc_pcm_ops->ioctl = platform->driver->ops->ioctl;
        soc_pcm_ops->copy = platform->driver->ops->copy;
        soc_pcm_ops->silence = platform->driver->ops->silence;
        soc_pcm_ops->ack = platform->driver->ops->ack;
        soc_pcm_ops->page = platform->driver->ops->page;
    }

    if (playback)
        snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_PLAYBACK, soc_pcm_ops); // 把 soc_pcm_ops 赋给 playback substream 的 ops 字段

    if (capture)
        snd_pcm_set_ops(pcm, SNDRV_PCM_STREAM_CAPTURE, soc_pcm_ops); // 把 soc_pcm_ops 赋给 capture substream 的 ops 字段

    // 回调 dma 驱动的 pcm_new()，进行 dma buffer 内存分配和 dma 设备初始化
    if (platform->driver->pcm_new) {
        ret = platform->driver->pcm_new(rtd);
        if (ret < 0) {
            pr_err("asoc: platform pcm constructor failed\n");
            return ret;
        }
    }

    pcm->private_free = platform->driver->pcm_free;
    printk(KERN_INFO "asoc: %s <-> %s mapping ok\n", codec_dai->name,
        cpu_dai->name);
    return ret;
}

```
可见 soc_new_pcm() 最主要的工作是创建 pcm 逻辑设备，创建回放子流和录制子流实例，并初始化回放子流和录制子流的 pcm 操作函数（数据搬运时，需要调用这些函数来驱动 codec、codec_dai、cpu_dai、dma 设备工作）。


####  （三）、 声卡控制之kcontrol

##### （1）、snd_kcontrol结构体
首先看看录音电路原理图：

![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/audio_codec_elec.png)
我们对着mic说话，声音会经过一系列线路进入声卡芯片的LINPUT1。看看声卡芯片wm8960

![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/wm8960.png)

LINPUT1 经过一系列部件经过ADC，对于这些部件我们需要设置相应的寄存器来启动他们，让声音转换成数字信号。
怎么设置这些寄存器呢？有没有统一的接口来设置他们？既是Kcontrol。

学习Kcontrol之前我们先来想想：
> 一个芯片有多个寄存器 
> 一个寄存器某些位表示某个功能 
> kcontrol有自己的读写函数
> 
> 一个声卡有多个kcontrol 
> 一个kcontrol对应一个功能，比如：音量，开关声音。
>  kcontrol有函数来设置功能

![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/snd_kcontrol.png)

##### （2）、tinymix 设置kcontrol

![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/tinymix_kcontrol.png)
可以看到有50个kcontrol，我们可以使用tinymix 来设置kcontrol
tinymix "Left Input Mixer Boost Switch" 1
tinymix "Capture Switch" 0

##### （3）、snd_soc_add_controls
首先看看snd_soc_add_controls
``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\codecs\wm8960.c
	snd_soc_add_controls(codec, wm8960_snd_controls,
				     ARRAY_SIZE(wm8960_snd_controls));
				 
```

```cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\soc-core.c
int snd_soc_add_controls(struct snd_soc_codec *codec,
	const struct snd_kcontrol_new *controls, int num_controls)
{
	struct snd_card *card = codec->card->snd_card;
	int err, i;

	for (i = 0; i < num_controls; i++) {
		const struct snd_kcontrol_new *control = &controls[i];
		err = snd_ctl_add(card, snd_soc_cnew(control, codec,
						     control->name,
						     codec->name_prefix));
		if (err < 0) {
			dev_err(codec->dev, "%s: Failed to add %s: %d\n",
				codec->name, control->name, err);
			return err;
		}
	}

	return 0;
}

```
snd_kcontrol 的值由谁提供。。？由对应芯片驱动程序提供,如WM8960芯片：

```cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\codecs\wm8960.c
static const struct snd_kcontrol_new wm8960_lin_boost[] = {
SOC_DAPM_SINGLE("LINPUT2 Switch", WM8960_LINPATH, 6, 1, 0),
SOC_DAPM_SINGLE("LINPUT3 Switch", WM8960_LINPATH, 7, 1, 0),
SOC_DAPM_SINGLE("LINPUT1 Switch", WM8960_LINPATH, 8, 1, 0),
};

```

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\soc-core.c

struct snd_kcontrol *snd_soc_cnew(const struct snd_kcontrol_new *_template,
				  void *data, char *long_name,
				  const char *prefix)
{
	struct snd_kcontrol_new template;
	struct snd_kcontrol *kcontrol;
	char *name = NULL;
	int name_len;

	memcpy(&template, _template, sizeof(template));
	template.index = 0;

	if (!long_name)
		long_name = template.name;

	if (prefix) {
		name_len = strlen(long_name) + strlen(prefix) + 2;
		name = kmalloc(name_len, GFP_ATOMIC);
		if (!name)
			return NULL;

		snprintf(name, name_len, "%s %s", prefix, long_name);

		template.name = name;
	} else {
		template.name = long_name;
	}

	kcontrol = snd_ctl_new1(&template, data);

	kfree(name);

	return kcontrol;
}

G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\core\control.c

struct snd_kcontrol *snd_ctl_new1(const struct snd_kcontrol_new *ncontrol,
				  void *private_data)
{
	struct snd_kcontrol kctl;
	unsigned int access;
	
	if (snd_BUG_ON(!ncontrol || !ncontrol->info))
		return NULL;
	memset(&kctl, 0, sizeof(kctl));
	kctl.id.iface = ncontrol->iface;
	kctl.id.device = ncontrol->device;
	kctl.id.subdevice = ncontrol->subdevice;
	if (ncontrol->name) {
		strlcpy(kctl.id.name, ncontrol->name, sizeof(kctl.id.name));
		if (strcmp(ncontrol->name, kctl.id.name) != 0)
			snd_printk(KERN_WARNING
				   "Control name '%s' truncated to '%s'\n",
				   ncontrol->name, kctl.id.name);
	}
	kctl.id.index = ncontrol->index;
	kctl.count = ncontrol->count ? ncontrol->count : 1;
	access = ncontrol->access == 0 ? SNDRV_CTL_ELEM_ACCESS_READWRITE :
		 (ncontrol->access & (SNDRV_CTL_ELEM_ACCESS_READWRITE|
				      SNDRV_CTL_ELEM_ACCESS_INACTIVE|
				      SNDRV_CTL_ELEM_ACCESS_TLV_READWRITE|
				      SNDRV_CTL_ELEM_ACCESS_TLV_COMMAND|
				      SNDRV_CTL_ELEM_ACCESS_TLV_CALLBACK));
	kctl.info = ncontrol->info;
	kctl.get = ncontrol->get;
	kctl.put = ncontrol->put;
	kctl.tlv.p = ncontrol->tlv.p;
	kctl.private_value = ncontrol->private_value;
	kctl.private_data = private_data;
	return snd_ctl_new(&kctl, access);
}

static struct snd_kcontrol *snd_ctl_new(struct snd_kcontrol *control,
					unsigned int access)
{
	struct snd_kcontrol *kctl;
	unsigned int idx;
	
	if (snd_BUG_ON(!control || !control->count))
		return NULL;

	if (control->count > MAX_CONTROL_COUNT)
		return NULL;

	kctl = kzalloc(sizeof(*kctl) + sizeof(struct snd_kcontrol_volatile) * control->count, GFP_KERNEL);
	if (kctl == NULL) {
		snd_printk(KERN_ERR "Cannot allocate control instance\n");
		return NULL;
	}
	*kctl = *control;
	for (idx = 0; idx < kctl->count; idx++)
		kctl->vd[idx].access = access;
	return kctl;
}

```


####  （四）、 DAPM_widget_route_path


概念：Dynamic Audio Power Management，动态音频电源管理，为移动 Linux 设备设计，使得音频系统任何时候都工作在最低功耗状态。

目的：使能最少的必要的部件，令音频系统正常工作。

原理：当音频路径发生改变（比如上层使用 tinymix 工具设置音频通路）时，或发生数据流事件（比如启动或停止播放）时，都会触发 dapm 去遍历所有邻近的音频部件，检查是否存在完整的音频路径（complete path：满足条件的音频路径，该路径上任意一个部件往前遍历能到达输入端点如 DAC/Mic/Linein，往后遍历能到达输出端点如 ADC/HP/SPK），如果存在完整的音频路径，则该路径上面的所有部件都是需要上电的，其他部件则下电。

部件上下电都是 dapm 根据策略自主控制的，外部无法干预，可以说 dapm 是一个专门为音频系统设计的自成体系的电源管理模块，独立于 Linux 电源管理之外。即使 SoC 休眠了，Codec 仍可以在正常工作，试想下这个情景：语音通话，modem_dai 连接到 codec_dai，语音数据不经过 SoC，因此这种情形下 SoC 可以进入睡眠以降低功耗，只保持 Codec 正常工作就行了。


![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/LINPUT1.png)



在这个例子中，codec 中的音频通路是：LINPUT1>LEFT Boost Mixer>ADC；

Mixer有多个输入源，只要其中的某个开关使能，顺便把其他三个也使能

##### （1）、widget Route Path概念：
![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/dapm_widget_route_path.png)


####  （五）、DAPM的kcontrol注册过程


![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/snd_soc_dapm_new_controls.png)



##### （1）、snd_soc_dapm_new_controls

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\soc-dapm.c
int snd_soc_dapm_new_controls(struct snd_soc_dapm_context *dapm,
	const struct snd_soc_dapm_widget *widget,
	int num)
{
	int i, ret;

	for (i = 0; i < num; i++) {
		ret = snd_soc_dapm_new_control(dapm, widget);
		if (ret < 0) {
			dev_err(dapm->dev,
				"ASoC: Failed to create DAPM control %s: %d\n",
				widget->name, ret);
			return ret;
		}
		widget++;
	}
	return 0;
}

int snd_soc_dapm_new_control(struct snd_soc_dapm_context *dapm,
	const struct snd_soc_dapm_widget *widget)
{
	struct snd_soc_dapm_widget *w;
	size_t name_len;

	if ((w = dapm_cnew_widget(widget)) == NULL)
		return -ENOMEM;

	name_len = strlen(widget->name) + 1;
	if (dapm->codec && dapm->codec->name_prefix)
		name_len += 1 + strlen(dapm->codec->name_prefix);
	w->name = kmalloc(name_len, GFP_KERNEL);
	if (w->name == NULL) {
		kfree(w);
		return -ENOMEM;
	}
	if (dapm->codec && dapm->codec->name_prefix)
		snprintf(w->name, name_len, "%s %s",
			dapm->codec->name_prefix, widget->name);
	else
		snprintf(w->name, name_len, "%s", widget->name);

	dapm->n_widgets++;
	w->dapm = dapm;
	w->codec = dapm->codec;
	INIT_LIST_HEAD(&w->sources);
	INIT_LIST_HEAD(&w->sinks);
	INIT_LIST_HEAD(&w->list);
	list_add(&w->list, &dapm->card->widgets);

	/* machine layer set ups unconnected pins and insertions */
	w->connected = 1;
	return 0;
}

```

##### （2）、snd_soc_dapm_new_widgets

``` cpp
int snd_soc_dapm_new_widgets(struct snd_soc_dapm_context *dapm)
{
	struct snd_soc_dapm_widget *w;
	unsigned int val;

	list_for_each_entry(w, &dapm->card->widgets, list)
	{
		if (w->new)
			continue;

		if (w->num_kcontrols) {
			w->kcontrols = kzalloc(w->num_kcontrols *
						sizeof(struct snd_kcontrol *),
						GFP_KERNEL);
			if (!w->kcontrols)
				return -ENOMEM;
		}

		switch(w->id) {
		case snd_soc_dapm_switch:
		case snd_soc_dapm_mixer:
		case snd_soc_dapm_mixer_named_ctl:
			w->power_check = dapm_generic_check_power;
			dapm_new_mixer(w);
			break;
		case snd_soc_dapm_mux:
		case snd_soc_dapm_virt_mux:
		case snd_soc_dapm_value_mux:
			w->power_check = dapm_generic_check_power;
			dapm_new_mux(w);
			break;
		case snd_soc_dapm_adc:
		case snd_soc_dapm_aif_out:
			w->power_check = dapm_adc_check_power;
			break;
		case snd_soc_dapm_dac:
		case snd_soc_dapm_aif_in:
			w->power_check = dapm_dac_check_power;
			break;
		case snd_soc_dapm_pga:
		case snd_soc_dapm_out_drv:
			w->power_check = dapm_generic_check_power;
			dapm_new_pga(w);
			break;
		case snd_soc_dapm_input:
		case snd_soc_dapm_output:
		case snd_soc_dapm_micbias:
		case snd_soc_dapm_spk:
		case snd_soc_dapm_hp:
		case snd_soc_dapm_mic:
		case snd_soc_dapm_line:
			w->power_check = dapm_generic_check_power;
			break;
		case snd_soc_dapm_supply:
			w->power_check = dapm_supply_check_power;
		case snd_soc_dapm_vmid:
		case snd_soc_dapm_pre:
		case snd_soc_dapm_post:
			break;
		}

		/* Read the initial power state from the device */
		if (w->reg >= 0) {
			val = snd_soc_read(w->codec, w->reg);
			val &= 1 << w->shift;
			if (w->invert)
				val = !val;

			if (val)
				w->power = 1;
		}

		w->new = 1;

		dapm_debugfs_add_widget(w);
	}

	dapm_power_widgets(dapm, SND_SOC_DAPM_STREAM_NOP);
	return 0;
}
```
a. 对于普通的snd_kcontrol:
snd_soc_add_controls : snd_kcontrol_new构造出snd_kcontrol, 放入card->controls链表

b. 对于DAPM的snd_kcontrol, 分2步:
b.1 snd_soc_dapm_new_controls  // 把widget放入card->widgets链表
b.2 在注册machine驱动时, 导致如下调用: 
    soc_probe_dai_link > soc_post_component_init > snd_soc_dapm_new_widgets

snd_soc_dapm_new_widgets:
   对于每一个widget, 设置它的power_check函数(用来判断该widget是否应该上电)  
   对于mixer widget, 取出其中的snd_kcontrol_new构造出snd_kcontrol, 放入card->controls链表
   对于mux widget,它只有一个snd_kcontrol_new, 构造出snd_kcontrol, 放入card->controls链表


####  （六）、route_path添加过程分析


##### （1）、snd_soc_dapm_add_routes

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\codecs\wm8960.c
static const struct snd_soc_dapm_route audio_paths[] = {
	{ "Left Boost Mixer", "LINPUT1 Switch", "LINPUT1" },
	{ "Left Boost Mixer", "LINPUT2 Switch", "LINPUT2" },
	{ "Left Boost Mixer", "LINPUT3 Switch", "LINPUT3" },

	{ "Left Input Mixer", "Boost Switch", "Left Boost Mixer", },
	{ "Left Input Mixer", NULL, "LINPUT1", },  /* Really Boost Switch */
	{ "Left Input Mixer", NULL, "LINPUT2" },
	{ "Left Input Mixer", NULL, "LINPUT3" },

	{ "Right Boost Mixer", "RINPUT1 Switch", "RINPUT1" },
	{ "Right Boost Mixer", "RINPUT2 Switch", "RINPUT2" },
	{ "Right Boost Mixer", "RINPUT3 Switch", "RINPUT3" },

	{ "Right Input Mixer", "Boost Switch", "Right Boost Mixer", },
	{ "Right Input Mixer", NULL, "RINPUT1", },  /* Really Boost Switch */
	{ "Right Input Mixer", NULL, "RINPUT2" },
	{ "Right Input Mixer", NULL, "LINPUT3" },

	{ "Left ADC", NULL, "Left Input Mixer" },
	{ "Right ADC", NULL, "Right Input Mixer" },
	......
}

int snd_soc_dapm_add_routes(struct snd_soc_dapm_context *dapm,
			    const struct snd_soc_dapm_route *route, int num)
{
	int i, ret;

	for (i = 0; i < num; i++) {
		ret = snd_soc_dapm_add_route(dapm, route);
		if (ret < 0) {
			dev_err(dapm->dev, "Failed to add route %s->%s\n",
				route->source, route->sink);
			return ret;
		}
		route++;
	}

	return 0;
}


static int snd_soc_dapm_add_route(struct snd_soc_dapm_context *dapm,
				  const struct snd_soc_dapm_route *route)
{
	struct snd_soc_dapm_path *path;
	struct snd_soc_dapm_widget *wsource = NULL, *wsink = NULL, *w;
	struct snd_soc_dapm_widget *wtsource = NULL, *wtsink = NULL;
	const char *sink;
	const char *control = route->control;
	const char *source;
	char prefixed_sink[80];
	char prefixed_source[80];
	int ret = 0;

	if (dapm->codec && dapm->codec->name_prefix) {
		snprintf(prefixed_sink, sizeof(prefixed_sink), "%s %s",
			 dapm->codec->name_prefix, route->sink);
		sink = prefixed_sink;
		snprintf(prefixed_source, sizeof(prefixed_source), "%s %s",
			 dapm->codec->name_prefix, route->source);
		source = prefixed_source;
	} else {
		sink = route->sink;
		source = route->source;
	}

	/*
	 * find src and dest widgets over all widgets but favor a widget from
	 * current DAPM context
	 */
	list_for_each_entry(w, &dapm->card->widgets, list) {
		if (!wsink && !(strcmp(w->name, sink))) {
			wtsink = w;
			if (w->dapm == dapm)
				wsink = w;
			continue;
		}
		if (!wsource && !(strcmp(w->name, source))) {
			wtsource = w;
			if (w->dapm == dapm)
				wsource = w;
		}
	}
	/* use widget from another DAPM context if not found from this */
	if (!wsink)
		wsink = wtsink;
	if (!wsource)
		wsource = wtsource;

	if (wsource == NULL || wsink == NULL)
		return -ENODEV;

	path = kzalloc(sizeof(struct snd_soc_dapm_path), GFP_KERNEL);
	if (!path)
		return -ENOMEM;

	path->source = wsource;
	path->sink = wsink;
	path->connected = route->connected;
	INIT_LIST_HEAD(&path->list);
	INIT_LIST_HEAD(&path->list_source);
	INIT_LIST_HEAD(&path->list_sink);

	/* check for external widgets */
	if (wsink->id == snd_soc_dapm_input) {
		if (wsource->id == snd_soc_dapm_micbias ||
			wsource->id == snd_soc_dapm_mic ||
			wsource->id == snd_soc_dapm_line ||
			wsource->id == snd_soc_dapm_output)
			wsink->ext = 1;
	}
	if (wsource->id == snd_soc_dapm_output) {
		if (wsink->id == snd_soc_dapm_spk ||
			wsink->id == snd_soc_dapm_hp ||
			wsink->id == snd_soc_dapm_line ||
			wsink->id == snd_soc_dapm_input)
			wsource->ext = 1;
	}

	/* connect static paths */
	if (control == NULL) {
		list_add(&path->list, &dapm->card->paths);
		list_add(&path->list_sink, &wsink->sources);
		list_add(&path->list_source, &wsource->sinks);
		path->connect = 1;
		return 0;
	}

	/* connect dynamic paths */
	switch (wsink->id) {
	case snd_soc_dapm_adc:
	case snd_soc_dapm_dac:
	case snd_soc_dapm_pga:
	case snd_soc_dapm_out_drv:
	case snd_soc_dapm_input:
	case snd_soc_dapm_output:
	case snd_soc_dapm_micbias:
	case snd_soc_dapm_vmid:
	case snd_soc_dapm_pre:
	case snd_soc_dapm_post:
	case snd_soc_dapm_supply:
	case snd_soc_dapm_aif_in:
	case snd_soc_dapm_aif_out:
		list_add(&path->list, &dapm->card->paths);
		list_add(&path->list_sink, &wsink->sources);
		list_add(&path->list_source, &wsource->sinks);
		path->connect = 1;
		return 0;
	case snd_soc_dapm_mux:
	case snd_soc_dapm_virt_mux:
	case snd_soc_dapm_value_mux:
		ret = dapm_connect_mux(dapm, wsource, wsink, path, control,
			&wsink->kcontrol_news[0]);
		if (ret != 0)
			goto err;
		break;
	case snd_soc_dapm_switch:
	case snd_soc_dapm_mixer:
	case snd_soc_dapm_mixer_named_ctl:
		ret = dapm_connect_mixer(dapm, wsource, wsink, path, control);
		if (ret != 0)
			goto err;
		break;
	case snd_soc_dapm_hp:
	case snd_soc_dapm_mic:
	case snd_soc_dapm_line:
	case snd_soc_dapm_spk:
		list_add(&path->list, &dapm->card->paths);
		list_add(&path->list_sink, &wsink->sources);
		list_add(&path->list_source, &wsource->sinks);
		path->connect = 0;
		return 0;
	}
	return 0;

err:
	dev_warn(dapm->dev, "asoc: no dapm match for %s --> %s --> %s\n",
		 source, control, sink);
	kfree(path);
	return ret;
}
```

![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/snd_soc_dapm_add_route.png)


##### （2）、route分类

#####  2.1、常规route

> {"sink"，NULL，"source"}          path->connect = 1
> 
#####  2.2、sink widget是mixer
> {"Mux"，name1，"source1"}    
> {"Mux"，name2，"source2"}    
(name1,name2)snd_kcontrol_new的名，可以通过操作某个kcontrol来打开某条path
![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/route_sink_widget_mixer.png)

#####  2.3、sink widget是Mux
> {"Mux"，value1，"source1"}    
> {"Mux"，value1，"source2"}    
![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/route_sink_widget_mux.png)

####  （七）、DAPM的情景分析_构造过程

a. kcontrol,route,path注册过程情景演示
![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/dapm_how_to_create_and_use.png)

#####   （1）、普通的kcontrol

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\soc-core.c
int snd_soc_add_controls(struct snd_soc_codec *codec,
	const struct snd_kcontrol_new *controls, int num_controls)
{
	struct snd_card *card = codec->card->snd_card;
	int err, i;

	for (i = 0; i < num_controls; i++) {
		const struct snd_kcontrol_new *control = &controls[i];
		err = snd_ctl_add(card, snd_soc_cnew(control, codec,
						     control->name,
						     codec->name_prefix));
		if (err < 0) {
			dev_err(codec->dev, "%s: Failed to add %s: %d\n",
				codec->name, control->name, err);
			return err;
		}
	}
	
```

#####   （2）、注册widget

```cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\codecs\wm8960.c
static int wm8960_add_widgets(struct snd_soc_codec *codec)
{
	struct wm8960_data *pdata = codec->dev->platform_data;
	struct wm8960_priv *wm8960 = snd_soc_codec_get_drvdata(codec);
	struct snd_soc_dapm_context *dapm = &codec->dapm;
	struct snd_soc_dapm_widget *w;

	snd_soc_dapm_new_controls(dapm, wm8960_dapm_widgets,
				  ARRAY_SIZE(wm8960_dapm_widgets));

	snd_soc_dapm_add_routes(dapm, audio_paths, ARRAY_SIZE(audio_paths));

	/* In capless mode OUT3 is used to provide VMID for the
	 * headphone outputs, otherwise it is used as a mono mixer.
	 */
	if (pdata && pdata->capless) {
		snd_soc_dapm_new_controls(dapm, wm8960_dapm_widgets_capless,
					  ARRAY_SIZE(wm8960_dapm_widgets_capless));

		snd_soc_dapm_add_routes(dapm, audio_paths_capless,
					ARRAY_SIZE(audio_paths_capless));
	} else {
		snd_soc_dapm_new_controls(dapm, wm8960_dapm_widgets_out3,
					  ARRAY_SIZE(wm8960_dapm_widgets_out3));

		snd_soc_dapm_add_routes(dapm, audio_paths_out3,
					ARRAY_SIZE(audio_paths_out3));
	}

	/* We need to power up the headphone output stage out of
	 * sequence for capless mode.  To save scanning the widget
	 * list each time to find the desired power state do so now
	 * and save the result.
	 */
	list_for_each_entry(w, &codec->card->widgets, list) {
		if (w->dapm != &codec->dapm)
			continue;
		if (strcmp(w->name, "LOUT1 PGA") == 0)
			wm8960->lout1 = w;
		if (strcmp(w->name, "ROUT1 PGA") == 0)
			wm8960->rout1 = w;
		if (strcmp(w->name, "OUT3 VMID") == 0)
			wm8960->out3 = w;
	}
	
	return 0;
}


G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\soc-core.c
int snd_soc_dapm_new_controls(struct snd_soc_dapm_context *dapm,
	const struct snd_soc_dapm_widget *widget,
	int num)
{
	int i, ret;

	for (i = 0; i < num; i++) {
		ret = snd_soc_dapm_new_control(dapm, widget);
		if (ret < 0) {
			dev_err(dapm->dev,
				"ASoC: Failed to create DAPM control %s: %d\n",
				widget->name, ret);
			return ret;
		}
		widget++;
	}
	return 0;
}
```

#####   （3）、route=>path注册

``` cpp
G:\Android-5.0.2\linux-3.0.86-20170221\linux-3.0.86\sound\soc\codecs\wm8960.c
//	snd_soc_dapm_add_routes(dapm, audio_paths, ARRAY_SIZE(audio_paths));

static const struct snd_soc_dapm_route audio_paths[] = {
	{ "Left Boost Mixer", "LINPUT1 Switch", "LINPUT1" },
	{ "Left Boost Mixer", "LINPUT2 Switch", "LINPUT2" },
	{ "Left Boost Mixer", "LINPUT3 Switch", "LINPUT3" },

	{ "Left Input Mixer", "Boost Switch", "Left Boost Mixer", },
	{ "Left Input Mixer", NULL, "LINPUT1", },  /* Really Boost Switch */
	{ "Left Input Mixer", NULL, "LINPUT2" },
	{ "Left Input Mixer", NULL, "LINPUT3" },

	{ "Right Boost Mixer", "RINPUT1 Switch", "RINPUT1" },
	{ "Right Boost Mixer", "RINPUT2 Switch", "RINPUT2" },
	{ "Right Boost Mixer", "RINPUT3 Switch", "RINPUT3" },

	{ "Right Input Mixer", "Boost Switch", "Right Boost Mixer", },
	{ "Right Input Mixer", NULL, "RINPUT1", },  /* Really Boost Switch */
	{ "Right Input Mixer", NULL, "RINPUT2" },
	{ "Right Input Mixer", NULL, "LINPUT3" },

	{ "Left ADC", NULL, "Left Input Mixer" },
	{ "Right ADC", NULL, "Right Input Mixer" },

	{ "Left Output Mixer", "LINPUT3 Switch", "LINPUT3" },
	{ "Left Output Mixer", "Boost Bypass Switch", "Left Boost Mixer"} ,
	{ "Left Output Mixer", "PCM Playback Switch", "Left DAC" },

	{ "Right Output Mixer", "RINPUT3 Switch", "RINPUT3" },
	{ "Right Output Mixer", "Boost Bypass Switch", "Right Boost Mixer" } ,
	{ "Right Output Mixer", "PCM Playback Switch", "Right DAC" },

	{ "LOUT1 PGA", NULL, "Left Output Mixer" },
	{ "ROUT1 PGA", NULL, "Right Output Mixer" },

	{ "HP_L", NULL, "LOUT1 PGA" },
	{ "HP_R", NULL, "ROUT1 PGA" },

	{ "Left Speaker PGA", NULL, "Left Output Mixer" },
	{ "Right Speaker PGA", NULL, "Right Output Mixer" },

	{ "Left Speaker Output", NULL, "Left Speaker PGA" },
	{ "Right Speaker Output", NULL, "Right Speaker PGA" },

	{ "SPK_LN", NULL, "Left Speaker Output" },
	{ "SPK_LP", NULL, "Left Speaker Output" },
	{ "SPK_RN", NULL, "Right Speaker Output" },
	{ "SPK_RP", NULL, "Right Speaker Output" },
};


```

#####   （4）、处理widget，确定power_check函数，对于Mux Mixer，根据kcontrol_new创建snd_kcontrol

``` cpp
static const struct snd_soc_dapm_widget wm8960_dapm_widgets[] = {
SND_SOC_DAPM_INPUT("LINPUT1"),
SND_SOC_DAPM_INPUT("RINPUT1"),
SND_SOC_DAPM_INPUT("LINPUT2"),
SND_SOC_DAPM_INPUT("RINPUT2"),
SND_SOC_DAPM_INPUT("LINPUT3"),
SND_SOC_DAPM_INPUT("RINPUT3"),

SND_SOC_DAPM_MICBIAS("MICB", WM8960_POWER1, 1, 0),

SND_SOC_DAPM_MIXER("Left Boost Mixer", WM8960_POWER1, 5, 0,
		   wm8960_lin_boost, ARRAY_SIZE(wm8960_lin_boost)),
SND_SOC_DAPM_MIXER("Right Boost Mixer", WM8960_POWER1, 4, 0,
		   wm8960_rin_boost, ARRAY_SIZE(wm8960_rin_boost)),

SND_SOC_DAPM_MIXER("Left Input Mixer", WM8960_POWER3, 5, 0,
		   wm8960_lin, ARRAY_SIZE(wm8960_lin)),
SND_SOC_DAPM_MIXER("Right Input Mixer", WM8960_POWER3, 4, 0,
		   wm8960_rin, ARRAY_SIZE(wm8960_rin)),

SND_SOC_DAPM_ADC("Left ADC", "Capture", WM8960_POWER2, 3, 0),
SND_SOC_DAPM_ADC("Right ADC", "Capture", WM8960_POWER2, 2, 0),

SND_SOC_DAPM_DAC("Left DAC", "Playback", WM8960_POWER2, 8, 0),
SND_SOC_DAPM_DAC("Right DAC", "Playback", WM8960_POWER2, 7, 0),

SND_SOC_DAPM_MIXER("Left Output Mixer", WM8960_POWER3, 3, 0,
	&wm8960_loutput_mixer[0],
	ARRAY_SIZE(wm8960_loutput_mixer)),
SND_SOC_DAPM_MIXER("Right Output Mixer", WM8960_POWER3, 2, 0,
	&wm8960_routput_mixer[0],
	ARRAY_SIZE(wm8960_routput_mixer)),

SND_SOC_DAPM_PGA("LOUT1 PGA", WM8960_POWER2, 6, 0, NULL, 0),
SND_SOC_DAPM_PGA("ROUT1 PGA", WM8960_POWER2, 5, 0, NULL, 0),

SND_SOC_DAPM_PGA("Left Speaker PGA", WM8960_POWER2, 4, 0, NULL, 0),
SND_SOC_DAPM_PGA("Right Speaker PGA", WM8960_POWER2, 3, 0, NULL, 0),

SND_SOC_DAPM_PGA("Right Speaker Output", WM8960_CLASSD1, 7, 0, NULL, 0),
SND_SOC_DAPM_PGA("Left Speaker Output", WM8960_CLASSD1, 6, 0, NULL, 0),

SND_SOC_DAPM_OUTPUT("SPK_LP"),
SND_SOC_DAPM_OUTPUT("SPK_LN"),
SND_SOC_DAPM_OUTPUT("HP_L"),
SND_SOC_DAPM_OUTPUT("HP_R"),
SND_SOC_DAPM_OUTPUT("SPK_RP"),
SND_SOC_DAPM_OUTPUT("SPK_RN"),
SND_SOC_DAPM_OUTPUT("OUT3"),
};

//	snd_soc_dapm_new_controls(dapm, wm8960_dapm_widgets,
//				  ARRAY_SIZE(wm8960_dapm_widgets));

int snd_soc_dapm_new_controls(struct snd_soc_dapm_context *dapm,
	const struct snd_soc_dapm_widget *widget,
	int num)
{
	int i, ret;

	for (i = 0; i < num; i++) {
		ret = snd_soc_dapm_new_control(dapm, widget);
		if (ret < 0) {
			dev_err(dapm->dev,
				"ASoC: Failed to create DAPM control %s: %d\n",
				widget->name, ret);
			return ret;
		}
		widget++;
	}
	return 0;
}
```

#####   （5）、widget上电过程
![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/dapm_power_check.png)



####  （八）、DAPM的情景分析_使用过程

a. tinymix,tinyplay,tinycap殊途同归,都会调用dapm_power_widgets
b. dapm的核心: complete path
某个widget是否要上电的判断过程 

##### （1）、tinymix调用过程

以SOC_DAPM_SINGLE("LINPUT1 Switch", WM8960_LINPATH, 8, 1, 0)为例:

``` cpp
snd_soc_dapm_put_volsw
	connect = xxx // 根据传入的val确定

	// 把kcontrol要设置的reg,val写入update
	update.kcontrol = kcontrol;
	update.widget = widget;
	update.reg = reg;
	update.mask = mask;
	update.val = val;
	widget->dapm->update = &update;
	
	dapm_mixer_update_power(widget, kcontrol, connect);
		// 找到path并设置connect
		path->connect = connect;
		
		// 调用此函数逐个widget进行判断、上电/关闭
		dapm_power_widgets(widget->dapm, SND_SOC_DAPM_STREAM_NOP);
	
	widget->dapm->update = NULL;
	
	
```

##### （2）、tinyplay,tinycap调用过程

播放/录音前都会调用 soc_pcm_prepare

``` cpp
soc_pcm_prepare
    // stream's name = Playback or Capture
	snd_soc_dapm_stream_event(rtd, stream's name, SND_SOC_DAPM_STREAM_START)
		soc_dapm_stream_event
			// 找出每一个widget
			// 如果strstr(w->sname, stream) // w->sname中含有Playback or Capture
			// w->active = 1
			
			dapm_power_widgets
```

大家都会调用dapm_power_widgets

> 		对于每一个widget 
>	
> 	power = w->power_check(w);  // 确定是否要上电

``` cpp
	// 放入不同的链表, 以后统一上是或关闭
	if (power)
		dapm_seq_insert(w, &up_list, true);
	else
		dapm_seq_insert(w, &down_list, false);
	
	// 关闭down_list上的所有widget
	dapm_seq_run(dapm, &down_list, event, false);
	
	// 根据dapm->update设置kcontrol, // update来自tinymix的调用
	dapm_widget_update(dapm);
	
	// 打开up_list上的所有widget
	dapm_seq_run(dapm, &up_list, event, true);

// 给链表中的widget上电或关闭
dapm_seq_run
	dapm_seq_run_coalesced
		snd_soc_update_bits
```

####  （九）、tinycap && tinyplay调用流程源码分析
#####  （1）、tiny4412声卡驱动录音功能调试
![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/WM8940_ADC_DAC.png)

![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/tiny4412_wm8960_fix_record.png)

#####  （2）、tinycap && tinyplay调用流程源码分析
![enter image description here](https://raw.githubusercontent.com/zzhoujinjian/PicGo/master/personal.website/tinycap_tinyplay.png)

tinycap /data/charlesvincent.wav   // 录音ok
tinyplay /data/charlesvincent.wav   // 播放ok

``` cpp
    //audio 打印 log：
	Line 1575: <4>[    4.628618] zjj.android.audio.sys platform register snd-soc-dummy
	Line 1577: <4>[    4.628627] zjj.android.audio.sys Registered platform 'snd-soc-dummy'
	Line 1579: <4>[    4.628872] zjj.android.audio.sys platform register samsung-audio
	Line 1581: <4>[    4.628878] zjj.android.audio.sys Registered platform 'samsung-audio'
	Line 1583: <4>[    4.628997] zjj.android.audio.sys samsung_i2s_probe snd_soc_register_dai0 i2s.c
	Line 1585: <4>[    4.629005] zjj.android.audio.sys i2s_alloc_dai i2s.c
	Line 1587: <4>[    4.629160] zjj.android.audio.sys samsung_i2s_probe snd_soc_register_dai1 i2s.c
	Line 1589: <4>[    4.629166] zjj.android.audio.sys dai register samsung-i2s.0
	Line 1591: <4>[    4.629174] zjj.android.audio.sys Registered DAI 'samsung-i2s.0'
	Line 1593: <4>[    4.629200] zjj.android.audio.sys samsung_i2s_probe snd_soc_register_dai0 i2s.c
	Line 1595: <4>[    4.629205] zjj.android.audio.sys dai register samsung-i2s.4
	Line 1597: <4>[    4.629210] zjj.android.audio.sys Registered DAI 'samsung-i2s.4'
	Line 1603: <4>[    4.629489] zjj.android.audio.sys platform register samsung-audio-idma
	Line 1605: <4>[    4.629496] zjj.android.audio.sys Registered platform 'samsung-audio-idma'
	Line 1973: <4>[    9.090438] zjj.android.audio.sys codec register 0-001a
	Line 1975: <4>[    9.090510] zjj.android.audio.sys dai register 0-001a #1
	Line 1977: <4>[    9.090572] zjj.android.audio.sys Registered codec 'wm8960-codec.0-001a'
	Line 1979: <4>[    9.114080] zjj.android.audio.sys platform_device_register soc-audio tiny4412_wm8960.c
	Line 1981: <4>[    9.114594] zjj.android.audio.sys soc_probe soc-core.c
	Line 1983: <4>[    9.114688] zjj.android.audio.sys soc_bind_dai_link soc-core.c
	Line 1985: <4>[    9.114753] zjj.android.audio.sys snd_card_create soc-core.c
	Line 1987: <4>[    9.116842] zjj.android.audio.sys soc-core.c probe S3C2440_UDA1341 dai link 0
	Line 1991: <4>[    9.124088] zjj.android.audio.sys wm8960_probe
	Line 1995: <4>[    9.234417] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 1997: <4>[    9.234545] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 1999: <4>[    9.234611] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2001: <4>[    9.234679] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2003: <4>[    9.235192] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2005: <4>[    9.246173] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2007: <4>[    9.246251] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2009: <4>[    9.252225] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2011: <4>[    9.258684] zjj.android.audio.sys soc-core.c registered pcm #0 100ask_UDA1341 wm8960-hifi-0
	Line 2013: <4>[    9.266114] zjj.android.audio.sys snd_pcm_new pcm.c
	Line 2019: <4>[    9.280976] zjj.android.audio.sys soc-core.c asoc: wm8960-hifi <-> samsung-i2s.0 mapping ok
	Line 2021: <4>[    9.281075] zjj.android.audio.sys snd_card_register soc-core.c
	Line 2023: <4>[    9.307787] zjj.android.audio.sys snd_soc_register_card soc-core.c
	Line 2025: <4>[    9.380530] zjj.android.audio.sys snd_soc_dapm_put_volsw soc-dapm.c
	Line 2027: <4>[    9.380608] zjj.android.audio.sys dapm_mixer_update_power soc-dapm.c
	Line 2029: <4>[    9.380698] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2031: <4>[    9.380979] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2033: <4>[    9.382107] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2035: <4>[    9.391457] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2037: <4>[    9.393502] zjj.android.audio.sys snd_soc_dapm_put_volsw soc-dapm.c
	Line 2039: <4>[    9.405092] zjj.android.audio.sys dapm_mixer_update_power soc-dapm.c
	Line 2041: <4>[    9.406440] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2043: <4>[    9.412262] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2045: <4>[    9.417412] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2047: <4>[    9.423600] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2251: <4>[   59.642371] zjj.android.audio.sys snd_pcm_capture_open pcm_native.c
	Line 2253: <4>[   59.642447] zjj.android.audio.sys snd_pcm_open pcm_native.c
	Line 2255: <4>[   59.643772] zjj.android.audio.sys snd_pcm_do_prepare pcm_native.c
	Line 2257: <4>[   59.643842] zjj.android.audio.sys soc_pcm_prepare soc-core.c
	Line 2259: <4>[   59.643920] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2261: <4>[   59.650158] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2263: <4>[   59.654885] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2265: <4>[   59.660862] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2267: <4>[   70.453890] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2269: <4>[   70.454754] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2271: <4>[   70.455971] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2273: <4>[   70.456096] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2275: <4>[   88.689064] zjj.android.audio.sys snd_pcm_playback_open pcm_native.c
	Line 2277: <4>[   88.689140] zjj.android.audio.sys snd_pcm_open pcm_native.c
	Line 2279: <4>[   88.689566] zjj.android.audio.sys snd_pcm_playback_open pcm_native.c
	Line 2281: <4>[   88.689638] zjj.android.audio.sys snd_pcm_open pcm_native.c
	Line 2283: <4>[   88.691460] zjj.android.audio.sys snd_pcm_do_prepare pcm_native.c
	Line 2285: <4>[   88.696896] zjj.android.audio.sys soc_pcm_prepare soc-core.c
	Line 2287: <4>[   88.702418] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2289: <4>[   88.708425] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2291: <4>[   88.713567] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2293: <4>[   88.719467] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2299: <4>[  104.510149] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2301: <4>[  104.510853] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2303: <4>[  104.512003] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2305: <4>[  104.512129] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2307: <4>[  115.184433] zjj.android.audio.sys snd_pcm_capture_open pcm_native.c
	Line 2309: <4>[  115.184510] zjj.android.audio.sys snd_pcm_open pcm_native.c
	Line 2311: <4>[  115.185324] zjj.android.audio.sys snd_pcm_do_prepare pcm_native.c
	Line 2313: <4>[  115.185397] zjj.android.audio.sys soc_pcm_prepare soc-core.c
	Line 2315: <4>[  115.185683] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2317: <4>[  115.191879] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2319: <4>[  115.196990] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2321: <4>[  115.202897] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2323: <4>[  116.077184] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2325: <4>[  116.077890] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2327: <4>[  116.079099] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2329: <4>[  116.079223] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2331: <4>[  117.836590] zjj.android.audio.sys snd_pcm_playback_open pcm_native.c
	Line 2333: <4>[  117.836666] zjj.android.audio.sys snd_pcm_open pcm_native.c
	Line 2335: <4>[  117.837085] zjj.android.audio.sys snd_pcm_playback_open pcm_native.c
	Line 2337: <4>[  117.837156] zjj.android.audio.sys snd_pcm_open pcm_native.c
	Line 2339: <4>[  117.838975] zjj.android.audio.sys snd_pcm_do_prepare pcm_native.c
	Line 2341: <4>[  117.844460] zjj.android.audio.sys soc_pcm_prepare soc-core.c
	Line 2343: <4>[  117.850000] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2345: <4>[  117.855998] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2347: <4>[  117.862549] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2349: <4>[  117.867224] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2355: <4>[  123.720164] zjj.android.audio.sys dapm_power_widgets soc-dapm.c
	Line 2357: <4>[  123.720905] zjj.android.audio.sys dapm_seq_run soc-dapm.c
	Line 2359: <4>[  123.722064] zjj.android.audio.sys dapm_widget_update soc-dapm.c
	Line 2361: <4>[  123.722188] zjj.android.audio.sys dapm_seq_run soc-dapm.c
```




####  （十）、参考文档：
[韦老师移植后的声卡驱动](https://github.com/weidongshan/DRV_0007_tiny4412_asoc_machine)
[Android音频模块启动流程分析](https://zhuanlan.zhihu.com/p/27480328)
[Jhuster的专栏​ Android音频开发](https://zhuanlan.zhihu.com/jhuster)
[高通audio offload学习 | Thinking](http://thinks.me/2016/09/13/audio_qcom_offload/)
[DroidPhone的专栏 - CSDN博客](https://blog.csdn.net/droidphone/article/category/1118446)
[alsa音频架构1-CSDN博客](https://blog.csdn.net/orz415678659/article/details/8866071)
[alsa音频架构2-ASoc - CSDN博客](https://blog.csdn.net/orz415678659/article/details/8982771)
[alsa音频架构3-Pcm - CSDN博客](https://blog.csdn.net/orz415678659/article/details/8995163)
[alsa音频架构4-声卡控制 - CSDN博客](https://blog.csdn.net/orz415678659/article/details/8999364)
[Linux ALSA 音频系统：逻辑设备篇 - CSDN博客](https://blog.csdn.net/zyuanyun/article/details/59180272)
[Linux ALSA 音频系统：物理链路篇 - CSDN博客](https://blog.csdn.net/zyuanyun/article/details/59170418)
[专栏：MultiMedia框架总结(基于6.0源码) - CSDN博客](https://blog.csdn.net/column/details/12812.html)
[Android 音频系统：从 AudioTrack 到 AudioFlinger - CSDN博客](https://blog.csdn.net/zyuanyun/article/details/60890534)
[AZURE - CSDN博客 - ALSA-Android Audio](https://blog.csdn.net/azloong/article/category/778492)
[AZURE - CSDN博客 - ANDROID音频系统](https://blog.csdn.net/azloong/article/category/777239)
[Audio驱动总结--ALSA | Winddoing's Blog](https://winddoing.github.io/2017/07/10/audio_alsa/)
[audio HAL - 牧 天 - 博客园](http://www.cnblogs.com/muhe221/articles/4461543.html)
[林学森的Android专栏 - CSDN博客](https://blog.csdn.net/xuesen_lin/article/category/1390680)
[深入剖析Android音频 - CSDN博客Yangwen123](https://blog.csdn.net/yangwen123/article/category/2589087)
[播放框架 - 标签 - Tocy - 博客园](http://www.cnblogs.com/tocy/tag/%E6%92%AD%E6%94%BE%E6%A1%86%E6%9E%B6/)
[Android-7.0-Nuplayer概述 - CSDN博客](https://blog.csdn.net/miaomiao12345678/article/details/57415505)
[Android-7.0-Nuplayer-启动流程 - CSDN博客](https://blog.csdn.net/miaomiao12345678/article/details/56961370)
[Android Media Player 框架分析-Nuplayer - CSDN博客](https://blog.csdn.net/harman_zjc/article/details/53386010)
[Android Media Player 框架分析-AHandler AMessage ALooper - CSDN博客](https://blog.csdn.net/harman_zjc/article/details/53397945)
[Android N Audio播放 start真面目- (六篇) CSDN博客](https://blog.csdn.net/liu1314you/article/category/6344583)
[深入理解Android音视频同步机制（五篇）NuPlayer的avsync逻辑 - CSDN博客](https://blog.csdn.net/nonmarking/article/details/78746671)
[wangyf的专栏 - CSDN博客-MT6737 Android N 平台 Audio系统学习](https://blog.csdn.net/u014310046/article/category/6571854)
[Android 7.0 Audio: Mediaplayer - CSDN博客](https://blog.csdn.net/xiashaohua/article/details/53638780)
[Android 7.0 Audio-相关类浅析- CSDN博客](https://blog.csdn.net/xiashaohua/article/category/6549668)
[Android N Audio播放六：如何读取buffer - CSDN博客](https://blog.csdn.net/liu1314you/article/details/59119144)
[Fuchsia OS中的RPC机制-FIDL - CSDN博客](https://blog.csdn.net/jinzhuojun/article/details/78007568)
[高通Audio中ASOC的codec驱动 - yooooooo - 博客园](http://www.cnblogs.com/linhaostudy/category/1145404.html)
