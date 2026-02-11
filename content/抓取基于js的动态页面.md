+++
title = "Scraping JavaScript-Based Dynamic Pages"
date = 2014-09-21
[taxonomies]
tags = ["selenum"]
+++

Recently worked on a project that required scraping dynamic web page content. Accumulated some experience, here's a brief summary of web scraping techniques.


<!-- more -->

# System Environment
* OS: ArchLinux@64bits
* Packages: python2-pip and pip-installed selenium, beautifulsoup4

# Fetching Static Content
Fetching static content is relatively simple - just ensure cookies are correct.


## Direct Fetch
```
import urllib2
r = urllib2.urlopen("http://www.baidu.com/")
print r.read()
```

## Fetching Static Content That Requires Login


```
import urllib2,urllib
cookie = urllib2.HTTPCookieProcessor()
opener = urllib2.build_opener(cookie)
## Construct login form data
user_pass = [("email","abc@example.com"),("password","lab"),('op','Log in')]
r = opener.open("http://example.com/index.php?",urllib.urlencode(user_pass))
print r.read()
```


## Fetching Static Content with OpenID Login



```
#Some sites require OpenID login. For OAuth protocol, first log in normally in Chrome, then export cookies to cookie.txt using the "cookie.txt export" extension
import urllib2,urllib,cookielib
cj = cookielib.FileCookieJar("./cookie.txt")
cookie = urllib2.HTTPCookieProcessor(cj)
opener = urllib2.build_opener(cookie)
r = opener.open("http://example.com/")
print r.read()
```


# Reading Dynamically Generated Content

For dynamically generated content (mainly pages generated via JavaScript), here are some approaches from Zhihu:
* Use phantomjs & casperjs - call JS code, JS calling JS, seamless integration...
* Use selenium - drive browsers for rendering and content extraction
* Use scrapy framework with pyV8 for large-scale scraping (Etao's approach)


## Scraping with PhantomJS

```
#Install phantomjs
#test.js file
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

PhantomJS has an interesting feature - it supports website screenshots, rendering pages to PNG. However, PhantomJS doesn't support file downloads. There's a download_support branch worth trying.


## Downloading Data with Selenium
The advantage of selenium is that anything a browser can do, a selenium script can do too. Common usage is automating button clicks and testing page responses, but here we use it for downloading.

```
from selenium import webdriver
import glob
import time
options = webdriver.ChromeOptions()
#Create an empty Chrome profile directory
options.add_argument('--user-data-dir=./chrome')
browser = webdriver.Chrome(chrome_options=options)
#Start downloading a file - opens a Chromium browser where you can log in, install extensions, then restart the script for normal downloading
browser.get("http://example.com/test.mp4")
#Check if file is downloaded by file extension
while True：
    //File downloading
    files = glob.glob(os.path.join("xxxx/Downloads","*.crdownload"))
    if len(files) == 0:
        time.sleep(0.1)
    else:
        break

//File download complete
while True:
    files = glob.glob(os.path.join("xxxx/Downloads","*.crdownload"))
    if len(files) > 0:
        time.sleep(0.1)
    else:
        break

//Move file to another location
try:
    out = glob.glob(os.path.join("xxxx/Downloads","*.mp4"))[0]
    shuitl.move(out,"/tmp/test.mp4")
except:
    print "failed to get %s" % out
```


## Scraping with Scrapy (TODO)


# Appendix: Beautiful Soup
Beautiful Soup is one of Python's most useful frameworks for parsing static HTML structure and finding relevant content. Common usage:

```
#Get Youku's featured videos
from bs4 import BeautifulSoup
import urllib2
r = urllib2.urlopen("http://www.youku.com/")
soup = BeautifulSoup(r.read())
r.close()
#Print featured video title
print soup.find("div","v-thumb").find("img").get("alt")
```
