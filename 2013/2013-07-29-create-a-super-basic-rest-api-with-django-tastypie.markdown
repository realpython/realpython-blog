# Create a Super Basic REST API with django-tastypie

One of my clients literally called thirty minutes ago (last Friday) needing a JSON payload based on a GET response from the data model. I installed [django-tastypie](http://tastypieapi.org/) and thirty minutes later had the project completed. Although this example is overly simplified, it's not far off from my real-world implementation.

> **Note:** Although this tutorial uses Django 1.5, the basic concepts will work on most versions greater than 1.3.

## Setup

> Either follow along below to create your sample Project or clone the repo from [Github](https://github.com/mjhea0/django-tastypie-tutorial).

Create a new directory, setup and activate virtualenv, install Django and the required dependencies:

```sh
$ mkdir django-tastypie-tutorial
$ cd django-tastypie-tutorial
$ virtualenv --no-site-packages env
$ source env/bin/activate
$ pip install Django==1.5
$ pip install django-tastypie==0.9.15
$ pip install defusedxml==0.4.1
$ pip install lxml==3.2.1
```

Create a basic Django Project and App:

```sh
$ django-admin.py startproject django15
$ cd django15
$ python manage.py startapp whatever
```

> Make sure to add the app to your `INSTALLED_APPS` section in *settings.py*.

Add support for SQLite (or your RDMS of choice) in *settings.py*:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': 'test.db',
    }
}
```

Enable the Django Admin, then update your *models.py* file:

```python
from django.db import models

class Whatever(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __unicode__(self):
        return self.title
```

Sync the DB:

```sh
$ python manage.py syncdb
```

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

## Setup Tasypie

Create a new file in your Project called *api.py*, then sync the DB again.

```python
from tastypie.resources import ModelResource
from tastypie.constants import ALL
from models import Whatever

class WhateverResource(ModelResource):
    class Meta:
        queryset = Whatever.objects.all()
        resource_name = 'whatever'
        filtering = { "title" : ALL }
```

Update *urls.py*:

```python
from django.conf.urls import patterns, include, url
from api import WhateverResource

whatever_resource = WhateverResource()

urlpatterns = patterns('',
   url(r'^api/', include(whatever_resource.urls)),
)
```

## Fire Away!

1. Fire up the server.
2. Navigate to [http://localhost:8000/api/whatever/?format=json](http://localhost:8000/api/whatever/?format=json) to get the data in JSON format
3. Navigate to [http://localhost:8000/api/whatever/?format=xml](http://localhost:8000/api/whatever/?format=json) to get the data in XML format

Remember the filer we put on the `WhateverResource` class?

```python
filtering = { "title" : ALL }
```

Well, we can filter the objects by title. Try out various keywords:

1. [http://localhost:8000/api/whatever/?format=xml&title__contains=what](http://localhost:8000/api/whatever/?format=xml&title__contains=what)
1. [http://localhost:8000/api/whatever/?format=xml&title__contains=test](http://localhost:8000/api/whatever/?format=xml&title__contains=test)

Simple, right!?

***

There is so much more that can configured with Tastypie. Check out the official [docs](http://tastypieapi.org/) if you need more info. Comment below if you have questions.

Again, you can download the code [here](https://github.com/mjhea0/django-tastypie-tutorial).