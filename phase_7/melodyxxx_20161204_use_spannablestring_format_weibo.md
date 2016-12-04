title: 使用 SpannableString 格式化微博内容
date: 2016-12-04 16:15:29
tags: [Android]
toc: true
---

SpannableString 配合 TextView 可以轻松实现对特定的文本做特定处理，例如可以修改文字颜色、背景色、将文字替换为图片实现，点击效果等。

<!--more-->

>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Melodyxxx](http://melodyxxx.com/)
>- 审阅者：[暂无]()

首先，上一张本文要最终实现的效果图:

<div align = center>
<img src="http://ww3.sinaimg.cn/large/df189f43gw1faeua1csqpj20u011e0xq.jpg" width = "60%" />
</div>

第一个卡片内的微博是原始文本信息，第二个卡片内的微博是第一个格式化后的文本内容，将微博内的"话题"、"表情"、"网页链接"、以及"@用户"都进行了处理，并可以点击，使其和官方微博展示的样式保持一致。

**要实现的效果**：
* 将话题进行变色并且可以点击提示对应的话题文本内容
* 将图片表情替换掉对应的表情关键字显示
* 将链接地址替换成一个链接的图片和"网页链接"四个字显示
* 将@的用户进行变色并且可以点击提示对应的话题文本内容

**需要**：
* 使用正则表达式提取文本内对应的"话题"、"表情"、"网页链接"、以及"@用户"内容
* 使用 SpannableString 格式化提取到的文本
* 给格式化的部分添加点击事件

## 定义正则表达式

首先定义"话题"、"表情"、"网页链接"、以及"@用户"对应的正则表达式和对应的 Pattern。SCHEME 下文会提到具体的用处的。
``` java
public class WeiboPattern {

    // #话题#
    public static final String REGEX_TOPIC = "#[\\p{Print}\\p{InCJKUnifiedIdeographs}&&[^#]]+#";
    // [表情]
    public static final String REGEX_EMOTION = "\\[(\\S+?)\\]";
    // url
    public static final String REGEX_URL = "http://[a-zA-Z0-9+&@#/%?=~_\\\\-|!:,\\\\.;]*[a-zA-Z0-9+&@#/%=~_|]";
    // @人
    public static final String REGEX_AT = "@[\\w\\p{InCJKUnifiedIdeographs}-]{1,26}";

    
    public static final Pattern PATTERN_TOPIC = Pattern.compile(REGEX_TOPIC);
    public static final Pattern PATTERN_EMOTION = Pattern.compile(REGEX_EMOTION);
    public static final Pattern PATTERN_URL = Pattern.compile(REGEX_URL);
    public static final Pattern PATTERN_AT = Pattern.compile(REGEX_AT);

    public static final String SCHEME_TOPIC = "topic:";
    public static final String SCHEME_URL = "url:";
    public static final String SCHEME_AT = "at:";

}
```

## 提取匹配部分并使用 SpannableString 格式化

我将此过程写到一个方法内了，下面直接上代码，代码中有详细的注释解释：

``` java
/**
 * 格式化微博文本
 *
 * @param context  上下文
 * @param source   源文本
 * @param textView 目标 TextView
 * @return SpannableStringBuilder
 */
public static SpannableStringBuilder formatWeiBoContent(Context context, String source, TextView textView) {

    // 获取到 TextView 的文字大小，后面的 ImageSpan 需要用到该值
    int textSize = (int) textView.getTextSize();

    // 若要部分 SpannableString 可点击，需要如下设置
    textView.setMovementMethod(LinkMovementMethod.getInstance());

    // 将要格式化的 String 构建成一个 SpannableStringBuilder
    SpannableStringBuilder value = new SpannableStringBuilder(source);

    // 使用正则匹配话题
    Linkify.addLinks(value, WeiboPattern.PATTERN_TOPIC, WeiboPattern.SCHEME_TOPIC);
    // 使用正则匹配链接
    Linkify.addLinks(value, WeiboPattern.PATTERN_URL, WeiboPattern.SCHEME_URL);
    // 使用正则匹配@用户
    Linkify.addLinks(value, WeiboPattern.PATTERN_AT, WeiboPattern.SCHEME_AT);

    // 自定义的匹配部分的点击效果
    MyClickableSpan clickSpan;

    // 获取上面到所有 addLinks 后的匹配部分(这里一个匹配项被封装成了一个 URLSpan 对象)
    URLSpan[] urlSpans = value.getSpans(0, value.length(), URLSpan.class);

    // 遍历所有的 URLSpan
    for (final URLSpan urlSpan : urlSpans) {
		// 点击匹配部分效果
        clickSpan = new MyClickableSpan() {
            @Override
            public void onClick(View view) {
                ToastUtils.makeShort(urlSpan.getURL());
            }
        };
        // 话题
        if (urlSpan.getURL().startsWith(WeiboPattern.SCHEME_TOPIC)) {
            int start = value.getSpanStart(urlSpan);
            int end = value.getSpanEnd(urlSpan);
            value.removeSpan(urlSpan);
            // 格式化话题部分文本
            value.setSpan(clickSpan, start, end, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
        }
        // @用户
        if (urlSpan.getURL().startsWith(WeiboPattern.SCHEME_AT)) {
            int start = value.getSpanStart(urlSpan);
            int end = value.getSpanEnd(urlSpan);
            value.removeSpan(urlSpan);
            // 格式化@用户部分文本
            value.setSpan(clickSpan, start, end, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
        }
        // 链接
        if (urlSpan.getURL().startsWith(WeiboPattern.SCHEME_URL)) {
            int start = value.getSpanStart(urlSpan);
            int end = value.getSpanEnd(urlSpan);
            value.removeSpan(urlSpan);
            SpannableStringBuilder urlSpannableString = getUrlTextSpannableString(context, urlSpan.getURL(), textSize);
            value.replace(start, end, urlSpannableString);
            // 格式化链接部分文本
            value.setSpan(clickSpan, start, start + urlSpannableString.length(), Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
        }
    }

    // 表情需要单独格式化
    Matcher emotionMatcher = WeiboPattern.PATTERN_EMOTION.matcher(value);
    while (emotionMatcher.find()) {
        String emotion = emotionMatcher.group();
        int start = emotionMatcher.start();
        int end = emotionMatcher.end();
        int resId = EmotionUtils.getImageByName(emotion);
        if (resId != -1) {  // 表情匹配
            L.e("find emotion: " + emotion);
            Drawable drawable = context.getResources().getDrawable(resId);
            drawable.setBounds(0, 0, (int) (textSize * 1.3), (int) (textSize * 1.3));
            // 自定义的 VerticalImageSpan ，可解决默认的 ImageSpan 不垂直居中的问题
            VerticalImageSpan imageSpan = new VerticalImageSpan(drawable);
            value.setSpan(imageSpan, start, end, Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
        }
    }

    return value;
}
```

``` java
private static SpannableStringBuilder getUrlTextSpannableString(Context context, String source, int size) {
    SpannableStringBuilder builder = new SpannableStringBuilder(source);
    String prefix = " ";
    builder.replace(0, prefix.length(), prefix);
    Drawable drawable = context.getResources().getDrawable(R.drawable.ic_status_link);
    drawable.setBounds(0, 0, size, size);
    builder.setSpan(new VerticalImageSpan(drawable), prefix.length(), source.length(), Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
    builder.append(" 网页链接");
    return builder;
}
```

`getUrlTextSpannableString()` ：方法是用来返回一个图标+"网页链接" SpannableString，用于替换链接文本

上面将"话题"、"表情"、"网页链接"都用了addLinks方法来标记的，然后统一处理。表情则是单独处理的。

表情则使用如下方法事先做好映射：
``` java
public class EmotionUtils {

    public static LinkedHashMap<String, Integer> sMap;

    static {
        sMap = new LinkedHashMap<>();
        sMap.put("[doge]", R.drawable.d_doge);
        sMap.put("[污]", R.drawable.d_wu);
    }

    public static int getImageByName(String name) {
        Integer integer = sMap.get(name);
        return integer == null ? -1 : integer;
    }

}

```

还有刚才说到的自定义 MyClickableSpan 修改默认的样式:
``` java
public class MyClickableSpan extends ClickableSpan {

    @Override
    public void onClick(View view) {

    }

    @Override
    public void updateDrawState(TextPaint ds) {
        super.updateDrawState(ds);
        ds.setColor(0xff03A9F4);
        ds.setUnderlineText(false);
    }

}
```

另外，由于默认的 ImageSpan 在 TextView 有使用`android:lineSpacingExtra`属性时，不会垂直居中，所以使用到了网上的一个继承自 ImageSpan 的 VerticalImageSpan 可以做到保持图片在 TextView 内保持垂直居中：

``` java
public class VerticalImageSpan extends ImageSpan {

    public VerticalImageSpan(Drawable drawable) {
        super(drawable);
    }

    /**
     * update the text line height
     */
    @Override
    public int getSize(Paint paint, CharSequence text, int start, int end,
                       Paint.FontMetricsInt fontMetricsInt) {
        Drawable drawable = getDrawable();
        Rect rect = drawable.getBounds();
        if (fontMetricsInt != null) {
            Paint.FontMetricsInt fmPaint = paint.getFontMetricsInt();
            int fontHeight = fmPaint.descent - fmPaint.ascent;
            int drHeight = rect.bottom - rect.top;
            int centerY = fmPaint.ascent + fontHeight / 2;

            fontMetricsInt.ascent = centerY - drHeight / 2;
            fontMetricsInt.top = fontMetricsInt.ascent;
            fontMetricsInt.bottom = centerY + drHeight / 2;
            fontMetricsInt.descent = fontMetricsInt.bottom;
        }
        return rect.right;
    }

    /**
     * see detail message in android.text.TextLine
     *
     * @param canvas the canvas, can be null if not rendering
     * @param text   the text to be draw
     * @param start  the text start position
     * @param end    the text end position
     * @param x      the edge of the replacement closest to the leading margin
     * @param top    the top of the line
     * @param y      the baseline
     * @param bottom the bottom of the line
     * @param paint  the work paint
     */
    @Override
    public void draw(Canvas canvas, CharSequence text, int start, int end,
                     float x, int top, int y, int bottom, Paint paint) {

        Drawable drawable = getDrawable();
        canvas.save();
        Paint.FontMetricsInt fmPaint = paint.getFontMetricsInt();
        int fontHeight = fmPaint.descent - fmPaint.ascent;
        int centerY = y + fmPaint.descent - fontHeight / 2;
        int transY = centerY - (drawable.getBounds().bottom - drawable.getBounds().top) / 2;
        canvas.translate(x, transY);
        drawable.draw(canvas);
        canvas.restore();
    }

}
```

然后直接调用该方法格式化：
``` java
mTextView.setText(formatWeiBoContent(this,mTextView.getText().toString(),mTextView))
```

最终的效果图和文章开头效果一样了，并且可以点击，这里展示了点击"网页链接"时弹出的 Toast 提示：
<div align = center>
<img src="http://ww1.sinaimg.cn/large/df189f43gw1faf0tt6elbj20u01hcjxk.jpg" width="60%"/>
</div>

## 总结
本文仅介绍了 SpannableString 常用的一些场景，例如修改特定文本的颜色，替换特定文本，特定文本的点击事件，但是 SpannableString 的强大远不止如此。SpannableString 的更多用法可阅读[官方文档](https://developer.android.com/reference/android/text/SpannableString.html)