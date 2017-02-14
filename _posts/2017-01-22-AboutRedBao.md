---
layout: post
title: "关于抢红包"
date: 2017-01-22
excerpt: "有什么事比抢红包还重要嘛？"
tags: [java, android, 安卓]
comments: true
---

## DIY抢红包“插件”

![红包View的Res-id](http://externie.com/redbao/2017-01-23_23_21_28.gif)

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
{% highlight java %}
package com.example.externie.redbaoproject;

import android.accessibilityservice.AccessibilityService;
import android.view.accessibility.AccessibilityEvent;
import android.view.accessibility.AccessibilityNodeInfo;
import android.widget.Toast;

import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Set;

/**
 * Created by externIE on 2017/1/22.
 */
public class RedBaoService extends AccessibilityService {

    private static final String RedBaoPackage    = "com.tencent.mm:id/a48";//红包的资源ID
    private static final String BtnKai           = "com.tencent.mm:id/be_";//按钮“开”的资源ID
    private static final String BtnBack          = "com.tencent.mm:id/gr"; //回退键的资源ID

    //该对象代表了整个窗口视图的快照
    private AccessibilityNodeInfo mRootNodeInfo = null;
    //已经拆过的红包收集容器
    private Set<AccessibilityNodeInfo> RB_OPENED_Set = new HashSet<AccessibilityNodeInfo>();
    //微信红包收集器
    private Set<AccessibilityNodeInfo> MC_RB_Set = new HashSet<AccessibilityNodeInfo>();

    private Set<AccessibilityNodeInfo> RootRes_Set = new HashSet<AccessibilityNodeInfo>();

    @Override
    public void onAccessibilityEvent(AccessibilityEvent accessibilityEvent) {
        mRootNodeInfo = accessibilityEvent.getSource();

        RootRes_Set.add(mRootNodeInfo);
        int size = RootRes_Set.size();


        if(mRootNodeInfo == null)
            return;//没有获得窗口视图快照

        if(accessibilityEvent.getEventType() == AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED){
            //窗口内容发生了变化
            this.onContentChange();
        }

        if(accessibilityEvent.getEventType() == AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED){
            //窗口状态发生了变化
            this.onWindowStateChange();
        }
    }

    @Override
    public void onInterrupt() {

    }

    /*
    * 窗口内容发生变化的时候*/
    private void onContentChange(){
        checkRedBaoAndClickIt(null);
        checkBtnBackAndClickIt();
    }

    /*
    * 窗口状态发生变化*/
    private void onWindowStateChange(){
        //checkRedBaoAndClickIt(null);
        checkKaiAndClickIt();
        checkBtnBackAndClickIt();
    }


    /*
    * 检查并拆掉没有拆过的红包
    *String strRedBaoSearchKeyWord 是搜索当前窗口中红包的关键字，微信红包的关键字为“领取红包”
    * 当参数为null的时候，采用resource-id寻找的方法*/
    private void checkRedBaoAndClickIt(String strRedBaoSearchKeyWord){
        List<AccessibilityNodeInfo> nodeList;
        if (strRedBaoSearchKeyWord != null){
            nodeList = getNodeListByText(strRedBaoSearchKeyWord);
        }else{
            nodeList =  getNodeListByResID(RedBaoPackage);
        }
        if (nodeList == null) return;

        if(nodeList.size()>0) {
            //至少找到一个红包的NodeInfo

            /*
            *想法一：找到用一个Set集合来存取打开过的NodeInfo，但我发现一个悲剧的事实，就是每次新事件到来
            * 所带来的NodeList里面的Node对象和存在Set里面的是一摸一样的，也就是说即便是新的红包出现
            * 只要当前界面的红包节点数小于set里面的红包节点数，set就会拒绝相同的节点存入啦，所以新的红包就被认为是拆过的
            * 很悲剧！！！所以想法一 Failed！！！
            */

            /*Iterator<AccessibilityNodeInfo> it = nodeList.iterator();
            while (it.hasNext()) {
                AccessibilityNodeInfo tempNode = it.next();
                if (RB_OPENED_Set.add(tempNode)) {//存在一个没有打开过的红包，那么我们现在贱贱的打开它吧，哈哈哈。
                    System.out.println("找到一个没有打开过的红包");
                    showTip("找到一个没有打开过的红包");
                    //点击该节点
                    tempNode.performAction(AccessibilityNodeInfo.ACTION_CLICK);
                    break;
                } else {
                    System.out.println("找到一个已经拆过的红包");
                    showTip("找到一个已经拆过的红包");
                }
            }*/

            /*
            * 想法二：不管三七二十一，把当前界面最下面的一个红包节点打开
            * 这样做的话，在windowState事件里面就不能有判断红包节点的动作，要不然就会无限循环啦
            * 可能有办法解决这样的无限循环
            * 于是我把windowState事件里面的红包检查给去掉了
            * 只剩下contentChange事件里面的红包检查
            * 这同样带来一个问题就是，你不能滑动你的屏幕啦，滑动就会默认去点击最后一个红包，即便它被拆过
            * 还有，当你第一次进入群的时候，service是不会去点击没有抢过的红包的
            * 只有有人新发了红包service才能正常工作
            * 也就是说在想法二下你需要这么做
            * 1。保持屏幕常亮
            * 2。不动手机
            * 3。默默等红包
            * 虽然很傻，但至少它开始工作了。。。
            * */
            AccessibilityNodeInfo lastOne = nodeList.get(nodeList.size() - 1);
            lastOne.performAction(AccessibilityNodeInfo.ACTION_CLICK);
            System.out.println("找到最新的一个，并拆开");
            showTip("找到最新的一个，并拆开");
        }
    }

    /*
    * 寻找“开”字按钮并点击它*/
    private void checkKaiAndClickIt(){
        boolean bSuccess = findViewByIDAndClickIt(BtnKai);
        if(bSuccess){
            System.out.println("点击开");
            showTip("点击开");
        }else{
            System.out.println("没找到开");
            showTip("没找到开");
        }
    }

    /*
    * 寻找“返回”按钮并点击它*/
    private void checkBtnBackAndClickIt(){
        boolean bSuccess = findViewByIDAndClickIt(BtnBack);
        if(bSuccess){
            System.out.println("点击返回");
            showTip("点击返回");
        }else{
            System.out.println("没找到返回");
            showTip("没找到返回");
        }
    }

    /*
    * 找到对应的res-id的控件并点击*/
    private boolean findViewByIDAndClickIt(String strID){
        List<AccessibilityNodeInfo> nodeList = getNodeListByResID(strID);
        if (nodeList == null) return false;
        if(nodeList.size()>0) {
            AccessibilityNodeInfo lastOne = nodeList.get(nodeList.size() - 1);
            lastOne.performAction(AccessibilityNodeInfo.ACTION_CLICK);
            return true;
        }
        return false;
    }

    /*
    * 根据资源ID来获取节点列表*/
    private List<AccessibilityNodeInfo> getNodeListByResID(String resID){
        if (mRootNodeInfo != null){
            //注意：findAccessibilityNodeInfosByViewId API18以上才可以用哦
            return mRootNodeInfo.findAccessibilityNodeInfosByViewId(resID);
        }else{
            return null;
        }
    }

    /*
    * 根据关键词来获取节点列表*/
    private List<AccessibilityNodeInfo> getNodeListByText(String strKeyWord){
        return getNodeListByResID(strKeyWord);
    }

    private void showTip(String strTip){
        Toast.makeText(this, strTip, Toast.LENGTH_SHORT).show();
        System.out.print("externIE: "+strTip);
    }
}

{% endhighlight %}

看了以上代码你可能会问`RedBaoPackage` `BtnKai` `BtnBack` 这三个resource-id是怎么来的？下面就告诉你怎么来。

***注意！*** resource-id 每台手机都不一定一样哦，所以直接拿上面的代码，很大可能是无效的，下面教你怎么找你手机的resource-id
{: .notice}

#### 神器的Android Device Monitor

ADM是AS的一个工具，可以帮助你把手机当前界面的UI视图树dump出来，只要把鼠标定位到相应的view上，你就可以看到该view的resource-id，牛逼吧。
![红包View的Res-id](http://externie.com/redbao/first.png)
![两个红包View Res-id的对比](http://externie.com/redbao/second.png)
![按钮“开”的Res-id](http://externie.com/redbao/third.png)
![返回键的Res-id](http://externie.com/redbao/fouth.png)
***注意！***返回键的res-id应该去ImageView的父视图LinnerLayout的res-id，上面的截图有误。
{: .notice}

#### 写一个启动Service的Activity

{% highlight java %}
package com.example.externie.redbaoproject;

import android.content.Intent;
import android.provider.Settings;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;

public class MainActivity extends AppCompatActivity {

    private Button btn_startService;
    private Intent accessibleIntent = new Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        btn_startService = (Button)findViewById(R.id.startService);
        btn_startService.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //跳到设置界面，让用户打开监听服务
                startActivity(accessibleIntent);
            }
        });
    }
}
{% endhighlight %}

### 工程的git地址

代码很糙。。Just for fun！！！嘻嘻。 https://github.com/externIE/RedBaoProject.git

