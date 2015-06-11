# Migrating your Django Project to Heroku

In this tutorial, we'll be taking a simple local Django project, backed by a MySQL database, and converting it to run on Heroku. Amazon S3 will be used to host our static files, while Fabric will automate the deployment process.

The Project is a simple message system. It could be a todo app or a blog or even a Twitter clone. To simulate a real-live scenario, the Project will first be created with a MySQL backend, then converted to Postgres for deployment on Heroku. I've personally had five or six projects where I've had to do this exact thing: convert a local Project, backed with MySQL, to a live app on Heroku.

## Setup

### Pre-requisites:

1. Read the official Django Quick Start guide over at [Heroku](https://devcenter.heroku.com/articles/django). Just read it. This will help you get a feel for what we'll be accomplishing in this tutorial. We'll be using the official tutorial as a guide for our own, more advanced deployment process.
2. Create an AWS account and set up an active S3 bucket.
3. Install MySQL.

### Let's begin:

Start by downloading the test Project [here](http://realpython.com/files/django_heroku_deploy.zip), unzip, then activate a virtualenv:

```sh
$ cd django_heroku_deploy
$ virtualenv --no-site-packages myenv
$ source myenv/bin/activate
```

Create a new repository on Github:

```sh
$ curl -u 'USER' https://api.github.com/user/repos -d '{"name":"REPO"}'
```

> Make sure to replace the all caps KEYWORDS with your own settings. For example - ```curl -u 'mjhea0' https://api.github.com/user/repos -d '{"name":"django-deploy-heroku-s3"}'```

Add a readme file, initialize the local Git repo, then PUSH the local copy to Github:

```sh
$ touch README.md
$ git init
$ git add .
$ git commit -am "initial"
$ git remote add origin https://github.com/username/Hello-World.git
$ git push origin master
```

> Be sure to change the URL to your repo's URL that you created in the previous step.

Set up a new MySQL database called *django_deploy*:

```sh
$ mysql.server start
$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g. Your MySQL connection id is 1
Type 'help;' or '\h' for help. Type '\c' to clear the buffer.
mysql>
mysql> CREATE DATABASE django_deploy;
Query OK, 1 row affected (0.01 sec)
mysql>
mysql> quit
Bye
```

Update *settings.py*:

```python
DATABASES = {
'default': {
    'ENGINE': 'django.db.backends.mysql',
    'NAME': 'django_deploy',
    'USER': 'root',
    'PASSWORD': 'your_password',
}
}
```

Install the dependencies:

```sh
$ pip install -r requirements.txt
$ python manage.py syncdb
$ python manage.py runserver
```

Run the server at [http://localhost:8000/admin/](http://localhost:8000/admin/), and make sure you can log in to the admin. Add a few items to the `Whatever` object. Kill the server.

## Convert from MySQL to Postgres

> Note: In this hypothetical situation, let's pretend that you have been working on this Project for a while using MySQL and now you want to convert it to Postgres.

Install dependencies:

```sh
$ pip install psycopg2
$ pip install py-mysql2pgsql
```

Set up a Postgres database:

```sh
$ psql -h localhost
psql (9.2.4)
Type "help" for help.
michaelherman=# CREATE DATABASE django_deploy;
CREATE DATABASE
michaelherman=# \q
```

Migrate data:

```sh
$ py-mysql2pgsql
```

This command creates a file called *mysql2pgsql.yml*, containing the following info:

```yaml
mysql:
  hostname: localhost
  port: 3306
  socket: /tmp/mysql.sock
  username: foo
  password: bar
  database: your_database_name
  compress: false
destination:
  postgres:
    hostname: localhost
    port: 5432
    username: foo
    password: bar
    database: your_database_name
```

> Update this for your configuration. This example just covers the basic conversion. You can also include or exclude certain tables. See the full example [here](https://github.com/philipsoutham/py-mysql2pgsql).

 Transfer the data:

```sh
$ py-mysql2pgsql -v -f mysql2pgsql.yml
```

Once the data is transferred, be sure to update your *settings.py* file:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql_psycopg2",
        "NAME": "your_database_name",
        "USER": "foo",
        "PASSWORD": "bar",
        "HOST": "localhost",
        "PORT": "5432",
    }
}
```

Finally, resync the database, run the test server, and add another item to the database to ensure the conversion succeeded.

## Add a local_settings.py file

By adding a *local_settings.py* file, you can extend the *settings.py* file with settings relevant to your local environment, while the main *settings.py* file is used solely for your staging and production environments.

Make sure you add *local_settings.py* to your *.gitignore* file in order to keep the file out of your repositories. Those who want to use or contribute to your project can then clone the repo and create their own *local_settings.py* file specific to their own local environment.

> Although this method of using two settings files has been convention for a number of years, many Python developers now use another pattern called [The One True Way](https://speakerdeck.com/jacobian/the-best-and-worst-of-django?slide=81). We may look at this pattern in a future tutorial.

### We need to make three changes to our current *settings.py* file:

Change `DEBUG` mode to false:

 ```python
 DEBUG = False
 ```
Add the following code to the bottom of the file:

```python
# # Allow all host hosts/domain names for this site
ALLOWED_HOSTS = ['*']

# Parse database configuration from $DATABASE_URL
import dj_database_url

DATABASES = { 'default' : dj_database_url.config()}

# Honor the 'X-Forwarded-Proto' header for request.is_secure()
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# try to load local_settings.py if it exists
try:
  from local_settings import *
except Exception as e:
  pass
```

Update the database settings:

```python
# we only need the engine name, as heroku takes care of the rest
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql_psycopg2",
    }
}
```

Create your *local_settings.py* file:

```sh
$ touch local_settings.py
$ pip install dj_database_url
```

Then add the following code:

```python
from settings import PROJECT_ROOT, SITE_ROOT
import os

DEBUG = True
TEMPLATE_DEBUG = True

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql_psycopg2",
        "NAME": "django_deploy",
        "USER": "foo",
        "PASSWORD": "bar",
        "HOST": "localhost",
        "PORT": "5432",
    }
}
```

Fire up the test server to make sure everything still works. Add a few more records to the database.

## Heroku setup

Add a Procfile to the main directory:

```sh
$ touch Procfile
```

and add the following code to the file:

```python
web: python manage.py runserver 0.0.0.0:$PORT --noreload
```

Install the Heroku toolbelt:

```sh
$ pip install django-toolbelt
```

Freeze the dependencies:

```sh
$ pip freeze > requirements.txt
```

Update the *wsgi.py* file:

```python
from django.core.wsgi import get_wsgi_application
from dj_static import Cling

application = Cling(get_wsgi_application())
```

Test your Heroku settings locally:

```sh
$ foreman start
```

Navigate to [http://localhost:5000/](http://localhost:5000/).

Looking good? Let's get Amazon S3 running.

## Amazon S3

Although it is hypothetically possible to host static files in your Heroku repo, it's best to use a third party host, especially if you have a customer-facing application. S3 is easy to use, and it requires only a few changes to your *settings.py* file.

Install dependencies:

```sh
$ pip install django-storages
$ pip install boto
```

Add `storages` and `boto` to your `INSTALLED_APPS` in "settings.py"

Add the following code to the bottom of "settings.py":

```python
#Storage on S3 settings are stored as os.environs to keep settings.py clean
if not DEBUG:
   AWS_STORAGE_BUCKET_NAME = os.environ['AWS_STORAGE_BUCKET_NAME']
   AWS_ACCESS_KEY_ID = os.environ['AWS_ACCESS_KEY_ID']
   AWS_SECRET_ACCESS_KEY = os.environ['AWS_SECRET_ACCESS_KEY']
   STATICFILES_STORAGE = 'storages.backends.s3boto.S3BotoStorage'
   S3_URL = 'http://%s.s3.amazonaws.com/' % AWS_STORAGE_BUCKET_NAME
   STATIC_URL = S3_URL
```

The AWS environment-dependent settings are stored as environmental variables. So we don't have to set these from the terminal each time we run the development server, we can set these in our virtualenv `activate` script. Grab the AWS Bucket name, Access Key ID, and Secret Access Key from S3. Open `myenv/bin/activate` and append the following code (make sure to add in your specific info you just pulled from S3):

```python
# s3 deployment info
export AWS_STORAGE_BUCKET_NAME=[YOUR AWS S3 BUCKET NAME]
export AWS_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXX
export AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXX
```

Deactivate and reactivate your virtualenv, then launch the local server to make sure the changes took affect:

```sh
$ foreman start
```

Kill the server, then update the *requirements.txt* file:

```sh
$ pip freeze > requirements.txt
```

## Push to Github and Heroku

Let's backup our files to Github before PUSHing to Heroku:

```sh
$ git add .
$ git commit -m "update project for heroku and S3"
$ git push -u origin master
```

Create a Heroku Project/Repo:

```sh
$ heroku create <name>
```

> Name it whatever you'd like.

PUSH to Heroku:

```sh
$ git push heroku master
```

Send the AWS environmental variables to Heroku

```sh
$ heroku config:set AWS_STORAGE_BUCKET_NAME=[YOUR AWS S3 BUCKET NAME]
$ heroku config:set AWS_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXX
$ heroku config:set AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXX
```

Collect the static files and send to Amazon:

```sh
$ heroku run python manage.py collectstatic
```

Add development database:

```sh
$ heroku addons:add heroku-postgresql:dev
Adding heroku-postgresql on deploy_django... done, v13 (free)
Attached as HEROKU_POSTGRESQL_COPPER_URL
Database has been created and is available
! This database is empty. If upgrading, you can transfer
! data from another database with pgbackups:restore.
Use `heroku addons:docs heroku-postgresql` to view documentation.
$ heroku pg:promote HEROKU_POSTGRESQL_COPPER_URL
Promoting HEROKU_POSTGRESQL_COPPER_URL to DATABASE_URL... done
```

Now sync the DB:

```sh
$ heroku run python manage.py syncdb
```

## Data transfer

We need to transfer the data from the local database to the production database.

Install the Heroku PGBackups add-on:

```sh
$ heroku addons:add pgbackups
```

Dump your local database:

```sh
$ pg_dump -h localhost  -Fc library  > db.dump
```

In order for Heroku to access db dump, you need to upload it to the Internet somewhere. You could use a personal website, dropbox, or S3. I simply uploaded it to the S3 bucket.

Import the dump to Heroku:

```sh
$ heroku pgbackups:restore DATABASE http://www.example.com/db.dump
```

## Test

Let's test to make sure everything works.

First, update your allowed hosts to your specific domain in *settings.py*:

```sh
ALLOWED_HOSTS = ['[your-project-name].herokuapp.com']
```

Check out your app:

```sh
$ heroku open
```

## Fabric

Fabric is used for automating deployment of your application.

Install:

```sh
$ pip install fabric
```

Create the fabfile:

```sh
$ touch fabfile.py
```

Then add the following code:

```python
from fabric.api import local

def deploy():
   local('pip freeze > requirements.txt')
   local('git add .')
   print("enter your git commit comment: ")
   comment = raw_input()
   local('git commit -m "%s"' % comment)
   local('git push -u origin master')
   local('heroku maintenance:on')
   local('git push heroku master')
   local('heroku maintenance:off')
```

Test:

```sh
$ fab deploy
```

<hr>

Please leave questions or comments? Thanks!











