---
title: Android 11 Display System源码分析（6）：Skia && hwui简要使用案例分析（V1）
cover: https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/post.cover.pictures/bing-wallpaper-2018.04.46.jpg
categories: 
 - Display
tags:
 - Display
toc: true
abbrlink: 20220916
date: 2022-11-16 16:16:16
---
注：文章都是通过阅读各位前辈总结的资料、Android 11 Rockchip平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网，如有侵权，请联系删除（◕‿◕），转载请注明出处（©Rockchip ©Android @Linux 版权所有），谢谢。

（==**文章基于 Android 11.0**==）

[【zhoujinjian.com博客原图链接】](https://github.com/zhoujinjiann) 

[【开发板】](https://wiki.radxa.com/Rockpi4)

[【开发板 源码链接】](https://wiki.radxa.com/Rockpi4/rockpi-android11)



正是由于前人的分析和总结，帮助我节约了大量的时间和精力，特别感谢，由于不喜欢图片水印，去除了水印，敬请谅解！！！

 [（1）FirstSkiaApp](https://github.com/nightelf3/FirstSkiaApp) 


--------------------------------------------------------------------------------

==源码（部分）==：

**xxx**

==源码（部分）==：

--------------------------------------------------------------------------------



## （一）、FirstSkiaApp-Demo(Windows)示例学习

### （1）、ExampleLayer1

![image-20220926102748367](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926102748367.png)

```c++
#include "include/Layers/ExampleLayer1.h"

void ExampleLayer1::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	// inititalize paint structure to fill shape with red color
	SkPaint paint;
	paint.setStyle(SkPaint::Style::kFill_Style);
	paint.setColor(SkColors::kRed);

	// draw 150x100 rect at (100, 100)
	SkRect rect = SkRect::MakeXYWH(100, 100, 150, 100);
	canvas->drawRect(rect, paint);
}
```

### （2）、ExampleLayer2

![image-20220926102826237](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926102826237.png)

```c++
#include "include/Layers/ExampleLayer2.h"
#include "include/Layers/Utils/Utils.h"

void ExampleLayer2::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	// inititalize paint structure to fill shape with red color
	SkPaint paint;
	paint.setStyle(SkPaint::Style::kFill_Style);
	paint.setColor(SkColors::kRed);

	// get canvas size and resuce it for 1/8
	SkRect rect = GetBounds(canvas);
	rect.inset(rect.width() / 8.f, rect.height() / 8.f);
	canvas->drawOval(rect, paint);
}
```

### （3）、ExampleLayer3

![image-20220926102928370](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926102928370.png)

```c++
#include "include/Layers/ExampleLayer3.h"

void ExampleLayer3::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

  SkPaint paint;
  paint.setStyle(SkPaint::kFill_Style);
  paint.setAntiAlias(true);
  paint.setStrokeWidth(4);
  paint.setColor(0xff4285F4);

  SkRect rect = SkRect::MakeXYWH(10, 10, 100, 160);
  canvas->drawRect(rect, paint);

  SkRRect oval;
  oval.setOval(rect);
  oval.offset(40, 80);
  paint.setColor(0xffDB4437);
  canvas->drawRRect(oval, paint);

  paint.setColor(0xff0F9D58);
  canvas->drawCircle(180, 50, 25, paint);

  rect.offset(80, 50);
  paint.setColor(0xffF4B400);
  paint.setStyle(SkPaint::kStroke_Style);
  canvas->drawRoundRect(rect, 10, 10, paint);

	__super::Draw(canvas);
}

```

### （4）、ExampleLayer4

![image-20220926103024130](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926103024130.png)

```c++
#include "include/Layers/ExampleLayer4.h"
#include "include/core/SkFont.h"
#include "include/core/SkFontMgr.h"

void ExampleLayer4::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	sk_sp<SkFontMgr> fontManager = SkFontMgr::RefDefault();
	sk_sp<SkTypeface> typeface(fontManager->matchFamilyStyle(nullptr, {}));

	SkFont font1(typeface, 64.0f, 1.0f, 0.0f);
	SkFont font2(typeface, 64.0f, 1.5f, 0.0f);
	font1.setEdging(SkFont::Edging::kAntiAlias);
	font2.setEdging(SkFont::Edging::kAntiAlias);

	SkPaint paint1;
	paint1.setAntiAlias(true);
	paint1.setColor(SkColorSetARGB(0xFF, 0x42, 0x85, 0xF4));
	canvas->drawString("Skia", 200.0f, 240.0f, font1, paint1);

	SkPaint paint2;
	paint2.setAntiAlias(true);
	paint2.setColor(SkColorSetARGB(0xFF, 0xDB, 0x44, 0x37));
	paint2.setStyle(SkPaint::kStroke_Style);
	paint2.setStrokeWidth(3.0f);
	canvas->drawString("Skia", 200.0f, 344.0f, font1, paint2);

	SkPaint paint3;
	paint3.setAntiAlias(true);
	paint3.setColor(SkColorSetARGB(0xFF, 0x0F, 0x9D, 0x58));
	canvas->drawString("Skia", 200.0f, 424.0f, font2, paint3);

	__super::Draw(canvas);

	m_FPS.Calc();
	m_FPS.Draw(canvas);
}
```

### （5）、ExampleLayer5

![image-20220926103332465](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926103332465.png)

```c++
#include "include/Layers/ExampleLayer5.h"
#include "include/core/SkFont.h"
#include "include/core/SkFontMgr.h"

namespace
{
	const char sDefaultFamily[] = "Default";
	const std::vector<std::wstring> sText = {
		L"一 (yī) – “one”",
		L"二 (èr) – “two”",
		L"三 (sān) – “three”",
		L"四 (sì) – “four”",
		L"五 (wǔ) – “five”",
		L"六 (liù) – “six”",
		L"七 (qī) – “seven",
		L"八 (bā) – “eight”",
		L"九 (jiǔ) – “nine”",
		L"十 (shí) – “ten”",
		L"文言文 - wényánwén",
		L"白话文 - Báihuà Wén",
		L"貞 zhēn became 贞 zhēn",
		L"贈 zèng became 赠 zèng"
	};

	template<typename TChar>
	SkScalar DrawAndMeatureString(SkCanvas* canvas, const TChar str[], const size_t lenght, const SkFont& font, const SkPaint& paint, SkScalar x, SkScalar y)
	{
		const auto encoding = sizeof(TChar) == 1 ? SkTextEncoding::kUTF8 : SkTextEncoding::kUTF16;
		SkRect measure;
		font.measureText(str, lenght * sizeof(TChar), encoding, &measure);
		canvas->drawSimpleText(str, lenght * sizeof(TChar), encoding, x, y, font, paint);
		return measure.height() + font.getSize() / 4.f;
	}

	void DrawSkText(SkCanvas* canvas, const std::vector<std::wstring>& sText, const char familyName[], SkScalar x, SkScalar y, const SkPaint& paint)
	{
		sk_sp<SkFontMgr> fontManager = SkFontMgr::RefDefault();
		sk_sp<SkTypeface> typeface(fontManager->matchFamilyStyle(familyName, {}));
		SkFont font(typeface, 20);

		if (!familyName)
			familyName = sDefaultFamily;

		y += DrawAndMeatureString(canvas, familyName, strlen(familyName), font, paint, x, y);
		for (auto& str : sText)
			y += DrawAndMeatureString(canvas, str.c_str(), str.length(), font, paint, x, y);
	}
}

void ExampleLayer5::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	SkPaint paint;
	paint.setColor(SkColors::kWhite);
	DrawSkText(canvas, sText, nullptr, 100, 100, paint);

	paint.setColor(SkColors::kGreen);
	DrawSkText(canvas, sText, "Yu Gothic UI", 400, 100, paint);
}
```

### （8）、ExampleLayer8

![image-20220926103616750](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926103616750.png)

```c++
#include "include/Layers/ExampleLayer8.h"

void ExampleLayer8::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	SkAutoCanvasRestore checkpoint(canvas, true);

	canvas->translate(m_Point.x(), m_Point.y());
	canvas->rotate(60.f);
	SkRect rect = SkRect::MakeXYWH(330, -330, 400, 300);

	SkPaint paint;
	paint.setAntiAlias(true);
	paint.setColor(0xff4285F4);
	canvas->drawRect(rect, paint);

	canvas->rotate(20.f);
	canvas->scale(1.2f, 1.2f);
	paint.setColor(0xffDB4437);
	canvas->drawRect(rect, paint);
}

bool ExampleLayer8::ProcessKey(Key key, InputState state, ModifierKey modifiers)
{
	constexpr SkScalar increment = 10.f;
	switch (key)
	{
	case Key::kUp:
		m_Point.fY -= increment;
		return true;

	case Key::kRight:
		m_Point.fX += increment;
		return true;

	case Key::kDown:
		m_Point.fY += increment;
		return true;

	case Key::kLeft:
		m_Point.fX -= increment;
		return true;
	}
	return false;
}
```

### （9）、ExampleLayer9

![image-20220926133423367](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926133423367.png)

```c++
#include "include/Layers/ExampleLayer9.h"
#include "include/Layers/Utils/Utils.h"
#include "include/core/SkMatrix.h"

ExampleLayer9::ExampleLayer9()
{
	m_Image = LoadImageFromFile(SkString("resources/doge.png"));
	m_Matrix.setIdentity();
}

void ExampleLayer9::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	SkAutoCanvasRestore checkpoint(canvas, true);
	canvas->setMatrix(m_Matrix);
	canvas->drawImage(m_Image, 200, 200);
}

bool ExampleLayer9::ProcessKey(Key key, InputState state, ModifierKey modifiers)
{
	constexpr SkScalar increment = 10.f;
	switch (key)
	{
	case Key::kUp:
		m_Matrix.preTranslate(0, -increment);
		return true;

	case Key::kRight:
		m_Matrix.preTranslate(increment, 0);
		return true;

	case Key::kDown:
		m_Matrix.preTranslate(0, increment);
		return true;

	case Key::kLeft:
		m_Matrix.preTranslate(-increment, 0);
		return true;
	}
	return false;
}

bool ExampleLayer9::ProcessMouseWheel(InputState state, ModifierKey modifiers)
{
	if (ModifierKey::kNone == modifiers)
	{
		if (InputState::kZoomIn == state)
			m_Matrix.preScale(1.1, 1.1);
		else
			m_Matrix.preScale(0.9, 0.9);
		return true;
	}
	else if (ModifierKey::kShift == modifiers)
	{
		if (InputState::kZoomIn == state)
			m_Matrix.preSkew(0.1, 0);
		else
			m_Matrix.preSkew(-0.1, 0);
		return true;
	}
	else
	{
		if (InputState::kZoomIn == state)
			m_Matrix.preSkew(0, 0.1);
		else
			m_Matrix.preSkew(0, -0.1);
		return true;
	}

	return false;
}
```

### （10）、ExampleLayer10

![image-20220926133632108](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926133632108.png)

```c++
#include "include/Layers/ExampleLayer10.h"
#include "include/core/SkFont.h"
#include "include/core/SkTextBlob.h"
#include "include/effects/SkGradientShader.h"

void ExampleLayer10::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	SkAutoCanvasRestore checkpoint(canvas, true);
	canvas->translate(200, 100);

	SkBlendMode modes[] = {
			SkBlendMode::kClear,
			SkBlendMode::kSrc,
			SkBlendMode::kDst,
			SkBlendMode::kSrcOver,
			SkBlendMode::kDstOver,
			SkBlendMode::kSrcIn,
			SkBlendMode::kDstIn,
			SkBlendMode::kSrcOut,
			SkBlendMode::kDstOut,
			SkBlendMode::kSrcATop,
			SkBlendMode::kDstATop,
			SkBlendMode::kXor,
			SkBlendMode::kPlus,
			SkBlendMode::kModulate,
			SkBlendMode::kScreen,
			SkBlendMode::kOverlay,
			SkBlendMode::kDarken,
			SkBlendMode::kLighten,
			SkBlendMode::kColorDodge,
			SkBlendMode::kColorBurn,
			SkBlendMode::kHardLight,
			SkBlendMode::kSoftLight,
			SkBlendMode::kDifference,
			SkBlendMode::kExclusion,
			SkBlendMode::kMultiply,
			SkBlendMode::kHue,
			SkBlendMode::kSaturation,
			SkBlendMode::kColor,
			SkBlendMode::kLuminosity,
	};

	SkRect rect = SkRect::MakeWH(64.0f, 64.0f);
	SkPaint stroke, src, dst;
	stroke.setStyle(SkPaint::kStroke_Style);
	SkFont font(nullptr, 24);
	SkPoint srcPoints[2] = {
			SkPoint::Make(0.0f, 0.0f),
			SkPoint::Make(64.0f, 0.0f)
	};
	SkColor srcColors[2] = {
			SK_ColorMAGENTA & 0x00FFFFFF,
			SK_ColorMAGENTA };
	src.setShader(SkGradientShader::MakeLinear(
		srcPoints, srcColors, nullptr, 2,
		SkTileMode::kClamp, 0, nullptr));

	SkPoint dstPoints[2] = {
			SkPoint::Make(0.0f, 0.0f),
			SkPoint::Make(0.0f, 64.0f)
	};
	SkColor dstColors[2] = {
			SK_ColorCYAN & 0x00FFFFFF,
			SK_ColorCYAN };
	dst.setShader(SkGradientShader::MakeLinear(
		dstPoints, dstColors, nullptr, 2,
		SkTileMode::kClamp, 0, nullptr));

	SkPaint textPaint;
	textPaint.setColor(SkColors::kWhite);

	size_t N = sizeof(modes) / sizeof(modes[0]);
	size_t K = (N - 1) / 3 + 1;
	for (size_t i = 0; i < N; ++i)
	{
		SkAutoCanvasRestore autoCanvasRestore(canvas, true);
		canvas->translate(192.0f * (i / K), 72.0f * (i % K));
		const char* desc = SkBlendMode_Name(modes[i]);
		canvas->drawTextBlob(SkTextBlob::MakeFromString(desc, font).get(), 68.0f, 30.0f, textPaint);
		canvas->clipRect(SkRect::MakeWH(64.0f, 64.0f));
		canvas->drawColor(SK_ColorLTGRAY);
		
		canvas->saveLayer(nullptr, nullptr);
		canvas->clear(SK_ColorTRANSPARENT);
		canvas->drawPaint(dst);
		src.setBlendMode(modes[i]);
		canvas->drawPaint(src);
		canvas->drawRect(rect, stroke);
	}
}
```



### （11）、ExampleLayer11

![image-20220926133732992](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926133732992.png)

```c++
#include "include/Layers/ExampleLayer11.h"
#include "include/core/SkPath.h"
#include "include/effects/SkGradientShader.h"
#include "include/effects/SkDiscretePathEffect.h"

void ExampleLayer11::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	SkPaint paint;
	paint.setAntiAlias(true);
	paint.setPathEffect(SkDiscretePathEffect::Make(10.0f, 4.0f));

	SkPoint points[2] = { SkPoint::Make(0.0f, 0.0f), SkPoint::Make(356.0f, 356.0f) };
	SkColor colors[2] = { SkColorSetRGB(66, 133, 244), SkColorSetRGB(15, 157, 88) };
	paint.setShader(SkGradientShader::MakeLinear(points, colors, nullptr, 2, SkTileMode::kClamp, 0, nullptr));

	const SkScalar R = 200.0f, C = 428.0f;
	SkPath path;
	path.moveTo(C + R, C);
	for (int i = 1; i < 15; i++)
	{
		SkScalar a = 0.44879895f * i;
		SkScalar r = R + R * (i % 2);
		path.lineTo(C + r * cos(a), C + r * sin(a));
	}
	canvas->drawPath(path, paint);
}
```

### （12）、ExampleLayer12

![image-20220926133910683](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926133910683.png)

```c++
#include "include/Layers/ExampleLayer12.h"
#include "include/effects/SkPerlinNoiseShader.h"

void ExampleLayer12::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	SkPaint paint;
	paint.setShader(SkPerlinNoiseShader::MakeFractalNoise(0.05f, 0.05f, 4, 0.0f, nullptr));
	canvas->drawPaint(paint);
}
```

### （13）、ExampleLayer13

![image-20220926134035499](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926134035499.png)

```c++
#include "include/Layers/ExampleLayer13.h"
#include "include/Layers/Utils/Utils.h"
#include "include/effects/SkGradientShader.h"
#include "include/effects/SkPerlinNoiseShader.h"

void ExampleLayer13::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

	const SkRect rBounds = GetBounds(canvas);
	SkColor colors[2] = {SK_ColorBLUE, SK_ColorYELLOW};
	SkPaint paint;
	paint.setShader(SkShaders::Blend(
		SkBlendMode::kDifference,
		SkGradientShader::MakeRadial(SkPoint::Make(rBounds.centerX(), rBounds.centerY()), std::min(rBounds.centerX(), rBounds.centerY()),
			colors, nullptr, 2, SkTileMode::kClamp, 0, nullptr),
		SkPerlinNoiseShader::MakeTurbulence(0.025f, 0.025f, 2, 0.0f, nullptr)));
	canvas->drawPaint(paint);
}

```

### （14）、ThankYouLayer

![image-20220926134159354](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926134159354.png)

```c++
#include "include/Layers/ThankYouLayer.h"
#include "include/Layers/Utils/Utils.h"
#include "include/core/SkFont.h"
#include "include/effects/SkGradientShader.h"
#include "include/effects/SkPerlinNoiseShader.h"

using namespace std::chrono;

ThankYouLayer::ThankYouLayer()
{
	SkString path;
	for (size_t i = 0; i < m_Images.size(); i++)
	{
		path.printf("resources/dance_%d.gif", i);
		m_Images[i] = LoadImageFromFile(path);
	}
}

void ThankYouLayer::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);
	const SkRect canvasBounds = GetBounds(canvas);

	time_point<system_clock> now = system_clock::now();
	if ((now - m_LastDraw) > 150ms)
	{
		m_Current = (m_Current + 1) % m_Images.size();
		m_LastDraw = now;
	}

	const SkIRect imageBounds = m_Images[m_Current]->bounds();
	canvas->drawImage(m_Images[m_Current], (canvasBounds.width() - imageBounds.width()) / 2.f, (canvasBounds.height() - imageBounds.height()) / 2.f, {});

	SkPaint paint;
	paint.setAntiAlias(true);

	static SkScalar offset = 0;
	const SkPoint points[2] = { SkPoint::Make(offset, offset), SkPoint::Make(256.0f + offset, 256.0f + offset) };
	const SkColor colors[2] = { SkColorSetRGB(66, 133, 244), SkColorSetRGB(15, 157, 88) };
	paint.setShader(SkShaders::Blend(
		SkBlendMode::kDifference,
		SkGradientShader::MakeLinear(points, colors, nullptr, 2, SkTileMode::kClamp, 0, nullptr),
		SkPerlinNoiseShader::MakeTurbulence(0.025f + offset / 1000.f, 0.025f + offset / 1000.f, 2, 0.0f, nullptr)));
	offset = std::fmodf(offset + rand() % 10, 100.f);

	const char str[] = "Thank you!!!";
	SkFont font(nullptr, 100.f);
	SkRect textBounds;
	font.measureText(str, strlen(str), SkTextEncoding::kUTF8, &textBounds, &paint);
	canvas->drawString(str, (canvasBounds.width() - textBounds.width()) / 2.f, (canvasBounds.height() - textBounds.height()) / 2.f - imageBounds.height() / 2.f, font, paint);
}
```



### （x）、Zero_off_dashing

![image-20220926102558662](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926102558662.png)

```c++
#include "include/Layers/Zero_off_dashing.h"

#include "include/effects/SkDashPathEffect.h"

static constexpr float kPi = 3.14159265358979323846f;

void Zero_off_dashing::Draw(SkCanvas* canvas)
{
	// clear canvas with black color
	canvas->clear(SkColors::kBlack);

    SkPaint p;
    canvas->drawCircle(128, 128, 60, p);

    p.setColor(0x88FF0000);
    p.setAntiAlias(true);
    p.setStyle(SkPaint::kStroke_Style);
    p.setStrokeCap(SkPaint::kSquare_Cap);
    p.setStrokeWidth(80);
    SkScalar interv[2] = { 120 * kPi / 6 - 0.05f, 0.0000f };
    p.setPathEffect(SkDashPathEffect::Make(interv, 2, 0.5));

    SkPath path, path2;
    path.addCircle(128, 128, 60);
    canvas->drawPath(path, p);

    p.setColor(0x8800FF00);
    SkScalar interv2[2] = { 120 * kPi / 6 - 0.05f, 10000.0000f };
    p.setPathEffect(SkDashPathEffect::Make(interv2, 2, 0));
    canvas->drawPath(path, p);

    p.getFillPath(path, &path2);
    p.setColor(0xFF000000);
    p.setStrokeWidth(0);
    p.setPathEffect(nullptr);
    canvas->drawPath(path2, p);
}
```

## （二）、Hwui demo展示：

![image-20220926142343794](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926142343794.png)

> ### Y:\home\zhoujinjian\android11_rockpi4\frameworks\base\libs\hwui\tests\common\scenes\OvalAnimation.cpp

```c++
#include "TestSceneBase.h"
#include "utils/Color.h"

class OvalAnimation;

static TestScene::Registrar _Oval(TestScene::Info{"oval", "Draws 1 oval.",
                                                  TestScene::simpleCreateScene<OvalAnimation>});

class OvalAnimation : public TestScene {
public:
    sp<RenderNode> card;
    void createContent(int width, int height, Canvas& canvas) override {
        canvas.drawColor(Color::White, SkBlendMode::kSrcOver);
        card = TestUtils::createNode(0, 0, 200, 200, [](RenderProperties& props, Canvas& canvas) {
            Paint paint;
            paint.setAntiAlias(true);
            paint.setColor(Color::Black);
            canvas.drawOval(0, 0, 200, 200, paint);
        });
        canvas.drawRenderNode(card.get());
    }

    void doFrame(int frameNr) override {
        int curFrame = frameNr % 150;
        card->mutateStagingProperties().setTranslationX(curFrame);
        card->mutateStagingProperties().setTranslationY(curFrame);
        card->setPropertyFieldsDirty(RenderNode::X | RenderNode::Y);
    }
};

```

## （三）、Hwui 分析

### （1）、DrawFrameTask::syncFrameState

![image-20220926143810860](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926143810860.png)

![image-20220926143947295](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926143947295.png)

![image-20220926144254764](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926144254764.png)

重点看看CanvasContext::prepareTree流程。

#### （1）、CanvasContext::prepareTree

![image-20220926144452156](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926144452156.png)

![image-20220926144612444](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926144612444.png)

#### （2）、RenderNode::prepareTreeImpl

![image-20220926144937997](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926144937997.png)

处理Properties变化，PrepareLayer。

![image-20220926145047378](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926145047378.png)

![image-20220926145218464](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926145218464.png)

![image-20220926145401131](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926145401131.png)

#### （3）、SkiaDisplayList::prepareListAndChildren

### （2）、CanvasContext::draw()

![image-20220926145954351](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926145954351.png)

#### （1）、mRenderPipeline->getFrame() (dequeueBuffer)

![image-20220926150348579](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926150348579.png)

![image-20220926152445609](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926152445609.png)

![image-20220926152607765](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926152607765.png)

#### （2）、mRenderPipeline->draw

![image-20220926152821530](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926152821530.png)

![image-20220926152702899](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926152702899.png)

![image-20220926152859542](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926152859542.png)

![image-20220926153057124](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926153057124.png)

![image-20220926160712537](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926160712537.png)

![image-20220926160850891](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926160850891.png)

> Y:\home\zhoujinjian\android11_rockpi4\frameworks\base\libs\hwui\RecordingCanvas.cpp

![image-20220926161110171](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926161110171.png)

![image-20220926161133211](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926161133211.png)

![image-20220926161242953](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926161242953.png)

![image-20220926161206863](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220926161206863.png)

上述操作还没真正绘制，只是记录各个图元绘制的一系列属性，draw哪些；如何draw。Skia绘制在下一小节分析。

#### （3）、mRenderPipeline->swapBuffers(queueBuffer)

现在绘制完了，交给SurfaceFlinger做进一步渲染合成。

![image-20220927154630554](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927154630554.png)

## （三）、Skia 简要分析

![image-20220927154848983](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927154848983.png)

### （1）、SkCanvas::drawPaint

> Y:\home\zhoujinjian\android11_rockpi4\external\skia\src\core\SkCanvas.cpp

![image-20220927155422585](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927155422585.png)

> Y:\home\zhoujinjian\android11_rockpi4\external\skia\src\gpu\SkGpuDevice.cpp

![image-20220927155532529](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927155532529.png)

> Y:\home\zhoujinjian\android11_rockpi4\external\skia\src\gpu\GrRenderTargetContext.cpp

![image-20220927155638575](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927155638575.png)

![image-20220927155719017](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927155719017.png)

![image-20220927155753899](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927155753899.png)

![image-20220927155924697](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927155924697.png)

> Y:\home\zhoujinjian\android11_rockpi4\external\skia\src\gpu\GrOpsTask.cpp

![image-20220927160024627](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160024627.png)

可以看到GrOpsTask也只是保存到绘制队列chain中。

### （2）、SkCanvas::drawOval

![image-20220927160210170](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160210170.png)

![image-20220927160228567](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160228567.png)

![image-20220927160254541](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160254541.png)

![image-20220927160339445](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160339445.png)

![image-20220927160406063](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160406063.png)

![image-20220927160440538](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160440538.png)

### （3）、SkCanvas::flush()

接下来才是GPU真正绘制的操作了。前面都是记录一系列draw op。

![image-20220927160550120](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160550120.png)

![image-20220927160910974](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160910974.png)

![image-20220927160949627](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927160949627.png)

![image-20220927161011642](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927161011642.png)

![image-20220927161130083](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927161130083.png)

![image-20220927161244120](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927161244120.png)

![image-20220927161337925](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927161337925.png)

![image-20220927161733678](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927161733678.png)

![image-20220927161810717](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927161810717.png)

![image-20220927162507562](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927162507562.png)

> Y:\home\zhoujinjian\android11_rockpi4\external\skia\src\gpu\GrOpsRenderPass.h

#### （1）、GrOpsRenderPass->begin()

![image-20220927175928835](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927175928835.png)

![image-20220927180051094](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927180051094.png)

![image-20220927180136644](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927180136644.png)

![image-20220927180313245](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927180313245.png)

![image-20220927180938249](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927180938249.png)

![image-20220927181127362](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927181127362.png)

![image-20220927181151892](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927181151892.png)

![image-20220927181234033](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927181234033.png)



#### （2）、GrGLGpu::flushGLState

![image-20220927181820373](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927181820373.png)

#### （3）、GrGLGpu::drawMeshes

![image-20220927181919490](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20220927181919490.png)

#### （4）、GrGLGpu-sendArrayMeshToGpu

![image-20221008151850462](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20221008151850462.png)

![image-20221008151922180](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20221008151922180.png)

![image-20221008155202064](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20221008155202064.png)

#### （5）、GrGLGpu-finishFlush

![image-20221008155427768](https://raw.githubusercontent.com/zhoujinjiann/zhoujinjian.com.images/master/Android_Display_System/Android11_Display06/image-20221008155427768.png)

## （四）、参考资料(特别感谢)：

 [（1）FirstSkiaApp](https://github.com/nightelf3/FirstSkiaApp) 
