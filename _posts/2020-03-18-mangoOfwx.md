---
title: Flask+IIS+微信小程序开发指南
description: 
 现在微信小程序越来越火，最主要的原因是用户无需下载安装app即可体验产品，产品推广和运营也比较方便，这些优点使得越来越多的企业和个人选择开发微信小程序，所以会写小程序也成为前端比较重要的技能，基于此，我将大学本科期间开发的「芒果虫害识别」APP移植到了『微信小程序』，期间遇到了很多问题，有的问题连续几天都让我一筹莫展，在这里我把「微信小程序」的开发过程和值得吐槽的地方记录下来，供后来人参考。
categories:
 - Tutorial
tags:
---

> 本项目已上传至Github，地址：[https://github.com/QAutumn/mangoOfwechat.git](https://github.com/QAutumn/mangoOfwechat.git)  

***

## 开发环境
> Windows Server 2016  数据中心版 64位
> 
> Python 3.7.6  
> 
> Flask 1.1.1  
> 
> Werkzeug 1.0.0

## 申请小程序账号

登录[『微信小程序官方网站』](https://mp.weixin.qq.com/)，根据指引填写信息和提交相应的资料，就可以拥有自己的小程序帐号。  

登录[「微信小程序管理页面」](https://mp.weixin.qq.com)，通过点击「设置」->『开发设置』即可看到小程序ID，此ID为该小程序的唯一标志符。

## 下载微信开发者工具

通过[「微信小程序官方文档」](https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html)下载微信开发者工具，我的版本为，`macOS 稳定版 Stable Build (1.02.1911180)`。

安装后点击新建项目，设置「项目名称」、『目录』。

**注意，在这里需要将AppID字段设置为刚才在「微信小程序管理页面」看到的AppID，否则小程序将无法发布。**

设置完毕后，点击「新建」便可看到小程序正式的开发界面，这里有几个需要注意的地方： 
 1.  微信小程序开发工具不允许直接将图片等资源直接拖入文件夹，需要打开新建项目时的目录，手动复制进去即可。
 2.  Pages文件夹中包含小程序的所有页面，每个页面包含的`.js`，`.wxml`以及`.wxss`分别代表前端中的`.js`，`.html`以及`.css`。
 3.  整个项目的大小不得超过2M，否则无法发布。 
 4.  小程序所需图片资源可通过Base64转换工具直接生成URL引用，无需包含到本地目录中。
 5.  若需和自己的服务器进行交互，小程序仅支持HTTPS协议，若在开发过程中勾选了「TLS版本以及HTTPS协议证书」，则仅在本地测试时可以通过，但无法发布。 
   
## Flask服务器

>Flask是一个轻量级的可定制框架，使用Python语言编写，较其他同类型框架更为灵活、轻便、安全且容易上手。它可以很好地结合MVC模式进行开发，开发人员分工合作，小型团队在短时间内就可以完成功能丰富的中小型网站或Web服务的实现。另外，Flask还有很强的定制性，用户可以根据自己的需求来添加相应的功能，在保持核心功能简单的同时实现功能的丰富与扩展，其强大的插件库可以让用户实现个性化的网站定制，开发出功能强大的网站。

先贴上服务器端的代码：
```python
from flask import Flask,request
import os
import Mytest as m
app = Flask(__name__)

@app.route('/',methods=['GET'])
def hello():
        return 'hello'

@app.route('/upload_image',methods = ['POST'])
def upload_image():
	print('get')
	print(request)
	print(request.files)
	img = request.files.get('picture')
	img.save('./674.jpg')
	ans=m.model_predict();
	print( ans)
	return ans


if __name__ == '__main__':
   app.run(debug = True,host='0.0.0.0',port=8080,ssl_context=("cert.pem","key.pem"))
```

这里定义了`upload_image()`这个方法用来处理微信小程序端的请求，将通过HTTPS协议上传的图片保存至`./674.jpg`后，调用`model_predict()`这个函数来对图片进行识别，并把结果作为`ans`返回给小程序。

>**这里有一个千万要注意的地方，那就是需要在Flask代码中手动设置SSL证书！！！**

 - 在服务器中安装IIS,步骤为：`「服务器管理器」->「添加角色」->『Web服务器(IIS)』`。
  
 - 众所周知，想要使用HTTPS协议，就必须先申请SSL证书，并将其绑定到服务器，这里我使用腾讯云申请的免费SSL证书，具体申请过程请见我的上一篇博客[「WinServer+jekyll+腾讯云实现自己的个人主页」](http://autumnqq.cn/tutorial/2020/03/18/Tencent-tutorial/)。

 - 打开IIS，在默认网站页面添加绑定，其中80端口绑定HTTP协议，443端口绑定HTTPS协议。(此步骤可省略)，因在上一步flask设置过程中已经将端口设置为8080。

 - **由于腾讯云申请的证书中，IIS文件夹中只有.pfx格式的证书，所以必须将其转为.pem证书，这是最关键的一步，否则小程序端将报错`TLS版本必须小于等于1.2`。本人在这里走了不少弯路，网上教程我没有找到任何一篇明确指出要这么做，纯粹是个人运气摸索出来的。附上.pfx证书转换为.pem格式证书的代码（首先要确保服务器已经安装OPENSSL，在此不再赘述）**  
  ```C
openssl pkcs12 -in certname.pfx -nocerts -out key.pem -nodes //提取公钥
openssl pkcs12 -in certname.pfx -nokeys -out cert.pem //提取私钥
  ```
 - 在服务器上安装`IISCrypto`或`SLLTools`这两个软件（任选其一）。确保只有TLS1.2协议被勾选，并重新启动服务器。
  
 - 运行app.py，如果正常的话，应该可以看到服务器已经在`https://0.0.0.0:8080/`端口上运行了，大功告成！

>本人目前为在校学生，能力一般，水平有限，如果您有任何问题或想法，欢迎随时与我联系。

More Details：[Setting up your GitHub Pages site locally with Jekyll](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/)


