---
title: 理解Python装饰器（Decorator）
id: 199
categories:
  - Python
date: 2017-04-14 00:06:46
tags:
---

Python的装饰器，顾名思义就是可以为已有的函数或对象起到装饰的作用，使得达到代码重用的目的。

### 从一个简单的例子出发

这个例子中我们已经拥有了若干个独立的函数。
```python
#! python 2
# coding: utf-8

def a():
　　print("a")

def b():
　　print("b")

def c():
　　print("c")

a()
b()
c()

```
输出结果是
```
a
b
c
```
而我们想要给这三个函数都加上打印当前日期的功能，倘若没有学习装饰器，那我们可能要为每一个函数都添加一行语句。
```python
#! python 2
# coding: utf-8

import time

def a():
　　print(time.asctime( time.localtime(time.time())))
　　print("a")

def b():
　　print(time.asctime( time.localtime(time.time())))
　　print("b")

def c():
　　print(time.asctime( time.localtime(time.time())))
　　print("c")

a()
b()
c()
```
这样看来需要添加的代码量似乎并不多，但如果需要被添加此功能的已经写好的模块已经有上百上千甚至上万？这样写岂不是过于繁杂，而有了装饰器，我们则可以像下面这样。
```python
#! python3
# coding: utf-8

import time

# return time
def rtime(func):
　　def wrapper():
　　　　print(time.asctime( time.localtime(time.time())))
　　　　return func()
　　return wrapper

@rtime
def a():
　　print("a")

@rtime
def b():
　　print("b")

@rtime
def c():
　　print("c")

a()
b()
c()

```
怎么样，是不是简洁明了了很多。

### 更加通用一点的装饰器

可以看到，上面的三个函数都没有接受参数，那如果rtime去装饰含有参数的函数会怎样呢？
```python
#! python3
# coding: utf-8

import time

# return time
def rtime(func):
　　def wrapper():
　　　　print(time.asctime(time.localtime(time.time())))
　　　　return func()
　　return wrapper

@rtime
def d(a):
　　print(a)

d(1)
```
显然，d函数包含了一个参数d，如果此时运行的话，则会出现报错
```
TypeError: wrapper() takes no arguments (1 given)
```
为了可以使得装饰器更加通用，我们可以像下面这样写：
```python
#! python3
# coding: utf-8

import time

# return time
def rtime(func):
　　def wrapper(*args, **kwargs):
　　　　print(time.asctime(time.localtime(time.time())))
　　　　return func(*args, **kwargs)
　　return wrapper

@rtime
def d(a):
　　print(a)

d(1)
```
可以看到，其中添加了*args和**kwargs，Python提供了可变参数*args和关键字参数**kwargs用于处理未知数量参数，这样就能解决被修饰函数中带参数得问题。<br>
那如果是装饰器本身想要带上参数呢，先记住这样一句话：本身需要支持参数得装饰器需要多一层的内嵌函数。下面看具体的代码实现：
```python
#! python3
# coding: utf-8

import time

# return time
def rtime(x):
    print(x)
    def wrapper(func):
        def inner_wrapper(*args, **kwargs):
            print(time.asctime(time.localtime(time.time())))
            return func(*args, **kwargs)
        return inner_wrapper
    return wrapper

@rtime(1) # x = 1
def d(a):
    print(a)

d(2)
```
输出结果为
```
1
Wed Apr 12 20:43:54 2017
2
```
### 调用多个装饰器
python的装饰器支持多次调用，且调用的顺序与在被装饰函数前声明装饰器的顺序相反，如若想先调用装饰器demo1和demo2，则装饰时应先@demo2再@demo1。
### 类实现的装饰器
若想要通过类来实现装饰器，则需要修改类的构造函数__init__()并重载__call__()函数。下面是一个简单的例子：
```python
#! python3
# coding: utf-8

import time

# return time
def rtime(x):
    print(x)
    def wrapper(func):
        def inner_wrapper(*args, **kwargs):
            print(time.asctime(time.localtime(time.time())))
            return func(*args, **kwargs)
        return inner_wrapper
    return wrapper


# return time 不带参数的类实现
class rtime1(object):
    def __init__(self, func):
        self.func = func
    
    def __call__(self, *args, **kwargs):
        print(time.asctime(time.localtime(time.time())))
        return self.func(*args, **kwargs)

# return time 带参数的类实现
class rtime2(object):
    def __init__(self, x):
        self.x = x
    
    def __call__(self, func):
        def wrapper(*args, **kwargs):
            print(self.x)
            print(time.asctime(time.localtime(time.time())))
            return func(*args, **kwargs)
        return wrapper

@rtime(1)
def d(a):
    print(a)

@rtime1
def e(a):
    print(a)

@rtime2(2)
def f(a):
    print(a)

d(3)
e(4)
f(5)
```
输出结果为
```
1
Thu Apr 13 11:32:22 2017
3
Thu Apr 13 11:32:22 2017
4
2
Thu Apr 13 11:32:22 2017
5
```
### Python内置装饰器
#### @property
内置装饰器property可以帮助我们为一个class写入属性
```python
#! python3
# coding: utf-8

import time


class test(object):
    
    @property
    def x(self):
        return self._x
    
    @x.setter
    def x(self, value):
        self._x = value

    @x.deleter
    def x(self):
        del self._x


temp = test()
temp.x = 1
print temp.x
```
输出结果为1，想必会有人疑惑为什么要这样写入属性，如果没有这样绑定属性直接将temp.x赋值的话，则属性x是不可控的，而通过property绑定属性之后，则可以在setter设定的时候添加对范围的判断，使得属性可控，property还有getter装饰器，不过getter装饰器和不带getter的属性装饰器效果一样。
#### @staticmethod &amp; @classmethod
通过staticmethod和classmethod装饰器可以使得我们在不实例化类的情况下直接调用类中的方法：class_name.method()即可直接调用。<br>
那么staticmethod和classmethod又有什么区别呢？<br>
@staticmethod不需要表示实例的self和自身类的cls参数，也就是说可不传递参数。<br>
@classmethod的第一个参数必须有且必须是表示自身类的cls参数。
