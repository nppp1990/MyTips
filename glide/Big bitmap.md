# Glide解决图片Bitmap too large to be uploaded into a texture问题

[TOC]

## 问题

项目中在用glide加载大图时，可能会报错Bitmap too large to be uploaded into a texture (7648x375, max=4096x4096)，并且图片无法显示。按字面的意识，就是手机对图片大小有限制，超过了这个值就不能显示

## 解决方法

### 方法一：android:hardwareAccelerated

加上android:hardwareAccelerated="false"禁止硬件加速，但是貌似在某些手机上不好使

### 方法二：BitmapRegionDecoder

利用BitmapRegionDecoder截取图片部分区域展示，超出的部分利用监听手势滑动的方式展示出。

```java
decodeRegion(Rect rect, BitmapFactory.Options options)
Decodes a rectangle region in the image specified by rect.
```

但是项目中需要把图片完整的展示在列表中，所以这种方法不合适

具体参考：https://developer.android.com/reference/android/graphics/BitmapRegionDecoder.html

###方法三：切割成多个Bitmap

rt，利用Bitmap.createBitmap切割成多个bitmap展示，实现效果应该是最好的，但是项目中大图时作为RecyclerView的一个item展示的，所以实现起来很麻烦，先放弃了

### 方法四：Transformation转换成小图展示

bitmap的高宽如果大于硬件加速允许的最大值就把bitmap缩放成小图展示

#### 读取硬件加速最大的高宽值

可以用android.opengl.GLES10获取openGl最大宽度，但是有的手机得到的值可能为0，这时候可以用canvas的getMaximumBitmapHeight获取

```java
    /**
     * Returns the maximum allowed height for bitmaps drawn with this canvas.
     * Attempting to draw with a bitmap taller than this value will result
     * in an error.
     *
     * @see #getMaximumBitmapWidth()
     */
    public int getMaximumBitmapHeight() {
        return MAXMIMUM_BITMAP_SIZE;
    }
```

最终方法如下

```java
public final class OpenGlUtils {

    private OpenGlUtils() {
    }

    private static int getGLESTextureLimitBelowLollipop() {
        int[] maxSize = new int[1];
        GLES10.glGetIntegerv(GLES10.GL_MAX_TEXTURE_SIZE, maxSize, 0);
        return maxSize[0];
    }

    private static int getGLESTextureLimitEqualAboveLollipop() {
        EGL10 egl = (EGL10) EGLContext.getEGL();
        EGLDisplay dpy = egl.eglGetDisplay(EGL10.EGL_DEFAULT_DISPLAY);
        int[] vers = new int[2];
        egl.eglInitialize(dpy, vers);
        int[] configAttr = {
                EGL10.EGL_COLOR_BUFFER_TYPE, EGL10.EGL_RGB_BUFFER,
                EGL10.EGL_LEVEL, 0,
                EGL10.EGL_SURFACE_TYPE, EGL10.EGL_PBUFFER_BIT,
                EGL10.EGL_NONE
        };
        EGLConfig[] configs = new EGLConfig[1];
        int[] numConfig = new int[1];
        egl.eglChooseConfig(dpy, configAttr, configs, 1, numConfig);
        if (numConfig[0] == 0) {// TROUBLE! No config found.
        }
        EGLConfig config = configs[0];
        int[] surfAttr = {
                EGL10.EGL_WIDTH, 64,
                EGL10.EGL_HEIGHT, 64,
                EGL10.EGL_NONE
        };
        EGLSurface surf = egl.eglCreatePbufferSurface(dpy, config, surfAttr);
        final int eglContextClientVersion = 0x3098;  // missing in EGL10
        int[] ctxAttrib = {
                eglContextClientVersion, 1,
                EGL10.EGL_NONE
        };
        EGLContext ctx = egl.eglCreateContext(dpy, config, EGL10.EGL_NO_CONTEXT, ctxAttrib);
        egl.eglMakeCurrent(dpy, surf, surf, ctx);
        int[] maxSize = new int[1];
        GLES10.glGetIntegerv(GLES10.GL_MAX_TEXTURE_SIZE, maxSize, 0);
        egl.eglMakeCurrent(dpy, EGL10.EGL_NO_SURFACE, EGL10.EGL_NO_SURFACE,
                EGL10.EGL_NO_CONTEXT);
        egl.eglDestroySurface(dpy, surf);
        egl.eglDestroyContext(dpy, ctx);
        egl.eglTerminate(dpy);
        return maxSize[0];
    }

    public static int getOpenGlImageWidth() {
        int max;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            max = getGLESTextureLimitEqualAboveLollipop();
        } else {
            max = getGLESTextureLimitBelowLollipop();
        }
        if (max == 0) {
            max = new Canvas().getMaximumBitmapHeight() / 8;
        }
        if (max == 0) {
            max = Integer.MAX_VALUE;
        }
        return max;
    }
}
```

#### 自定义BitmapTransformation

```java
public class BigBitmapTransformation extends BitmapTransformation {
    private static final String ID = "BigBitmapTransformation";
    private static final byte[] ID_BYTES = ID.getBytes(CHARSET);
    private static final Paint DEFAULT_PAINT = new Paint(TransformationUtils.PAINT_FLAGS);

    private int maxWidth;

    public BigBitmapTransformation(int maxWidth) {
        this.maxWidth = maxWidth;
    }

    @Override
    protected Bitmap transform(@NonNull BitmapPool pool, @NonNull Bitmap toTransform, int outWidth, int outHeight) {
        if (toTransform.getHeight() <= maxWidth) {
            return toTransform;
        }
        return TransformationUtils.centerCrop(pool, toTransform,
                toTransform.getWidth() * maxWidth / toTransform.getHeight(), maxWidth);
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof BigBitmapTransformation;
    }

    @Override
    public int hashCode() {
        return ID.hashCode();
    }

    @Override
    public void updateDiskCacheKey(MessageDigest messageDigest) {
        messageDigest.update(ID_BYTES);
    }
}
```

#### 使用BitmapTransformation

```java
request.load(url).asBitmap()
        .diskCacheStrategy(DiskCacheStrategy.ALL)
        .transform(new BigBitmapTransformation(context,maxOpenGlWidth))
```

