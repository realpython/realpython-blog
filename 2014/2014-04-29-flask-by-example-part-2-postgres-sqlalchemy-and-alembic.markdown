# Flask by Example - Setting up Postgres, SQLAlchemy, and Alembic

*The following is a collaboration piece between Cam Linke, co-founder of [Startup Edmonton](http://startupedmonton.com/), and the folks at Real Python.*

**Updated 02/22/2015:** Added Python 3 support.

<br>

**In this section we're going to get our database set up to store the results of our word counts. Along the way we'll set up a Postgres database, add SQLAlchemy to our app for use as an ORM, and use Alembic for our data migrations.**

Remember, here's what we're building: A Flask app that calculates word-frequency pairs based on the text from a given URL. This is a full-stack tutorial.

1. [Part One](http://www.realpython.com/blog/python/flask-by-example-part-1-project-setup): Setup a local development environment and then deploy both a staging environment and a production environment on Heroku.
1. **Part Two: Setup a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations. (current)**
1. [Part Three](https://realpython.com/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/): Add in the back-end logic to scrape and then process the counting of words from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
1. [Part Four](https://realpython.com/blog/python/flask-by-example-implementing-a-redis-task-queue/): Implement a Redis task queue to handle the text processing.
1. [Part Five](https://realpython.com/blog/python/flask-by-example-integrating-flask-and-angularjs/): Setup Angular on the front-end to continuously poll the back-end to see if the request is done.
1. [Part Six](https://realpython.com/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis, detailing how to run two processes (web and worker) on a single Dyno.
1. [Part Seven](https://realpython.com/blog/python/flask-by-example-updating-the-ui/): Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Install Requirements

Tools we'll use in this part:

- **Postgres** - [http://www.postgresql.org/](http://www.postgresql.org/)
- **Psycopg2** - [http://initd.org/psycopg/](http://initd.org/psycopg/)
- **SQLAlchemy** - [http://www.sqlalchemy.org/](http://www.sqlalchemy.org/)
- **Alembic** - [http://alembic.readthedocs.org/en/latest/](http://alembic.readthedocs.org/en/latest/)
- **Flask-Migrate** - [http://flask-migrate.readthedocs.org/en/latest/](http://flask-migrate.readthedocs.org/en/latest/)

To get started install Postgres on your local computer if you don't have it already. Since Heroku uses Postgres it will be good for us to develop locally on the same database. If you don't have Postgres installed, [Postgres.app](http://postgresapp.com/) is an easy way to get up and running quick for Mac OSX users. Once you have Postgres installed and running, create a database called *wordcount_dev* to use as our local development database. In order to use our newly created database in the Flask app we're going to need to install a few things:

```sh
$ cd wordcounts
$ pip install psycopg2 Flask-SQLAlchemy Flask-Migrate
$ pip freeze > requirements.txt
```

Psycopg is a Python adapter for Postgres, SQLAlchemy is an awesome Python ORM, and Flask-Migrate will install both that extension and Alembic which we'll use for our database migrations.

> If you're on Mavericks and having trouble installing psycopg2 check out [this](http://stackoverflow.com/questions/22313407/clang-error-unknown-argument-mno-fused-madd-python-package-installation-fa) Stack Overflow article.

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
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
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

## Model

Set up a basic model to hold the results of the wordcount by adding a *models.py* file:

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

What we're doing here is creating a table to store the results of our word counts. We first import the database connection that we created in our *app.py* file as well as JSON from SQLAlchemy's [PostgreSQL dialects](http://docs.sqlalchemy.org/en/latest/dialects/postgresql.html#sqlalchemy.dialects.postgresql.JSON). JSON columns are fairly new to Postgres and are not available in every database supported by SQLAlchemy so we need to import it specifically.

Next we create a `Result()` class and assign it a table name of `results`. We then set the attributes that we want to store for a result - the `id` of the result we stored, the `url` that we counted the words from, a full list of words that we counted, and a list of words that we counted minus stop words (more on this later).

We then create an `__init__()` method that will run the first time we create a new result and, finally, a `__repr__()` method to represent the object when we query for it.

## Local Migration

We are going to use [Alembic](https://alembic.readthedocs.org/) and [Flask-Migrate](https://flask-migrate.readthedocs.org/) to migrate our database to the latest version. Alembic is migration library for SQLAlchemy and could be used without Flask-Migrate if you want. However Flask-Migrate does help with some of the setup and makes things easier.

Create a new file called *manage.py*:

```python
from flask.ext.script import Manager
from flask.ext.migrate import Migrate, MigrateCommand
import os

from app import app, db
app.config.from_object(os.environ['APP_SETTINGS'])

migrate = Migrate(app, db)
manager = Manager(app)

manager.add_command('db', MigrateCommand)

if __name__ == '__main__':
    manager.run()
```

In order to use Flask-Migrate we need to import `Manager` as well as `Migrate` and `MigrateCommand` to our *manage.py* file. We also import `app` and `db` so we have access to them within the script.

First we set our config to get our environment - based on the environment variable - and create a migrate instance with `app` and `db` as the arguments and set up a `manager` command to initialize a `Manager` instance for our app. Finally we add the `db` command to our manager so that we can run our migrations from the command line.

In order to run our migrations initialize Alembic:

```sh
$ python manage.py db init
  Creating directory /Users/michael/repos/realpython/flask-by-example/migrations ... done
  Creating directory /Users/michael/repos/realpython/flask-by-example/migrations/versions ... done
  Generating /Users/michael/repos/realpython/flask-by-example/migrations/alembic.ini ... done
  Generating /Users/michael/repos/realpython/flask-by-example/migrations/env.py ... done
  Generating /Users/michael/repos/realpython/flask-by-example/migrations/README ... done
  Generating /Users/michael/repos/realpython/flask-by-example/migrations/script.py.mako ... done
  Please edit configuration/connection/logging settings in '/Users/michael/repos/realpython/flask-by-
  example/migrations/alembic.ini' before proceeding.
```

After you run the database initialization you will see a new folder called "migrations" in the project. This holds the setup necessary for Alembic to run migrations on the project. Inside of "migrations" you will see that it has a folder called "versions", containing the migration scripts as they are created.

Let's create our first migration by running the `migrate` command.

```sh
$ python manage.py db migrate
  INFO  [alembic.migration] Context impl PostgresqlImpl.
  INFO  [alembic.migration] Will assume transactional DDL.
  INFO  [alembic.autogenerate.compare] Detected added table 'results'
  Generating /Users/michael/repos/realpython/flask-by-example/migrations/versions/53c94f1fe77_.py ... done
```

Now you'll notice in your "versions" folder there is a migration file. This file is autogenerated by Alembic based on the model. You could generate (or edit) this file yourself; however, for a lot of cases the autogenerated file will do.

Now we'll apply our upgrades to our database using the `db upgrade` command:

```sh
$ python manage.py db upgrade
  INFO  [alembic.migration] Context impl PostgresqlImpl.
  INFO  [alembic.migration] Will assume transactional DDL.
  INFO  [alembic.migration] Running upgrade  -> 53c94f1fe77, empty message
```

Our database is now ready for us to use in our app.

## Remote Migration

Finally, let's apply the migrations to our Heroku databases. First, though, we need to add the details of our staging and production databases to our *config.py* file. To check if you have a database set up on your staging server run:

```
$ heroku config --app wordcounts-stage
=== wordcount-stage Config Vars
APP_SETTINGS: config.StagingConfig
```

> Make sure to replace `wordcount-stage` with the name of your staging app.

Since we don't see anything about a database, we need to add the Postgres addon to the staging server. To do so, run the following command to add the Postgres addon to your Heroku app:

```sh
$ heroku addons:create heroku-postgresql:hobby-dev --app wordcounts-stage
  Adding heroku-postgresql:dev on wordcount-stage... done, v8 (free)
  Attached as HEROKU_POSTGRESQL_AMBER_URL
  Database has been created and is available
   ! This database is empty. If upgrading, you can transfer
   ! data from another database with pgbackups:restore.
  Use `heroku addons:docs heroku-postgresql:dev` to view documentation.
```

> **NOTE**: `hobby-dev` is the [free tier](https://addons.heroku.com/heroku-postgresql#hobby-dev) Heroku Postgres addon.

Now when we run Heroku config again we should see the connection settings for our URL.

For example:

```sh
APP_SETTINGS:               config.StagingConfig
DATABASE_URL:               postgres://srxgxusbbyadyc:8NA0X-zdGZfPwPhfOtLuLZ5hzI@ec2-54-221-243-6.compute-1.amazonaws.com:5432/d7j5r8c30kbvuh
```

Next we need to commit the changes that you've made to git and push to your staging server:

```sh
$ git push stage master
```

Run the migrations that we created to migrate our staging database. We do this by using the `heroku run` command to run python scripts within our Heroku app. We will use this to run the same `db upgrade` command from our *manage.py* file.

```sh
$ heroku run python manage.py db upgrade --app wordcounts-stage
  Running `python manage.py db upgrade` attached to terminal... up, run.4755
  INFO  [alembic.migration] Context impl PostgresqlImpl.
  INFO  [alembic.migration] Will assume transactional DDL.
  INFO  [alembic.migration] Running upgrade None -> 20ff8063fe45, empty message
```

> Notice how we only ran the upgrade, not the `init` or `migrate` commands like before. We already have our migration setup and ready to go, we only need to run it on our Heroku database.

Let's now do the same for our production site. Set up a database for your production app on Heroku, just like you did for staging. Push your changes to your production site. Notice how you don't have to make any changes to the config file - it's setting the database based on the newly created `DATABASE_URL` environment variable.

```sh
$ heroku addons:create heroku-postgresql:hobby-dev --app wordcounts-pro
$ git push pro master
$ heroku run python manage.py db upgrade --app wordcounts-pro
```

Now both our staging and production sites have their databases set up and are migrated - and ready to go!

> When you apply a new migration to your production database, there could be down time. If this is an issue, you can setup database replication by adding a "follower" (commonly known as a slave) database. For more on this, check out the official Heroku [documentation](https://devcenter.heroku.com/articles/heroku-postgres-follower-databases).

## Sanity Check

Remember in Part 1, when we tested the environment variables to make sure the right environment was being detected by adding a print statement to *app.py* - `print(os.environ['APP_SETTINGS'])`? Well, let's do the same thing, but test the Database URIs by adding a `print` function to the bottom of *config.py*:

```python
print(os.environ['DATABASE_URL'])
```

Now let's test.

**Local**:

```
$ python config.py
postgresql://localhost/wordcount_dev
```

Commit and push again to staging and production. Now let's test it out...

**Staging**:

```sh
$ heroku run python config.py  --app wordcounts-stage
Running `python config.py` attached to terminal... up, run.1572
postgres://eccqpmccvlokrj:d0iLgQB8naQ2Pg8HL4q61G9gOd@ec2-54-235-250-41.compute-1.amazonaws.com:5432/dep90ehmacu89e
```

**Production**:

```sh
$ heroku run python config.py  --app wordcounts-pro
Running `python config.py` attached to terminal... up, run.3993
postgres://rsjezmhdfavadr:_ams4r9uEHXcGCZOcnDqqD6Pxs@ec2-54-235-250-41.compute-1.amazonaws.com:5432/d6dpkb5kmd7bg9
```

The URIs for the staging and production should match the URIs displayed when we ran the `heroku` config commands. Test this out again:

```sh
$ heroku config --app wordcounts-stage
```

and

```sh
$ heroku config --app wordcount-pro
```

## Conclusion

That's it for part 2. I hope database migrations make better sense now. Please comment below with questions. In [Part 3](https://realpython.com/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/) we're going to build the word counting functionality and have it sent to a request queue to deal with the longer running wordcount processing. See you next time. Cheers!

Don't forget to remove the print statement from the config file when done, commit, and then push back up to your various environments.
