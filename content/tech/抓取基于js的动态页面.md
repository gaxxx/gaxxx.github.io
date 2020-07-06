+++
title = "抓取基于js的动态页面"
date = 2014-09-21
[taxonomies]
tags = ["selenum"]
+++

# Overview

最新做一个项目，需要抓取动态网页的内容，积累了一些经验，以下对于网页抓取做一个简单的总结


<!-- more -->

# 系统环境
* OS: ArchLinux@64bits
* Packages: python2-pip 以及基于pip安装的 selenium, beautifulsoup4

# 对于静态内容的获取
对于静态内容的获取相对简单，只需要保证cookie正确就好了


## 无脑直接获取
```
import urllib2
r = urllib2.urlopen("http://www.baidu.com/")
print r.read()
```

## 对于需要登录的静态内容的获取


```
import urllib2,urllib
cookie = urllib2.HTTPCookieProcessor()
opener = urllib2.build_opener(cookie)
## 构造登录form格式的数据
user_pass = [("email","abc@example.com"),("password","lab"),('op','Log in')]
r = opener.open("http://example.com/index.php?",urllib.urlencode(user_pass))
print r.read()
```


## 对于使用OpenID登录的静态内容的获取



```
#有些网站需要openid登录，采用oauth协议的话，我们可以先在chrome上正常登录，然后通过插件"cookie.txt export",将cookie导出成cookie.txt
import urllib2,urllib,cookielib
cj = cookielib.FileCookieJar("./cookie.txt")
cookie = urllib2.HTTPCookieProcessor(cj)
opener = urllib2.build_opener(cookie)
r = opener.open("http://example.com/")
print r.read()
```


# 对于动态生成内容的读取

对于动态生成的内容，主要指通过js动态生成的页面代码,参考知乎意见
* 使用phantomjs & caperjs , 调用js代码，js调用js，无缝结合效果好....
* 采用selenium,调用浏览器进行渲染和内容获取
* 采用scapy框架，集成pyV8进行大数据量抓取 (一淘的做法)


## 使用phantomjs抓取内容

```
#安装phantomjs
#test.js文件
var page = require('webpage').create();
page.onResourceReceived = function(data) {
    console.log("Got a url:" + data.url);
    console.log("Got a type:" + data.contentType);
};
page.open('http://www.baidu.com',function(status){
        phantom.exit()
})

#phantomjs test.js
```

phantomjs 比较有意思的一点，支持网站截图，可以把页面渲染成png
但phantomjs虽好，但是不支持文件下载，有个download_support branch可以尝试一下


## 使用selenium下载数据
使用selenium的好处就是，浏览器能干的事情，selenium脚本也能干。
常见用法就是使用selenium自动调用button，测试页面反馈，但这里我们使用它对某网站进行下载。

```
from selenium import webdriver
import glob
import time
options = webdriver.ChromeOptions()
#新建一个空的chrome配置目录
options.add_argument('--user-data-dir=./chrome')
browser = webdriver.Chrome(chrome_options=options)
#开始下载某个文件，会打开一个chromium浏览器,可以在上面完成，登录，下载插件等操作,然后退出本脚本，重新运行，就可以正常下载内容了
browser.get("http://example.com/test.mp4")
#通过文件后缀判断文件是否下载
while True：
    //文件下载中
    files = glob.glob(os.path.join("xxxx/Downloads","*.crdownload"))
    if len(files) == 0:
        time.sleep(0.1)
    else:
        break

//文件下载完成
while True:
    files = glob.glob(os.path.join("xxxx/Downloads","*.crdownload"))
    if len(files) > 0:
        time.sleep(0.1)
    else:
        break

//文件转移到其他地方
try:
    out = glob.glob(os.path.join("xxxx/Downloads","*.mp4"))[0]
    shuitl.move(out,"/tmp/test.mp4")
except:
    print "failed to get %s" % out
```


## 使用scrapy进行抓取 (TODO)


# 附录： 一碗靓汤
Beautiful Soup是python特别好用的框架之一，可以用来分析静态html的结构，找出相关的内容
常用方式

```
#获取youku首推的视频
from bs4 import BeautifulSoup
import urllib2
r = urllib2.urlopen("http://www.youku.com/")
soup = BeautifulSoup(r.read())
r.close()
#打印首推视频的标题
print soup.find("div","v-thumb").find("img").get("alt")
```


