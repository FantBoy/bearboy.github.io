---
layout: post
title: 【Exception】 finally - local variable 's_path' referenced before assignment
date: 2017-04-19 20:35:11.000000000 +09:00
updated: 2017-04-19 20:35:11.000000000 +09:00
comments: true
tags:
- 异常记录
categories:
- Python
---

> 使用`try...except...finally...`做异常检测和处理时，在`finally`中出现了变量不存在的异常
> `local variable 's_path' referenced before assignment`

<!--more-->

## try...except...finally... 的正确打开方式
### try/except捕捉异常

try/except语句用来检测try语句块中的错误，从而让except语句捕获异常信息并处理。
让程序在出现异常时不强制退出程序

``` python
try:
    name = 'bearboyxu'
    age = 27
    print int(name)

except ValueError, e:
    print e
```

上述代码，很明显是存在bug的，`name`属于字符型字符串，并不是数字型字符串，使用`int`会触发异常

```
invalid literal for int() with base 10: 'bearboyxu'
```

### try...except...else

``` python
try:
    <语句>        #运行别的代码
except <名字>：
    <语句>        #如果在try部份引发了'name'异常
except <名字>，<数据>:
    <语句>        #如果引发了'name'异常，获得附加的数据
else:
    <语句>        #如果没有异常发生
```

try的工作原理是，当开始一个try语句后，python就在当前程序的上下文中作标记，这样当异常出现时就可以回到这里。
 - 如果当try后的语句执行时发生异常，python就跳回到try并执行第一个匹配该异常的except子句，异常处理完毕，控制流就通过整个try语句（除非在处理异常时又引发新的异常）。
 - 如果在try后的语句里发生了异常，却没有匹配的except子句，异常将被递交到上层的try，或者到程序的最上层（这样将结束程序，并打印缺省的出错信息）。
 - 如果在try子句执行时没有发生异常，python将执行else语句后的语句（如果有else的话），然后控制流通过整个try语句。

实例：
``` python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

try:
    fh = open("testfile", "w")
    fh.write("这是一个测试文件，用于测试异常!!")
except IOError:
    print "Error: 没有找到文件或读取文件失败"
else:
    print "内容写入文件成功"
    fh.close()
```

### try-finally
try-finally 语句无论是否发生异常都将执行最后finally的代码。

``` python
try:
    <语句>
finally:
    <语句>    #退出try时总会执行
raise
```

`finally`中的语句是无论如何都会执行的。即使在`try`或`except`中执行了`return`操作。

## finally 中出现变量不存在的异常
程序在运行过程中，如果抛出了异常，当程序进入finally时，变量还未被定义：
``` python
local variable 's_path' referenced before assignment
```

### 解决方法
``` python
if locals().has_key('s_path'):
    #...
```
