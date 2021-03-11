---
title: cpp对象内存布局.md
date: 2019-05-21 19:42:43
draft: true
tags:
    - cpp
    - 备忘
keywords: cpp, c++, 内存布局, 继承, 多重继承, 虚继承
---

### 前言

本文主要是分析一下cpp对象在单继承、多重继承、虚继承下的内存布局，给自己做个笔记...

### 正文

#### 0.C结构体

先从C结构体说起，内存布局原则：**成员变量按其被声明的顺序排列，按具体实现所规定的对齐原则在内存地址上对齐**
```C
struct A {
    char c;
    int n;
}
```

[图]
(挖坑待填...)

#### 1.简单cpp结构
```cpp
struct B {
public:
    int b1;
private:
    char b2;
public:
    static int b3;
    void bf1();
    static void bf2();
}
```

(挖坑待填...)

#### 2.单继承

(挖坑待填...)
