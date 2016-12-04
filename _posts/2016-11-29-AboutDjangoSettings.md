---
layout: post
title: "关于Django-Settings.py的一些事"
date: 2016-11-29
excerpt: "关于Django-Settings看这篇就够了"
tags: [python, django, settings]
comments: true
---

## Django settings 概述
Django settings是个神奇的py文件，它包含了整个django站点的相关设置，其可选项是很多的，采用import * 覆写的方式，配置属于自己的站点，也就是说其实django本来就已经管理有一份默认的settings配置。
这份文件基本上决定了你的服务器是否要输出调试信息，采用什么数据库，站点内都有什么app，有什么中间件，你也可以自定义一些字段，以便全站使用。所以这分文件是及其重要的，也是及其私密的，不可随意公开。

## 一些基本知识
settings文件其实就是一个python带模块级别变量的模块罢了。
下面是一些常见的配置：
{% highlight python %}
ALLOWED_HOSTS = ['www.externie.com']
DEBUG = False
DEFAULT_FROM_EMAIL = 'xxx@163.com'
{% endhighlight %}
**注意!** 如果DEBUG==False那么ALLOWE_HOSTS就一定要有值，不能是空数组
{: .notice}
由于settings文件其实是个python模块，所以它也具备python模块的一些特性：
* 不能有python语法错误
* 可以用正确的python语句给settings动态赋值，如下：
{% highlight python %}
MY_SETTING = [str(i) for i in range(30)]
{% endhighlight %}
* 你也可以从其它的settings文件中导入（import）一些变量

## 指定使用哪份settings文件
### DJANGO_SETTINGS_MODULE
当你在用django的时候，你必须告诉django你需要用的settings文件。这时候就需要用上`DJANGO_SETTINGS_MODULE`这个环境变量。

DJANGO_SETTINGS_MODULE需要用python的点路径，比如`e.g.mysite.settings`。当然了settings这个文件必须在python的搜索路径下，换句话说就是sys.path数组中一定要有你要用的settings模块的路径。

### django-admin
当你在用django-admin的时候，你也可以只设置一次`DJANGO_SETTINGS_MODULE`就可以了，之后你的每次运行，django-admin都会使用你所设置的settings文件。

例子（类Unix）：
{% highlight python %}
export DJANGO_SETTINGS_MODULE = mysite.settings
django-admin runserver
{% endhighlight %}

例子（windows）：
{% highlight python %}
set DJANGO_SETTINGS_MODULE = mysite.settings
django-admin runserver
{% endhighlight %}

当然你也可以用--settings命令行来指定当前需要的特定的settings文件，如下：
{% highlight python %}
django-admin runserver --settings=mysite.settings
{% endhighlight %}

### 动态指定
在你的动态服务器环境下，你需要告诉WSGI应用当前用的是那一份settings文件，你可以这么做：
{% highlight python %}
import os
os.environ['DJANGO_SETTINGS_MODULE'] = 'mysite.settings'
{% endhighlight %}
[关于Django mod_wsgi的文档](https://docs.djangoproject.com/en/dev/howto/deployment/wsgi/modwsgi/)

## 默认的settings文件
一份settings文件中的一些字段如果你不需要配置的，你完全可以不写出来，因为django会有一份默认的settings文件，它包含了一些默认值，如果你没有重写这些字段的话就会是默认值。
这份更加神奇的settings文件在django/conf/global_settings.py 
** 关于settings的“继承机制”：
* 先从global_settings.py中加载所有配置
* 再加载你指定的settings文件，并根据你的settings中的内容覆盖global_settings中的一些字段
**注意!** 你并不需要在自己的settings文件中import global_settings，django已经帮你做好了
{: .notice}

## 未完待续。。。。