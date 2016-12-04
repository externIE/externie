---
layout: post
title: "关于virtualenv的一些事"
date: 2016-12-04
excerpt: "关于virtualenv看这篇就够了"
tags: [python, virtualenv]
comments: true
---
## virtualenv 的概述
由于python包的繁多，每个包的版本之间存在不兼容，包括py2.x和3.x也存在不兼容现象。如果我们同时在开发多个py应用的时候可能就会产生这样一个尴尬的现象：
你的工程A需要使用`某包2.x`而你的工程B需要使用`某包3.x`，这两个版本之由于不兼容你需要不断的在开发过程中来回切换，是不是很麻烦呢。
virtualenv的作用就在于给你的每一个python程序提供了一个沙盒式的运行环境，简单的说你可以为你的某个python程序提供一个特殊的python环境。
你不仅可以拥有一份独立的lib库，还可以拥有一个独立的解释器interpreter，也就是说你可以在工程A中使用py2.7，在工程B中使用py2.8。

## 安装virtualenv
{% highlight python %}
pip install --upgrade virtualenv
{% endhighlight %}
**注意!** 在macOS 10.12和10.11下你可能得先关闭rootless功能，并且使用下面命令
{: .notice}
{% highlight python %}
sudo pip install --upgrade virtualenv
{% endhighlight %}

## 创建沙盒虚拟环境
cd 到你想放置虚拟环境的目录，在终端中输入下面命令
{% highlight python %}
virtualenv env
{% endhighlight %}
`env`是你建的虚拟环境的名字，在当前目录下你会发现virtualenv帮你建了一个venv的目录，事实上这个就是你的虚拟环境。

## 进入创建好的虚拟环境
{% highlight python %}
source env/bin/activate
{% endhighlight %}
进入成功后终端的前缀就是你现在所使用的那一份虚拟环境

## 退出当前虚拟环境
在当前环境下输入
{% highlight python %}
deactivate
{% endhighlight %}

## 删除虚拟环境
直接把你所建的虚拟环境对应的文件夹删除就好了