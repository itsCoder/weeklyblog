---
title: 使用贝塞尔曲线实现“一键下班”功能
date: 2016-11-13 22:01:10
categories: Android
tags: [Android, Bezier, animation]
---

# 前言
什么是“一键下班”？

![qq bubble](http://ww2.sinaimg.cn/mw690/005X6W83jw1f9qtzfqtj2j30ge07x0sw.jpg)

就是这个 QQ 消息气泡清除效果, QQ 气泡拖拽会产生粘性效果，拖拽到一定长度之后，连线断裂，气泡随之消失。大家都知道这用到了贝塞尔曲线，用来画链接线部分，ok，来学习一下贝塞尔曲线吧。

# 贝塞尔曲线
如果想用计算机来画一条直线，很简单，只要确定两个点，起点和终点就可以了，但是如果想要让计算机画一条曲线，那计算机该如何做呢？咳咳，肯定是通过数学公式来呗，
1962年，法国数学家Pierre Bézier第一个研究了这种矢量绘制曲线的方法，并给出了详细的计算公式，因此按照这样的公式绘制出来的曲线就用他的姓氏来命名是为贝塞尔曲线。那么可以说贝塞尔曲线就是通过某种公式绘制出的一条光滑曲线

## 一介贝塞尔曲线（直线）
方程：
![bezier1](http://a.hiphotos.baidu.com/baike/s%3D325/sign=391820be267f9e2f74351b0a2a31e962/91529822720e0cf34f59dca30b46f21fbe09aa38.jpg)

曲线：
![bezier1_line](http://ww1.sinaimg.cn/mw690/005X6W83jw1f9rzdypx1vg306o02s74z.gif)

一介贝塞尔曲线就是一条直线，确定两个点，得到一条直线。Android 中对应的方法为`lineTo(float x, float y)`

## 二介贝塞尔曲线
方程：
![bezier2](http://c.hiphotos.baidu.com/baike/s%3D317/sign=9aefef4b08f79052eb1f413f3bf2d738/11385343fbf2b21129581916cb8065380cd78e70.jpg)

曲线：
![bezier2_line](http://ww3.sinaimg.cn/mw690/005X6W83jw1f9rzdz67fgg306o02sgnc.gif)

二介贝塞尔曲线需要一对起始点以及一个控制点，如图控制点 p1 控制着曲线的拉伸程度。对应于 Android 中方法`quadTo(float x1, float y1, float x2, float y2)`

## 三介贝塞尔曲线
方程：
![bezier3](http://e.hiphotos.baidu.com/baike/s%3D421/sign=9a6521eab8014a90853e47bf98763971/f603918fa0ec08fad54f8dff58ee3d6d55fbda1f.jpg)

曲线：
![bezier3_line](http://ww2.sinaimg.cn/mw690/005X6W83jw1f9rzdzqbqlg306o02s76z.gif)

三介贝塞尔曲线需要一对起始点以及两个额外的控制点，控制点 p1,p2 控制着曲线的弯曲程度以及弯曲方向。对应于 Android 中方法`cubicTo(float x1, float y1, float x2, float y2,float x3, float y3)`

## 任意介贝塞尔曲线
方程：
![bezierx](http://f.hiphotos.baidu.com/baike/s%3D801/sign=a9e1f30835a85edffe8cf323785509d8/f9dcd100baa1cd11675be878b812c8fcc2ce2dfc.jpg)

高阶实际用的不多，可以通过多次低阶来实现。

# “一键下班”设计
需求---分析---实现
详见[QQ手机版 5.0“一键下班”设计小结](http://isux.tencent.com/qq-mobile-off-duty.html)

# 计算连接线算法
算法产考了一篇博客，原文网址找不到了，找了一个内容一样的博客，应该也是原作者博客，在此先贴出[博客地址](http://blog.csdn.net/xieyupeng520/article/details/50374561)，根据博客介绍，总结出几个重要控制点的计算公式，如下图（凑合着看，呵呵）

![no diao use](http://ww3.sinaimg.cn/mw690/005X6W83jw1f9s1bl264yj30eo0ag40d.jpg)

