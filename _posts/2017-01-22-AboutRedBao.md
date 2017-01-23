---
layout: post
title: "关于抢红包"
date: 2016-11-29
excerpt: "有什么事比抢红包还重要嘛？"
tags: [java, android, 安卓]
comments: true
---

## DIY抢红包“插件”

### 背景

忘了什么时候开始出现微信红包这种东西了，也不知道当初是哪个伟大的产品经理的伟大想法，后来发现抢红包这种行为逐渐地成为过年过节的一种习惯。但是呢，亲朋好友聚会的时候，搓麻将抽不开身的时候，自己蒙头戳手机似乎不太礼貌，但是呢红包又不想错过，怎么办呢？！让我来告诉你怎么办。

### 需求

我们需要一个能够实现自动抢红包的工具

### 准备

包饺子得先准备好面，写程序也一样，准备清单如下：
* JDK（java的开发环境）
* Android Studio（安卓开发官方IDE，我们不仅会在上面码代码，还会用它的一个牛逼工具来分析微信界面~）
* 一台安卓手机
* 一颗爱折腾的心（我相信在以上环境配置和以后的码代码中，你可能会遇到一些问题，never mind，遇到的问题都是你将学到的知识）

### 开始

开始码代码了

#### 新建一个Android工程

选择模板Empty Activity就可以了，后面我们只需要在这个Activity里面放一个按钮，让用户能够跳转到设置界面开启抢红包的Service就足够了。

#### 新建一个Java class

我取名叫RedBaoService（家里穷，学不起英语，蟹蟹），RedBaoService继承AccessibilityService（这个类厉害了，后面一单元介绍一下）。继承AccessibilityService后必须实现两个
抽象方法`onAccessibilityEvent`和`onInterrupt`。AS给我们提供了一个很方便的实现抽象方法的工具，具体步骤如下
1. 光标定到类名RedBaoService
2. 快捷键alt+insert
3. 选中implement
4. 勾选需要实现的方法
5. 确定

#### 什么是AccessibilityService

**注意!** 如果你只是急于抢红包可以忽略这一单元。
{: .notice}

Accessibility services是android中用来给帮助一些残障人士使用安卓手机或者安卓app的。它运行在后台并且当有AccessibilityEvents发生的时候系统会调用Accessibility services中相应的回调方法。
就好像页面的切换，焦点的变换，按钮的点击等等，Accessibility服务可以帮你分析当前界面，你甚至可以用它做一些自动化操作。很神奇不是吗？

#### AndroidManifest.xml

在AndroidManifest.xml文件中注册一个service如下：

{% highlight javascript %}
<service
    android:name=".RedBaoService"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
    >
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService">

        </action>
    </intent-filter>
    <meta-data android:name="android.accessibilityservice"
        android:resource="@xml/accessible_service_config"
        ></meta-data>
</service>
{% endhighlight %}

把以上代码插入到<application>标签中

这时候我相信如果你的IDE是正常的并且眼睛也是的话，`android:resource="@xml/accessible_service_config"`这段内容应该会被标红，因为我们还没有新建这个配置文件。

#### 新建配置文件accessible_service_config
accessibility-service

1. res文件夹右键
2. New -》 Android Resource file
3. Resource type选择xml
4. 输入File name： accessible_service_config
5. 输入Root element： accessibility-service
6. OK

{% highlight javascript %}
<accessibility-service
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:description="@string/app_name"
    android:accessibilityEventTypes="typeWindowContentChanged|typeWindowStateChanged"
    android:accessibilityFeedbackType="feedbackAllMask"

    android:notificationTimeout="10"
    android:accessibilityFlags=""
    android:canRetrieveWindowContent="true"
    android:packageNames="com.tencent.mm">
</accessibility-service>
{% endhighlight %}

在accessible_service_config中插入以上标签
值得注意的是android:packageNames和android:accessibilityEventTypes这两个属性，packageNames填写你想要监听app的包名，这里监听的是微信，所以填的com.tencent.mm。
android:accessibilityEventTypes填的是你要监听的事件类型，typeWindowContentChanged是窗口内容变化，typeWindowStateChanged是窗口的状态变化。

#### 进入正文

下面我们要开始动手写RedBaoService了。