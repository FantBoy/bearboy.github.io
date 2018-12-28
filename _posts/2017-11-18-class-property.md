---
layout: post
title: 类的属性管理
date: 2017-11-18 15:18
toc: true
description: 编写高质量python代码之类的属性管理
comments: true
tags:
- Python
categories:
- 编写高质量python代码
---


我们在定义一些基础类的时候，往往会有一些参数代表了这个类的属性，而这些属性我们不希望直接暴露给类对象，并且在类对象修改类属性值的时候，能够做参数校验。类的属性管理在这里就显得很重要。python提供了`@property` 属性装饰器来装饰类属性，使类的属性管理变得极为简单。`@property`可以让调用者写出简短的代码，又能很好的保证对参数进行必要的检查。

<!--more-->


## 属性装饰器的使用 

### 为class添加属性(getter)

Python为我们提供了**属性函数(property)**来设置类属性，property实际上是一个装饰器，使用极为方便。

``` python
class UserItem(object):
    @property
    def name(self):
        """
            @property 使name函数可以像访问类属性一样访问
        """
        return self._name
```

在函数 name() 上设置`@property`装饰器，该方法即成为了类UserItem的属性，可直接通过`UserItem.name`直接访问



### 为属性设置修改限制(setter)

当类对象想要设置属性值时，我们可以进行数据校验，甚至是拒绝修改，实现属性的只读。

``` python
class UserItem(object):
    @name.setter
    def name(self, name):
        """
            @name.setter 使_name的值，可以通过属性值设置的方式进行修改
            并可对待修改的值做逻辑判断
            也可以做显示修改的逻辑，使该属性成为 只读属性
        """
        if not isinstance(name, unicode):
            raise ValueError('name must be a string')
        self._name = name
```

跟添加属性类似，使用 {func_name}.setter 作为属性的装饰器，当类对象试图修改类属性值的时候，会走到该逻辑，对待修改的值进行数据校验和赋值。



>  如果属性没有设置 {func_name}.setter 装饰器的话，该属性将不能被修改(只读属性)




### 完整实例

``` python
class UserItem(object):

    @property
    def name(self):
        """
            @property 使name函数可以像访问类属性一样访问
        """
        return self._name

    @name.setter
    def name(self, name):
        """
            @name.setter 使_name的值，可以通过属性值设置的方式进行修改
            并可对待修改的值做逻辑判断
            也可以做显示修改的逻辑，使该属性成为 只读属性
        """
        if not isinstance(name, unicode):
            raise ValueError('name must be a string')
        self._name = name

    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, age):
        if not isinstance(age, int):
            raise ValueError('age must be a integer')
        if age < 0 or age > 120:
            raise ValueError('age must between [0, 120]')
        self._age = age
        
user = UserItem()
user.name = 'bear'
user.age = 27
user.age = 234
```


## 对历史代码中类参数的属性化改造 

面对一些遗留代码，可能通过自定义的get方法和set方法，实现了对类参数的访问和设置。像这类现象，通过property方法对类参数进行属性化改造也是很方便。

### 遗留代码

``` python
class HistoryUserItem(object):
    def __init__(self):
        self._name = 'bearboy'
        
    def get_name(self):
        return self._name
        
    def set_name(self, name):
        if not isinstance(name, unicode):
            raise ValueError('name must be a string')
        self._name = name
```

这种类参数的访问和设置都显得较麻烦，但是通过看property装饰后就不一样了。

将参数的get和set方法通过property装饰后，便可直接按照类属性的方式访问了。



### 改造

通过`property(get_name, set_name)` 改造参数 `_name`为类属性

``` python
class HistoryUserItem(object):
    def __init__(self):
        self._name = 'bearboy'
    def get_name(self):
        return self._name
    def set_name(self, name):
        if not isinstance(name, unicode):
            raise ValueError('name must be a string')
        self._name = name

    name = property(get_name, set_name)

user = HistoryUserItem()
user.name = 'bearboyxu'
print user.name
```







