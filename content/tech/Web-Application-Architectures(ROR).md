+++
title = "Web Application Architectures(ROR)"
date = 2014-08-23
[taxonomies]
tags = ["ruby", "rails", "ror"]

+++

# Overview
最近学了一门[Web Application Architectures](https://class.coursera.org/webapplications-002),本来是打算借鉴一下其中的架构，然后用golang做一个简单CMS类应用实现的，但是学完发现，ROR用来开发的效率还不错，而且能自动生成不少代码。

<!-- more -->

即使相对于号称快捷开发的django,同样的任务，ROR的编码量还要更少一些,但有些人认为，django的架构，要更容易控制一些，参考很有意思的一篇blog, [RAILS VS DJANGO: AN IN-DEPTH TECHNICAL COMPARISON](https://bernardopires.com/2014/03/rails-vs-django-an-in-depth-technical-comparison/).


## 安装
在Archlinux下，直接安装ruby

```
root# pacman -S ruby
root# gem install rails
```

在其他操作系统上，包括ubuntu，ruby的版本太老了....可以参考[rvm](http://rvm.io/rvm/install),可以在[railscast](http://railscasts.com/episodes/310-getting-started-with-rails)上找到相关视频

## 创建应用
安装完rails之后，可以直接创建应用了

```
root# rails new blog
```

应用自动建立了相关目录，主要有
* config 配置目录，主要有数据库的配置，router的配置    
* db 数据库的shema
* public 404文件等
* app MVC相关的东西，其中
    * assets js和css文件
    * models (MVC 中的M)
    * views (MVC 中的V)
    * controller  (MVC中的C)


## 添加数据库支持
rails默认使用sqlite，也可以使用mysql,甚至mongodb,以mysql为例
首先安装mysql以及相应的gem

* 安装mysql(以Archlinux为例)


```
root# pacman -S  mysql ; gem instamysql2
```

* 编辑config/database.yml  (注意看默认配置文件的 <<: \*default 的代码复用)


```
development:
    adapter: mysql2
    #使用的数据库
    database: db_name_dev
    #mysql用户名
    username: woo
    #mysql密码
    password: aabb
```
 
* 增加post和comment数据结构


```
root# rails generate scaffold post title:string body:text
root# rails generate scaffold comment post_id:integer body:text
root# rake db:migrate
```

这样就增加了post和comment的MVC以及相关的测试用例，更有趣的是，增加了post和comment的restful风格的界面
同时可以通过rake routes查看目前支持的restful的url,同时可以启动ror(默认使用3000端口),以及访问http://127.0.0.1:3000/posts/ 

```
root# rails s 
```

## 修改界面逻辑

* 增加post与comment之间的关联，命令太多了，直接上[asciinema](https://asciinema.org)
<script type="text/javascript" src="https://asciinema.org/a/11671.js" id="asciicast-11671" async></script>
* 修改相关的router和view，在post页面展现comments,继续上asciinema
<script type="text/javascript" src="https://asciinema.org/a/11672.js" id="asciicast-11672" async></script>
* 修改controller，每次提交comment之后重定向到post页面,修改comment的create函数
<script type="text/javascript" src="https://asciinema.org/a/11674.js" id="asciicast-11674" async></script>



## 完整的示例
见[Bitbucket](https://bitbucket.org/gaxxx/blog_ror)
