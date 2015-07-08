---
layout: post
title: "Asynchronous Tasks with Django and Celery"
date: 2015-07-08 07:55:26 -0600
toc: true
comments: true
category_side_bar: true
categories: [python, django]

keywords: "python, web development, django, celery, celery tasks, asynchronous tasks, redis, task queue"
description: "This tutorial shows how to integrate Celery and Django and create Periodic Tasks"
---

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-and-celery/django-celery.png" style="max-width: 100%;" alt="Django and Celery">
</div>

<br>

When I was new to Django, one of the most frustrating things I experienced was the need to run a bit of code periodically. I wrote a nice function that performed an action that needed to run daily at 12am. Easy, right? Wrong. This turned out to be a huge problem to me since at the time I was used to "[Cpanel](http://en.wikipedia.org/wiki/Web_hosting_control_panel)-type" web hosting where there was a nice handy GUI for setting up cron jobs for this very purpose.

After much research, I found a nice solution - [Celery](http://www.celeryproject.org/), a powerful asynchronous job queue used for running tasks in the background. But this led to additional problems, since I couldn't find an easy set of instructions to integrate Celery into a Django Project.

Of course I eventually did manage to figure it - which is what this article will cover: **How to integrate Celery into a Django Project and create Periodic Tasks.**

> This project utilizes Python [3.4](https://www.python.org/download/releases/3.4.0/), Django [1.8.2](https://docs.djangoproject.com/en/1.8/releases/1.8.2/), Celery [3.1.18](http://celery.readthedocs.org/en/latest/changelog.html), and Redis [3.0.2](https://raw.githubusercontent.com/antirez/redis/3.0/00-RELEASENOTES).

## Overview

For your convenience, since this is such a large post, please refer back to this table for brief info on each step and to grab the associated code.

<table style="
    font-size: 16px;border-spacing: 10px 0px;border-collapse: separate;">
<thead>
<tr>
<th></th>
<th> Step              </th>
<th> Overview                      </th>
<th> Git Tag                                                   </th>
</tr>
</thead>
<tbody>
<tr>
<td></td>
<td> Boilerplate       </td>
<td> Download boilerplate          </td>
<td> <a href="https://github.com/realpython/Picha/releases/tag/v1">v1</a></td>
</tr>
<tr>
<td></td>
<td> Setup             </td>
<td> Integrate Celery with Django  </td>
<td> <a href="https://github.com/realpython/Picha/releases/tag/v2">v2</a></td>
</tr>
<tr>
<td></td>
<td> Celery Tasks      </td>
<td> Add basic Celery Task         </td>
<td> <a href="https://github.com/realpython/Picha/releases/tag/v3">v3</a></td>
</tr>
<tr>
<td></td>
<td> Periodic Tasks    </td>
<td> Add Periodic Task             </td>
<td> <a href="https://github.com/realpython/Picha/releases/tag/v4">v4</a></td>
</tr>
<tr>
<td></td>
<td> Running Locally   </td>
<td> Run our app locally           </td>
<td> <a href="https://github.com/realpython/Picha/releases/tag/v5">v5</a></td>
</tr>
<tr>
<td></td>
<td> Running Remotely  </td>
<td> Run our app remotely          </td>
<td> <a href="https://github.com/realpython/Picha/releases/tag/v5">v5</a></td>
</tr>
</tbody>
</table>

## What is Celery?

"[Celery](http://www.celeryproject.org/) is an asynchronous task queue/job queue based on distributed message passing. It is focused on real-time operation, but supports scheduling as well." For this post, we will focus on the scheduling feature to periodically run a job/task.

**Why is this useful?**

- Think of all the times you have had to run a certain task in the future. Perhaps you needed to access an API every hour. Or maybe you needed to send a batch of emails at the end of the day. Large or small, Celery makes scheduling such periodic tasks easy.
- You never want end users to have to wait unnecessarily for pages to load or actions to complete. If a long process is part of your application's workflow, you can use Celery to execute that process in the background, as resources become available, so that your application can continue to respond to client requests. This keeps the task out of the application's context.

## Setup

Before diving into Celery, grab the starter project from the Github [repo](https://github.com/realpython/Picha/releases/tag/v1). Make sure to activate a virtualenv, install the requirements, and run the migrations. Then fire up the server and navigate to [http://localhost:8000/](http://localhost:8000/) in your browser. You should see the familiar "Congratulations on your first Django-powered page" text. When done, kill the sever.

Next, let's install Celery:

```sh
$ pip install celery==3.1.18
$ pip freeze > requirements.txt
```

Now we can integrate Celery into our Django Project in just three easy steps.

### Step 1: Add *celery.py*

Inside the "picha" directory, create a new file called *celery.py*:

```python
from __future__ import absolute_import
import os
from celery import Celery
from django.conf import settings

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'picha.settings')
app = Celery('picha')

# Using a string here means the worker will not have to
# pickle the object when using Windows.
app.config_from_object('django.conf:settings')
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)


@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```

Take note of the comments in the code.

### Step 2: Import your new Celery app

To ensure that the Celery app is loaded when Django starts, add the following code into the *\_\_init\_\_.py* file that sits next to your *settings.py* file:

```python
from __future__ import absolute_import

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app
```

Having done that, your project layout should now look like:

```sh
├── manage.py
├── picha
│   ├── __init__.py
│   ├── celery.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── requirements.txt
```

### Step 3: Install Redis as a Celery "Broker"

Celery uses "[brokers](http://celery.readthedocs.org/en/latest/getting-started/brokers/)" to pass messages between a Django Project and the Celery [workers](http://celery.readthedocs.org/en/latest/userguide/workers.html). In this tutorial, we will use [Redis](http://celery.readthedocs.org/en/latest/getting-started/brokers/redis.html) as the message broker.

First, install [Redis](http://redis.io/) from the official download [page](http://redis.io/download) or via brew (`brew install redis`) and then turn to your terminal, in a new terminal window, fire up the server:

```sh
$ redis-server
```

You can test that Redis is working properly by typing this into your terminal:

```sh
$ redis-cli ping
```

Redis should reply with `PONG` - try it!

Once Redis is up, add the following code to your settings.py file:

```python
# CELERY STUFF
BROKER_URL = 'redis://localhost:6379'
CELERY_RESULT_BACKEND = 'redis://localhost:6379'
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Africa/Nairobi'
```

You also need to add Redis as a dependency in the Django Project:

```sh
$ pip install redis==2.10.3
$ pip freeze > requirements.txt
```

That's it! You should now be able to use Celery with Django. For more information on setting up Celery with Django, please check out the official Celery [documentation](http://docs.celeryproject.org/en/latest/django/first-steps-with-django.html#using-celery-with-django).

Before moving on, let's run a few sanity checks to ensure all is well...

Test that the Celery worker is ready to receive tasks:

```sh
$ celery -A picha worker -l info
...
[2015-07-07 14:07:07,398: INFO/MainProcess] Connected to redis://localhost:6379//
[2015-07-07 14:07:07,410: INFO/MainProcess] mingle: searching for neighbors
[2015-07-07 14:07:08,419: INFO/MainProcess] mingle: all alone
```

Kill the process with CTRL-C. Now, test that the Celery task scheduler is ready for action:

```sh
$ celery -A picha beat -l info
...
[2015-07-07 14:08:23,054: INFO/MainProcess] beat: Starting...
```

Boom!

Again, kill the process when done.

## Celery Tasks

Celery utilizes [tasks](http://celery.readthedocs.org/en/latest/userguide/tasks.html), which can be thought of as regular Python functions that are called with Celery.

For example, let's turn this basic function into a Celery task:

```python
def add(x, y):
    return x + y
```

First, add a decorator:

```python
from celery.decorators import task

@task(name="sum_two_numbers")
def add(x, y):
    return x + y
```

Then you can run this task asynchronously with Celery like so:

```python
add.delay(7, 8)
```

Simple, right?

So, these types of tasks are perfect for when you want to load a web page without making the user wait for some background process to complete.

Let's look at an example...

<hr>

Going back to the Django Project, grab version [three](https://github.com/realpython/Picha/releases/tag/v3), which includes an app that accepts feedback from users, aptly called `feedback`:

```sh
├── feedback
│   ├── __init__.py
│   ├── admin.py
│   ├── emails.py
│   ├── forms.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── manage.py
├── picha
│   ├── __init__.py
│   ├── celery.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── requirements.txt
└── templates
    ├── base.html
    └── feedback
        ├── contact.html
        └── email
            ├── feedback_email_body.txt
            └── feedback_email_subject.txt
```

Install the new requirements, fire up the app, and navigate to [http://localhost:8000/feedback/](http://localhost:8000/feedback/). You should see:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-and-celery/feedback-app.png" style="max-width: 100%;" alt="django feedback app">
</div>

<br>

Let's wire up the Celery task.

### Add the Task

Basically, after the user submits the feedback form, we want to immediately let him continue on his merry way while we process the feedback, send an email, etc., all in the background.

To accomplish this, first add a file called *tasks.py* to the "feedback" directory:

```python
from celery.decorators import task
from celery.utils.log import get_task_logger

from feedback.emails import send_feedback_email

logger = get_task_logger(__name__)


@task(name="send_feedback_email_task")
def send_feedback_email_task(email, message):
    """sends an email when feedback form is filled successfully"""
    logger.info("Sent feedback email")
    return send_feedback_email(email, message)
```

Then update *forms.py* like so:

```python
from django import forms
from feedback.tasks import send_feedback_email_task


class FeedbackForm(forms.Form):
    email = forms.EmailField(label="Email Address")
    message = forms.CharField(
        label="Message", widget=forms.Textarea(attrs={'rows': 5}))
    honeypot = forms.CharField(widget=forms.HiddenInput(), required=False)

    def send_email(self):
        # try to trick spammers by checking whether the honeypot field is
        # filled in; not super complicated/effective but it works
        if self.cleaned_data['honeypot']:
            return False
        send_feedback_email_task.delay(
            self.cleaned_data['email'], self.cleaned_data['message'])
```

In essence, the `send_feedback_email_task.delay(email, message)` function processes and sends the feedback email in the background as the user continues to use the site.

> **NOTE**: The `success_url` in *views.py* is set to redirect the user to `/`, which does not exist yet. We'll set this endpoint up in the next section.

## Periodic Tasks

Often, you'll need to schedule a task to run at a specific time every so often - i.e., a web scraper may need to run daily, for example. Such tasks, called [periodic tasks](http://celery.readthedocs.org/en/latest/userguide/periodic-tasks.html), are easy to set up with Celery.

Celery uses "celery beat" to schedule periodic tasks. Celery beat runs tasks at regular intervals, which are then executed by celery workers.

For example, the following task is scheduled to run every fifteen minutes:

```python
from celery.task.schedules import crontab
from celery.decorators import periodic_task


@periodic_task(run_every=(crontab(minute='*/15')), name="some_task", ignore_result=True)
def some_task():
    # do something
```

Let's look at more robust example by adding this functionality into the Django Project...

<hr>

Back to the Django Project, grab version [four](https://github.com/realpython/Picha/releases/tag/v4), which includes another new app, called `photos`, that uses the [Flickr API](https://www.flickr.com/services/api/) to get new photos for display on the site:

```sh
├── feedback
│   ├── __init__.py
│   ├── admin.py
│   ├── emails.py
│   ├── forms.py
│   ├── models.py
│   ├── tasks.py
│   ├── tests.py
│   └── views.py
├── manage.py
├── photos
│   ├── __init__.py
│   ├── admin.py
│   ├── models.py
│   ├── settings.py
│   ├── tests.py
│   ├── utils.py
│   └── views.py
├── picha
│   ├── __init__.py
│   ├── celery.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── requirements.txt
└── templates
    ├── base.html
    ├── feedback
    │   ├── contact.html
    │   └── email
    │       ├── feedback_email_body.txt
    │       └── feedback_email_subject.txt
    └── photos
        └── photo_list.html
```

Install the new requirements, run the migrations, and then fire up the server to make sure all is well. Try testing out the feedback form again. This time it should redirect just fine.

What's next?

Well, since we would need to call the Flickr API periodically to add more photos to our site, we can add a Celery task.

### Add the Task

Add a *tasks.py* to the `photos` app:

```python
from celery.task.schedules import crontab
from celery.decorators import periodic_task
from celery.utils.log import get_task_logger

from photos.utils import save_latest_flickr_image

logger = get_task_logger(__name__)


@periodic_task(
    run_every=(crontab(minute='*/15')),
    name="task_save_latest_flickr_image",
    ignore_result=True
)
def task_save_latest_flickr_image():
    """
    Saves latest image from Flickr
    """
    save_latest_flickr_image()
    logger.info("Saved image from Flickr")
```

Here, we run the `save_latest_flickr_image()` function every fifteen minutes by wrapping the function call in a `task`. The `@periodic_task` decorator abstracts out the code to run the Celery task, leaving the *tasks.py* file clean and easy to read!

## Running Locally

Ready to run this thing?

With your Django App and Redis running, open two new terminal windows/tabs. In each new window, navigate to your project directory, activate your virtualenv, and then run the following commands (one in each window):

```sh
$ celery -A picha worker -l info
$ celery -A picha beat -l info
```

When you visit the site on [http://127.0.0.1:8000/](http://127.0.0.1:8000/) you should now see one image. Our app gets one image from Flickr every 15 minutes:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-and-celery/photos-app.png" style="max-width: 100%;" alt="django photos app">
</div>

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-and-celery/celery-beat.png" style="max-width: 100%;" alt="get image celery task">
</div>

<br>

Take a look at `photos/tasks.py` to see the code. Clicking on the "Feedback" button allows you to... send some feedback:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-and-celery/added-feeback.png" style="max-width: 100%;" alt="added feedback to django app">
</div>

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-and-celery/send-email-celery.png" style="max-width: 100%;" alt="send email celery task">
</div>

<br>

This works via a celery task. Have a look at `feedback/tasks.py` for more.

That's it, you have the Picha project up and running!

This is good for testing while developing your Django Project locally, but does not work so well when you need to deploy to production - like on [DigitalOcean](https://www.digitalocean.com/?refcode=d8f211a4b4c2), perhaps. For that, it is recommended that you run the Celery worker and scheduler in the background as a daemon with [Supervisor](http://supervisord.org/).

## Running Remotely

Installation is simple. Grab version [five](https://github.com/realpython/Picha/releases/tag/v5) from the repo (if you don't already have it). Then SSH into your remote server and run:

```sh
$ sudo apt-get install supervisor
```

We then need to tell Supervisor about our Celery workers by adding configuration files to the "/etc/supervisor/conf.d/" directory on the remote server. In our case, we need two such configuration files - one for the Celery worker and one for the Celery scheduler.

Locally, create a folder called "supervisor" in the project root. Then add the following files...

**Celery Worker: *picha_celery.conf***

```
; ==================================
;  celery worker supervisor example
; ==================================

; the name of your supervisord program
[program:pichacelery]

; Set full path to celery program if using virtualenv
command=/home/mosh/.virtualenvs/picha/bin/celery worker -A picha --loglevel=INFO

; The directory to your Django project
directory=/home/mosh/sites/picha

; If supervisord is run as the root user, switch users to this UNIX user account
; before doing any processing.
user=mosh

; Supervisor will start as many instances of this program as named by numprocs
numprocs=1

; Put process stdout output in this file
stdout_logfile=/var/log/celery/picha_worker.log

; Put process stderr output in this file
stderr_logfile=/var/log/celery/picha_worker.log

; If true, this program will start automatically when supervisord is started
autostart=true

; May be one of false, unexpected, or true. If false, the process will never
; be autorestarted. If unexpected, the process will be restart when the program
; exits with an exit code that is not one of the exit codes associated with this
; process’ configuration (see exitcodes). If true, the process will be
; unconditionally restarted when it exits, without regard to its exit code.
autorestart=true

; The total number of seconds which the program needs to stay running after
; a startup to consider the start successful.
startsecs=10

; Need to wait for currently executing tasks to finish at shutdown.
; Increase this if you have very long running tasks.
stopwaitsecs = 600

; When resorting to send SIGKILL to the program to terminate it
; send SIGKILL to its whole process group instead,
; taking care of its children as well.
killasgroup=true

; if your broker is supervised, set its priority higher
; so it starts first
priority=998
```

**Celery Scheduler: *picha_celerybeat.conf***

```
; ================================
;  celery beat supervisor example
; ================================

; the name of your supervisord program
[program:pichacelerybeat]

; Set full path to celery program if using virtualenv
command=/home/mosh/.virtualenvs/picha/bin/celerybeat -A picha --loglevel=INFO

; The directory to your Django project
directory=/home/mosh/sites/picha

; If supervisord is run as the root user, switch users to this UNIX user account
; before doing any processing.
user=mosh

; Supervisor will start as many instances of this program as named by numprocs
numprocs=1

; Put process stdout output in this file
stdout_logfile=/var/log/celery/picha_beat.log

; Put process stderr output in this file
stderr_logfile=/var/log/celery/picha_beat.log

; If true, this program will start automatically when supervisord is started
autostart=true

; May be one of false, unexpected, or true. If false, the process will never
; be autorestarted. If unexpected, the process will be restart when the program
; exits with an exit code that is not one of the exit codes associated with this
; process’ configuration (see exitcodes). If true, the process will be
; unconditionally restarted when it exits, without regard to its exit code.
autorestart=true

; The total number of seconds which the program needs to stay running after
; a startup to consider the start successful.
startsecs=10

; if your broker is supervised, set its priority higher
; so it starts first
priority=999
```

> Make sure to update the paths in these files to match the remote server's filesystem.

Basically, these supervisor configuration files tell supervisord how to run and manage our 'programs' (as they are called by supervisord).

In the examples above, we have created two supervisord programs named "pichacelery" and "pichacelerybeat".

Now just copy these files to the remote server in the "/etc/supervisor/conf.d/" directory.

We also need to create the log files that are mentioned in the above scripts on the remote server:

```sh
$ touch /var/log/celery/picha_worker.log
$ touch /var/log/celery/picha_beat.log
```

Finally, run the following commands to make Supervisor aware of the programs - e.g., `pichacelery` and `pichacelerybeat`:

```sh
$ sudo supervisorctl reread
$ sudo supervisorctl update
```

Run the following commands to stop, start, and/or check the status of the `pichacelery` program:

```sh
$ sudo supervisorctl stop pichacelery
$ sudo supervisorctl start pichacelery
$ sudo supervisorctl status pichacelery
```

You can read more about Supervisor from the [official documentation](http://supervisord.org).

## Final Tips

1. *Do not pass Django model objects to Celery tasks.* To avoid cases where the model object has already changed before it is passed to a Celery task, pass the object's primary key to Celery. You would then, of course, have to use the primary key to get the object from the database before working on it.
1. *The default Celery scheduler creates some files to store its schedule locally.* These files would be "celerybeat-schedule.db" and "celerybeat.pid". If you are using a version control system like Git (which you should!), it is a good idea to ignore this files and not add them to your repository since they are for running processes locally.

## Next steps

Well, that's it for the basic introduction to integrating Celery into a Django Project.

Want more?

1. Dive into the official [Celery User Guide](http://celery.readthedocs.org/en/latest/userguide/) to learn more.
1. Create a [Fabfile](http://www.fabfile.org/) to setup Supervisor and the configuration files. Make sure to add the commands to `reread` and `update` Supervisor.
1. Fork the Project from the [repo](https://github.com/realpython/Picha) and open a Pull Request to add a new Celery task.

Happy coding!
