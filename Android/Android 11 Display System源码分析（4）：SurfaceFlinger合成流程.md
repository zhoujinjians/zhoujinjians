---
title: Android 11 Display System源码分析（4）：SurfaceFlinger合成流程（V1）
cover: https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/post.cover.pictures/bing-wallpaper-2018.04.44.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20220916
date: 2022-09-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【zhoujinjianx.com博客原图链接】](https://github.com/zhoujinjianx) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

 [xxx](xxx) 




--------------------------------------------------------------------------------

==源码（部分）==：

**xxx**



==源码（部分）==：

--------------------------------------------------------------------------------

## 一、SurfaceFlinger::onMessageInvalidate

### SurfaceFlinger接收到MessageQueue::INVALIDATE消息后。

![image-20220830153250163](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830153250163.png)

![image-20220830153416358](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830153416358.png)

![image-20220830153630145](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830153630145.png)

![image-20220830153659446](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830153659446.png)

调用handlePageFlip()查看是否有刷新的必要，如果有发送刷新信号signalRefresh()。

（1）、handlePageFlip

![image-20220830153957918](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830153957918.png)

遍历mDrawingState里面的layer是否有准备好的。如果有并且需要立即显示加入到mLayersWithQueuedFrames里面。

### invalidateLayerStack

判断需要刷新的layer是否属于当前Output。

![image-20220830154218307](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830154218307.png)

## 二、SurfaceFlinger::onMessageRefresh

![image-20220830155105025](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830155105025.png)

### CompositionRefreshArgs构造

获取mDrawingState每个layer的CompositionEngineLayerFE，加入到layers中

![image-20220830155519327](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830155519327.png)

前面主要搜集需要送显的layer的情况，封装成compositionengine::CompositionRefreshArgs参数传递给CompositionEngine做进一步操作。

## 三、CompositionEngine->present(refreshArgs)

### （1）、onPreComposition

SurfaceFlinger主要合成及调用Hal composer送显的逻辑都在这里了。

![image-20220830161151773](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830161151773.png)

![image-20220830174653376](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830174653376.png)

调用了layer->onPreComposition做composition前的准备。进去看没做啥实际操作。

### （2）、Output::prepare

![image-20220830190449852](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830190449852.png)

![image-20220830191817972](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830191817972.png)

![image-20220830191845274](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830191845274.png)

收集可见的Layers。

![image-20220830191955805](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830191955805.png)

记得之前版本是放在SurfaceFlinger::computeVisibleRegions函数里面的。

### （3）、Output::updateLayerStateFromFE

![image-20220830192811578](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830192811578.png)

![image-20220830192849127](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830192849127.png)

遍历每个output中的layer。调用getLayerFE().prepareCompositionState

![image-20220830193102037](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830193102037.png)

### （4）、Output::updateInfoForHwc2On1Adapter

![image-20220830194414490](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830194414490.png)

![image-20220830194441817](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830194441817.png)

更新colorMode，Composition状态，然后写入hwc。

![image-20220830194610346](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830194610346.png)

![image-20220830194748461](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220830194748461.png)

将Composition模式，Color状态写入HWC。

### （5）、Output::presentForHwc2On1Adapter

![image-20220901104924305](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220901104924305.png)

####     5.1、Output::beginFrame



![image-20220901105129431](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220901105129431.png)



![image-20220901105233973](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220901105233973.png)

```c++
Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\services\surfaceflinger\CompositionEngine\src\RenderSurface.cpp
status_t RenderSurface::beginFrame(bool mustRecompose) {
	ATRACE_NAME("RenderSurface::beginFrame");
    return mDisplaySurface->beginFrame(mustRecompose);
}
Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\services\surfaceflinger\DisplayHardware\FramebufferSurface.cpp
status_t FramebufferSurface::beginFrame(bool /*mustRecompose*/) {
	ATRACE_NAME("FramebufferSurface::beginFrame");
    return NO_ERROR;
}
```



####     5.2、Output::prepareFrame



![image-20220901105518049](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220901105518049.png)

```c++
Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\services\surfaceflinger\CompositionEngine\src\Output.cpp
void Output::prepareFrame() {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
    ATRACE_NAME("Output::prepareFrame");

    const auto& outputState = getState();
    if (!outputState.isEnabled) {
        return;
    }

    chooseCompositionStrategy();

    mRenderSurface->prepareFrame(outputState.usesClientComposition,
                                 outputState.usesDeviceComposition);
}

void Output::chooseCompositionStrategy() {
    // The base output implementation can only do client composition
    ATRACE_NAME("Output::chooseCompositionStrategy");
    auto& outputState = editState();
    outputState.usesClientComposition = true;
    outputState.usesDeviceComposition = false;
    outputState.reusedClientComposition = false;
}
Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\services\surfaceflinger\CompositionEngine\src\Display.cpp
void Display::chooseCompositionStrategy() {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
	ATRACE_NAME("Display::chooseCompositionStrategy");

    // Default to the base settings -- client composition only.
    Output::chooseCompositionStrategy();

    // If we don't have a HWC display, then we are done
    if (!mId) {
        return;
    }

    // Get any composition changes requested by the HWC device, and apply them.
    std::optional<android::HWComposer::DeviceRequestedChanges> changes;
    auto& hwc = getCompositionEngine().getHwComposer();
    if (status_t result = hwc.getDeviceCompositionChanges(*mId, anyLayersRequireClientComposition(),
                                                          &changes);
        result != NO_ERROR) {
        ALOGE("chooseCompositionStrategy failed for %s: %d (%s)", getName().c_str(), result,
              strerror(-result));
        return;
    }
    if (changes) {
        applyChangedTypesToLayers(changes->changedTypes);
        applyDisplayRequests(changes->displayRequests);
        applyLayerRequestsToLayers(changes->layerRequests);
        applyClientTargetRequests(changes->clientTargetProperty);
    }

    // Determine what type of composition we are doing from the final state
    auto& state = editState();
    state.usesClientComposition = anyLayersRequireClientComposition();
    state.usesDeviceComposition = !allLayersRequireClientComposition();
}

```

首先将成策略给HWC看是否接受，然后如果有变化就将变化应用于layer。

![image-20220901110831426](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220901110831426.png)

####     5.3、Output::finishFrame

前面都是做准备工作，接下来才真正composeSurfaces。

![image-20220901113451103](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220901113451103.png)

接下来看看composeSurfaces。

##### 1)、首先dequeueBuffer获取一个Graphic Buffer。

![image-20220901113626556](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220901113626556.png)

![image-20220901113842626](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220901113842626.png)

##### 2)、为此输出上的图层生成客户端组合请求。

![image-20220920192155384](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920192155384.png)

![image-20220920191645958](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920191645958.png)

##### 3)、GLESRenderEngine::drawLayers

![image-20220920191749360](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920191749360.png)

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\native\libs\renderengine\gl\GLESRenderEngine.cpp

重点来看看

查看有没有需要blur的layer

![image-20220920192823984](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920192823984.png)

![image-20220920192913922](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920192913922.png)

![image-20220920193028215](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920193028215.png)

![image-20220920193059514](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920193059514.png)



![image-20220920194801539](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920194801539.png)

drawMesh可以dump layer debug。

![image-20220920204552384](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920204552384.png)

导出drawMesh:

\ppm\texture_out431.ppm（壁纸）

![image-20220920204614735](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920204614735.png)

\ppm\texture_out432.ppm（壁纸 + launcher）

![image-20220920204647955](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920204647955.png)

\ppm\texture_out433.ppm（壁纸 + launcher + statusbar）

![image-20220920204800635](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920204800635.png)

\ppm\texture_out433.ppm（壁纸 + launcher + statusbar+navigationbar）

![image-20220920204833579](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920204833579.png)

##### 4)、mRenderSurface->queueBuffer

![image-20220920194218105](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220920194218105.png)

#### 5.4、Output::postFramebuffer

![image-20220921133836896](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220921133836896.png)



![image-20220921153808731](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220921153808731.png)

![image-20220921153709409](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220921153709409.png)

![image-20220921154132543](https://raw.githubusercontent.com/zhoujinjianx/PicGo2/master/Android_Display_System/Android11_Display04/image-20220921154132543.png)
