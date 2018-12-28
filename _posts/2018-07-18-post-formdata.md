---
layout: post
title: Django中POST请求的正确执行方式
filename: post-formdata
datetime: '2018-01-02 14:46'
description: Django中POST请求的正确执行方式
toc: true
tags:
  - POST
categories:
  - Django
---


在Django的HTTP请求中，GET类请求能够安全无误被执行，但是POST请求会收到服务器的403（拒绝访问）错误码。原因实际很简单，Django会对POST请求执行CSRF校验，来保证该次POST请求的安全性。

而Django这样做的目的，主要就是为了防止CSRF，即POST请求的安全性，比如伪造的跨域请求等。

<!--more-->

## CSRF校验失败结果
![403](/images/posts/post-formdata/403.png)


## 为请求添加CSRF校验

### 为form表单添加 csrf_token

在前端页面中form表单有两种形式，一是html代码中的form，另一种则是由js代码动态生成的form表单数据，添加的方法类似

html代码中的form表单

在form行尾部追加 csrf_token 。 当用户点击登陆，发出的POST会自动携带用于csrf校验的token
``` html
    <div class="container">
    
          <form class="form-signin" action="/login/" method='post'>{\% csrf_token \%}
            <h2 class="form-signin-heading">Please sign in</h2>
            <label class="sr-only" for="inputUserName">Email address/UserName</label>
            <input type="text" autofocus="" required="" placeholder="Email address/UserName" class="form-control" id="inputUserName" name="inputUserName">
            <label class="sr-only" for="inputPassword">Password</label>
            <input type="password" required="" placeholder="Password" class="form-control" id="inputPassword">
            <div class="checkbox">
              <label>
                <input type="checkbox" value="remember-me"> Remember me
              </label>
            </div>
            <button type="submit" class="btn btn-lg btn-primary btn-block">Sign in</button>
          </form>
    
        </div> <!-- /container -->
```

![csrftoken_in_post](/images/posts/post-formdata/csrftoken_in_post.png)
如上所示，POST请求体中，csrfmiddlewaretoken 即为 csrf_token 生成的校验token

### js动态生成FormData

构造ForData对象，并添加 csrfmiddlewaretoken 属性
``` javascript
    var formData = new FormData($new_blob);
        formData.append('file', $new_blob);
        formData.append('url', url);
        formData.append('csrfmiddlewaretoken', '{{ csrf_token }}');
        $.ajax({
                url: 'upload',  //server script to process data
                type: 'POST',
                data: formData,
                cache: false,
                contentType: false,
                processData: false
            });
```
## 后台处理

注释掉django自带的强校验

在工程setting文件中的中间件列表中注释掉 django.middleware.csrf.CsrfViewMiddleware

### 后台返回静态模板

后台返回静态模板常用方法有两类，一类是通过 TemplateView.as_view() 方法返回；另一类是通过 render_to_response 方法返回。

这两个方法没对 csrf_token 的自动生成没有感知（不知道是否是我没弄对，两个都试了，没用）。

最终使用 render 方法，实现了在前端页面对 csrf_token 的获取

### 后台返回模板的方法

``` python
    def post_index(request):
        return render(request, 'test.html', response_data)
```
当在浏览器请求页面时，页面自动生成了 csrf_token （隐藏的input标签）
![csrf_token](/images/posts/post-formdata/csrf_token.png)

