---
layout: post
title: 高质量python代码之类属性的访问方法
date: 2017-03-31 19:30:00
updated: 2017-03-31 19:30:00
comments: true
description: 编写高质量python代码之python的类属性访问方法
feature: images/tag-plugins/python.jpg
tags:
- Python
categories:
- 编写高质量python代码
---


python类常用的有三类访问类属性的魔法方式，这三个魔方方法让python类的使用更加灵活,拦截器、动态代理都可以很耗的实现。[Python魔法方法指南][0]

python类的魔法方法为访问class的属性提供了诸多方便。主要分为三类：
 - object.__getattr__(self, key) ：在通过`__getattribute__`访问不到属性的时候，会调用，返回一个值或`AttributeError`异常
 - object.__getattribute__(self, key) ：无条件的被调用，访问类/实例的属性。如果class中还定义了`__getattr__`，则`__getattr__`不会被访问，（除非显示调用或者引发了`AttributeError`异常）
 - object.__get__(self, instance, owner) ：如果class定义了它，则这个class就被成为`descriptor`,owner是所有者的类，instance是访问descriptor的实例，如果不是通过实例访问，而是通过类访问的话，instance则为None
 - object.__setattr__(self, item, value)：当试图为对象的item赋值时调用

<!--more-->

## object.__getattribute__(self, key)

方法`__getattribute__`是绝对、无条件的。访问类中属性和方法时，无论什么情况，都会访问该方法。

``` python
class SuperDynamo(object):
    def __getattribute__(self, key):
        if key == 'name':
            return 'bearboyxu'
        else:
            return AttributeError  #属性未知，抛出AttributeError异常

superdemo = SuperDynamo()
print superdemo.name         #无条件隐式调用__getattribute__方法，属性已知，返回bearboyxu
print superdemo.age          #属性未知，抛出AttributeError异常：<type 'exceptions.AttributeError'>
superdemo.name = 'xiaoyaya'  #显示的设置属性值
print superdemo.name         #即使被定义，但还是无条件的调用__getattribute__方法
```

**__getattribute__方法跟__getattr__方法可以结合使用，以达到跟踪属性值的目的，否则在创建实例后显示设置的值将会消失**


## object.__getattr__(self, key)

只有__getattribute__找不到的时候,才会调用__getattr__

当使用`__getattr__`访问类属性时，如果属性未定义，则会调用该方法

**如果类对象中已有该属性值或者已经显示的设置了该属性的值，则直接返回不访问**

``` python
class Dydemo(object):
    def __getattr__(self, key):
        if key == 'name':
            return 'bearboyxu'
        else:
            return AttributeError

demo = Dydemo()
print demo.name         #类对象demo中没有定义name属性，则调用__getattr__方法,输出 bearboyxu
demo.name = 'xiaoyaya'  #显示的设置属性值
print demo.name         #已经被定义，不访问__getattr__方法直接返回，输出 xiaoyaya
```

## object.__get__(self, instance, owner) [Python描述器引导][1]

如果一个对象同时定义了 __get__() 和 __set__(),它叫做资料描述器(data descriptor)。

仅定义了 __get__() 的描述器叫非资料描述器(常用于方法，当然其他用途也是可以的)

``` python
class RevealAccess(object):
    """A data descriptor that sets and returns values
       normally and prints a message logging their access.
    """

    def __init__(self, initval=None, name='var'):
        self.val = initval
        self.name = name

    def __get__(self, obj, objtype):
        print 'Retrieving', self.name
        return self.val

    def __set__(self, obj, val):
        print 'Updating' , self.name
        self.val = val

>>> class MyClass(object):
    x = RevealAccess(10, 'var "x"')
    y = 5

>>> m = MyClass()
>>> m.x
Retrieving var "x"
10
>>> m.x = 20
Updating var "x"
>>> m.x
Retrieving var "x"
20
>>> m.y
5
```

[0]:http://pyzh.readthedocs.io/en/latest/python-magic-methods-guide.html
[1]:http://pyzh.readthedocs.io/en/latest/Descriptor-HOW-TO-Guide.html
