#一、apache 部署 django admin css消失问题
##问题描述
根据**《The Django Book 2rd》**开发出的网站在*django*自己的*web server*上跑时，*admin*管理后台没有问题，但是使用*Apache*部署的时候，再打开*admin*却很奇怪地发现没有样式。
习惯性地打开chrome的调试选项，会发现css文件都是`file not found`，这说明文件路径配置不对，那应该怎么配置呢？
##查资料得知django中相关配置项

####开发环境和发布环境
 注意区分两个环境，不同环境下对*static*文件的配置是不同的。
 
###使用 staticfiles
在 Django 1.7 中，*staticfiles* 默认是启用的。
如果没启用，可以使用下面的方法开启：
*INSTALLED_APPS += ('django.contrib.staticfiles',)*
####*STATICFILES_FINDERS *
**开发环境** 下 *staticfiles* 查找静态资源的顺序取决于 *STATICFILES_FINDERS *配置项，*STATICFILES_FINDERS* 缺省设置如下：

`(
"django.contrib.staticfiles.finders.FileSystemFinder",
 "django.contrib.staticfiles.finders.AppDirectoriesFinder"
 ) `
`django.contrib.staticfiles.finders.FileSystemFinder `用来从*STATICFILES_DIRS * 指定的路径中查找额外的静态文件；
`django.contrib.staticfiles.finders.AppDirectoriesFinder` 是从 *INSTALLED_APPS* 列表内的 *APP *所在包的*static* 目录中查找资源文件。
默认情况下（如果没有修改*STATICFILES_FINDERS*的话），*Django*首先会在*STATICFILES_DIRS*配置的文件夹中寻找静态文件，然后再从每个app的static子目录下查找，并且返回找到的第一个文件。所以我们可以将全局的静态文件放在*STATICFILES_DIRS*配置的目录中，将app独有的静态文件放在app的static子目录中。存放的时候按类别存放在static目录的子目录下，如图片都放在images文件夹中，所有的CSS都放在css文件夹中，所有的js文件都放在js文件夹中。
**配置全局的静态文件目录**
默认情况下，*STATICFILES_DIRS*为空，我们可以这样配置：
`
import os.path
STATICFILES_DIRS = (
    # Put strings here, like "/home/html/static" or "C:/www/django/static".
    # Always use forward slashes, even on Windows.
    # Don't forget to use absolute paths, not relative paths.
    os.path.join(os.path.dirname(__file__), 'static').replace('\\','/'),
)`
**配置app的静态文件目录**
直接在每个*app*目录下创建*static*目录，将文件直接放进去就可以了。如果*Django*在*STATICFILES_DIRS*当中没有找到的话，会到app的目录下的static目录去查找的。

####*STATIC_ROOT *配置项
当**工程部署**之后，处理静态文件较好的方法是交给**Web容器**来做，比如**Apache**，所以我们要将所有的静态文件放在一起，然后交给*Apache*托管，*Django*提供了一个命令`collectstatic`用来把所有的静态文件收集到*STATIC_ROOT*当中。所以有一个问题就是，**所有的静态文件，最好不要有重名**。

*STATIC_ROOT *被用来指定执行 `manage.py collectstatic `时静态文件存放的路径。在生产环境下，集中存放静态资源有用利于使用 *Lighttpd / Nginx *托管静态资源。为了方便调试，通常设置如下：
`
SITE_ROOT = os.path.dirname(os.path.abspath(__file__))
SITE_ROOT = os.path.abspath(os.path.join(SITE_ROOT, '../'))
STATIC_ROOT = os.path.join(SITE_ROOT, 'collectedstatic')
`
####*STATIC_URL* 配置项

在 *Django* 模板中，我们可以引用` {{ STATIC_URL }}` 变量避免把路径写死。通常设置如下：
`
STATIC_URL = '/static/'
`
#####在模板中使用 STATIC_URL

`<link rel="stylesheet" href="{{ STATIC_URL }}css/core.css">`

#####开发环境访问静态资源

设置 `urls.py`

`from django.contrib.staticfiles.urls import staticfiles_urlpatterns 
 urlpatterns += staticfiles_urlpatterns()
`
注意：无须对上面的代码使用` if settings.DEBUG: `判断，`staticfiles_urlpatterns() `函数会自己处理。

#####发布环境访问静态资源

首先应该执行 `manage.py collectstatic `命令把所有*APP* （包括 `django.contrib.admin`）的静态资源收集到 *STATIC_ROOT *指定的目录

接着，我们需要配置*lighttpd/nginx/Apache* **服务器**，将 `/static/ `路径指向 *STATIC_ROOT* 目录，提供静态资源的服务。详细细节请参考：[Serving static files in production](https://docs.djangoproject.com/en/1.7/howto/static-files/#serving-static-files-in-productio)

####`findstatic`命令
另外，Django提供了一个`findstatic`命令来查找指定的静态文件所在的目录，例如：`
D:\TestDjango>python manage.py findstatic Chrome.jpg
('D:/TestDjango/TestDjango/templates',)
Found 'Chrome.jpg' here:
D:\TestDjango\TestDjango\static\Chrome.jpg
`
##问题的解决
###环境：win7+django1.7.1+python2.7.6+apache2.4
###步骤
- 设置`settings.py`中的*STATICFILES_FINDERS*、*STATIC_ROOT*
`STATICFILES_FINDERS = (
    'django.contrib.staticfiles.finders.FileSystemFinder',
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
)
STATIC_ROOT = "D:/mycode/Python/MyWorks/Django/The-Django-Book/mysite/mysite/static"
STATIC_URL = '/static/'
`
- 执行`manage.py collectstatic `命令，将static文件收集到项目文件夹下
- 修改`urls.py`
`
import settings
`
在`urlpatterns`中添加：
` url(r'^static/(?P<path>.*)$', 'django.views.static.serve', {'document_root': settings.STATIC_ROOT}),
 `
#####其他
其实有尝试修改Apache的配置文件`httpd.conf`：

1.
\#-----------------------------将样式路径指向django默认的样式路径----------------------
	`Alias /static/admin D:/Python27/Lib/site-packages/django-1.7.1-py2.7.egg/django/contrib/admin/static/admin
	<Directory "D:/Python27/Lib/site-packages/django-1.7.1-py2.7.egg/django/contrib/admin/static/admin"> 
        AllowOverride None 
        Options None 
        Order allow,deny 
        Allow from all 
    </Directory> 
    <Location "/static/">
        SetHandler None 
    </Location> 
    <LocationMatch "\.(jpg|gif|png|txt|ico|pdf|css|jpeg)$"> 
        SetHandler None 
    </LocationMatch>
</IfModule>`
2.
\#------------static location without cgi but itself--------------
`<Location "/static/">
  SetHandler None
</Location>`

发现并无效果，最后修改django配置文件最后才解决了。
不过根据上面的一些配置知识可知，这样的修改是不够完美的，如果是为了以后的大项目做准备，应该遵循上面提到的**开发环境和发布环境分开配置**的原则。
####参考
[ref1](http://m.blog.csdn.net/blog/a657941877/8953233)
[ref2](http://blog.yangyubo.com/2012/07/26/django-staticfiles/)
