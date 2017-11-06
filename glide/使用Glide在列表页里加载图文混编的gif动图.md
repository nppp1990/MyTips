

[TOC]

### 前言

项目中需要再列表里加上gif动图，并且没一个list的item是包含数目不定的gif表情图和文字的混编

下面的具体实现基本参考https://github.com/bumptech/glide/issues/1062 

PS：我是一个勤劳的代码搬运工。。。。啦啦啦

### 准备

#### Glide加载gif的通用方法

```java
Glide.with(context).load("").asGif()
  .diskCacheStrategy(DiskCacheStrategy.SOURCE)//这里diskCacheStrategy需要选择source，
  不然加载会很慢，网上说的，我没有验证
  .into(imageView);
```

但是这种方法只能是ImageView，由于需求里需要多个表情图且数目不一定，位置也不一定，所以不推荐使用ImageView的方法

#### Glide加载gif的方法的SimpleTarget<GifDrawable>

glide加载gif可以不绑定ImageView而是只获取gifDrawable

```
private final class GifTarget extends SimpleTarget<GifDrawable> {

    @Override
    public void onResourceReady(GifDrawable resource, GlideAnimation<? super GifDrawable> glideAnimation) {
    }

    @Override
    public void onLoadCleared(Drawable placeholder) {
        super.onLoadCleared(placeholder);
    }
}
```

#### Drawable.Callback

```java
/**
 * Implement this interface if you want to create an animated drawable that
 * extends {@link android.graphics.drawable.Drawable Drawable}.
 * Upon retrieving a drawable, use
 * {@link Drawable#setCallback(android.graphics.drawable.Drawable.Callback)}
 * to supply your implementation of the interface to the drawable; it uses
 * this interface to schedule and execute animation changes.
 */
public interface Callback {
    /**
     * Called when the drawable needs to be redrawn.  A view at this point
     * should invalidate itself (or at least the part of itself where the
     * drawable appears).
     *
     * @param who The drawable that is requesting the update.
     */
    void invalidateDrawable(@NonNull Drawable who);

    /**
     * A Drawable can call this to schedule the next frame of its
     * animation.  An implementation can generally simply call
     * {@link android.os.Handler#postAtTime(Runnable, Object, long)} with
     * the parameters <var>(what, who, when)</var> to perform the
     * scheduling.
     *
     * @param who The drawable being scheduled.
     * @param what The action to execute.
     * @param when The time (in milliseconds) to run.  The timebase is
     *             {@link android.os.SystemClock#uptimeMillis}
     */
    void scheduleDrawable(@NonNull Drawable who, @NonNull Runnable what, long when);

    /**
     * A Drawable can call this to unschedule an action previously
     * scheduled with {@link #scheduleDrawable}.  An implementation can
     * generally simply call
     * {@link android.os.Handler#removeCallbacks(Runnable, Object)} with
     * the parameters <var>(what, who)</var> to unschedule the drawable.
     *
     * @param who The drawable being unscheduled.
     * @param what The action being unscheduled.
     */
    void unscheduleDrawable(@NonNull Drawable who, @NonNull Runnable what);
}
```

```java
drawable.setCallback(textView);
```

Drawable.Callback可以用来监听animation drawable的变化，我们可以用setCallback来绑定textview，在invalidateDrawable方法里调用textview的invalidate，这样就可以实时的更新textView里的image来实现gif动画效果

#### Html.ImageGetter

```java
public static interface ImageGetter {
    /**
     * This method is called when the HTML parser encounters an
     * &lt;img&gt; tag.  The <code>source</code> argument is the
     * string from the "src" attribute; the return value should be
     * a Drawable representation of the image or <code>null</code>
     * for a generic replacement image.  Make sure you call
     * setBounds() on your Drawable if it doesn't already have
     * its bounds set.
     */
    public Drawable getDrawable(String source);
}
```

getDrawable函数是在解析html的img标签时会被调用，这时候用Glide来下载展示gif表情图

#### Html.fromHtml

```java
/**
 * Returns displayable styled text from the provided HTML string. Any &lt;img&gt; tags in the
 * HTML will use the specified ImageGetter to request a representation of the image (use null
 * if you don't want this) and the specified TagHandler to handle unknown tags (specify null if
 * you don't want this).
 *
 * <p>This uses TagSoup to handle real HTML, including all of the brokenness found in the wild.
 */
public static Spanned fromHtml(String source, int flags, ImageGetter imageGetter,
        TagHandler tagHandler) {
```

我们可以用这个函数构造TextView的span

####解析gif文本

每个item的接口给的内容可能如下形式

```java
点赞啊啦啦啦[#8][#7][#2]。。。。。
```

类似于微信表情，这样每个gif图用一个约定好的文本表示实现图文混编的文本

然后利用正则匹配把[#7]替换成<img src="xxxx"/>,下面是我的实现

```java
    private static final String REGEX = "\\[#\\d+\\]";
    private static final Pattern PATTERN = Pattern.compile(REGEX);
    public static String replaceHtml(String origin) {
        Matcher matcher = PATTERN.matcher(origin);
        while (matcher.find()) {
            String key = matcher.group();
            int start = matcher.start();
            int end = start + key.length();
            String realKey = key.replace("[#", "").replace("]", "");
            String url = getFaceUrl(realKey);
            if (url != null) {
                origin = origin.replace(key, "<img src=\"" + url + "\">");
            }
        }
        return origin;
    }
```

### 实现列表页中gif图文混编整体流程

1. 获取每个item的内容content，利用正则匹配gif表情将content替换成有<img/>标签的文本
2. 放content的容器TextView实现invalidateDrawable方法调用更新invalidate
3. 编写自定义的Html.ImageGetter，在getDrawable里用glide加载gif
4. 在onResourceReady里获取GifDrawable对象resource，绑定textview，开启动画


下面是核心代码

```java
public class FaceImageGetter implements Html.ImageGetter {
    private Set<SimpleTarget> targets;
    private Set<GifDrawable> gifDrawables;
    private final TextView textView;
    private RequestManager glide;

    public static FaceImageGetter get(View view) {
        if (null == view) {
            return null;
        }
        Object tag = view.getTag(R.id.mkz_face_image_flag);
        if (null != tag) {
            return (FaceImageGetter) tag;
        } else {
            return null;
        }
    }

    private static void stopGif(Set<GifDrawable> gifList) {
        if (gifList == null) {
            return;
        }
        for (GifDrawable glideDrawable : gifList) {
            if (null != glideDrawable) {
                glideDrawable.stop();
                glideDrawable.setCallback(null);
            }
        }
        gifList.clear();
    }

    public static void pauseGif(TextView textView) {
        FaceImageGetter currentGlideImage = get(textView);
        if (currentGlideImage != null) {
            if (currentGlideImage.gifDrawables != null) {
                for (GifDrawable gifDrawable : currentGlideImage.gifDrawables) {
                    if (gifDrawable != null) {
                        gifDrawable.stop();
                        gifDrawable.setCallback(null);
                    }
                }
            }
        }
    }

    public static void resumeGif(TextView textView) {
        FaceImageGetter currentGlideImage = get(textView);
        if (currentGlideImage != null) {
            if (currentGlideImage.gifDrawables != null) {
                for (GifDrawable gifDrawable : currentGlideImage.gifDrawables) {
                    if (gifDrawable != null) {
                        gifDrawable.setCallback(textView);
                        gifDrawable.start();
                    }
                }
            }
        }
    }

    public static void clearGifInsideView(TextView textView) {
        FaceImageGetter currentGlideImage = get(textView);
        textView.setText(null);
        if (currentGlideImage == null) {
            return;
        }
        stopGif(currentGlideImage.gifDrawables);
        clearTarget(currentGlideImage.targets);
        textView.setTag(null);
    }

    private static void clearTarget(Set<SimpleTarget> mTargets) {
        if (null == mTargets) {
            return;
        }
        for (SimpleTarget target : mTargets) {
            Glide.clear(target);
        }
    }


    public FaceImageGetter(RequestManager glide, TextView textView) {
        this.glide = glide;
        this.textView = textView;
        targets = new HashSet<>();
        gifDrawables = new HashSet<>();
        textView.setTag(R.id.mkz_face_image_flag, this);
    }

    @Override
    public Drawable getDrawable(String source) {
        final UrlDrawable urlDrawable = new UrlDrawable();
        GenericRequestBuilder load;
        final SimpleTarget target;
        if (isGif(source)) {
            load = glide.load(ImageQualityUtil.getQualityUrl(source, ImageQualityUtil.GIF_SIZE)).diskCacheStrategy(DiskCacheStrategy.SOURCE);
            target = new GifTarget(urlDrawable);
        } else {
            load = glide.load(ImageQualityUtil.getQualityUrl(source, ImageQualityUtil.AVATAR_SIZE)).asBitmap();
            target = new BitmapTarget(urlDrawable);
        }
        targets.add(target);
        load.into(target);
        return urlDrawable;
    }

    private static boolean isGif(String path) {
        // TODO
        return true;
//        int index = path.lastIndexOf('.');
//        return index > 0 && "gif".toUpperCase().equals(path.substring(index + 1).toUpperCase());
    }


    private final class GifTarget extends SimpleTarget<GifDrawable> {
        private final UrlDrawable urlDrawable;


        private GifTarget(UrlDrawable urlDrawable) {
            this.urlDrawable = urlDrawable;

        }

        @Override
        public void onResourceReady(GifDrawable resource, GlideAnimation<? super GifDrawable> glideAnimation) {
            int width = DisplayUtils.dip2px(textView.getContext(), 25);
            Rect rect = new Rect(0, 0, width, width);
            resource.setBounds(rect);
            urlDrawable.setBounds(rect);
            urlDrawable.setDrawable(resource);
            gifDrawables.add(resource);
            resource.setCallback(textView);
            resource.start();
            resource.setLoopCount(GlideDrawable.LOOP_FOREVER);
            textView.setText(textView.getText());
            textView.invalidate();
        }

        @Override
        public void onLoadCleared(Drawable placeholder) {
            if (placeholder instanceof GifDrawable) {
                GifDrawable drawable = (GifDrawable) placeholder;
                drawable.stop();
            }
            super.onLoadCleared(placeholder);
        }
    }

    private class BitmapTarget extends SimpleTarget<Bitmap> {
        private final UrlDrawable urlDrawable;

        public BitmapTarget(UrlDrawable urlDrawable) {
            this.urlDrawable = urlDrawable;
        }

        @Override
        public void onResourceReady(Bitmap resource, GlideAnimation<? super Bitmap> glideAnimation) {
            Ln.e("----bitmap");
            // TODO
        }
    }
}
```

```java
public class FaceTextView extends TextView implements View.OnAttachStateChangeListener {

    public FaceTextView(Context context) {
        super(context);
    }

    public FaceTextView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public FaceTextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    public void onViewAttachedToWindow(View v) {

    }

    @Override
    public void onViewDetachedFromWindow(View v) {
        FaceImageGetter.clearGifInsideView(this);
    }

    @Override
    public void invalidateDrawable(@NonNull Drawable drawable) {
//        super.invalidateDrawable(drawable);
        invalidate();
    }
```

### 性能问题

按照上面的思路，功能是完成了，但是用久了内存直线上涨，显然有内存泄露问题；于是乎加上了性能方面的优化，内存基本没有问题了，主要以下几点

1. 页面退出时：停止动画，清空所有的glide的target对象
2. view滑出列表时：停止对应的动画，清空对应的target对象
3. 列表滑动时：停止所有动画；列表停止滑动时：恢复动画

退出页面时

```java
@Override
public void onDestroy() {
    super.onDestroy();
    operateListGifView(TYPE_CLEAR_GIF);
}
```

view滑出时

```java
getListView().setRecyclerListener(new AbsListView.RecyclerListener() {
    @Override
    public void onMovedToScrapHeap(View view) {
        operateGifView(view, TYPE_CLEAR_GIF);
    }
});
```

列表滑动监听

```java
addOnScrollListener(new AbsListView.OnScrollListener() {
    boolean isFastScroll;

    @Override
    public void onScrollStateChanged(AbsListView view, int scrollState) {
        isFastScroll = scrollState == SCROLL_STATE_FLING;
        if (scrollState == SCROLL_STATE_IDLE) {
            operateListGifView(TYPE_RESUME_GIF);
        }
    }

    @Override
    public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount, int totalItemCount) {
        if (isFastScroll) {
            operateListGifView(TYPE_PAUSE_GIF);
        } else {
            operateListGifView(TYPE_RESUME_GIF);
        }
    }
});
```

#### todo

gif表情图比较多的页面在滑动时，还是比较卡顿