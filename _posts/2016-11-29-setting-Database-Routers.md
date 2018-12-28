---
layout: post
date: 2016-11-29 23:23:0.000000000 +09:00
title: Django路由设之Database_Routers
comments: true
toc: true
tags: 
- Django
- Python
categories:
- Django 
---

在一些复杂的django服务中，一般都需要访问多个数据库的数据：自身库+若干外部库。
django提供了针对数据库的路由功能：[Database routers][1]。本文针对该方法，提供一种
是使用多数据库的方法、多个数据库的联用以及多数据库时数据的导入导出方法

<!--more-->

## 路由方式实现多数据库访问
### 为每个app都可以单独设置一个数据库

在settings.py中有数据库的相关设置，有一个默认的数据库 default  (*default 是必不可少的*)，
还可以额外添加其他数据库的配置[setting.py的官方配置文档][2]

``` python
# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': '',
        'USER': '',
        'PASSWORD': '',
        'HOST': '',
        'PORT': '',
    },
    'db1': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': '',
        'USER': '',
        'PASSWORD': '',
        'HOST': '',
        'PORT': '',
    },
}
 
# use multi-database in django
DATABASE_ROUTERS = ['project_name.database_router.DatabaseAppsRouter']
DATABASE_APPS_MAPPING = {
    # example:
    #'app_name':'database_name',
    'app1': 'db1',
}
```

 - *DATABASES* 为数据库服务器信息
 - *DATABASE_ROUTERS* 为数据库路由信息，指向路由类
 - *DATABASE_APPS_MAPPING* 为app与数据库实例的映射关系，不同的APP用不用的数据库，也可以用相同的数据库

### DatabaseAppsRouter 定义
在在project_name文件夹中创建 database_router.py 文件

``` python
# -*- coding: utf-8 -*-
from django.conf import settings
 
DATABASE_MAPPING = settings.DATABASE_APPS_MAPPING
 
 
class DatabaseAppsRouter(object):
    """
    A router to control all database operations on models for different
    databases.
 
    In case an app is not set in settings.DATABASE_APPS_MAPPING, the router
    will fallback to the `default` database.
 
    Settings example:
 
    DATABASE_APPS_MAPPING = {'app1': 'db1', 'app2': 'db2'}
    """
 
    def db_for_read(self, model, **hints):
        """"Point all read operations to the specific database."""
        if model._meta.app_label in DATABASE_MAPPING:
            return DATABASE_MAPPING[model._meta.app_label]
        return None
 
    def db_for_write(self, model, **hints):
        """Point all write operations to the specific database."""
        if model._meta.app_label in DATABASE_MAPPING:
            return DATABASE_MAPPING[model._meta.app_label]
        return None
 
    def allow_relation(self, obj1, obj2, **hints):
        """Allow any relation between apps that use the same database."""
        db_obj1 = DATABASE_MAPPING.get(obj1._meta.app_label)
        db_obj2 = DATABASE_MAPPING.get(obj2._meta.app_label)
        if db_obj1 and db_obj2:
            if db_obj1 == db_obj2:
                return True
            else:
                return False
        return None
 
    def allow_syncdb(self, db, model):
        """Make sure that apps only appear in the related database."""
 
        if db in DATABASE_MAPPING.values():
            return DATABASE_MAPPING.get(model._meta.app_label) == db
        elif model._meta.app_label in DATABASE_MAPPING:
            return False
        return None
```

## 使用using方法实现多数据库访问
在查询的语句后面用 `using(dbname)` 来指定要操作的数据库
``` python
# 查询
YourModel.objects.using('db1').all() 
或者 YourModel.objects.using('db2').all()
 
# 保存 或 删除
user_obj.save(using='new_users')
user_obj.delete(using='legacy_users')
```

## 多个数据库联用时数据导入导出
使用的时候和一个数据库的区别是:

**如果不是defalut(默认数据库）要在命令后边加 --database=数据库对应的settings.py中的名称** 

 - 如： --database=db1  或 --database=db2

数据库同步（创建表）
``` python
python manage.py syncdb #同步默认的数据库，和原来的没有区别
 
#同步数据库 db1 (注意：不是数据库名是db1,是settings.py中的那个db1，不过你可以使这两个名称相同，容易使用)
python manage.py syncdb --database=db1
```

数据导出
``` shell
python manage.py dumpdata app1 --database=db1 > app1_fixture.json
python manage.py dumpdata app2 --database=db2 > app2_fixture.json
python manage.py dumpdata auth > auth_fixture.json
```

数据库导入
``` shell
python manage.py loaddata app1_fixture.json --database=db1
python manage.py loaddata app2_fixture.json --database=db2
```

[1]: http://devdocs.io/django~1.8/topics/db/multi-db#topics-db-multi-db-routing
[2]: https://docs.djangoproject.com/en/1.8/ref/settings/#databases
