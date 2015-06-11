# Testing in Django (part 2) - Model Mommy vs. Django Testing Fixtures

In the last [post](http://www.realpython.com/blog/python/testing-in-django-part-1-best-practices-and-examples), I introduced you to testing in Django and we looked at best practices as well as a few examples. This time, I'll show you a bit more complicated example and introduce you to [Model Mommy](https://github.com/vandersonmota/model_mommy) for creating sample data.

## Why should you care?

In the last post, I said that, "[factory_boy](https://github.com/rbarrois/factory_boy), [model_mommy](https://github.com/vandersonmota/model_mommy), and [mock](https://pypi.python.org/pypi/mock) are  all are used in place of fixtures or the ORM for populating needed data for testing. Both fixtures and the ORM can be slow and need to be updated whenever your model changes."

To summarize, the Django Testing Fixtures are problematic because they-

- must be updated each time your model/schema changes,
- are really, really slow, and
- sometimes hard-coded data can cause your tests to fail in the future.

So, by using Model Mommy you can create fixtures that load quicker and are much easier to maintain over time.

## Django Testing Fixtures

Let's start by looking at our example for testing the model in the last post:

```python


class WhateverTest(TestCase):

    def create_whatever(self, title="only a test", body="yes, this is only a test"):
        return Whatever.objects.create(title=title, body=body, created_at=timezone.now())

    def test_whatever_creation(self):
        w = self.create_whatever()
        self.assertTrue(isinstance(w, Whatever))
        self.assertEqual(w.__unicode__(), w.title)
```

Here we simply created a `Whatever()` object and asserted that the created title matched the expected title.

If you downloaded the Project from the [repo](https://github.com/mjhea0/testing-in-django), fire up the server and run the tests:

```sh
$ coverage run manage.py test whatever -v 2
```

You will see that the above tests pass:

```sh
test_whatever_creation (whatever.tests.WhateverTest) ... ok
```

Now, instead of having to create a new instance, with each attribute, each time (boring!), we can use Model Mommy to streamline the process.

## Model Mommy

Install:

```sh
$ pip install model_mommy
```

Remember what our model looks like?

```python
class Whatever(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __unicode__(self):
        return self.title
```

Now we can re-write the above test in a much easier fashion:

```python
# model mommy

from model_mommy import mommy

class WhateverTestMommy(TestCase):

    def test_whatever_creation_mommy(self):
        what = mommy.make(Whatever)
        self.assertTrue(isinstance(what, Whatever))
        self.assertEqual(what.__unicode__(), what.title)
```

Run it. Did it pass?

How easy was that? No need to pass in arguments.

## New Model

Let's look at a bit more complicated example.

### Setup

Create a new app:

```sh
$ python manage.py startapp whatevs
```

Add the app to the `Installed_Apps` in the *settings.py* file

Create the model:

```sh
from django.db import models
from django.contrib.auth.models import User
from django.contrib import admin

class Forum(models.Model):
    title = models.CharField(max_length=100)

    def __unicode__(self):
        return self.title

class Post(models.Model):
    title = models.CharField(max_length=100)
    created = models.DateTimeField(auto_now_add=True)
    creator = models.ForeignKey(User, blank=True, null=True)
    forum = models.ForeignKey(Forum)
    body = models.TextField(max_length=10000)

    def __unicode__(self):
        return unicode(self.creator) + " - " + self.title
```

Sync the DB.

What does our coverage report look like?

![cover](https://realpython.com/images/blog_images/model-mommy.png)

### Test

Add the tests:

```python
from model_mommy import mommy
from django.test import TestCase
from whatevs.models import Forum, Thread


class WhateverTestMommy(TestCase):

    def test_forum_creation_mommy(self):
        new_forum = mommy.make('whatevs.Forum')
        new_thread = mommy.make('whatevs.Thread')
        self.assertTrue(isinstance(new_forum, Forum))
        self.assertTrue(isinstance(new_thread, Thread))
        self.assertEqual(new_forum.__unicode__(), new_forum.title)
        self.assertEqual(new_thread.__unicode__(), (str(new_thread.forum) + " - " + str(new_thread.title)))
```

Re-run your tests (which should  pass), then create the coverage report:

```sh
$ coverage run manage.py test whatevs -v 2
$ coverage html
```

## Well?

Care to try running the above tests using JSON fixtures to see just how to set the tests up using the Django Testing Fixtures?

I'm not sure what we'll have in store for the next tutorial, so let me know what you'd like to see. Grab the code [here](https://github.com/mjhea0/testing-in-django). Be sure to comment below if you have questions. Cheers!