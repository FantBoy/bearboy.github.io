---
layout: post
filename: python-configuration
title: python之配置文件
datetime: 2018-10-16 00:14:32
description: python之配置文件
comments: true
tags:
 - Python
categories:
 - Python
 
---

在项目开发过程中，经常会编写配置文件。python项目也一样，而对于python项目，常见的配置文件以`*.py`文件格式，包含python代码的单独配置文件出现，如Django的`settings.py`，。其实，除此之外python还可以支持其他一些主流的配置文件格式:

- `*config.py` for Python files
- `*.yaml` or `*.yml` if the configuration is done in YAML format
- `*.json` for configuration files written in JSON format
- `*.cfg` or `*.conf` to indicate that it is a configuration file
- `*.ini` for "initialization" are quite widespread (see [Wiki](https://en.wikipedia.org/wiki/INI_file))



##  *.py 配置文件

### 配置文件样例 

`databaseconfig.py`

```
#!/usr/bin/env python
import preprocessing
mysql = {'host': 'localhost',
         'user': 'root',
         'passwd': 'my secret password',
         'db': 'write-math'}
preprocessing_queue = [preprocessing.scale_and_center,
                       preprocessing.dot_reduction,
                       preprocessing.connect_lines]
use_anonymous = True
```

### 使用方法

```
#!/usr/bin/env python
import databaseconfig as cfg
connect(cfg.mysql['host'], cfg.mysql['user'], cfg.mysql['password'])
```

##  *.json 配置文件

在某些场景，通过json格式的配置文件，可以更方便的一次性加载所有的相关配置

### 配置文件样例
`config.json`

```
{
    "mysql":{
        "host":"localhost",
        "user":"root",
        "passwd":"my secret password",
        "db":"write-math"
    },
    "other":{
        "preprocessing_queue":[
            "preprocessing.scale_and_center",
            "preprocessing.dot_reduction",
            "preprocessing.connect_lines"
            ],
        "use_anonymous":true
    }
}
```

### 使用方法

```
import json

with open('config.json') as json_data_file:
    data = json.load(json_data_file)
print(data)
```

### 配置信息数据生成

```
import json
with open('config.json', 'w') as outfile:
    json.dump(data, outfile)
```



## *.yaml 配置文件

> YAML (rhymes with camel) is a human-readable data serialization format that takes concepts from programming languages such as C, Perl, and Python, and ideas from XML and the data format of electronic mail (RFC 2822). YAML was first proposed by Clark Evans in 2001, who designed it together with Ingy döt Net and Oren Ben-Kiki. It is available for several programming languages.



### 配置文件样例

`config.yml`

```
mysql:
    host: localhost
    user: root
    passwd: my secret password
    db: write-math
other:
    preprocessing_queue:
        - preprocessing.scale_and_center
        - preprocessing.dot_reduction
        - preprocessing.connect_lines
    use_anonymous: yes
```

### 使用方法

```
import yaml

with open("config.yml", 'r') as ymlfile:
    cfg = yaml.load(ymlfile)

for section in cfg:
    print(section)
print(cfg['mysql'])
print(cfg['other'])
```

#### output

```
other
mysql
{'passwd': 'my secret password', 'host': 'localhost', 'db': 'write-math', 'user': 'root'}
{'preprocessing_queue': ['preprocessing.scale_and_center', 'preprocessing.dot_reduction', 'preprocessing.connect_lines'], 'use_anonymous': True}
```



**yaml提供了 `yaml.dump` 方法，可用于生成*.yaml配置文件**



## *.ini 配置文件

### 配置文件样例

`config.ini`

```
[mysql]
host=localhost
user=root
passwd=my secret password
db=write-math

[other]
preprocessing_queue=["preprocessing.scale_and_center",
                     "preprocessing.dot_reduction",
                     "preprocessing.connect_lines"]
use_anonymous=yes
```

### 调用方法

```
#!/usr/bin/env python

import ConfigParser
import io

# Load the configuration file
with open("config.ini") as f:
    sample_config = f.read()
config = ConfigParser.RawConfigParser(allow_no_value=True)
config.readfp(io.BytesIO(sample_config))

# List all contents
print("List all contents")
for section in config.sections():
    print("Section: %s" % section)
    for options in config.options(section):
        print("x %s:::%s:::%s" % (options,
                                  config.get(section, options),
                                  str(type(options))))

# Print some contents
print("\nPrint some contents")
print(config.get('other', 'use_anonymous'))  # Just get the value
print(config.getboolean('other', 'use_anonymous'))  # You know the datatype?
```

#### output

```
List all contents
Section: mysql
x host:::localhost:::<type 'str'>
x user:::root:::<type 'str'>
x passwd:::my secret password:::<type 'str'>
x db:::write-math:::<type 'str'>
Section: other
x preprocessing_queue:::["preprocessing.scale_and_center",
"preprocessing.dot_reduction",
"preprocessing.connect_lines"]:::<type 'str'>
x use_anonymous:::yes:::<type 'str'>

Print some contents
yes
True
```

### 配置信息数据生成

`ConfigParser`同样提供了`*.ini`配置文件的生成方法

```
import os
configfile_name = "config.ini"

# Check if there is already a configurtion file
if not os.path.isfile(configfile_name):
    # Create the configuration file as it doesn't exist yet
    cfgfile = open(configfile_name, 'w')

    # Add content to the file
    Config = ConfigParser.ConfigParser()
    Config.add_section('mysql')
    Config.set('mysql', 'host', 'localhost')
    Config.set('mysql', 'user', 'root')
    Config.set('mysql', 'passwd', 'my secret password')
    Config.set('mysql', 'db', 'write-math')
    Config.add_section('other')
    Config.set('other',
               'preprocessing_queue',
               ['preprocessing.scale_and_center',
                'preprocessing.dot_reduction',
                'preprocessing.connect_lines'])
    Config.set('other', 'use_anonymous', True)
    Config.write(cfgfile)
    cfgfile.close()
```



## *.xml 配置文件

`*.xml`文件读取可以使用`BeautifulSoup `来完成

### 配置文件样例

```
<config>
    <mysql>
        <host>localhost</host>
        <user>root</user>
        <passwd>my secret password</passwd>
        <db>write-math</db>
    </mysql>
    <other>
        <preprocessing_queue>
            <li>preprocessing.scale_and_center</li>
            <li>preprocessing.dot_reduction</li>
            <li>preprocessing.connect_lines</li>
        </preprocessing_queue>
        <use_anonymous value="true" />
    </other>
</config>
```

### 读取方法

```
from BeautifulSoup import BeautifulSoup

with open("config.xml") as f:
    content = f.read()

y = BeautifulSoup(content)
print(y.mysql.host.contents[0])
for tag in y.other.preprocessing_queue:
    print(tag)
```












