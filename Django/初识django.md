### 初识Django

##### 设计模型

使用 <u>数据-模型语句</u> 来描述你的数据模型。

```python
# mysite/news/models.py
from django.db import models

class Reporter(models.Model):
    full_name = models.CharField(max_length=70)

    def __str__(self):
        return self.full_name

class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)

    def __str__(self):
        return self.headline
```



##### 应用数据模型

运行 Django 命令行实用程序以自动创建数据库表：

```
python manage.py makemigrations
python manage.py migrate
```

该 [`makemigrations`](https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/#django-admin-makemigrations) 命令查找所有可用的模型，为任意一个在数据库中不存在对应数据表的模型创建迁移脚本文件。[`migrate`](https://docs.djangoproject.com/zh-hans/3.1/ref/django-admin/#django-admin-migrate) 命令则运行这些迁移来自动创建数据库表。



##### 享用便捷的API

使用一套便捷而丰富的 [Python API](https://docs.djangoproject.com/zh-hans/3.1/topics/db/queries/) 访问你的数据。API 是动态创建的，不需要代码生成：

```python
# Import the models we created from our "news" app
>>> from news.models import Article, Reporter

# No reporters are in the system yet.
>>> Reporter.objects.all()
<QuerySet []>

# Create a new Reporter.
>>> r = Reporter(full_name='John Smith')

# Save the object into the database. You have to call save() explicitly.
>>> r.save()

# Now it has an ID.
>>> r.id
1

# Now the new reporter is in the database.
>>> Reporter.objects.all()
<QuerySet [<Reporter: John Smith>]>

# Fields are represented as attributes on the Python object.
>>> r.full_name
'John Smith'

# Django provides a rich database lookup API.
>>> Reporter.objects.get(id=1)
<Reporter: John Smith>
>>> Reporter.objects.get(full_name__startswith='John')
<Reporter: John Smith>
>>> Reporter.objects.get(full_name__contains='mith')
<Reporter: John Smith>
>>> Reporter.objects.get(id=2)
Traceback (most recent call last):
    ...
DoesNotExist: Reporter matching query does not exist.

# Create an article.
>>> from datetime import date
>>> a = Article(pub_date=date.today(), headline='Django is cool',
...     content='Yeah.', reporter=r)
>>> a.save()

# Now the article is in the database.
>>> Article.objects.all()
<QuerySet [<Article: Django is cool>]>

# Article objects get API access to related Reporter objects.
>>> r = a.reporter
>>> r.full_name
'John Smith'

# And vice versa: Reporter objects get API access to Article objects.
>>> r.article_set.all()
<QuerySet [<Article: Django is cool>]>

# The API follows relationships as far as you need, performing efficient
# JOINs for you behind the scenes.
# This finds all articles by a reporter whose name starts with "John".
>>> Article.objects.filter(reporter__full_name__startswith='John')
<QuerySet [<Article: Django is cool>]>

# Change an object by altering its attributes and calling save().
>>> r.full_name = 'Billy Goat'
>>> r.save()

# Delete an object with delete().
>>> r.delete()
```



##### 动态管理接口

当你的模型完成定义，Django 就会自动生成一个专业的生产级 [管理接口](https://docs.djangoproject.com/zh-hans/3.1/ref/contrib/admin/) ——一个允许认证用户添加、更改和删除对象的 Web 站点。你只需在管理站点上注册你的模型即可：

```python
# mysite/news/models.py
from django.db import models

class Article(models.Model):
    pub_date = models.DateField()
    headline = models.CharField(max_length=200)
    content = models.TextField()
    reporter = models.ForeignKey(Reporter, on_delete=models.CASCADE)
```

```python
# mysite/news/admin.py
from django.contrib import admin

from . import models

admin.site.register(models.Article)
```

创建 Django 应用的典型流程是，先建立数据模型，然后搭建管理站点，之后你的员工（或者客户）就可以向网站里填充数据了。



##### 规划URLs

为了设计你自己的 [URLconf](https://docs.djangoproject.com/zh-hans/3.1/topics/http/urls/) ，你需要创建一个叫做 URLconf 的 Python 模块。这是网站的目录，它包含了一张 URL 和 Python 回调函数之间的映射表。URLconf 也有利于将 Python 代码与 URL 进行解耦。

下面这个 URLconf 适用于前面 `Reporter`/`Article` 的例子：

```python
# mysite/news/urls.py
from django.urls import path

from . import views

urlpatterns = [
    path('articles/<int:year>/', views.year_archive),
    path('articles/<int:year>/<int:month>/', views.month_archive),
    path('articles/<int:year>/<int:month>/<int:pk>/', views.article_detail),
]
```

上述代码将 URL 路径映射到了 Python 回调函数（“视图”）。路径字符串使用参数标签从URL中“捕获”相应值。当用户请求页面时，Django 依次遍历路径，直至初次匹配到了请求的 URL。(如果无匹配项，Django 会调用 404 视图。) 这个过程非常快，因为路径在加载时就编译成了正则表达式。

一旦有 URL 路径匹配成功，Django 会调用相应的视图函数。每个视图函数会接受一个请求对象——包含请求元信息——以及在匹配式中获取的参数值。

例如，当用户请求了这样的 URL "/articles/2005/05/39323/"，Django 会调用 `news.views.article_detail(request, year=2005, month=5, pk=39323)`。



##### 编写视图

视图函数的执行结果只可能有两种：返回一个包含请求页面元素的 [`HttpResponse`](https://docs.djangoproject.com/zh-hans/3.1/ref/request-response/#django.http.HttpResponse) 对象，或者是抛出 [`Http404`](https://docs.djangoproject.com/zh-hans/3.1/topics/http/views/#django.http.Http404) 这类异常。至于执行过程中的其它的动作则由你决定。

通常来说，一个视图的工作就是：从参数获取数据，装载一个模板，然后将根据获取的数据对模板进行渲染。下面是一个 `year_archive` 的视图样例：

```python
# mysite/news/views.py
from django.shortcuts import render

from .models import Article

def year_archive(request, year):
    a_list = Article.objects.filter(pub_date__year=year)
    context = {'year': year, 'article_list': a_list}
    return render(request, 'news/year_archive.html', context)
```



##### 设计模版

上面的代码加载了 `news/year_archive.html` 模板。

Django 允许设置搜索模板路径，这样可以最小化模板之间的冗余。在 Django 设置中，你可以通过 [`DIRS`](https://docs.djangoproject.com/zh-hans/3.1/ref/settings/#std:setting-TEMPLATES-DIRS) 参数指定一个路径列表用于检索模板。如果第一个路径中不包含任何模板，就继续检查第二个，以此类推。

让我们假设 `news/year_archive.html` 模板已经找到。它看起来可能是下面这个样子：

```html
<!--mysite/news/templates/news/year_archive.html-->
{% extends "base.html" %}

{% block title %}Articles for {{ year }}{% endblock %}

{% block content %}
<h1>Articles for {{ year }}</h1>

{% for article in article_list %}
    <p>{{ article.headline }}</p>
    <p>By {{ article.reporter.full_name }}</p>
    <p>Published {{ article.pub_date|date:"F j, Y" }}</p>
{% endfor %}
{% endblock %}
```

我们看到变量都被双大括号括起来了。 `{{ article.headline }}` 的意思是“输出 article 的 headline 属性值”。这个“点”还有更多的用途，比如查找字典键值、查找索引和函数调用。

我们注意到 `{{ article.pub_date|date:"F j, Y" }}` 使用了 Unix 风格的“管道符”（“|”字符）。这是一个模板过滤器，用于过滤变量值。在这里过滤器将一个 Python datetime 对象转化为指定的格式（就像 PHP 中的日期函数那样）。

你可以将多个过滤器连在一起使用。你还可以使用你 [自定义的模板过滤器](https://docs.djangoproject.com/zh-hans/3.1/howto/custom-template-tags/#howto-writing-custom-template-filters) 。你甚至可以自己编写 [自定义的模板标签](https://docs.djangoproject.com/zh-hans/3.1/howto/custom-template-tags/) ，相关的 Python 代码会在使用标签时在后台运行。

Django 使用了 ''模板继承'' 的概念。这就是 `{% extends "base.html" %}` 的作用。它的含义是''先加载名为 base 的模板，并且用下面的标记块对模板中定义的标记块进行填充''。简而言之，模板继承可以使模板间的冗余内容最小化：每个模板只需包含与其它文档有区别的内容。

下面是 base.html 可能的样子，它使用了 [静态文件](https://docs.djangoproject.com/zh-hans/3.1/howto/static-files/) ：

```html
<!--mysite/templates/base.html-->
{% load static %}
<html>
<head>
    <title>{% block title %}{% endblock %}</title>
</head>
<body>
    <img src="{% static 'images/sitelogo.png' %}" alt="Logo">
    {% block content %}{% endblock %}
</body>
</html>
```

简而言之，它定义了这个网站的外观（利用网站的 logo），并且给子模板们挖好了可以填的”坑“。这就意味着你可以通过修改基础模板以达到重新设计网页的目的。

