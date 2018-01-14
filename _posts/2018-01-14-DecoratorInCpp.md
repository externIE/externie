---
layout: post
title: "C++泛型实现类似于Python的装饰器"
date: 2018-01-14
excerpt: "用C++0x的特性实现类似于Python的装饰器"
tags: [C++, C++11, 泛型编程, 装饰器]
comments: true
---

```
//
//  FuncWapper.hpp
//  FuncWapper
//
//  Created by externIE on 2018/1/13.
//  Copyright © 2018年 externIE. All rights reserved.
//

#ifndef FuncWapper_hpp
#define FuncWapper_hpp

#include <stdio.h>
#include <time.h>

class AutoCountTime
{
private:
    clock_t m_tStart;
public:
    AutoCountTime(){
        m_tStart = clock();
    }
    ~AutoCountTime()
    {
        int dTime = clock() - m_tStart;
        printf("【耗时】%d毫秒\n", dTime);
    }
};

// 支持普通函数rettype func(args); ... ARGS 是模板变参
template<typename RET, typename... ARGS>
RET LYK_LogFuncTimeWapper(RET(*func)(ARGS...), ARGS... args)
{
    AutoCountTime t;
    return (*func)(args...);
}

// 为了支持可变参数的函数例如printf(const char* , ...);
template<typename RET, typename... ARGS, typename... ARGS2>
RET LYK_LogFuncTimeWapper(RET(*func)(ARGS2..., ...), ARGS... args)
{
    AutoCountTime t;
    return (*func)(args...);
}

// 适用于普通成员函数
template<typename RET, typename CLS, typename... ARGS>
RET LYK_LogFuncTimeWapper(RET(CLS::*func)(ARGS...), CLS* pObj, ARGS... args)
{
    AutoCountTime t;
    return (pObj->*func)(args...);
}

// 适用于变参成员函数
template<typename RET, typename CLS, typename... ARGS, typename... ARGS2>
RET LYK_LogFuncTimeWapper(RET(CLS::*func)(ARGS2..., ...), CLS* pObj, ARGS... args)
{
    AutoCountTime t;
    return (pObj->*func)(args...);
}


#endif /* FuncWapper_hpp */

```