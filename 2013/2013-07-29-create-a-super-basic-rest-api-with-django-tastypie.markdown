# Create a Super Basic REST API with Django Tastypie

Let's set up a RESTful API with [Django Tastypie](http://tastypieapi.org/).

*Updates:*

  - 07/10/2016: Upgraded to the latest versions of Python (v[3.5.1](https://www.python.org/downloads/release/python-351/)), Django (v[1.9.7](https://docs.djangoproject.com/en/1.9/releases/1.9.7/)), and django-tastypie (v[13.3](https://github.com/django-tastypie/django-tastypie/releases/tag/v0.13.3)).

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-tastypie/django-tastypie-logo.png" style="max-width: 100%;" alt="Django Tastypie">
</div>

<br>

## Project set up

> Either follow along below to create your sample Project or clone the repo from [Github](https://github.com/mjhea0/django-tastypie-tutorial).

Create a new project directory, create and activate a virtualenv, install Django and the required dependencies:

```sh
$ mkdir django-tastypie-tutorial
$ cd django-tastypie-tutorial
$ pyvenv-3.5 env
$ source env/bin/activate
$ pip install Django==1.9.7
$ pip install django-tastypie==0.13.3
$ pip install defusedxml==0.4.1
$ pip install lxml==3.6.0
```

Create a basic Django Project and App:

```sh
$ django-admin.py startproject django19
$ cd django19
$ python manage.py startapp whatever
```

Make sure to add the app to your `INSTALLED_APPS` section in *settings.py*:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'whatever',
]
```

Add support for SQLite (or your RDBMS of choice) in *settings.py*:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'test.db'),
    }
}
```

Update your *models.py* file:

```python
from django.db import models


class Whatever(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

Create the migrations:

```sh
$ python manage.py makemigrations
```

Now migrate them:

```sh
$ python manage.py migrate --fake-initial
```

> **Note**: The `fake-initial` optional argument is required if you have to troubleshoot the existing migrations. Omit if no migrations exist.

Fire up the Django Shell and populate the database:

```sh
$ python manage.py shell
>>> from whatever.models import Whatever
>>> w = Whatever(title="What Am I Good At?", body="What am I good at? What is my talent? What makes me stand out? These are the questions we ask ourselves over and over again and somehow can not seem to come up with the perfect answer. This is because we are blinded, we are blinded by our own bias on who we are and what we should be. But discovering the answers to these questions is crucial in branding yourself.")
>>> w.save()
>>>
>>> w = Whatever(title="Charting Best Practices: Proper Data Visualization", body="Charting data and determining business progress is an important part of measuring success. From recording financial statistics to webpage visitor tracking, finding the best practices for charting your data is vastly important for your company’s success. Here is a look at five charting best practices for optimal data visualization and analysis.")
>>> w.save()
>>>
>>> w = Whatever(title="Understand Your Support System Better With Sentiment Analysis", body="There’s more to evaluating success than monitoring your bottom line. While analyzing your support system on a macro level helps to ensure your costs are going down and earnings are rising, taking a micro approach to your business gives you a thorough appreciation of your business’ performance. Sentiment analysis helps you to clearly see whether your business practices are leading to higher customer satisfaction, or if you’re on the verge of running clients away.")
>>> w.save()
```

Exit the shell when done.

## Tastypie set up

Create a new file in your App called *api.py*.

```python
from tastypie.resources import ModelResource
from tastypie.constants import ALL

from whatever.models import Whatever


class WhateverResource(ModelResource):
    class Meta:
        queryset = Whatever.objects.all()
        resource_name = 'whatever'
        filtering = {'title': ALL}
```

Update *urls.py*:

```python
from django.conf.urls import url, include
from django.contrib import admin

from django19.api import WhateverResource

whatever_resource = WhateverResource()

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^api/', include(whatever_resource.urls)),
]
```

## Fire away!

1. Fire up the server.
1. Navigate to [http://localhost:8000/api/whatever/?format=json](http://localhost:8000/api/whatever/?format=json) to get the data in JSON format
1. Navigate to [http://localhost:8000/api/whatever/?format=xml](http://localhost:8000/api/whatever/?format=json) to get the data in XML format

Remember the filter we put on the `WhateverResource` class?

```python
filtering = {'title': ALL}
```

Well, we can filter the objects by title. Try out various keywords:

1. [http://localhost:8000/api/whatever/?format=json&title__contains=what](http://localhost:8000/api/whatever/?format=json&title__contains=what)
1. [http://localhost:8000/api/whatever/?format=json&title__contains=test](http://localhost:8000/api/whatever/?format=json&title__contains=test)

Simple, right!?!

***

There is so much more that can configured with Tastypie. Check out the official [docs](http://tastypieapi.org/) for more info. Comment below if you have questions.

Again, you can download the code from the [repo](https://github.com/mjhea0/django-tastypie-tutorial).