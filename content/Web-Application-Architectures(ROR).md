+++
title = "Web Application Architectures (ROR)"
date = 2014-08-23
[taxonomies]
tags = ["ruby", "rails", "ror"]
+++

Recently completed a course on [Web Application Architectures](https://class.coursera.org/webapplications-002). I originally planned to borrow some architectural ideas and build a simple CMS application with Go, but after finishing the course, I found that ROR development efficiency is quite good and it auto-generates a lot of code.

<!-- more -->

Even compared to Django, which is known for rapid development, ROR requires less coding for the same tasks. However, some argue that Django's architecture is easier to control. See this interesting blog post: [RAILS VS DJANGO: AN IN-DEPTH TECHNICAL COMPARISON](https://bernardopires.com/2014/03/rails-vs-django-an-in-depth-technical-comparison/).


## Installation
On Archlinux, install ruby directly:

```
root# pacman -S ruby
root# gem install rails
```

On other systems including Ubuntu, the Ruby version is too old... See [rvm](http://rvm.io/rvm/install) for installation. Related videos can be found on [railscast](http://railscasts.com/episodes/310-getting-started-with-rails).

## Creating an Application
After installing Rails, you can create an application directly:

```
root# rails new blog
```

The application automatically creates the directory structure:
* config - Configuration directory, mainly database and router config
* db - Database schema
* public - 404 files etc.
* app - MVC-related files:
    * assets - JS and CSS files
    * models (M in MVC)
    * views (V in MVC)
    * controller (C in MVC)


## Adding Database Support
Rails uses SQLite by default but can also use MySQL or even MongoDB. Using MySQL as an example:

* Install MySQL (Archlinux example):

```
root# pacman -S  mysql ; gem instamysql2
```

* Edit config/database.yml (note the <<: \*default code reuse in the default config):

```
development:
    adapter: mysql2
    #database name
    database: db_name_dev
    #mysql username
    username: woo
    #mysql password
    password: aabb
```

* Add post and comment data structures:

```
root# rails generate scaffold post title:string body:text
root# rails generate scaffold comment post_id:integer body:text
root# rake db:migrate
```

This creates the MVC and test cases for post and comment, and also generates RESTful-style interfaces. You can view supported RESTful URLs with rake routes, start the server (default port 3000), and visit http://127.0.0.1:3000/posts/

```
root# rails s
```

## Modifying UI Logic

* Add association between post and comment - too many commands, using [asciinema](https://asciinema.org) directly:
<script type="text/javascript" src="https://asciinema.org/a/11671.js" id="asciicast-11671" async></script>
* Modify router and view to show comments on post page:
<script type="text/javascript" src="https://asciinema.org/a/11672.js" id="asciicast-11672" async></script>
* Modify controller to redirect to post page after submitting a comment:
<script type="text/javascript" src="https://asciinema.org/a/11674.js" id="asciicast-11674" async></script>



## Complete Example
See [Bitbucket](https://bitbucket.org/gaxxx/blog_ror)
