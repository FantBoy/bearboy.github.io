---
layout: post
title: Mac下基于iTerm2的终端美化
datetime: 2019-04-021 22:21:32
description: macos下基于iTerm2的终端美化
comments: true
tags:
 - iTerm2
categories:
 - Mac
---

Mac下的终端美化，一直想做，但是没有找到合适的方法，最近才在网上借鉴了SuperDanny的[iTerm 2 && Oh My Zsh【DIY教程——亲身体验过程】](https://www.jianshu.com/p/7de00c73a2bb)美化了一下自己的终端界面，仅以此记录美化过程。

<!--more-->

## 先上一下最终的截图：
> iTerm2的相关配置已经放到了[github](https://github.com/FantBoy/MyLocalConfig/blob/master/zsh/.zshrc)


![](/images/posts/macos_iterm2_style/iterm2_finally.png)

## 美化流程
1. 下载并安装[iTerm2](http://www.iterm2.com/)
2. Mac默认使用dash作为终端，切换至zsh(Mac默认安装zsh)
    ``` bash
    Last login: Sun Apr 21 23:14:42 on ttys000
    (base) bearboyxudembp:~ bear$ echo $SHELL
    /bin/bash
    (base) bearboyxudembp:~ bear$ chsh -s /bin/zsh
    Changing shell for bear.
    Password for bear:
    (base) bearboyxudembp:~ bear$ source ~/.zshrc
    ```

3. 安装`oh-my-zsh`

    `curl -L https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh | sh`,默认将`oh-my-zsh`安装至`~/.oh-my-zsh`

4. 安装`Powerline`

    `pip install powerline-status`

5. 安装字体库
    从[https://github.com/powerline/fonts](https://github.com/powerline/fonts)克隆所有字体，执行`./install.sh`默认将所有字体安装到`/Users/Bear/Library/Fonts`目录下

6. 设置iTerm2的字体
    安装字体库后，将iTerm2的`Regular Font`和`Non-ASCII Font`的字体都设置成 Powerline的字体
    ![](/images/posts/macos_iterm2_style/iTerm2_font.png)

7. 安装并使用agnoster主题
    从[https://github.com/fcamblor/oh-my-zsh-agnoster-fcamblor](https://github.com/fcamblor/oh-my-zsh-agnoster-fcamblor)克隆agnoster主题，执行`./install`会将主题安装到`~/.oh-my-zsh/themes`下

    修改`~/.zshrc`中的`ZSH_THEME`配置项为`agnoster`

8. 安装高亮插件
    ``` bash
    cd ~/.oh-my-zsh/custom/plugins/
    git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
    ```
    
    配置`~/.zshrc`中的`plugins`配置项,**其中zsh-syntax-highlighting必须是最后一项**
    ``` bash
    plugins=(
        git
        zsh-syntax-highlighting
    )
    ```
    在`~/.zshrc`末尾添加`source ~/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh`使插件正式生效

9. 安装代码补全插件
    ``` bash
    cd ~/.oh-my-zsh/custom/plugins/
    git clone https://github.com/zsh-users/zsh-autosuggestions
    ```

    和高亮插件一样，配置`~/.zshrc`中的`plugins`配置项,**其中zsh-syntax-highlighting必须是最后一项**
    ``` bash
    plugins=(
        git
        zsh-autosuggestions
        zsh-syntax-highlighting
    )
    ```
    在`~/.zshrc`末尾添加`source /Users/bear/.oh-my-zsh/custom/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh`使插件正式生效

    > Preferences -> Profiles -> Colors 界面的 ANSI Colors 中Bright的第一个是补全的字体颜色
    
    
    ![](/images/posts/macos_iterm2_style/iTerm2_autosuggestions.png)

10. 设置背景图
     使用SuperDanny提供的图片作为我的背景图。[下载地址](http://wx1.sinaimg.cn/large/81f8a509gy1fnjdvkkwgoj20zk0m8ak8.jpg)

     > 更换背景图片方式：iTerm2 -> Preferences -> Profiles -> Window -> BackGround Image选择图片

## 路径前缀太长，显示缩减路径的方法

在`~/.oh-my-zsh/themes`路径下找到`agnoster.zsh-theme`文件，可使用文本工具打开，将里面的`build_prompt`下的`prompt_context`字段在前面加#注释掉即可