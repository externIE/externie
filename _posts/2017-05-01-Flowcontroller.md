---
layout: project
title: "换个姿势表白吧"
date: 2017-02-14
excerpt: "你还在送花，送包，送手机吗"
tags: [javascript, html]
comments: true
---

# 一个流程控制应用
目标：做一个面向工作流的ToDoList

现状：工作中往往会有数条工作流在同时进行，每条工作流一般都会有固定的步骤和必须要执行的任务，光靠脑袋记忆经常容易出错，就像开发模式一样，应该有一套流程管理的东西来约束工作流的并发进行不出纰漏。

需求：
- 提供工作流模版
  - 工作流模板可定制
  - 可修改
  - 可删除
- 提供工作流实例的创建和删除
  - 工作流实例可以编辑修改
  - 工作流实例可以被保存为工作流模版
- 一条工作流由若干（>=1个）步骤组成
  - 步骤可以被创建，删除，修改
- 关于步骤
  - 每条工作流正在进行的步骤只有一个
  - 多条工作流的当前步骤可以同时进行
  - 每个步骤包括其生命周期内的每一天的ToDoList
  - 每个步骤有固定要完成的任务（来自工作流模板）
  - 完成步骤的固定任务才可以激活下个步骤
  - 激活下个步骤的同时必须关闭当前步骤
  - 每个步骤有包含有注意事项，注意事项可以不强制要求完成
- 关于任务
  - 任务分为必须完成，不必须完成两种
  - 任务被分配在步骤中则必须完成
  - 用户新建的任务被分配到当天的ToDoList中，不必须完成，第二天的ToDoList会自动加载前一天未完成的任务
  - 任务可以被修改，删除，创建
  - 用户可以将创建的任务转化为某条工作流的步骤中的必须完成的任务
 - 关于ToDoList
  - ToDoList负责正在进行的任务，当天结束的任务，当前步骤任务的展示 
  
## 开发
- 框架：vuejs
- 样式：materialize.css
- 构建：vue-cli
- 调试：chrome
- 编辑器：vscode+vue插件
- 代码管理：git

## 设计
- App
  - 工作流模板管理组件
  - 工作区组件
    - 工作流组件
      - 步骤组件
    - ToDoList组件
      - ToDoList组件（正在进行的任务）
        - Task组件 
      - FinishedList组件
        - Task组件

![设计图](http://externie.com/github/flowcontroller/design.png)

## 开发
### 第一个模块：ToDoList
ToDoList由两个组件组成
- ToDoList组件（正在进行）
- FinishedList组件

ToDoList组件（正在进行）和FinishedList组件又由Task组件组成

App组件持有所有任务信息，负责任务信息的存储和获取，并将任务信息发送给ToDoList和FinishList组件，然后由这两个子组件调用其子组建Task显示每条任务。Uwl图如下：

![ToDoList设计](http://externie.com/github/flowcontroller/externie-todolist.png)
