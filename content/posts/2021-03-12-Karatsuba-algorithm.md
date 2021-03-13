---
title: "Karatsuba大数乘法算法"
date: 2021-03-12T21:35:03+08:00
draft: true
---

## 0x00 Python中的大整数乘法
最近在看Python源码，Python3中的整数对象`PyLongObject`是一个大整数实现。

```c
// Include/longobject.h

struct _longobject {
    PyObject_VAR_HEAD
    digit ob_digit[1];
};

...

typedef struct _longobject PyLongObject; /* Revealed in longintrepr.h */
```

`PyLongObject`是一个变长对象，对于比较大的整数， Python 将其拆成若干部分，保存在 ob_digit 数组中。

此处仅分析Python整数对象大整数乘法的实现，如有兴趣了解Python整数对象具体实现，可以前往 [Python中的整数对象](https://github.com/Junnplus/blog/issues/12)


以下是Python整数乘法的简略代码
``` c
// Objects/longobject.c
static PyObject *
long_mul(PyLongObject *a, PyLongObject *b)
{
    ...
    
    z = k_mul(a, b);
    
    ...
    
    return (PyObject *)z;
}

...

/* Karatsuba multiplication.  Ignores the input signs, and returns the
 * absolute value of the product (or NULL if error).
 * See Knuth Vol. 2 Chapter 4.3.3 (Pp. 294-295).
 */
static PyLongObject *
k_mul(PyLongObject *a, PyLongObject *b)
{
...
}
```
k_mul函数是[Karatsuba multiplication算法](https://en.wikipedia.org/wiki/Karatsuba_algorithm)的实现，一种快速相乘算法。



> From Wikipedia, the free encyclopedia
>
> The **Karatsuba algorithm** is a fast [multiplication algorithm](https://en.wikipedia.org/wiki/Multiplication_algorithm). It was discovered by [Anatoly Karatsuba](https://en.wikipedia.org/wiki/Anatoly_Karatsuba) in 1960 and published in 1962.[[1\]](https://en.wikipedia.org/wiki/Karatsuba_algorithm#cite_note-kara1962-1)[[2\]](https://en.wikipedia.org/wiki/Karatsuba_algorithm#cite_note-kara1995-2)[[3\]](https://en.wikipedia.org/wiki/Karatsuba_algorithm#cite_note-knuthV2-3) It reduces the multiplication of two *n*-digit numbers to at most $n^{log_23}$ approx $n^{1.58}$ single-digit multiplications in general (and exactly $n^{log_23}$  when *n* is a power of 2). It is therefore [asymptotically faster](https://en.wikipedia.org/wiki/Asymptotic_complexity) than the [traditional](https://en.wikipedia.org/wiki/Long_multiplication) algorithm, which requires $n^2$ single-digit products. For example, the Karatsuba algorithm requires 310 = 59,049 single-digit multiplications to multiply two 1024-digit numbers (*n* = 1024 = 210), whereas the traditional algorithm requires (210)2 = 1,048,576 (a speedup of 17.75 times).

## 0x01