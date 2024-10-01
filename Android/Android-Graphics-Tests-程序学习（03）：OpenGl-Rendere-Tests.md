---
title: Android Graphics Tests 程序学习（03）：OpenGl Renderer Tests
cover:  https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/hexo.themes/bing-wallpaper-2018.04.35.jpg
categories: 
  - OpenGL
tags:
  - Android
  - OpenGL
toc: true
abbrlink: 20190424
date: 2019-04-24 09:25:00
---


--------------------------------------------------------------------------------

注：文章都是通过阅读各位前辈总结的资料 Android 8.x && Linux（kernel 4.x）Qualcomm平台源码、加上自己的思考分析总结出来的，其中难免有理解不对的地方，欢迎大家批评指正。文章为个人学习、研究、欣赏之用，图文内容整理自互联网（◕‿◕），如有侵权，请联系删除，禁止转载（©Qualcomm ©Android @Linux 版权所有），谢谢。

正是由于前人的分析和总结，帮助我节约了大量的时间和精力，再次感谢！！！

Google Pixel、Pixel XL 内核代码（==**文章基于 Kernel-4.x**==）：
 [Kernel source for Pixel 2 (walleye) and Pixel 2 XL (taimen) - GitHub](https://github.com/nathanchance/wahoo)

AOSP 源码（==**文章基于 Android 8.x**==）：
 [ Android 系统全套源代码分享 (更新到 8.1.0_r1)](https://testerhome.com/topics/2229)

--------------------------------------------------------------------------------
==源码（部分）==：

>opengl

- android\frameworks\base\tests\HwAccelerationTest

--------------------------------------------------------------------------------

#### (一)、Animation


##### 1.1、Circle Props
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/CirclePropActivity00.png)
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/CirclePropActivity01.png)


源码：

``` java
\android\frameworks\base\tests\HwAccelerationTest\src\com\android\test\hwui\CirclePropActivity.java

public class CirclePropActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        final LinearLayout layout = new LinearLayout(this);
        layout.setOrientation(LinearLayout.VERTICAL);

        ProgressBar spinner = new ProgressBar(this, null, android.R.attr.progressBarStyleLarge);
        layout.addView(spinner, new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT));
        // For testing with a functor in the tree
//        WebView wv = new WebView(this);
//        wv.setWebViewClient(new WebViewClient());
//        wv.setWebChromeClient(new WebChromeClient());
//        wv.loadUrl("http://theverge.com");
//        layout.addView(wv, new LayoutParams(LayoutParams.MATCH_PARENT, 100));

        layout.addView(new CircleView(this),
                new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT));

        setContentView(layout);
    }

    static class CircleView extends View {
        static final int DURATION = 500;

        private boolean mToggle = false;
        ArrayList<RenderNodeAnimator> mRunningAnimations = new ArrayList<RenderNodeAnimator>();

        CanvasProperty<Float> mX;
        CanvasProperty<Float> mY;
        CanvasProperty<Float> mRadius;
        CanvasProperty<Paint> mPaint;

        CircleView(Context c) {
            super(c);
            setClickable(true);

            mX = CanvasProperty.createFloat(200.0f);
            mY = CanvasProperty.createFloat(200.0f);
            mRadius = CanvasProperty.createFloat(150.0f);

            Paint p = new Paint();
            p.setAntiAlias(true);
            p.setColor(0xFFFF0000);
            p.setStyle(Style.STROKE);
            p.setStrokeWidth(60.0f);
            mPaint = CanvasProperty.createPaint(p);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);

            if (canvas.isHardwareAccelerated()) {
                DisplayListCanvas displayListCanvas = (DisplayListCanvas) canvas;
                displayListCanvas.drawCircle(mX, mY, mRadius, mPaint);
            }
        }

        @Override
        public boolean performClick() {
            for (int i = 0; i < mRunningAnimations.size(); i++) {
                mRunningAnimations.get(i).cancel();
            }
            mRunningAnimations.clear();

            mToggle = !mToggle;

            mRunningAnimations.add(new RenderNodeAnimator(
                    mX, mToggle ? 400.0f : 200.0f));

            mRunningAnimations.add(new RenderNodeAnimator(
                    mY, mToggle ? 600.0f : 200.0f));

            mRunningAnimations.add(new RenderNodeAnimator(
                    mRadius, mToggle ? 250.0f : 150.0f));

            mRunningAnimations.add(new RenderNodeAnimator(
                    mPaint, RenderNodeAnimator.PAINT_STROKE_WIDTH,
                    mToggle ? 5.0f : 60.0f));

            mRunningAnimations.add(new RenderNodeAnimator(
                    mPaint, RenderNodeAnimator.PAINT_ALPHA, 64.0f));

            // Will be "chained" to run after the above
            mRunningAnimations.add(new RenderNodeAnimator(
                    mPaint, RenderNodeAnimator.PAINT_ALPHA, 255.0f));

            for (int i = 0; i < mRunningAnimations.size(); i++) {
                RenderNodeAnimator anim = mRunningAnimations.get(i);
                anim.setDuration(1000);
                anim.setTarget(this);
                if (i == (mRunningAnimations.size() - 1)) {
                    // "chain" test
                    anim.setStartValue(64.0f);
                    anim.setStartDelay(anim.getDuration());
                }
                anim.start();
            }

            if (mToggle) {
                post(new Runnable() {
                    @Override
                    public void run() {
                        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "pretendBusy");
                        try {
                            Thread.sleep(DURATION);
                        } catch (InterruptedException e) {
                        }
                        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                    }
                });
            }
            return true;
        }
    }
}

```

##### 1.2、Reveal animation
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/RevealActivity00.png)
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/RevealActivity01.png)

源码：

``` java
\LINUX\android\frameworks\base\tests\HwAccelerationTest\src\com\android\test\hwui\RevealActivity.java

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        final LinearLayout layout = new LinearLayout(this);
        layout.setOrientation(LinearLayout.VERTICAL);

        ProgressBar spinner = new ProgressBar(this, null, android.R.attr.progressBarStyleLarge);
        layout.addView(spinner, new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT));
        View revealView = new MyView(this);
        layout.addView(revealView,
                new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT));
        setContentView(layout);

        revealView.setOnClickListener(this);
    }

    @Override
    public void onClick(View view) {
        Animator animator = ViewAnimationUtils.createCircularReveal(view,
                view.getWidth() / 2, view.getHeight() / 2,
                0, Math.max(view.getWidth(), view.getHeight()));
        Log.d("Reveal", "Calling start...");
        animator.addListener(mListener);
        if (mIteration < 2) {
            animator.setDuration(DURATION);
            animator.start();
        } else {
            AnimatorSet set = new AnimatorSet();
            set.playTogether(animator);
            set.setDuration(DURATION);
            set.addListener(mListener);
            set.start();
        }

        mIteration = (mIteration + 1) % 4;
        mShouldBlock = !mShouldBlock;
        if (mShouldBlock) {
            view.post(sBlockThread);
        }
    }

```

#### (二)、Bitmaps


##### 2.1、Draw Bitmaps
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/BitmapsActivity.png)


源码：

``` java
\android\frameworks\base\tests\HwAccelerationTest\src\com\android\test\hwui\BitmapsActivity.java

public class BitmapsActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        final BitmapsView view = new BitmapsView(this);
        final FrameLayout layout = new FrameLayout(this);
        layout.addView(view, new FrameLayout.LayoutParams(480, 800, Gravity.CENTER));
        setContentView(layout);
        
        ScaleAnimation a = new ScaleAnimation(1.0f, 2.0f, 1.0f, 2.0f,
                ScaleAnimation.RELATIVE_TO_SELF, 0.5f,
                ScaleAnimation.RELATIVE_TO_SELF,0.5f);
        a.setDuration(2000);
        a.setRepeatCount(Animation.INFINITE);
        a.setRepeatMode(Animation.REVERSE);
        view.startAnimation(a);
    }

    static class BitmapsView extends View {
        private Paint mBitmapPaint;
        private final Bitmap mBitmap1;
        private final Bitmap mBitmap2;
        private final PorterDuffXfermode mDstIn;

        BitmapsView(Context c) {
            super(c);

            mBitmap1 = BitmapFactory.decodeResource(c.getResources(), R.drawable.sunset1);
            mBitmap2 = BitmapFactory.decodeResource(c.getResources(), R.drawable.sunset2);
            
            Log.d("Bitmap", "mBitmap1.isMutable() = " + mBitmap1.isMutable());
            Log.d("Bitmap", "mBitmap2.isMutable() = " + mBitmap2.isMutable());

            BitmapFactory.Options opts = new BitmapFactory.Options();
            opts.inMutable = true;
            Bitmap bitmap = BitmapFactory.decodeResource(c.getResources(), R.drawable.sunset1, opts);
            Log.d("Bitmap", "bitmap.isMutable() = " + bitmap.isMutable());
            
            mBitmapPaint = new Paint();
            mDstIn = new PorterDuffXfermode(PorterDuff.Mode.DST_IN);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);

            canvas.translate(120.0f, 50.0f);
            canvas.drawBitmap(mBitmap1, 0.0f, 0.0f, mBitmapPaint);

            canvas.translate(0.0f, mBitmap1.getHeight());
            canvas.translate(0.0f, 25.0f);
            canvas.drawBitmap(mBitmap2, 0.0f, 0.0f, null);
            
            mBitmapPaint.setAlpha(127);
            canvas.translate(0.0f, mBitmap2.getHeight());
            canvas.translate(0.0f, 25.0f);
            canvas.drawBitmap(mBitmap1, 0.0f, 0.0f, mBitmapPaint);
            
            mBitmapPaint.setAlpha(255);
            canvas.translate(0.0f, mBitmap1.getHeight());
            canvas.translate(0.0f, 25.0f);
            mBitmapPaint.setColor(0xffff0000);
            canvas.drawRect(0.0f, 0.0f, mBitmap2.getWidth(), mBitmap2.getHeight(), mBitmapPaint);
            mBitmapPaint.setXfermode(mDstIn);
            canvas.drawBitmap(mBitmap2, 0.0f, 0.0f, mBitmapPaint);

            mBitmapPaint.reset();
        }
    }
}

```

##### 2.2、Rect

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/BitmapsRectActivity.png)

源码：

``` cpp
\android\frameworks\base\tests\HwAccelerationTest\src\com\android\test\hwui\BitmapsRectActivity.java

public class BitmapsRectActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        final BitmapsView view = new BitmapsView(this);
        setContentView(view);
    }

    static class BitmapsView extends View {
        private Paint mBitmapPaint;
        private final Bitmap mBitmap1;
        private final Bitmap mBitmap2;
        private final Rect mSrcRect;
        private final RectF mDstRect;
        private final RectF mDstRect2;

        BitmapsView(Context c) {
            super(c);

            mBitmap1 = BitmapFactory.decodeResource(c.getResources(), R.drawable.sunset1);
            mBitmap2 = BitmapFactory.decodeResource(c.getResources(), R.drawable.sunset2);

            mBitmapPaint = new Paint();
            mBitmapPaint.setFilterBitmap(true);

            final float fourth = mBitmap1.getWidth() / 4.0f;
            final float half = mBitmap1.getHeight() / 2.0f;
            mSrcRect = new Rect((int) fourth, (int) (half - half / 2.0f),
                    (int) (fourth + fourth), (int) (half + half / 2.0f));
            mDstRect = new RectF(fourth, half - half / 2.0f, fourth + fourth, half + half / 2.0f);
            mDstRect2 = new RectF(fourth, half - half / 2.0f,
                    (fourth + fourth) * 3.0f, (half + half / 2.0f) * 3.0f);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);

            canvas.translate(120.0f, 50.0f);
            canvas.drawBitmap(mBitmap1, mSrcRect, mDstRect, mBitmapPaint);

            canvas.translate(0.0f, mBitmap1.getHeight());
            canvas.translate(-100.0f, 25.0f);
            canvas.drawBitmap(mBitmap1, mSrcRect, mDstRect2, mBitmapPaint);
        }
    }
}
```


#### (三)、View
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/ViewPropertyAlphaActivity.png)

``` java
public class ViewPropertyAlphaActivity extends Activity {
    
    MyView myViewAlphaDefault, myViewAlphaHandled;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.view_properties);

        getWindow().getDecorView().postDelayed(new Runnable() {
            @Override
            public void run() {
                startAnim(R.id.button);
                startAnim(R.id.textview);
                startAnim(R.id.spantext);
                startAnim(R.id.edittext);
                startAnim(R.id.selectedtext);
                startAnim(R.id.textviewbackground);
                startAnim(R.id.layout);
                startAnim(R.id.imageview);
                startAnim(myViewAlphaDefault);
                startAnim(myViewAlphaHandled);
                EditText selectedText = findViewById(R.id.selectedtext);
                selectedText.setSelection(3, 8);
            }
        }, 2000);
        
        Button invalidator = findViewById(R.id.invalidateButton);
        invalidator.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                findViewById(R.id.textview).invalidate();
                findViewById(R.id.spantext).invalidate();
            }
        });

        TextView textView = findViewById(R.id.spantext);
        if (textView != null) {
            SpannableStringBuilder text =
                    new SpannableStringBuilder("Now this is a short text message with spans");

            text.setSpan(new BackgroundColorSpan(Color.RED), 0, 3,
                    Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
            text.setSpan(new ForegroundColorSpan(Color.BLUE), 4, 9,
                    Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
            text.setSpan(new SuggestionSpan(this, new String[]{"longer"}, 3), 11, 16,
                    Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
            text.setSpan(new UnderlineSpan(), 17, 20,
                    Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
            text.setSpan(new ImageSpan(this, R.drawable.icon), 21, 22,
                    Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);

            textView.setText(text);
        }
        
        LinearLayout container = findViewById(R.id.container);
        myViewAlphaDefault = new MyView(this, false);
        myViewAlphaDefault.setLayoutParams(new LinearLayout.LayoutParams(75, 75));
        container.addView(myViewAlphaDefault);
        myViewAlphaHandled = new MyView(this, true);
        myViewAlphaHandled.setLayoutParams(new LinearLayout.LayoutParams(75, 75));
        container.addView(myViewAlphaHandled);
    }

    private void startAnim(View target) {
        ObjectAnimator anim = ObjectAnimator.ofFloat(target, View.ALPHA, 0);
        anim.setRepeatCount(ValueAnimator.INFINITE);
        anim.setRepeatMode(ValueAnimator.REVERSE);
        anim.setDuration(1000);
        anim.start();
    }
    private void startAnim(int id) {
        startAnim(findViewById(id));
    }
    
    private static class MyView extends View {
        private int mMyAlpha = 255;
        private boolean mHandleAlpha;
        private Paint mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        
        private MyView(Context context, boolean handleAlpha) {
            super(context);
            mHandleAlpha = handleAlpha;
            mPaint.setColor(Color.RED);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            if (mHandleAlpha) {
                mPaint.setAlpha(mMyAlpha);
            }
            canvas.drawCircle(30, 30, 30, mPaint);
        }

        @Override
        protected boolean onSetAlpha(int alpha) {
            if (mHandleAlpha) {
                mMyAlpha = alpha;
                return true;
            }
            return super.onSetAlpha(alpha);
        }
    }

}

```

#### (四)、ColorFilters


##### 4.1、ColorFilters/Mutate Filters
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/ColorFiltersMutateActivity.png)


源码：

``` java
public class ColorFiltersMutateActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        final BitmapsView view = new BitmapsView(this);
        setContentView(view);
    }

    static class BitmapsView extends View {
        private final Bitmap mBitmap1;
        private final Bitmap mBitmap2;
        private final Paint mColorMatrixPaint;
        private final Paint mLightingPaint;
        private final Paint mBlendPaint;

        private float mSaturation = 0.0f;
        private int mLightAdd = 0;
        private int mLightMul = 0;
        private int mPorterDuffColor = 0;

        BitmapsView(Context c) {
            super(c);

            mBitmap1 = BitmapFactory.decodeResource(c.getResources(), R.drawable.sunset1);
            mBitmap2 = BitmapFactory.decodeResource(c.getResources(), R.drawable.sunset2);

            mColorMatrixPaint = new Paint();
            final ColorMatrix colorMatrix = new ColorMatrix();
            colorMatrix.setSaturation(0);
            mColorMatrixPaint.setColorFilter(new ColorMatrixColorFilter(colorMatrix));

            mLightingPaint = new Paint();
            mLightingPaint.setColorFilter(new LightingColorFilter(0, 0));

            mBlendPaint = new Paint();
            mBlendPaint.setColorFilter(new PorterDuffColorFilter(0, PorterDuff.Mode.SRC_OVER));

            ObjectAnimator sat = ObjectAnimator.ofFloat(this, "saturation", 1.0f);
            sat.setDuration(1000);
            sat.setRepeatCount(ObjectAnimator.INFINITE);
            sat.setRepeatMode(ObjectAnimator.REVERSE);
            sat.start();

            ObjectAnimator light = ObjectAnimator.ofInt(this, "lightAdd", 0x00101030);
            light.setEvaluator(new ArgbEvaluator());
            light.setDuration(1000);
            light.setRepeatCount(ObjectAnimator.INFINITE);
            light.setRepeatMode(ObjectAnimator.REVERSE);
            light.start();

            ObjectAnimator mult = ObjectAnimator.ofInt(this, "lightMul", 0x0060ffff);
            mult.setEvaluator(new ArgbEvaluator());
            mult.setDuration(1000);
            mult.setRepeatCount(ObjectAnimator.INFINITE);
            mult.setRepeatMode(ObjectAnimator.REVERSE);
            mult.start();

            ObjectAnimator color = ObjectAnimator.ofInt(this, "porterDuffColor", 0x7f990040);
            color.setEvaluator(new ArgbEvaluator());
            color.setDuration(1000);
            color.setRepeatCount(ObjectAnimator.INFINITE);
            color.setRepeatMode(ObjectAnimator.REVERSE);
            color.start();
        }

        public int getPorterDuffColor() {
            return mPorterDuffColor;
        }

        public void setPorterDuffColor(int porterDuffColor) {
            mPorterDuffColor = porterDuffColor;
            final PorterDuffColorFilter filter =
                    (PorterDuffColorFilter) mBlendPaint.getColorFilter();
            filter.setColor(mPorterDuffColor);
            invalidate();
        }

        public int getLightAdd() {
            return mLightAdd;
        }

        public void setLightAdd(int lightAdd) {
            mLightAdd = lightAdd;
            final LightingColorFilter filter =
                    (LightingColorFilter) mLightingPaint.getColorFilter();
            filter.setColorAdd(lightAdd);
            invalidate();
        }

        public int getLightMul() {
            return mLightAdd;
        }

        public void setLightMul(int lightMul) {
            mLightMul = lightMul;
            final LightingColorFilter filter =
                    (LightingColorFilter) mLightingPaint.getColorFilter();
            filter.setColorMultiply(lightMul);
            invalidate();
        }

        public void setSaturation(float saturation) {
            mSaturation = saturation;
            final ColorMatrixColorFilter filter =
                    (ColorMatrixColorFilter) mColorMatrixPaint.getColorFilter();
            final ColorMatrix m = new ColorMatrix();
            m.setSaturation(saturation);
            filter.setColorMatrix(m);
            invalidate();
        }

        public float getSaturation() {
            return mSaturation;
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);

            canvas.drawARGB(255, 255, 255, 255);

            canvas.save();
            canvas.translate(120.0f, 50.0f);
            canvas.drawBitmap(mBitmap1, 0.0f, 0.0f, mColorMatrixPaint);

            canvas.translate(0.0f, 50.0f + mBitmap1.getHeight());
            canvas.drawBitmap(mBitmap1, 0.0f, 0.0f, mLightingPaint);

            canvas.translate(0.0f, 50.0f + mBitmap1.getHeight());
            canvas.drawBitmap(mBitmap1, 0.0f, 0.0f, mBlendPaint);
            canvas.restore();

            canvas.save();
            canvas.translate(120.0f + mBitmap1.getWidth() + 120.0f, 50.0f);
            canvas.drawBitmap(mBitmap2, 0.0f, 0.0f, mColorMatrixPaint);

            canvas.translate(0.0f, 50.0f + mBitmap2.getHeight());
            canvas.drawBitmap(mBitmap2, 0.0f, 0.0f, mLightingPaint);

            canvas.translate(0.0f, 50.0f + mBitmap2.getHeight());
            canvas.drawBitmap(mBitmap2, 0.0f, 0.0f, mBlendPaint);
            canvas.restore();
        }
    }
}

```


#### (五)、Draw


##### 5.1、Animated 3D transform
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/Animated3dActivity.png)


``` java
LINUX\android\frameworks\base\tests\HwAccelerationTest\src\com\android\test\hwui\Animated3dActivity.java
public class Animated3dActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        ImageView view = new ImageView(this);
        view.setImageResource(R.drawable.large_photo);

        setContentView(view, new FrameLayout.LayoutParams(
                FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT
        ));

        ObjectAnimator animator = ObjectAnimator.ofFloat(view, "rotationY", 0.0f, 360.0f);
        animator.setDuration(4000);
        animator.setRepeatCount(ObjectAnimator.INFINITE);
        animator.setRepeatMode(ObjectAnimator.REVERSE);
        animator.start();
    }
}


```


##### 5.2、Xfermodes
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/XfermodeActivity.png)

``` java
LINUX\android\frameworks\base\tests\HwAccelerationTest\src\com\android\test\hwui\XfermodeActivity.java
import static android.graphics.PorterDuff.Mode;

SuppressWarnings({"UnusedDeclaration"})
public class XfermodeActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        setContentView(new XfermodesView(this));
    }

    static class XfermodesView extends View {
        private final Paint mBluePaint;
        private final Paint mRedPaint;

        XfermodesView(Context c) {
            super(c);

            mBluePaint = new Paint();
            mBluePaint.setColor(0xff0000ff);
            
            mRedPaint = new Paint();
            mRedPaint.setColor(0x7fff0000);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);
            //canvas.drawRGB(255, 255, 255);

            canvas.translate(100.0f, 100.0f);
            
            // SRC modes
            canvas.save();

            drawRects(canvas, Mode.SRC_OVER);
            canvas.translate(0.0f, 100.0f);

            drawRects(canvas, Mode.SRC_IN);
            canvas.translate(0.0f, 100.0f);            

            drawRects(canvas, Mode.SRC_OUT);
            canvas.translate(0.0f, 100.0f);

            drawRects(canvas, Mode.SRC_ATOP);
            canvas.translate(0.0f, 100.0f);
            
            drawRects(canvas, Mode.SRC);

            canvas.restore();
            
            canvas.translate(100.0f, 0.0f);
            
            // DST modes
            canvas.save();

            drawRects(canvas, Mode.DST_OVER);
            canvas.translate(0.0f, 100.0f);

            drawRects(canvas, Mode.DST_IN);
            canvas.translate(0.0f, 100.0f);            

            drawRects(canvas, Mode.DST_OUT);
            canvas.translate(0.0f, 100.0f);

            drawRects(canvas, Mode.DST_ATOP);
            canvas.translate(0.0f, 100.0f);
            
            drawRects(canvas, Mode.DST);

            canvas.restore();
            
            canvas.translate(100.0f, 0.0f);
            
            // Other modes
            canvas.save();

            drawRects(canvas, Mode.CLEAR);
            canvas.translate(0.0f, 100.0f);

            drawRects(canvas, Mode.XOR);
            
            canvas.translate(0.0f, 100.0f);
            
            mBluePaint.setAlpha(127);
            canvas.drawRect(0.0f, 0.0f, 50.0f, 50.0f, mBluePaint);
            
            canvas.translate(0.0f, 100.0f);
            
            mBluePaint.setAlpha(10);
            mBluePaint.setColor(0x7f0000ff);
            canvas.drawRect(0.0f, 0.0f, 50.0f, 50.0f, mBluePaint);
            
            mBluePaint.setColor(0xff0000ff);
            mBluePaint.setAlpha(255);

            canvas.restore();
        }

        private void drawRects(Canvas canvas, PorterDuff.Mode mode) {
            canvas.drawRect(0.0f, 0.0f, 50.0f, 50.0f, mBluePaint);

            canvas.save();
            canvas.translate(25.0f, 25.0f);
            mRedPaint.setXfermode(new PorterDuffXfermode(mode));
            canvas.drawRect(0.0f, 0.0f, 50.0f, 50.0f, mRedPaint);
            canvas.restore();
        }
    }
}

```


#### (六)、Gradients


##### 6.1、Gradients

![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/GradientsActivity.png)

``` java
\android\frameworks\base\tests\HwAccelerationTest\src\com\android\test\hwui\GradientsActivity.java

public class GradientsActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        final FrameLayout layout = new FrameLayout(this);

        final ShadersView shadersView = new ShadersView(this);
        final GradientView gradientView = new GradientView(this);
        final RadialGradientView radialGradientView = new RadialGradientView(this);
        final SweepGradientView sweepGradientView = new SweepGradientView(this);
        final BitmapView bitmapView = new BitmapView(this);

        final SeekBar rotateView = new SeekBar(this);
        rotateView.setMax(360);
        rotateView.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            public void onStopTrackingTouch(SeekBar seekBar) {
            }

            public void onStartTrackingTouch(SeekBar seekBar) {
            }

            public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
                gradientView.setRotationY((float) progress);
                radialGradientView.setRotationX((float) progress);
                sweepGradientView.setRotationY((float) progress);
                bitmapView.setRotationX((float) progress);
            }
        });

        layout.addView(shadersView);
        layout.addView(gradientView, new FrameLayout.LayoutParams(
                200, 200, Gravity.CENTER));

        FrameLayout.LayoutParams lp = new FrameLayout.LayoutParams(200, 200, Gravity.CENTER);
        lp.setMargins(220, 0, 0, 0);
        layout.addView(radialGradientView, lp);

        lp = new FrameLayout.LayoutParams(200, 200, Gravity.CENTER);
        lp.setMargins(440, 0, 0, 0);
        layout.addView(sweepGradientView, lp);

        lp = new FrameLayout.LayoutParams(200, 200, Gravity.CENTER);
        lp.setMargins(220, -220, 0, 0);
        layout.addView(bitmapView, lp);

        layout.addView(rotateView, new FrameLayout.LayoutParams(
                300, FrameLayout.LayoutParams.WRAP_CONTENT,
                Gravity.CENTER_HORIZONTAL | Gravity.BOTTOM));

        setContentView(layout);
    }

    static class BitmapView extends View {
        private final Paint mPaint;

        BitmapView(Context c) {
            super(c);

            Bitmap texture = BitmapFactory.decodeResource(c.getResources(), R.drawable.sunset1);
            BitmapShader shader = new BitmapShader(texture, Shader.TileMode.REPEAT,
                    Shader.TileMode.REPEAT);
            mPaint = new Paint();
            mPaint.setShader(shader);
        }

        @Override
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
            setMeasuredDimension(200, 200);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);
            canvas.drawRect(0.0f, 0.0f, getWidth(), getHeight(), mPaint);
        }
    }

    static class GradientView extends View {
        private final Paint mPaint;

        GradientView(Context c) {
            super(c);

            LinearGradient gradient = new LinearGradient(0, 0, 200, 0, 0xFF000000, 0,
                    Shader.TileMode.CLAMP);
            mPaint = new Paint();
            mPaint.setShader(gradient);
        }

        @Override
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
            setMeasuredDimension(200, 200);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);
            canvas.drawRect(0.0f, 0.0f, getWidth(), getHeight(), mPaint);
        }
    }

    static class RadialGradientView extends View {
        private final Paint mPaint;

        RadialGradientView(Context c) {
            super(c);

            RadialGradient gradient = new RadialGradient(0.0f, 0.0f, 100.0f, 0xff000000, 0xffffffff,
                    Shader.TileMode.MIRROR);
            mPaint = new Paint();
            mPaint.setShader(gradient);
        }

        @Override
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
            setMeasuredDimension(200, 200);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);
            canvas.drawRect(0.0f, 0.0f, getWidth(), getHeight(), mPaint);
        }
    }

    static class SweepGradientView extends View {
        private final Paint mPaint;

        SweepGradientView(Context c) {
            super(c);

            SweepGradient gradient = new SweepGradient(100.0f, 100.0f, 0xff000000, 0xffffffff);
            mPaint = new Paint();
            mPaint.setShader(gradient);
        }

        @Override
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
            setMeasuredDimension(200, 200);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);
            canvas.drawRect(0.0f, 0.0f, getWidth(), getHeight(), mPaint);
        }
    }

    static class ShadersView extends View {
        private final Paint mPaint;
        private final float mDrawWidth;
        private final float mDrawHeight;
        private final LinearGradient mGradient;
        private final LinearGradient mGradientStops;
        private final Matrix mMatrix;

        ShadersView(Context c) {
            super(c);

            mDrawWidth = 200;
            mDrawHeight = 200;

            mGradient = new LinearGradient(0, 0, 0, 1, 0xFF000000, 0, Shader.TileMode.CLAMP);
            mGradientStops = new LinearGradient(0, 0, 0, 1,
                    new int[] { 0xFFFF0000, 0xFF00FF00, 0xFF0000FF }, null, Shader.TileMode.CLAMP);

            mMatrix = new Matrix();

            mPaint = new Paint();
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);
            canvas.drawRGB(255, 255, 255);

            // Gradients
            canvas.save();
            float top = 40.0f;
            float right = 40.0f + mDrawWidth;
            float left = 40.0f;
            float bottom = 40.0f + mDrawHeight;

            mMatrix.setScale(1, mDrawWidth);
            mMatrix.postRotate(90);
            mMatrix.postTranslate(right, top);
            mGradient.setLocalMatrix(mMatrix);
            mPaint.setShader(mGradient);
            canvas.drawRect(right - mDrawWidth, top, right, top + mDrawHeight, mPaint);

            top += 40.0f + mDrawHeight;
            bottom += 40.0f + mDrawHeight;

            mMatrix.setScale(1, mDrawHeight);
            mMatrix.postTranslate(left, top);
            mGradient.setLocalMatrix(mMatrix);
            mPaint.setShader(mGradient);
            canvas.drawRect(left, top, right, top + mDrawHeight, mPaint);

            left += 40.0f + mDrawWidth;
            right += 40.0f + mDrawWidth;
            top -= 40.0f + mDrawHeight;
            bottom -= 40.0f + mDrawHeight;

            mMatrix.setScale(1, mDrawHeight);
            mMatrix.postRotate(180);
            mMatrix.postTranslate(left, bottom);
            mGradient.setLocalMatrix(mMatrix);
            mPaint.setShader(mGradient);
            canvas.drawRect(left, bottom - mDrawHeight, right, bottom, mPaint);

            top += 40.0f + mDrawHeight;
            bottom += 40.0f + mDrawHeight;

            mMatrix.setScale(1, mDrawWidth);
            mMatrix.postRotate(-90);
            mMatrix.postTranslate(left, top);
            mGradient.setLocalMatrix(mMatrix);
            mPaint.setShader(mGradient);
            canvas.drawRect(left, top, left + mDrawWidth, bottom, mPaint);

            right = left + mDrawWidth;
            left = 40.0f;
            top = bottom + 20.0f;
            bottom = top + 50.0f;

            mMatrix.setScale(1, mDrawWidth);
            mMatrix.postRotate(90);
            mMatrix.postTranslate(right, top);
            mGradientStops.setLocalMatrix(mMatrix);
            mPaint.setShader(mGradientStops);
            canvas.drawRect(left, top, right, bottom, mPaint);
            
            canvas.restore();
        }
    }
}

```


#### (七)、Path


##### 7.1、Shapes
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/ShapesActivity.png)

``` cpp
public class ShapesActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(new ShapesView(this));
    }

    static class ShapesView extends View {
        private final Paint mNormalPaint;
        private final Paint mStrokePaint;
        private final Paint mFillPaint;
        private final RectF mRect;
        private final RectF mOval;
        private final RectF mArc;
        private final Path mTriangle;

        ShapesView(Context c) {
            super(c);

            mRect = new RectF(0.0f, 0.0f, 160.0f, 90.0f);

            mNormalPaint = new Paint();
            mNormalPaint.setAntiAlias(true);
            mNormalPaint.setColor(0xff0000ff);
            mNormalPaint.setStrokeWidth(6.0f);
            mNormalPaint.setStyle(Paint.Style.FILL_AND_STROKE);

            mStrokePaint = new Paint();
            mStrokePaint.setAntiAlias(true);
            mStrokePaint.setColor(0xff0000ff);
            mStrokePaint.setStrokeWidth(6.0f);
            mStrokePaint.setStyle(Paint.Style.STROKE);
            
            mFillPaint = new Paint();
            mFillPaint.setAntiAlias(true);
            mFillPaint.setColor(0xff0000ff);
            mFillPaint.setStyle(Paint.Style.FILL);

            mOval = new RectF(0.0f, 0.0f, 80.0f, 45.0f);
            mArc = new RectF(0.0f, 0.0f, 100.0f, 120.0f);

            mTriangle = new Path();
            mTriangle.moveTo(0.0f, 90.0f);
            mTriangle.lineTo(45.0f, 0.0f);
            mTriangle.lineTo(90.0f, 90.0f);
            mTriangle.close();
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);

            canvas.save();
            canvas.translate(50.0f, 50.0f);
            canvas.drawRoundRect(mRect, 6.0f, 6.0f, mNormalPaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawRoundRect(mRect, 6.0f, 6.0f, mStrokePaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawRoundRect(mRect, 6.0f, 6.0f, mFillPaint);
            canvas.restore();

            canvas.save();
            canvas.translate(250.0f, 50.0f);
            canvas.drawCircle(80.0f, 45.0f, 45.0f, mNormalPaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawCircle(80.0f, 45.0f, 45.0f, mStrokePaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawCircle(80.0f, 45.0f, 45.0f, mFillPaint);
            canvas.restore();

            canvas.save();
            canvas.translate(450.0f, 50.0f);
            canvas.drawOval(mOval, mNormalPaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawOval(mOval, mStrokePaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawOval(mOval, mFillPaint);
            canvas.restore();

            canvas.save();
            canvas.translate(625.0f, 50.0f);
            canvas.drawRect(0.0f, 0.0f, 160.0f, 90.0f, mNormalPaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawRect(0.0f, 0.0f, 160.0f, 90.0f, mStrokePaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawRect(0.0f, 0.0f, 160.0f, 90.0f, mFillPaint);
            canvas.restore();

            canvas.save();
            canvas.translate(825.0f, 50.0f);
            canvas.drawArc(mArc, -30.0f, 70.0f, true, mNormalPaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawArc(mArc, -30.0f, 70.0f, true, mStrokePaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawArc(mArc, -30.0f, 70.0f, true, mFillPaint);
            canvas.restore();
            
            canvas.save();
            canvas.translate(950.0f, 50.0f);
            canvas.drawArc(mArc, 30.0f, 100.0f, false, mNormalPaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawArc(mArc, 30.0f, 100.0f, false, mStrokePaint);

            canvas.translate(0.0f, 110.0f);
            canvas.drawArc(mArc, 30.0f, 100.0f, false, mFillPaint);
            canvas.restore();

            canvas.save();
            canvas.translate(50.0f, 400.0f);
            canvas.drawPath(mTriangle, mNormalPaint);

            canvas.translate(110.0f, 0.0f);
            canvas.drawPath(mTriangle, mStrokePaint);

            canvas.translate(110.0f, 0.0f);
            canvas.drawPath(mTriangle, mFillPaint);
            canvas.restore();
        }
    }
}

```

#### (八)、Window


##### 8.1、PixelCopyWindow
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/PixelCopyWindow.png)


``` cpp
public class PixelCopyWindow extends Activity {

    private Handler mHandler;
    private ImageView mImage;
    private TextView mText;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mHandler = new Handler();

        LinearLayout layout = new LinearLayout(this);
        TextView text = new TextView(this);
        text.setText("Hello, World!");
        Button btn = new Button(this);
        btn.setText("Screenshot!");
        btn.setOnClickListener((v) -> takeScreenshot());
        mImage = new ImageView(this);
        mText = new TextView(this);

        layout.setOrientation(LinearLayout.VERTICAL);
        layout.addView(text);
        layout.addView(btn);
        layout.addView(mImage);
        layout.addView(mText);
        final float density = getResources().getDisplayMetrics().density;
        layout.setBackground(new Drawable() {
            Paint mPaint = new Paint();

            @Override
            public void draw(Canvas canvas) {
                mPaint.setStyle(Style.STROKE);
                mPaint.setStrokeWidth(4 * density);
                mPaint.setColor(Color.BLUE);
                final Rect bounds = getBounds();
                canvas.drawRect(bounds, mPaint);
                mPaint.setColor(Color.RED);
                canvas.drawLine(bounds.centerX(), 0, bounds.centerX(), bounds.height(), mPaint);
                mPaint.setColor(Color.GREEN);
                canvas.drawLine(0, bounds.centerY(), bounds.width(), bounds.centerY(), mPaint);
            }

            @Override
            public void setAlpha(int alpha) {
            }

            @Override
            public void setColorFilter(ColorFilter colorFilter) {
            }

            @Override
            public int getOpacity() {
                return PixelFormat.TRANSLUCENT;
            }
        });
        setContentView(layout);
    }

    private void takeScreenshot() {
        View decor = getWindow().getDecorView();
        Rect srcRect = new Rect();
        decor.getGlobalVisibleRect(srcRect);
        final Bitmap bitmap = Bitmap.createBitmap(
                (int) (srcRect.width() * .25), (int) (srcRect.height() * .25), Config.ARGB_8888);
        PixelCopy.request(getWindow(), srcRect, bitmap, (result) -> {
            if (result != PixelCopy.SUCCESS) {
                mText.setText("Copy failed, result: " + result);
                mImage.setImageBitmap(null);
            } else {
                mText.setText("");
                mImage.setImageBitmap(bitmap);
            }
        }, mHandler);
    }
}

```


#### (九)、Text


##### 9.1、Scaled
![Alt text | center](https://raw.githubusercontent.com/zhoujinjianm/zhoujinjian.com.images/master/others/ScaledTextActivity.png)

``` cpp
public class ScaledTextActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        final ScaledTextView view = new ScaledTextView(this);
        setContentView(view);

        ObjectAnimator animation = ObjectAnimator.ofFloat(view, "textScale", 1.0f, 10.0f);
        animation.setDuration(3000);
        animation.setRepeatCount(ObjectAnimator.INFINITE);
        animation.setRepeatMode(ObjectAnimator.REVERSE);
        animation.start();

    }

    public static class ScaledTextView extends View {
        private static final String TEXT = "Hello libhwui! ";

        private final Paint mPaint;
        private final Paint mShadowPaint;
        private final Path mPath;

        private float mScale = 1.0f;

        public ScaledTextView(Context c) {
            super(c);
            setLayerType(LAYER_TYPE_HARDWARE, null);

            mPath = makePath();

            mPaint = new Paint();
            mPaint.setAntiAlias(true);
            mPaint.setTextSize(20.0f);

            mShadowPaint = new Paint();
            mShadowPaint.setAntiAlias(true);
            mShadowPaint.setShadowLayer(3.0f, 0.0f, 3.0f, 0xff000000);
            mShadowPaint.setTextSize(20.0f);
        }

        public float getTextScale() {
            return mScale;
        }

        public void setTextScale(float scale) {
            mScale = scale;
            invalidate();
        }

        private static Path makePath() {
            Path path = new Path();
            buildPath(path);
            return path;
        }

        private static void buildPath(Path path) {
            path.moveTo(0.0f, 0.0f);
            path.cubicTo(0.0f, 0.0f, 100.0f, 150.0f, 100.0f, 200.0f);
            path.cubicTo(100.0f, 200.0f, 50.0f, 300.0f, -80.0f, 200.0f);
            path.cubicTo(-80.0f, 200.0f, 100.0f, 200.0f, 200.0f, 0.0f);
        }

        @Override
        protected void onDraw(Canvas canvas) {
            super.onDraw(canvas);

            canvas.drawText(TEXT, 30.0f, 30.0f, mPaint);
            mPaint.setTextAlign(Paint.Align.CENTER);
            canvas.drawText(TEXT, 30.0f, 50.0f, mPaint);
            mPaint.setTextAlign(Paint.Align.RIGHT);
            canvas.drawText(TEXT, 30.0f, 70.0f, mPaint);

            canvas.save();
            canvas.translate(400.0f, 0.0f);
            canvas.scale(3.0f, 3.0f);
            mPaint.setTextAlign(Paint.Align.LEFT);
            mPaint.setStrikeThruText(true);
            canvas.drawText(TEXT, 30.0f, 30.0f, mPaint);
            mPaint.setStrikeThruText(false);
            mPaint.setTextAlign(Paint.Align.CENTER);
            canvas.drawText(TEXT, 30.0f, 50.0f, mPaint);
            mPaint.setTextAlign(Paint.Align.RIGHT);
            canvas.drawText(TEXT, 30.0f, 70.0f, mPaint);
            canvas.restore();

            mPaint.setTextAlign(Paint.Align.LEFT);
            canvas.translate(0.0f, 100.0f);

            canvas.save();
            canvas.scale(mScale, mScale);
            canvas.drawText(TEXT, 30.0f, 30.0f, mPaint);
            canvas.restore();

            canvas.translate(0.0f, 250.0f);
            canvas.save();
            canvas.scale(3.0f, 3.0f);
            canvas.drawText(TEXT, 30.0f, 30.0f, mShadowPaint);
            canvas.translate(100.0f, 0.0f);
//            canvas.drawTextOnPath(TEXT + TEXT + TEXT, mPath, 0.0f, 0.0f, mPaint);
            canvas.restore();

            float width = mPaint.measureText(TEXT);

            canvas.translate(500.0f, 0.0f);
            canvas.rotate(45.0f, width * 3.0f / 2.0f, 0.0f);
            canvas.scale(3.0f, 3.0f);
            canvas.drawText(TEXT, 30.0f, 30.0f, mPaint);
        }
    }
}

```


#### 参考文档：
[Android Region代码分析](https://blog.csdn.net/fuyajun01/article/details/25551717)