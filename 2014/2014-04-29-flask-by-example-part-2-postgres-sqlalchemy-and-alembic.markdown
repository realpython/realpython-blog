# Flask by Example - Setting up Postgres, SQLAlchemy, and Alembic

**In this part we're going to set up a Postgres database to store the results of our word counts as well as SQLAlchemy, an Object Relational Mapper, and Alembic to handle database migrations.**

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask_by_example/flask-by-example-part2.png" style="max-width: 100%;" alt="flask by example part 2">
</div>

<br>

*Updates:*

  - 03/22/2016: Upgraded to Python version [3.5.1](https://www.python.org/downloads/release/python-351/) as well as the latest versions of Psycopg2, Flask-SQLAlchemy, and Flask-Migrate. See [below](#install-requirements) for details.
  - 02/22/2015: Added Python 3 support.

<hr><br>

Remember: Here's what we're building - A Flask app that calculates word-frequency pairs based on the text from a given URL.

<br>

1. [Part One](/blog/python/flask-by-example-part-1-project-setup/): Set up a local development environment and then deploy both a staging and a production environment on Heroku.
1. **Part Two: Set up a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations. (_current_)**
1. [Part Three](/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/): Add in the back-end logic to scrape and then process the word counts from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
1. [Part Four](/blog/python/flask-by-example-implementing-a-redis-task-queue/): Implement a Redis task queue to handle the text processing.
1. [Part Five](/blog/python/flask-by-example-integrating-flask-and-angularjs/): Set up Angular on the front-end to continuously poll the back-end to see if the request is done processing.
1. [Part Six](/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis and detailing how to run two processes (web and worker) on a single Dyno.
1. [Part Seven](/blog/python/flask-by-example-updating-the-ui/): Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Install Requirements

Tools used in this part:

- PostgreSQL ([9.4](https://wiki.postgresql.org/wiki/What's_new_in_PostgreSQL_9.4))
- Psycopg2 ([2.6.1](https://pypi.python.org/pypi/psycopg2/2.6.1)) - a Python adapter for Postgres
- Flask-SQLAlchemy ([2.1](http://flask-sqlalchemy.pocoo.org/2.1/)) - Flask extension that provides [SQLAlchemy](http://www.sqlalchemy.org/) support
- Flask-Migrate ([1.8.0](https://pypi.python.org/pypi/Flask-Migrate/1.8.0)) - extension that supports SQLAlchemy database migrations via [Alembic](https://pypi.python.org/pypi/alembic/0.8.5)

To get started, install Postgres on your local computer, if you don't have it already. Since Heroku uses Postgres, it will be good for us to develop locally on the same database. If you don't have Postgres installed, [Postgres.app](http://postgresapp.com/) is an easy way to get up and running for Mac OS X users. Consult the [download page](http://www.postgresql.org/download/) for more info.

Once you have Postgres installed and running, create a database called `wordcount_dev` to use as our local development database:

```sh
$ psql
# create database wordcount_dev;
CREATE DATABASE
# \q
```

In order to use our newly created database within the Flask app we to need to install a few things:

```sh
$ cd flask-by-example
```

> `cd`ing into the directory should activate the virtual environment and set the environment variables found in the `.env` file via [autoenv](https://pypi.python.org/pypi/autoenv/1.0.0), which we set up in [part 1](/blog/python/flask-by-example-part-1-project-setup/).

```sh
$ pip install psycopg2==2.6.1 Flask-SQLAlchemy===2.1 Flask-Migrate==1.8.0
$ pip freeze > requirements.txt
```

> If you're on OS X and having trouble installing psycopg2 check out [this](http://stackoverflow.com/questions/22313407/clang-error-unknown-argument-mno-fused-madd-python-package-installation-fa) Stack Overflow article.

## Update Configuration

Add `SQLALCHEMY_DATABASE_URI` field to the `Config()` class in your *config.py* file to set your app to use the newly created database in development (local), staging, and production:

```python
import os

class Config(object):
    ...
    SQLALCHEMY_DATABASE_URI = os.environ['DATABASE_URL']
```


Your *config.py* file should now look like this:

```python
import os
basedir = os.path.abspath(os.path.dirname(__file__))


class Config(object):
    DEBUG = False
    TESTING = False
    CSRF_ENABLED = True
    SECRET_KEY = 'this-really-needs-to-be-changed'
    SQLALCHEMY_DATABASE_URI = os.environ['DATABASE_URL']


class ProductionConfig(Config):
    DEBUG = False


class StagingConfig(Config):
    DEVELOPMENT = True
    DEBUG = True


class DevelopmentConfig(Config):
    DEVELOPMENT = True
    DEBUG = True


class TestingConfig(Config):
    TESTING = True
```

Now when our config is loaded into our app the appropriate database will be connected to it as well.

Similar to how we added an environment variable in the last post, we are going to add a `DATABASE_URL` variable. Run this in the terminal:


```sh
$ export DATABASE_URL="postgresql://localhost/wordcount_dev"
```

And then add that line into your *.env* file.

In your *app.py* file import SQLAlchemy and connect to the database:

```python
from flask import Flask
from flask.ext.sqlalchemy import SQLAlchemy
import os


app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

from models import Result


@app.route('/')
def hello():
    return "Hello World!"


@app.route('/<name>')
def hello_name(name):
    return "Hello {}!".format(name)


if __name__ == '__main__':
    app.run()
```

## Data Model

Set up a basic model by adding a *models.py* file:

```python
from app import db
from sqlalchemy.dialects.postgresql import JSON


class Result(db.Model):
    __tablename__ = 'results'

    id = db.Column(db.Integer, primary_key=True)
    url = db.Column(db.String())
    result_all = db.Column(JSON)
    result_no_stop_words = db.Column(JSON)

    def __init__(self, url, result_all, result_no_stop_words):
        self.url = url
        self.result_all = result_all
        self.result_no_stop_words = result_no_stop_words

    def __repr__(self):
        return '<id {}>'.format(self.id)
```

Here we created a table to store the results of the word counts.

We first import the database connection that we created in our *app.py* file as well as JSON from SQLAlchemy's [PostgreSQL dialects](http://docs.sqlalchemy.org/en/latest/dialects/postgresql.html#sqlalchemy.dialects.postgresql.JSON). JSON columns are fairly new to Postgres and are not available in every database supported by SQLAlchemy so we need to import it specifically.

Next we created a `Result()` class and assigned it a table name of `results`. We then set the attributes that we want to store for a result-

  - the `id` of the result we stored
  - the `url` that we counted the words from
  - a full list of words that we counted
  - a list of words that we counted minus stop words (more on this later)

We then created an `__init__()` method that will run the first time we create a new result and, finally, a `__repr__()` method to represent the object when we query for it.

## Local Migration

We are going to use [Alembic](https://pypi.python.org/pypi/alembic/0.8.5), which is part of [Flask-Migrate](https://pypi.python.org/pypi/Flask-Migrate/1.8.0), to manage database migrations to update a database's schema.

Create a new file called *manage.py*:

```python
import os
from flask.ext.script import Manager
from flask.ext.migrate import Migrate, MigrateCommand

from app import app, db


app.config.from_object(os.environ['APP_SETTINGS'])

migrate = Migrate(app, db)
manager = Manager(app)

manager.add_command('db', MigrateCommand)


if __name__ == '__main__':
    manager.run()
```

In order to use Flask-Migrate we imported `Manager` as well as `Migrate` and `MigrateCommand` to our *manage.py* file. We also imported `app` and `db` so we have access to them from within the script.

First, we set our config to get our environment - based on the environment variable - created a migrate instance, with `app` and `db` as the arguments, and set up a `manager` command to initialize a `Manager` instance for our app. Finally, we added the `db` command to the `manager` so that we can run the migrations from the command line.

In order to run the migrations initialize Alembic:

```sh
$ python manage.py db init
  Creating directory /flask-by-example/migrations ... done
  Creating directory /flask-by-example/migrations/versions ... done
  Generating /flask-by-example/migrations/alembic.ini ... done
  Generating /flask-by-example/migrations/env.py ... done
  Generating /flask-by-example/migrations/README ... done
  Generating /flask-by-example/migrations/script.py.mako ... done
  Please edit configuration/connection/logging settings in
  '/flask-by-example/migrations/alembic.ini' before proceeding.
```

After you run the database initialization you will see a new folder called "migrations" in the project. This holds the setup necessary for Alembic to run migrations against the project. Inside of "migrations" you will see that it has a folder called "versions", which will contain the migration scripts as they are created.

Let's create our first migration by running the `migrate` command.

```sh
$ python manage.py db migrate
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  INFO  [alembic.autogenerate.compare] Detected added table 'results'
    Generating /flask-by-example/migrations/versions/63dba2060f71_.py
    ... done
```

Now you'll notice in your "versions" folder there is a migration file. This file is auto-generated by Alembic based on the model. You could generate (or edit) this file yourself; however, for most cases the auto-generated file will do.

Now we'll apply the upgrades to the database using the `db upgrade` command:

```sh
$ python manage.py db upgrade
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  INFO  [alembic.runtime.migration] Running upgrade  -> 63dba2060f71, empty message
```

The database is now ready for us to use in our app:

```sh
$ psql
# \c wordcount_dev
You are now connected to database "wordcount_dev" as user "michaelherman".
# \dt

                List of relations
 Schema |      Name       | Type  |     Owner
--------+-----------------+-------+---------------
 public | alembic_version | table | michaelherman
 public | results         | table | michaelherman
(2 rows)

# \d results
                                     Table "public.results"
        Column        |       Type        |                      Modifiers
----------------------+-------------------+------------------------------------------------------
 id                   | integer           | not null default nextval('results_id_seq'::regclass)
 url                  | character varying |
 result_all           | json              |
 result_no_stop_words | json              |
Indexes:
    "results_pkey" PRIMARY KEY, btree (id)
```

## Remote Migration

Finally, let's apply the migrations to the databases on Heroku. First, though, we need to add the details of the staging and production databases to the *config.py* file.

To check if we have a database set up on the staging server run:

```
$ heroku config --app wordcount-stage
=== wordcount-stage Config Vars
APP_SETTINGS: config.StagingConfig
```

> Make sure to replace `wordcount-stage` with the name of your staging app.

Since we don't see a database environment variable, we need to add the Postgres addon to the staging server. To do so, run the following command:

```sh
$ heroku addons:create heroku-postgresql:hobby-dev --app wordcount-stage
  Creating postgresql-cubic-86416... done, (free)
  Adding postgresql-cubic-86416 to wordcount-stage... done
  Setting DATABASE_URL and restarting wordcount-stage... done, v8
  Database has been created and is available
   ! This database is empty. If upgrading, you can transfer
   ! data from another database with pg:copy
  Use `heroku addons:docs heroku-postgresql` to view documentation.
```

> `hobby-dev` is the [free tier](https://addons.heroku.com/heroku-postgresql#hobby-dev) of the Heroku Postgres addon.

Now when we run `heroku config --app wordcount-stage` again we should see the connection settings for the database:

```sh
=== wordcount-stage Config Vars
APP_SETTINGS: config.StagingConfig
DATABASE_URL: postgres://azrqiefezenfrg:Zti5fjSyeyFgoc-U-yXnPrXHQv@ec2-54-225-151-64.compute-1.amazonaws.com:5432/d2kio2ubc804p7
```

Next we need to commit the changes that you've made to git and push to your staging server:

```sh
$ git push stage master
```

Run the migrations that we created to migrate our staging database by using the `heroku run` command:

```sh
$ heroku run python manage.py db upgrade --app wordcount-stage
  Running python manage.py db upgrade on wordcount-stage... up, run.5677
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  INFO  [alembic.runtime.migration] Running upgrade  -> 63dba2060f71, empty message
```

> Notice how we only ran the `upgrade`, not the `init` or `migrate` commands like before. We already have our migration file set up and ready to go; we just need to apply it against the Heroku database.

Let's now do the same for production.

1. Set up a database for your production app on Heroku, just like you did for staging: `heroku addons:create heroku-postgresql:hobby-dev --app wordcount-pro`
1. Push your changes to your production site: `git push pro master` Notice how you don't have to make any changes to the config file - it's setting the database based on the newly created `DATABASE_URL` environment variable.
1. Apply the migrations: `heroku run python manage.py db upgrade --app wordcount-pro`

Now both our staging and production sites have their databases set up and are migrated - and ready to go!

> When you apply a new migration to the production database, there could be down time. If this is an issue, you can set up database replication by adding a "follower" (commonly known as a slave) database. For more on this, check out the official Heroku [documentation](https://devcenter.heroku.com/articles/heroku-postgres-follower-databases).

## Conclusion

That's it for part 2. Comment below with questions.

*In [Part 3](/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/) we're going to build the word counting functionality and have it sent to a task queue to deal with the longer running word count processing.*

See you next time. Cheers!

<br>

<p style="font-size: 14px;">
  <em>This is a collaboration piece between Cam Linke, co-founder of <a href="http://startupedmonton.com/">Startup Edmonton</a>, and the folks at Real Python</em>
</p>