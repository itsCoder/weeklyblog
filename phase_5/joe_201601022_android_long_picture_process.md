---
title: Android 中的长图片处理
date: 2016-10-22 16:38:21
categories: Android
tags:
---

最近太忙了，好久没有写东西了，itsCoder 也缺席了一期，感觉自己也好久没有学习新的东西了。不管再忙还是应该抽出时间来学习和锻炼，不然时刻感觉身体被掏空啊～

### 前言ß

图片处理在 Android 开发中是一个比较重要的技术，处理稍有不慎就可能会出现 oom 等情况，网上也有很多图片处理相关的文章，我在[Volley 中的ImageLoader源码学习笔记](http://extremej.itscoder.com/volley_imageloader_source/)这篇文章中也分析了一些解决方案。同时开源社区也有一些优秀的图片处理框架－Glide,Fresco,Picasso 等。这篇文章不会再讲这些解决方案。

我自己在业余时间做了一款图片应用，在实际的开发中我发现有一种特殊的情况，加载超长或者超宽的图片，并且随着拼图的越来越普及，以后这种长图的出现可能会变得更加的常见，这篇文章就来讲讲我在处理长图显示中遇到的坑。

### Max Texture Size 问题

在实际的开发中我使用的是 Glide 作为图片框架，但是我发现依然会有一些图片无法显示出来。通过 log 定位，我发现这些图片都比较长，并且抓到了这样一个异常：

```
[WARN] : OpenGLRenderer: Bitmap too large to be uploaded into a texture (752x8546, max=8192x8192)
```

从字面意思来理解是当前图片的尺寸超过了 `OpenGL` 的渲染上限。最大值是 8192 而图片长度明显超过了最大长度。

- `OpenGL` 是什么？

> **Open Graphics Library** (**OpenGL**)[[3\]](https://en.wikipedia.org/wiki/OpenGL#cite_note-3)[[4\]](https://en.wikipedia.org/wiki/OpenGL#cite_note-4) is a [cross-language](https://en.wikipedia.org/wiki/Language-independent_specification), [cross-platform](https://en.wikipedia.org/wiki/Cross-platform) [application programming interface](https://en.wikipedia.org/wiki/Application_programming_interface) (API) for rendering [2D](https://en.wikipedia.org/wiki/2D_computer_graphics) and[3D](https://en.wikipedia.org/wiki/3D_computer_graphics) [vector graphics](https://en.wikipedia.org/wiki/Vector_graphics). The API is typically used to interact with a [graphics processing unit](https://en.wikipedia.org/wiki/Graphics_processing_unit) (GPU), to achieve [hardware-accelerated](https://en.wikipedia.org/wiki/Hardware_acceleration)[rendering](https://en.wikipedia.org/wiki/Rendering_(computer_graphics)).

维基百科上面是这样解释的。

我自己对 `OpenGL` 并没有什么了解，这里也只是本着解决问题的想法，所以想要深入了解 `OpenGL` 的同学可以自行学习。简单来说就是图片的尺寸超过了 `OpenGL` 渲染的限制。

### 解决方案

知道了问题的所在，就可以开始寻找解决方案了，寻找解决方案的过程就不细说了，直接抛出三个可行的方案来解决。

- 调整图片尺寸
- 局部显示图片
- 关闭硬件加速

目前我所寻找到的解决方案是这三种，其中第一种是最容易想到的，也是本篇文章重点讲解的。

Android 在使用 `OpenGL` 去渲染图片的时候实际上开了硬件加速（不知道这么说对不对），所以最简单粗暴的方法就是在 `AndroidManifest.xml` 文件中去关闭硬件加速，当然这么做是不怎么清真的，可能会引起其他的问题。

### 调整图片尺寸

resize 图片的尺寸大概每个开发者都会，因此这个方案里面最为重要的一个步骤是获取 max texture size ，通过测试发现不同的手机这个尺寸可能是不一样的。那么如何精确的获取当前手机的渲染限制大小呢？

上 StackOverFlow 上面搜了一下，大部分答案都给出了如下代码

```java
int[] maxTextureSize = new int[1];
GLES10.glGetIntegerv(GL10.GL_MAX_TEXTURE_SIZE, maxTextureSize, 0);
```

先别急着去试，如果你稍微仔细一点会发现这些答案后面基本都会有疑惑，这样获取到的值始终是 0。

所以又换了个问题，为什么会是 0？搜到了这个答案：

http://stackoverflow.com/questions/26985858/gles10-glgetintegerv-returns-0-in-lollipop-only

> call to OpenGL ES API with no current context (logged once per thread)
>
> You need a current OpenGL context in your thread before you can make **any** OpenGL calls, which includes your `glGetIntegerv()` call. This was always true. But it seems like in pre-Lollipop, there was an OpenGL context that was created in the frameworks, and that was sometimes (always?) current when app code was called.

在调用 OpenGL ES API 的时候没有一个 context。所以问题又变成了如何创建一个 OpenGL 的 context？因为这个回答已经相当详细，我只简单的翻译一下。

- 创建 `GLSurfaceView`，这是最简单最方便的一种方法，但是该方法只有当你需要使用 OpenGL 来渲染展示画面的时候才具有实际意义。
- 使用 `EGL14`来完成调用 `OpenGl` 的相关 API 前的准备工作，并且不需要真正的渲染画面。

第一种方法在官方文档中给出了非常详细的教程－[GLSurfaceView](https://developer.android.com/reference/android/opengl/GLSurfaceView.html)

第二种方法中的 `EGL14` 只支持到 API 17，如果要兼容到更低的系统需要使用 `EGL10`。

#### 创建 OpenGL Context

- 初始化 `EGLDisplay`

```java
EGLDisplay dpy = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);
int[] vers = new int[2];
EGL14.eglInitialize(dpy, vers, 0, vers, 1);
```

- 做一些配置，因为我们只是获取下 texture size ，不需要真正的渲染，所以这些配置不是特别重要

```java
int[] configAttr = {
    EGL14.EGL_COLOR_BUFFER_TYPE, EGL14.EGL_RGB_BUFFER,
    EGL14.EGL_LEVEL, 0,
    EGL14.EGL_RENDERABLE_TYPE, EGL14.EGL_OPENGL_ES2_BIT,
    EGL14.EGL_SURFACE_TYPE, EGL14.EGL_PBUFFER_BIT,
    EGL14.EGL_NONE
};
EGLConfig[] configs = new EGLConfig[1];
int[] numConfig = new int[1];
EGL14.eglChooseConfig(dpy, configAttr, 0,
                      configs, 0, 1, numConfig, 0);
if (numConfig[0] == 0) {
    // TROUBLE! No config found.
}
EGLConfig config = configs[0];
```

- 为了成为当前的 context，需要渲染一个 `SurfaceView`，当是我们又不想真的去渲染一个 `view` ，因此创建一个在屏幕外的小 `surface`

```java
int[] surfAttr = {
    EGL14.EGL_WIDTH, 64,
    EGL14.EGL_HEIGHT, 64,
    EGL14.EGL_NONE
};
EGLSurface surf = EGL14.eglCreatePbufferSurface(dpy, config, surfAttr, 0);
```

- 创建 `OpenGL context`

```java
int[] ctxAttrib = {
    EGL14.EGL_CONTEXT_CLIENT_VERSION, 2,
    EGL14.EGL_NONE
};
EGLContext ctx = EGL14.eglCreateContext(dpy, config, EGL14.EGL_NO_CONTEXT, ctxAttrib, 0);
```

- Make Current

```java
EGL14.eglMakeCurrent(dpy, surf, surf, ctx);
```

到这儿我们的准备工作就做完了。

接下来就是获取 max texture size

```java
int[] maxSize = new int[1];
GLES20.glGetIntegerv(GLES20.GL_MAX_TEXTURE_SIZE, maxSize, 0);
```

有了这个值，怎么去调整图片就看你自己了～

#### 销毁 OpenGL Context

获取到了这个值以后就可以将这个上下文给销毁了

```java
EGL14.eglMakeCurrent(dpy, EGL14.EGL_NO_SURFACE, EGL14.EGL_NO_SURFACE,
                     EGL14.EGL_NO_CONTEXT);
EGL14.eglDestroySurface(dpy, surf);
EGL14.eglDestroyContext(dpy, ctx);
EGL14.eglTerminate(dpy);
```

### 局部显示图片

第一种方案虽然解决了大部分长图的显示问题，但是依然会面临 OOM 的风险。尤其是当对图片的显示要求比较高时（高清大图），上一种方案似乎也不够完美。

`BitmapRegionDecoder` 似乎是一个不错的选择。

> BitmapRegionDecoder can be used to decode a rectangle region from an image. BitmapRegionDecoder is particularly useful when an original image is large and you only need parts of the image.

这个类也是我最近才接触的，还没有用到实际的项目中去，但是感觉会是一个不错解决方案。目前在网上有一篇写的比较好的文章－[Android 高清加载巨图方案 拒绝压缩图片](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/1021/3607.html)，介绍的非常详细，也有 demo 可以学习。这个方案我就不再细说了，等具体用到项目中了再来看有没有什么值得补充的。

### 总结

这篇文章并没有太多原创的内容，毕竟涉及到问题解决的时候，一般都可以从网上找到答案。而这个问题是我实际开发中所碰到的，索性把较好的解决方案抛出来，也方便有同样需求的同学，如果你有更好的解决方案也非常希望你能分享出来。