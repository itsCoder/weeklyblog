## StaticLayout 源码分析
Android 控件中，看起来最简单、最基础的 TextView 实际上是很复杂的，很多常见的控件都是其子类，例如 Botton、EditText、CheckBox 等，由于作为一个基础控件类，TextView 需要考虑到子类的各种使用场景，满足子类的需求。源码中，TextView 单个类源码就多达 1万行，而且其工作时还依赖很多辅助类。其文本的排版、折行处理，以及最终的显示，均是交给辅助类 Layout 类来处理的。

#### 前言
由于 Canvas 本身提供的 drawText 绘制文本是不支持换行的，所以在文本需要换行显示时，就需要用到 Layout 类。我们可以看到官方对 Layout 类的描述：

> A base class that manages text layout in visual elements on the screen.

一个用于管理屏幕上文本布局的基类。

其直接子类有 StaticLayout、DynamicLayout、BoringLayout，在官方的文档中提到，如果文本内容会被编辑，应该使用 DynamicLayout，如果文本显示之后不会发生改变，应该使用 StaticLayout，而 BoringLayout 则使用场景极为有限：当你确保你的文本只有一行，且所有的字符均是从左到右显示的（某些语言的文字是从右到左显示的），你才可以使用 BoringLayout。

本文将会简单地深入 StaticLayout 的源码，分析下具体是如何工作的。

#### 概述
先看 StaticLayout 类的注释：StaticLayout 是一个为不可编辑的文本布局的类，这意味着一旦布局完成，文本内容就不可以改变，如果需要改变的话，应该使用 DynamicLayout 来布局。同时你不应该直接使用 StaticLayout 类，除非你需要实现一个自定义的控件或者自定义显示对象，否则，你应该直接调用  `Canvas.drawText()`。因此，在正常的开发工作中，你接触 StaticLayout 的机会应该不多。

在 TextView 初始化时，会通过 `makeNewLayout()` 方法，根据文本的特点，是否包含 Span，是否单行等，决定创建具体的 Layout 类型。在单纯地使用TextView来展示静态文本的时候，创建的就是 StaticLayout。StaticLayout 的初始化是通过内部类 `StaticLayout.Builder` 完成的，然后调用 `generate()` 方法完成段落、折行以及缩进之类的处理，在 `generate()` 方法中调用了 `out()` 方法，完成文本显示的行距、顶部底部留白、省略文本等的处理，这两个方法也是 StaticLayout 源码中两个主要的方法，完成了一系列的文本处理。在 TextView 的 `onDraw(Canvas canvas)` 方法中，调用父类 Layout 的 `draw()` 方法，改方法会依次调用 `drawBackground()` 和 `drawText()` 完成背景和文本的绘制。

#### 构造方法
StaticLayout 有多个构造方法，最完整的构造方法(其他构造方法最终也是调用的这个构造方法)如下所示：

```java
 public StaticLayout(CharSequence source, int bufstart, int bufend,
                        TextPaint paint, int outerwidth,
                        Alignment align, TextDirectionHeuristic textDir,
                        float spacingmult, float spacingadd,
                        boolean includepad,
                        TextUtils.TruncateAt ellipsize, int ellipsizedWidth, int maxLines) {
        super((ellipsize == null)
                ? source
                : (source instanceof Spanned)
                    ? new SpannedEllipsizer(source)
                    : new Ellipsizer(source),
              paint, outerwidth, align, textDir, spacingmult, spacingadd);

        Builder b = Builder.obtain(source, bufstart, bufend, paint, outerwidth)
            .setAlignment(align)
            .setTextDirection(textDir)
            .setLineSpacing(spacingadd, spacingmult)
            .setIncludePad(includepad)
            .setEllipsizedWidth(ellipsizedWidth)
            .setEllipsize(ellipsize)
            .setMaxLines(maxLines);
        if (ellipsize != null) {
            Ellipsizer e = (Ellipsizer) getText();

            e.mLayout = this;
            e.mWidth = ellipsizedWidth;
            e.mMethod = ellipsize;
            mEllipsizedWidth = ellipsizedWidth;
            mColumns = COLUMNS_ELLIPSIZE;
        } else {
            mColumns = COLUMNS_NORMAL;
            mEllipsizedWidth = outerwidth;
        }

        mLineDirections = ArrayUtils.newUnpaddedArray(Directions.class, 2 * mColumns);
        mLines = new int[mLineDirections.length];
        mMaximumVisibleLineCount = maxLines;

        generate(b, b.mIncludePad, b.mIncludePad);

        Builder.recycle(b);
    }
```

参数说明：

- `CharSequence source` 文本内容
- ` int bufstart, int bufend,` 开始位置和结束位置
- ` TextPaint paint` 文本画笔对象
-  `int outerwidth` 布局宽度，超出宽度换行显示
- `Alignment align` 对齐方式，默认是`Alignment.ALIGN_LEFT`
- `TextDirectionHeuristic textDir` 文本显示方向
- `float spacingmult` 行间距倍数，默认是1
- `float spacingadd` 行距增加值，默认是0
- `boolean includepad` 文本顶部和底部是否留白
- `TextUtils.TruncateAt ellipsize` 文本省略方式，有 START、MIDDLE、 END、MARQUEE 四种省略方式（其实还有一个 END_SMALL，但是 Google 并未开放出来）。
- `int ellipsizedWidth` 省略宽度
- `int maxLines` 最大行数

细节分析：

- 构造方法的开始，在调用父类 Layout 构造方法的时候，判断了文本是否需要省略，如果需要省略，则创建一个 Ellipsizer 对象，Ellipsizer 是 Layout 的嵌套内部类，实现了 CharSequence 和 GetChars 接口。该类就是用来对文本进行省略处理的，具体的处理方法是由其 `getChars()` 方法完成的。
- 在创建 Ellipsizer 对象之前，还判断了一下需要显示的文本是否是 Spanned ,如果是的话则创建 SpannedEllipsizer 对象，SpannedEllipsizer 类继承 Ellipsizer ，同时实现了 Spanned 接口。
- StaticLayout.Builder 对象的创建是通过 `Builder.obtain()` 方法创建的，在该方法内部可以看到 Builder 对象通过 SynchronizedPool 对象池来管理的，起到缓存的作用，避免 Builder 对象的重复创建，在 StaticLayout 的构造方法的最后也可以看到 `Builder.recycle(b)` 的调用，回收 Builder 对象。
    Builder 的构造方法如下所示：

    ``` java
   private Builder() {
        mNativePtr = nNewBuilder();
    }
    ```
    其调用了 JNI 层的 `nNewBuilder()` 方法，新建了一个 LineBreak 对象，并将其指针指向 java 层，赋值给 Builder 对象的 mNativePtr 字段 ，后面调用 native 方法时，均需要将 mNativePtr 作为参数传递过去。
   
 ```cpp
 static jlong nNewBuilder(JNIEnv*, jclass) {
        return reinterpret_cast<jlong>(new LineBreaker);
 }
 ```
- mLineDirections 需要结合到后面每行文本处理来理解，这里可以大致说一下，StaticLayout 源码中声明了以下的常量：

 ``` java
 int COLUMNS_NORMAL = 4;
 int COLUMNS_ELLIPSIZE = 6;
 int START = 0;
 int DIR = START;
 int TAB = START;
 int TOP = 1;
 int DESCENT = 2;
 int HYPHEN = 3;
 int ELLIPSIS_START = 4;
 int ELLIPSIS_COUNT = 5;
 ```
其中 COLUMNS_NORMAL 和 COLUMNS_ELLIPSIZE 会赋值给全局变量 mColumns，正如你在构造方法中看到的那样，这个在没一行处理时会用到，每一行文本处理时需要记录四个值，start，top，desent，hyphen 值，当文本需要省略时，还需要记录 ellipsis_start 和 ellipsis_count 值，因此正常的 mColumn 值为4，省略时则是6，因此 mLineDirections 数组大小始终是 mColumn 的倍数，mLine 数组的大小和其保持一致(从后面的分析来看，mLineDirections 数组的大小没必要这么大)。

#### generate 方法分析
StaticLayout 中的 `generate()` 方法近 300 行，其完成了文本的段落、折行的处理，建议自行对照源码来阅读下面的分析，本文不贴太多代码。

接受的参数：

- `StaticLayout.Builder b`  StaticLayout.Builder 对象
- `boolean includepad`是否上下保留空白
- `boolean trackpad`

细节分析：

1. 在方法的开始，创建了很多的局部变量，并将 Builder 对象对应的值赋值给这些变量。
       - 其中有个 `Paint.FontMetricsInt fm` 变量，FontMetricsInt 是 Paint 的内部类，主要用来完成字体测量，其和 `FontMetrics` 非常类似，只是在文字测量时，对应的数值均是 int 类型，FontMetrics 是 float 类型。FontMetricsInt 类主要包含保存了字体测量相关的数据，源码如下：
     
     ```java
       public static class FontMetricsInt {
             public int   top;
             public int   ascent;
             public int   descent;
             public int   bottom;
             public int   leading;
    }
    ```
    每个值的含义如下图所示，在 baseline 之上为负值，baseline 之下为正值，leading 表示两行文本 baseline 之间的距离，这个值可以由行间距倍数和行间距增加值来调整：
    ![](http://ac-qygvx1cc.clouddn.com/00a715d3dc637c92.png)
在接下来的字体测量中，会使用 fmCache 数组来缓存字体测量的信息，缓存 top, bottom, ascent, 和 descen 四个值，因此 fmCache 数组的大小始终是4的倍数。

2. 接下来就是按照一个个段落来处理文本：
    
    ``` java
    for (int paraStart = bufStart; paraStart <= bufEnd; paraStart = paraEnd) {
        paraEnd = TextUtils.indexOf(source, CHAR_NEW_LINE, paraStart, bufEnd);
        if (paraEnd < 0)
            paraEnd = bufEnd;
        else
            paraEnd++;
        ...
    }
    ```
    
	通过查找换行符，确定每个段落的起止位置，接下来的处理，均是对该段落文本的处理。

3. span 文本的处理

4. 处理段落文本 :
    
    ```java
    measured.setPara(source, paraStart, paraEnd, textDir, b);
    char[] chs = measured.mChars;
    float[] widths = measured.mWidths;
    byte[] chdirs = measured.mLevels;
    int dir = measured.mDir;
    boolean easy = measured.mEasy;
    ```
    
5. 处理制表位，这里的制表位是使用 `TabStopSpan` 方式插入到文本中的，通过 Spanned 接口提供的 `getSpans(int start, int end, Class<T> type)` 方法来获取到 TabStopSpan，排序后将所有的制表位的位置存在 variableTabStops 数组中。
    
    ```java
    int[] variableTabStops = null;
    if (spanned != null) {
        TabStopSpan[] spans = getParagraphSpans(spanned, paraStart,
                paraEnd, TabStopSpan.class);
        if (spans.length > 0) {
            int[] stops = new int[spans.length];
            for (int i = 0; i < spans.length; i++) {
                stops[i] = spans[i].getTabStop();
            }
            Arrays.sort(stops, 0, stops.length);
            variableTabStops = stops;
        }
    }
    ```
    
6. 完成以上处理后，就是交给 JNI 层来处理段落文本，主要处理了段落的制表行缩进、折行等；需要再分析。
   
    ``` java
     nSetupParagraph(b.mNativePtr, chs, paraEnd - paraStart,
                    firstWidth, firstWidthLineCount, restWidth,
                    variableTabStops, TAB_INCREMENT, b.mBreakStrategy, b.mHyphenationFrequency);
    ```
    
7. 处理缩进的源码如下：
    
    ``` java
    if (mLeftIndents != null || mRightIndents != null) {
        int leftLen = mLeftIndents == null ? 0 : mLeftIndents.length;
        int rightLen = mRightIndents == null ? 0 : mRightIndents.length;
        int indentsLen = Math.max(1, Math.min(leftLen, rightLen) - mLineCount);
        int[] indents = new int[indentsLen];
        for (int i = 0; i < indentsLen; i++) {
            int leftMargin = mLeftIndents == null ? 0 :
                    mLeftIndents[Math.min(i + mLineCount, leftLen - 1)];
            int rightMargin = mRightIndents == null ? 0 :
                    mRightIndents[Math.min(i + mLineCount, rightLen - 1)];
            indents[i] = leftMargin + rightMargin;
        }
        nSetIndents(b.mNativePtr, indents);
    }        
    ```
    
	开始的条件判断使用的 mLeftIndents 和 mRightIndents 变量是通过 Builder 对象来赋值的：
    
   ```java
   mLeftIndents = b.mLeftIndents;
   mRightIndents = b.mRightIndents;
   ```
    
	但是比较困惑的是，源码中并没有对 Builder 对象这两个字段赋值的地方，因此这里的条件判断结果都是 false，实际 debug 测试了下，这个地方的判断确实始终是 false，所以具体的逻辑还需要再分析下。可以看见的是，在方法的最后，同样是调用 JNI 层的方法设置缩进。

8. 缓存字体测量信息，源码如下：

    ```java
    for (int spanStart = paraStart, spanEnd; spanStart < paraEnd; spanStart = spanEnd) {
        if (fmCacheCount * 4 >= fmCache.length) {
            int[] grow = new int[fmCacheCount * 4 * 2];
            System.arraycopy(fmCache, 0, grow, 0, fmCacheCount * 4);
            fmCache = grow;
        }
        if (spanEndCacheCount >= spanEndCache.length) {
            int[] grow = new int[spanEndCacheCount * 2];
            System.arraycopy(spanEndCache, 0, grow, 0, spanEndCacheCount);
            spanEndCache = grow;
        }
        if (spanned == null) {
            spanEnd = paraEnd;
            int spanLen = spanEnd - spanStart;
            measured.addStyleRun(paint, spanLen, fm);
        } else {
            spanEnd = spanned.nextSpanTransition(spanStart, paraEnd,
                    MetricAffectingSpan.class);
            int spanLen = spanEnd - spanStart;
            MetricAffectingSpan[] spans =
                    spanned.getSpans(spanStart, spanEnd, MetricAffectingSpan.class);
            spans = TextUtils.removeEmptySpans(spans, spanned, MetricAffectingSpan.class);
            measured.addStyleRun(paint, spans, spanLen, fm);
        }
        // the order of storage here (top, bottom, ascent, descent) has to match the code belo
        // where these values are retrieved
        fmCache[fmCacheCount * 4 + 0] = fm.top;
        fmCache[fmCacheCount * 4 + 1] = fm.bottom;
        fmCache[fmCacheCount * 4 + 2] = fm.ascent;
        fmCache[fmCacheCount * 4 + 3] = fm.descent;
        fmCacheCount++;
        spanEndCache[spanEndCacheCount] = spanEnd;
        spanEndCacheCount++;
    }
    ```
	
	fmCache 的初始化时的大小是 16，因此在每次循环开始时，需要判断下是否需要对 fmCache 扩容，这里的扩容同样保证了 fmCache 的大小是4的倍数，同时每次扩容时都是双倍扩容。
这里也会对文本中的 Span 的结束位置使用 spanEndCache 缓存记录下来，这里处理的 span 具体类型是 MetricAffectingSpan，顾名思义就是对字体会有影响的 Span，需要单独拿出来处理，缓存字体测量信息。

	具体的测量则是交给 MeasuredText 类的 `addStyleRun(TextPaint paint, int len, Paint.FontMetricsInt fm)` 和 `addStyleRun(TextPaint paint, MetricAffectingSpan[] spans, int len, Paint.FontMetricsInt fm)` 方法来处理，具体的处理涉及到文字的排版，感兴趣的可以自己查看源码，这里不再详细分析了。

	测量完成后，字体测量信息的值4个一组地存储在 fmCache 数组中，spanEnd 值存储在 spanEndCache 数组中。
	
9. 计算每行宽度和折行处理，宽度的计算和折行的处理分别借助 JNI 层的 `nGetWidths()` 和 `nComputeLineBreaks()` 方法来处理。

    ``` java
    nGetWidths(b.mNativePtr, widths);
    // 得到当前行内包含的折行数目
    int breakCount = nComputeLineBreaks(b.mNativePtr, lineBreaks, lineBreaks.breaks,
            lineBreaks.widths, lineBreaks.flags, lineBreaks.breaks.length);
            
    int[] breaks = lineBreaks.breaks;
    float[] lineWidths = lineBreaks.widths;
    int[] flags = lineBreaks.flags;
    
    // 得到剩下的行数 = 最大允许行数 - 当前行数
    final int remainingLineCount = mMaximumVisibleLineCount - mLineCount;
    final boolean ellipsisMayBeApplied = ellipsize != null
            && (ellipsize == TextUtils.TruncateAt.END
                || (mMaximumVisibleLineCount == 1
                        && ellipsize != TextUtils.TruncateAt.MARQUEE));
    // 如果剩下的行数小于当前行包含的折行数目，则需要将最后一行和超出的行处理成单行
    if (remainingLineCount > 0 && remainingLineCount < breakCount &&
            ellipsisMayBeApplied) {
        // Treat the last line and overflowed lines as a single line.
        breaks[remainingLineCount - 1] = breaks[breakCount - 1];
        // Calculate width and flag.
        float width = 0;
        int flag = 0;
       // 计算 width 和 flag 值
        for (int i = remainingLineCount - 1; i < breakCount; i++) {
            width += lineWidths[i];
            flag |= flags[i] & TAB_MASK;
        }
        lineWidths[remainingLineCount - 1] = width;
        flags[remainingLineCount - 1] = flag;
        // 设置当前行中的折行数为可用的行数
        breakCount = remainingLineCount;
    }
    ```
处理完折行后，会判断下是否需要省略处理，如果需要，则根据允许的最大行数和当前行包含的折行数目来确定需要处理成省略的那一行，并设置相关的 width 和 flag 信息。

10. 处理文本中 Span 和折行：
    
    ``` java
    int here = paraStart;
    int fmTop = 0, fmBottom = 0, fmAscent = 0, fmDescent = 0;
    int fmCacheIndex = 0;
    int spanEndCacheIndex = 0;
    int breakIndex = 0;
    for (int spanStart = paraStart, spanEnd; spanStart < paraEnd; spanStart = spanEnd) {
        // 从之前存储的数据中获取 span 结束位置
        spanEnd = spanEndCache[spanEndCacheIndex++];
        // 恢复之前存储的字体测量信息
        // retrieve cached metrics, order matches above
        fm.top = fmCache[fmCacheIndex * 4 + 0];
        fm.bottom = fmCache[fmCacheIndex * 4 + 1];
        fm.ascent = fmCache[fmCacheIndex * 4 + 2];
        fm.descent = fmCache[fmCacheIndex * 4 + 3];
        fmCacheIndex++;
        // 参照前面提到的字体测量的几个值的说明，这里的 top 和 ascent 取值小的，bottom 和 descent 取值大的
        // 保证文本均可以正常显示
        if (fm.top < fmTop) {
            fmTop = fm.top;
        }
        if (fm.ascent < fmAscent) {
            fmAscent = fm.ascent;
        }
        if (fm.descent > fmDescent) {
            fmDescent = fm.descent;
        }
        if (fm.bottom > fmBottom) {
            fmBottom = fm.bottom;
        }
        // 跳过 span 之前的折行
        while (breakIndex < breakCount && paraStart + breaks[breakIndex] < spanStart) {
            breakIndex++;
        }
        // 处理 span 中的折行
        while (breakIndex < breakCount && paraStart + breaks[breakIndex] <= spanEnd) {
            int endPos = paraStart + breaks[breakIndex];
            boolean moreChars = (endPos < bufEnd);
            v = out(source, here, endPos,
                    fmAscent, fmDescent, fmTop, fmBottom,
                    v, spacingmult, spacingadd, chooseHt,chooseHtv, fm, flags[breakIndex],
                    needMultiply, chdirs, dir, easy, bufEnd, includepad, trackpad,
                    chs, widths, paraStart, ellipsize, ellipsizedWidth,
                    lineWidths[breakIndex], paint, moreChars);
            if (endPos < spanEnd) {
                // 如果 Span 文本还未处理完成，则恢复当前的 fontMetrics 信息
                // 否则归零处理，处理下一段 Span
                // preserve metrics for current span
                fmTop = fm.top;
                fmBottom = fm.bottom;
                fmAscent = fm.ascent;
                fmDescent = fm.descent;
            } else {
                fmTop = fmBottom = fmAscent = fmDescent = 0;
            }
            here = endPos;
            breakIndex++;
            // 如果处理该段落时行数已经超过最大可见行数，则直接终止后面的处理
            if (mLineCount >= mMaximumVisibleLineCount) {
                return;
            }
        }
    }
    // 如果段落结束就是整个文本的结束，则跳出处理段落的循环，否则处理下一段。
    if (paraEnd == bufEnd)
        break;
}
    
    ```
至此，以段落为单位的文本就处理完毕，包括文本的折行、Span 的处理都已完成。

11. 当需要处理的文本起止位置相同时（即需要处理的文本为空），且前面是换行符时，此时也需要将该空白处理成一个段落。代码如下：

	``` java
    if ((bufEnd == bufStart || source.charAt(bufEnd - 1) == CHAR_NEW_LINE) &&
            mLineCount < mMaximumVisibleLineCount) {
       
        measured.setPara(source, bufEnd, bufEnd, textDir, b);
        paint.getFontMetricsInt(fm);
        v = out(source,
                bufEnd, bufEnd, fm.ascent, fm.descent,
                fm.top, fm.bottom,
                v,
                spacingmult, spacingadd, null,
                null, fm, 0,
                needMultiply, measured.mLevels, measured.mDir, measured.mEasy, bufEnd,
                includepad, trackpad, null,
                null, bufStart, ellipsize,
                ellipsizedWidth, 0, paint, false);
    }
    ```
第10点和第11点分析中均出现了 `out()` 方法，前面提到，该方法也是 StaticLayout 源码中的一个重要的方法，接下来会分析下 `out` 方法中做了什么处理。 

#### out 方法分析
`out()` 方法在我看来，就是 layout 中的 out。如果说 `generate()` 大部分是处理一些折行、段落相关的数据，那么 `out()` 方法就是将这些数据使用起来，真正地布局出来（注意，布局不是显示，显示的话还是在父类的 `drawText()` 方法中进行的）。

1. 方法接收的参数如下所示，很多参数都是在 `generate()` 中处理获得，参数的含义和前面提到的基本相同。
    
    ``` java
     out(CharSequence text, int start, int end,
       int above, int below, int top, int bottom, int v,
       float spacingmult, float spacingadd,
       LineHeightSpan[] chooseHt, int[] chooseHtv,
       Paint.FontMetricsInt fm, int flags,
       boolean needMultiply, byte[] chdirs, int dir,
       boolean easy, int bufEnd, boolean includePad,
       boolean trackPad, char[] chs,
       float[] widths, int widthStart, TextUtils.TruncateAt ellipsize,
       float ellipsisWidth, float textWidth,
       TextPaint paint, boolean moreChars) 
    ```
    
2. 对 mLineDirections 和 mLine 扩容处理，根据当前行数，判断下 mLine 数组大小是否足够储存当前行的信息，如果不够，则扩容，对应的 mLineDirections 也进行扩容处理（两个数组大小相同）。
   
    ``` java
    int j = mLineCount;
    int off = j * mColumns;
    int want = off + mColumns + TOP;
    int[] lines = mLines;
    if (want >= lines.length) {
        Directions[] grow2 = ArrayUtils.newUnpaddedArray(
                Directions.class, GrowingArrayUtils.growSize(want))
        System.arraycopy(mLineDirections, 0, grow2, 0,
                         mLineDirections.length);
        mLineDirections = grow2;
        int[] grow = new int[grow2.length];
        System.arraycopy(lines, 0, grow, 0, lines.length);
        mLines = grow;
        lines = grow;
    }
    ```
3. 待分析
   
    ``` java
    if (chooseHt != null) {
        fm.ascent = above;
        fm.descent = below;
        fm.top = top;
        fm.bottom = bottom;
        for (int i = 0; i < chooseHt.length; i++) {
            if (chooseHt[i] instanceof LineHeightSpan.WithDensity) {
                ((LineHeightSpan.WithDensity) chooseHt[i]).
                    chooseHeight(text, start, end, chooseHtv[i], v, fm, paint);
            } else {
                chooseHt[i].chooseHeight(text, start, end, chooseHtv[i], v, fm);
            }
        }
        above = fm.ascent;
        below = fm.descent;
        top = fm.top;
        bottom = fm.bottom;
    }    
    ```
4. 第一行和最后一行的特殊处理，以及行间距的处理

	```java
	// 判断是否是第一行
	boolean firstLine = (j == 0);
	// 判断是否是最后一行：全部文本的最后一行或者行数等于可见的最大的行数
   boolean currentLineIsTheLastVisibleOne = (j + 1 == mMaximumVisibleLineCount);
    boolean lastLine = currentLineIsTheLastVisibleOne || (end == bufEnd);
    // 第一行需要处理上面的留白
    if (firstLine) {
        if (trackPad) {
            mTopPadding = top - above;
        }
        if (includePad) {
            above = top;
        }
    }
    int extra;
    // 最后一行需要处理下面的留白
    if (lastLine) {
        if (trackPad) {
            mBottomPadding = bottom - below;
        }
        if (includePad) {
            below = bottom;
        }
    }
   	// 处理行间距
    if (needMultiply && !lastLine) {
       double ex = (below - above) * (spacingmult - 1) + spacingadd;
       if (ex >= 0) {
           extra = (int)(ex + EXTRA_ROUNDING);
       } else {
           extra = -(int)(-ex + EXTRA_ROUNDING);
       }
   } else {
       extra = 0;
   }
	```
	
5. 接下来就是记录每行的文本的信息，需要注意到的是，每行的信息由 lines 中的连续的值来记录，值的数量等于 mColumns 的大小( mColumns 的取值前面有提到)。
	
	``` java
	// 记录每行的起止位置，顶部和底部位置
	lines[off + START] = start;
   lines[off + TOP] = v;
   lines[off + DESCENT] = below + extra;
   // 记录下一行的起始位置和顶部位置，v 值会作为返回值返回给调用的地方。
   v += (below - above) + extra;
   lines[off + mColumns + START] = end;
   lines[off + mColumns + TOP] = v;
   // TODO: could move TAB to share same column as HYPHEN, simplifying this code and gaining
   // one bit for start field
   // 通过位运算记录 tab 和文本方向信息
   lines[off + TAB] |= flags & TAB_MASK;
   lines[off + HYPHEN] = flags;
   lines[off + DIR] |= dir << DIR_SHIFT;
   Directions linedirs = DIRS_ALL_LEFT_TO_RIGHT;
   // easy means all chars < the first RTL, so no emoji, no nothing
   // XXX a run with no text or all spaces is easy but might be an empty
   // RTL paragraph.  Make sure easy is false if this is the case.
   // 记录文本的方向
   if (easy) {
       mLineDirections[j] = linedirs;
   } else {
       mLineDirections[j] = AndroidBidi.directions(dir, chdirs, start - widthStart, chs,
               start - widthStart, end - start);
   }
	```
	
6. 文本省略的处理：
	
	``` java
	// 判读是否需要省略
	if (ellipsize != null) {
       // If there is only one line, then do any type of ellipsis except when it is MARQUEE
       // if there are multiple lines, just allow END ellipsis on the last line
       boolean forceEllipsis = moreChars && (mLineCount + 1 == mMaximumVisibleLineCount);
       boolean doEllipsis =
                   (((mMaximumVisibleLineCount == 1 && moreChars) || (firstLine && !moreChars)) &&
                           ellipsize != TextUtils.TruncateAt.MARQUEE) ||
                   (!firstLine && (currentLineIsTheLastVisibleOne || !moreChars) &&
                           ellipsize == TextUtils.TruncateAt.END);
       if (doEllipsis) {
           calculateEllipsis(start, end, widths, widthStart,
                   ellipsisWidth, ellipsize, j,
                   textWidth, paint, forceEllipsis);
       }
   }
	```
	如 Google 的工程师注释所说的那样，如果是指定了最大行数是1，则任何省略方式都可以，如果指定的最大行数不是1，但是只有单行文本时，除了 `MARQUEE` 的省略方式不支持以外，其他的省略方式都是支持的。如果是多行省略，且不止一行文本时，只支持在可见的最后一行的最后省略，即 `END` 省略方式。
	
	省略的计算是通过 `calculateEllipsis()` 方法实现的，其内部处理完成会将省略的起始位置和计数复制给 mLines 对应的每行数据的第5和第6个数据（省略时每行的记录的数据个数为6个，即 mColumns 赋的值是 COLUMNS_ELLIPSIZE 的值，即6），`calculateEllipsis()`方法的实现这里就不作具体分析了。
	

#### 总结
至此，StaticLayout 的源码大致分析了一遍，后面需要结合 TextView 和 Layout 来具体看一下，文字到底是怎么绘制到屏幕上的。