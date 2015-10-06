# LinkedIn Social Authentication in Django

Social Authentication (or Social Login) is a way to simplify logins for end users by using existing login information from the popular social networking services such as [Facebook](https://www.facebook.com/), [Twitter](https://www.twitter.com/), [Google+](https://plus.google.com/), [LinkedIn](https://www.linkedin.com/) (focus of this article), and so on. Most websites that require a user to login, utilize social login platforms for a better authentication/registeration experience in lieu of developing their own systems.

[Python Social Auth](http://psa.matiasaguirre.net/) provides a mechanism for easily setting up an authentication/registration system, which supports several frameworks and auth providers. **In this tutorial, we’ll demonstrate in detail how to integrate this library into your Django Project to provide user authentication through LinkedIn using OAuth 2.0.**

<div class="center-text">
  <img class="no-border" src="/images/blog_images/social-auth/django-linkedin.png" style="max-width: 100%;" alt="django-linkedin">
</div>

<br>

## What is OAuth 2.0?

[OAuth 2.0](http://oauth.net/2/) is the authorization framework which allows applications to gain access to an end user's account for authenticating/registering via popular social networking services. The end user gets to choose which details the application has access to. It focuses on simplifying the development workflow while providing specific authorization flows for web applications and desktop applications, mobile phones, and IOT (Internet of Things) devices.

## Environment and Django Set up

We'll be using:

  - python v3.4.2
  - Django v1.8.4
  - python-social-auth v0.2.12

> If you already have a [virtualenv](http://docs.python-guide.org/en/latest/dev/virtualenvs/) and a Django Project set up and ready to go, feel free to skip this section.

### Create a virtualenv

```sh
$ mkvirtualenv --python='/usr/bin/python3.4' django_social_project
$ pip install django==1.8.4
```

### Start a new Django Project

Bootstrap a Django Application:

```sh
$ django-admin.py startproject django_social_project
$ cd django_social_project
$ python manage.py startapp django_social_app
```

Don't forget to add the app to the `INSTALLED_APPS` tuple in *settings.py*, for our Project to know that we have created an app that needs to be served as a part of our Django Project.

### Set up the initial tables

```sh
$ python manage.py migrate
Operations to perform:
  Synchronize unmigrated apps: messages, staticfiles
  Apply all migrations: sessions, admin, auth, contenttypes
Synchronizing apps without migrations:
  Creating tables...
    Running deferred SQL...
  Installing custom SQL...
Running migrations:
  Rendering model states... DONE
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying sessions.0001_initial... OK
```

### Add a superuser

```sh
$ python manage.py syncdb
```

### Create a templates directory

Create a new directory called "templates" in the project root, and then add the correct path to the `TEMPLATES` in the *settings.py* file:

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

### Run a sanity check

Fire up the development server - `python manage.py runserver` to ensure all is in order and then navigate to [http://127.0.0.1:8000/](http://127.0.0.1:8000/). You should see the "It worked!" page.

Your project should look like this:

```sh
└── django_social_project
    ├── db.sqlite3
    ├── django_social_app
    │   ├── __init__.py
    │   ├── admin.py
    │   ├── migrations
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    ├── django_social_project
    │   ├── __init__.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── templates
```

## Python Social Auth Set up

Follow the steps below and/or the [official installation guide](http://psa.matiasaguirre.net/docs/installing.html) to install and set up the basic configuration in order to make our app handle the social login via any social networking service.

### Installation

Install with pip:

```sh
$ pip install python-social-auth==0.2.12
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
    'django_social_app',
    'social.apps.django_app.default',
)

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.core.context_processors.debug',
                'django.core.context_processors.i18n',
                'django.core.context_processors.media',
                'django.core.context_processors.static',
                'django.core.context_processors.tz',
                'django.contrib.messages.context_processors.messages',
                'social.apps.django_app.context_processors.backends',
                'social.apps.django_app.context_processors.login_redirect',
            ],
        },
    },
]
```

> **NOTE**: Since we're using LinkedIn Social Authentication, we need the Linkedin OAuth2 backend:

```python
AUTHENTICATION_BACKENDS = (
    'social.backends.linkedin.LinkedinOAuth2',
    'django.contrib.auth.backends.ModelBackend',
)
```

### Run a migration

Once registered, update the database:

```sh
$ python manage.py makemigrations
$ python manage.py migrate
```

### Update the URLs

Within the Project's *urls.py* file, update the urlpatters to inclue the main auth URLs:

```python
urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url('', include('social.apps.django_app.urls', namespace='social')),
]
```

Next we need to grab the required authentication keys from a LinkedIn Application. This process is similar for many popular social networks - like Twitter, Facebook and Google.

### LinkedIn Authentication Keys

In order for our app to distinctly recognize the LinkedIn social login that we implemented, we need to have some app-specific credentials to differentiate our app's login from other social logins on the web.

Create a new application at [https://www.linkedin.com/developer/apps](https://www.linkedin.com/developer/apps) and make sure to use a callback/redirect URL of [http://127.0.0.1:8000/complete/linkedin-oauth2/](http://127.0.0.1:8000/complete/linkedin-oauth2/) (very important!). This URL is specific to OAuth 2.0, remember that.

> **NOTE**: The callback URL used above is valid only in local development and will need be changed when you move to a production or staging environment.

In the "django_social_project" directory, add a new file called *config.py*. Then grab the Consumer Key (API Key) and the Consumer Secret (API Secret) from LinkedIn and add them to the file:

```python
SOCIAL_AUTH_LINKEDIN_OAUTH2_KEY = 'update me'
SOCIAL_AUTH_LINKEDIN_OAUTH2_SECRET = 'update me'
```

Let’s also add the following URLs to the *config.py* file to specify the login and the redirect URLs (after a user authenticates):

```python
SOCIAL_AUTH_LOGIN_REDIRECT_URL = '/home/'
SOCIAL_AUTH_LOGIN_URL = '/'
```

Add the following imports to *settings.py*

```python
from config import *
```

In order for our app to complete the login process successfully we need to define the Templates and Views associated with those URLs. Let’s do that now.

## Friendly Views

For checking whether our app is working or not we just need two views - *login* and *home*.

### URLs

Update the urlpatterns in the Project's *urls.py* file, to map our URLs to our views that we'll be seeing in the further sections:

```python
urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url('', include('social.apps.django_app.urls', namespace='social')),
    url(r'^$', 'django_social_app.views.login'),
    url(r'^home/$', 'django_social_app.views.home'),
    url(r'^logout/$', 'django_social_app.views.logout'),
]
```

### Views

Now, add the views to the App's *views.py* for making our routes know what exactly to do when a particular route is hit.

```python
from django.shortcuts import render_to_response, redirect
from django.contrib.auth import logout as auth_logout
from django.contrib.auth.decorators import login_required
from django.template.context import RequestContext


def login(request):
    return render_to_response('login.html', context=RequestContext(request))


@login_required(login_url='/')
def home(request):
    return render_to_response('home.html')


def logout(request):
    auth_logout(request)
    return redirect('/')
```

So, in the login function we fetched the logged-in user using the `RequestContext`.

### Templates

Add two templates to the "templates" folder - *login.html*:

{% raw %}
```html
<!-- login.html -->

{% if user and not user.is_anonymous %}
  <a>Hello, {{ user.get_full_name }}!</a>
  <br>
  <a href="/logout">Logout</a>
{% else %}
  <a href="{% url 'social:begin' backend='linkedin-oauth2' %}">Login with Linkedin</a>
{% endif %}
```
{% endraw %}

And *home.html*:

```html
<!-- home.html -->

<h1>Welcome</h1>
<br>
<p><a href="/logout">Logout</a>
```

Your project should now look like this:

```sh
└── django_social_project
    ├── db.sqlite3
    ├── django_social_app
    │   ├── __init__.py
    │   ├── admin.py
    │   ├── migrations
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    ├── django_social_project
    │   ├── __init__.py
    │   ├── config.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── templates
        ├── home.html
        └── login.html
```

## Test!

Now just fire up the server again to test:

```sh
$ python manage.py runserver
```

Just browse to [http://127.0.0.1:8000/](http://127.0.0.1:8000/) and you will see a "Login with Linkedin" hyperlink. Test this out to make sure all is well.

Grab the code [here](https://github.com/realpython/python-social-auth). Cheers!
