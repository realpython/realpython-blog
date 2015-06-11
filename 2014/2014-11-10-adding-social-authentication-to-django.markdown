# Adding Social Authentication to Django

[Python Social Auth](https://github.com/omab/python-social-auth) is a library that provides "an easy-to-setup social authentication/registration mechanism with support for several [frameworks](http://psa.matiasaguirre.net/docs/intro.html#supported-frameworks) and [auth providers](http://psa.matiasaguirre.net/docs/intro.html#auth-providers)". In this tutorial, we'll detail how to integrate this library into a Django project to provide user authentication.

**What we're using**:

- Django==1.7.1
- python-social-auth==0.2.1

## Django Setup

> If you already have a project set up and ready to go, feel free to skip this section.

Create and activate a virtualenv, install Django, and then start a new Django project:

```sh
$ django-admin.py startproject django_social_project
$ cd django_social_project
$ python manage.py startapp django_social_app
```

Set up the initial tables and add a superuser:

```sh
$ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, contenttypes, auth, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying sessions.0001_initial... OK

$ python manage.py createsuperuser
Username (leave blank to use 'michaelherman'): admin
Email address: ad@min.com
Password:
Password (again):
Superuser created successfully.
```

Create a new directory in the Project root called "templates", and then add the correct path to the *settings.py* file:

```python
TEMPLATE_DIRS = (
    os.path.join(BASE_DIR, 'templates'),
)
```

Run the development server to ensure all is in order and then navigate to [http://localhost:8000/](http://localhost:8000/). You should see the "It worked!" page.

Your project should look like this:

```
└── django_social_project
    ├── db.sqlite3
    ├── django_social_app
    │   ├── __init__.py
    │   ├── admin.py
    │   ├── migrations
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    ├── django_social_project
    │   ├── __init__.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── templates
```

## Python Social Auth Setup

Follow the steps below and/or the [official installation guide](http://psa.matiasaguirre.net/docs/index.html) to install and setup the basic configuration.

### Installation

Install with pip:

```sh
$ pip install python-social-auth==0.2.1
```

### Configuration

Update *settings.py* to include/register the library in our project:

```python
INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django_social_project',
    'social.apps.django_app.default',
)

TEMPLATE_CONTEXT_PROCESSORS = (
    'django.contrib.auth.context_processors.auth',
    'django.core.context_processors.debug',
    'django.core.context_processors.i18n',
    'django.core.context_processors.media',
    'django.core.context_processors.static',
    'django.core.context_processors.tz',
    'django.contrib.messages.context_processors.messages',
    'social.apps.django_app.context_processors.backends',
    'social.apps.django_app.context_processors.login_redirect',
)

AUTHENTICATION_BACKENDS = (
    'social.backends.facebook.FacebookOAuth2',
    'social.backends.google.GoogleOAuth2',
    'social.backends.twitter.TwitterOAuth',
    'django.contrib.auth.backends.ModelBackend',
)
```

Once registered, update the database:

```sh
$ python manage.py makemigrations
Migrations for 'default':
  0002_auto_20141109_1829.py:
    - Alter field user on usersocialauth

$ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, default, contenttypes, auth, sessions
Running migrations:
  Applying default.0001_initial... OK
  Applying default.0002_auto_20141109_1829... OK
```

Update the Project's `urlpatterns` in *urls.py* to include the main auth URLs:

```sh
urlpatterns = patterns(
    '',
    url(r'^admin/', include(admin.site.urls)),
    url('', include('social.apps.django_app.urls', namespace='social')),
)
```

Next, you need to grab the required authentication keys form each social application that you want to include. The process is similar for many of the popular social networks - like Twitter, Facebook, and Google. Let's look at Twitter as an example...

## Twitter Authentication Keys

Create a new application at [https://apps.twitter.com/app/new](https://apps.twitter.com/app/new) and make sure to use a callback url of [http://127.0.0.1:8000/complete/twitter](http://127.0.0.1:8000/complete/twitter).

In the "django_social_project" directory, add a new file called *config.py*. Grab the `Consumer Key (API Key)` and the `Consumer Secret (API Secret)` from Twitter under the "Keys and Access Tokens" tab and add them to the config file like so:

```python
SOCIAL_AUTH_TWITTER_KEY = 'update me'
SOCIAL_AUTH_TWITTER_SECRET = 'update me'
```

Let's also add the following URLs to *config.py* to specify the login and redirect URLs (after a user authenticates):

```python
SOCIAL_AUTH_LOGIN_REDIRECT_URL = '/home/'
SOCIAL_AUTH_LOGIN_URL = '/'
```

Add the following import to *settings.py*:

```python
from config import *
```

Make sure to add *config.py* to your *.gitignore* file, as you do *not* want this file added to version control since it contains sensitive information.

For more info, check the [official docs](http://python-social-auth.readthedocs.org/en/latest/backends/twitter.html?highlight=twitter).

## Sanity Check

Let's test this out. Fire up the server and navigate to [http://127.0.0.1:8000/login/twitter](http://127.0.0.1:8000/login/twitter), authorize the app, and if all works out, you should be redirected to [http://127.0.0.1:8000/home/](http://127.0.0.1:8000/home/) (the URL associated with `SOCIAL_AUTH_LOGIN_REDIRECT_URL`). You should see a 404 error since we do not have a route, view, or template set up yet.

Let's do that now...

## Friendly Views

For now, we just need two views - login and home.

### URLs

Update the URL patterns in *urls.py*:

```python
urlpatterns = patterns(
    '',
    url(r'^admin/', include(admin.site.urls)),
    url('', include('social.apps.django_app.urls', namespace='social')),
    url(r'^$', 'django_social_app.views.login'),
    url(r'^home/$', 'django_social_app.views.home'),
    url(r'^logout/$', 'django_social_app.views.logout'),
)
```

Besides the `/` and `home/` routes, we also added the `logout/` route.

### Views

Next, add the following view functions:

```python
from django.shortcuts import render_to_response, redirect, render
from django.contrib.auth import logout as auth_logout
from django.contrib.auth.decorators import login_required
# from django.template.context import RequestContext


def login(request):
    # context = RequestContext(request, {
    #     'request': request, 'user': request.user})
    # return render_to_response('login.html', context_instance=context)
    return render(request, 'login.html')


@login_required(login_url='/')
def home(request):
    return render_to_response('home.html')


def logout(request):
    auth_logout(request)
    return redirect('/')
```

In the `login()` function, we grab the logged in user with the `RequestContext`. For your reference, the more explicit method of achieving this is commented out.

### Templates

Add two templates *home.html* and *login.html*.
<br>

**home.html**

```html
<h1>Welcome</h1>
<p><a href="/logout">Logout</a>
```

**login.html**

{% raw %}
```html
{% if user and not user.is_anonymous %}
  <a>Hello, {{ user.get_full_name }}!</a>
  <br>
  <a href="/logout">Logout</a>
{% else %}
  <a href="{% url 'social:begin' 'twitter' %}?next={{ request.path }}">Login with Twitter</a>
{% endif %}
```
{% endraw %}


Your project should now look like this:

```
└── django_social_project
    ├── db.sqlite3
    ├── django_social_app
    │   ├── __init__.py
    │   ├── admin.py
    │   ├── migrations
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    ├── django_social_project
    │   ├── __init__.py
    │   ├── config.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── templates
        ├── home.html
        └── login.html
```

Test it out again. Fire up the server. Make sure to first logout, since a user should already be logged in from the last test, and then test logging in and logging out. After login, the user should be redirected to `/home`.

## Next Steps

At this point, you'll probably want to add a few more authentication providers - like [Facebook](http://python-social-auth.readthedocs.org/en/latest/backends/facebook.html) and [Google](http://python-social-auth.readthedocs.org/en/latest/backends/google.html). The workflow is simple for adding a new social auth provider:

{% raw %}
1. Create a new application on the provider's website.
1. Set the callback URL.
1. Grab the keys/tokens and add them to *config.py*.
1. Add the new provider to the `AUTHENTICATION_BACKENDS ` tuple in *settings.py*.
1. Update the login template by adding the new URL like so - `<a href="{% url 'social:begin' 'ADD AUTHENTICATION PROVIDER NAME' %}?next={{ request.path }}">Login with AUTHENTICATION PROVIDER NAME</a>`.
{% endraw %}

Please review the [official docs](http://psa.matiasaguirre.net/docs/) for more information. Leave comments and questions below. Thanks for reading!

Oh - and grab the code from the [repo](https://github.com/realpython/django-social-auth-example).
