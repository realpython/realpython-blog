# Flask by Example - Project Setup

**Welcome! Today we're going to start building a Flask app that calculates word-frequency pairs based on the text from a given URL.** This is a full-stack tutorial.

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask_by_example/flask-by-example-part1.png" style="max-width: 100%;" alt="flask by example part 1">
</div>

<br>

*Updates:*

  - 03/22/2016: Upgraded to Python version [3.5.1](https://www.python.org/downloads/release/python-351/), and added autoenv version [1.0.0](https://pypi.python.org/pypi/autoenv/1.0.0).
  - 02/22/2015: Added Python 3 support.

<hr><br>

1. **Part One: Set up a local development environment and then deploy both a staging and a production environment on Heroku. (_current_)**
1. [Part Two](/blog/python/flask-by-example-part-2-postgres-sqlalchemy-and-alembic): Set up a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations.
1. [Part Three](/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/): Add in the back-end logic to scrape and then process the word counts from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
1. [Part Four](/blog/python/flask-by-example-implementing-a-redis-task-queue/): Implement a Redis task queue to handle the text processing.
1. [Part Five](/blog/python/flask-by-example-integrating-flask-and-angularjs/): Set up Angular on the front-end to continuously poll the back-end to see if the request is done processing.
1. [Part Six](/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis and detailing how to run two processes (web and worker) on a single Dyno.
1. [Part Seven](/blog/python/flask-by-example-updating-the-ui/): Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Project Setup

We'll start with a *basic "Hello World" app on Heroku with staging (or pre-production) and production environments*.

For the initial setup, you should have some familiarity with the following tools:

- **Virtualenv** - [http://www.virtualenv.org/en/latest/](http://www.virtualenv.org/en/latest/)
- **Flask** - [http://flask.pocoo.org/](http://flask.pocoo.org/)
- **git/Github** - [http://try.github.io/levels/1/challenges/1](http://try.github.io/levels/1/challenges/1)
- **Heroku (basics)** - [https://devcenter.heroku.com/articles/getting-started-with-python](https://devcenter.heroku.com/articles/getting-started-with-python)

First things first, let's get a working directory set up:

```sh
$ mkdir flask-by-example && cd flask-by-example
```

Initialize a new git repo within your working directory:

```sh
$ git init
```

Set up a virtual environment to use for our application:

```sh
$ pyvenv-3.5 env
$ source env/bin/activate
```

You should now see you `(env)` to the left of the prompt in the terminal, indicating that you are now working in a virtual environment.

> In order to leave your virtual environment, just run `deactivate`, and then run `source env/bin/activate` when you are ready to work on your project again.

Next we're going to get our basic structure for our app set up. Add the following files to your "flask-by-example" folder:

```sh
$ touch app.py .gitignore README.md requirements.txt
```

This will give you the following structure:

```sh
├── .gitignore
├── app.py
├── README.md
└── requirements.txt
```

Be sure to update the *.gitignore* file from the [repo](https://github.com/realpython/flask-by-example).

Next install Flask:

```sh
$ pip install Flask==0.10.1
```

Add the installed libraries to our *requirements.txt* file:

```sh
$ pip freeze > requirements.txt
```

Open up *app.py* in your favorite editor and add the following code:

```python
from flask import Flask
app = Flask(__name__)


@app.route('/')
def hello():
    return "Hello World!"

if __name__ == '__main__':
    app.run()
```

Run the app:

```sh
$ python app.py
```

And you should see your basic "Hello world" app in action on [http://localhost:5000/](http://localhost:5000/). Kill the server when done.

Next we're going to set up our Heroku environments for both our production and staging app.

## Heroku Setup

If you haven't already, create a Heroku account, download and install the Heroku [Toolbelt](https://toolbelt.heroku.com/), and then in your terminal run `heroku login` to log in to Heroku.

Once done, create a [Procfile](https://devcenter.heroku.com/articles/procfile) in your project's root directory:

```sh
$ touch Procfile
```

Add the following line to your newly created file

```sh
web: gunicorn app:app
```

Make sure to add [gunicorn](http://gunicorn.org/) to your *requirments.txt* file

```sh
$ pip install gunicorn==19.4.5
$ pip freeze > requirements.txt
```

We also need to specify a Python version so that Heroku uses the right [Python Runtime](https://devcenter.heroku.com/articles/python-runtimes) to run our app with. Simply create a file called *runtime.txt* with the following code:

```sh
python-3.5.1
```

Commit your changes in git (and optionally push to Github), then create two new Heroku apps.

One for production:

```sh
$ heroku create wordcount-pro
```

And one for staging:

```sh
$ heroku create wordcount-stage
```

> These names are now taken, so you will have to make your Heroku app name unique.

Add your new apps to your git remotes. Make sure to name one remote *pro* (for "production") and the other *stage* (for "staging"):

```sh
$ git remote add pro git@heroku.com:YOUR_APP_NAME.git
$ git remote add stage git@heroku.com:YOUR_APP_NAME.git
```

Now we can push both of our apps live to Heroku.

- For staging: `git push stage master`
- For production: `git push pro master`

Once both of those have been pushed, open the URLs up in your web browser and if all went well you should see your app live in both environments.

## Staging/Production Workflow

Let's make a change to our app and push only to staging:

```python
from flask import Flask
app = Flask(__name__)


@app.route('/')
def hello():
    return "Hello World!"


@app.route('/<name>')
def hello_name(name):
    return "Hello {}!".format(name)

if __name__ == '__main__':
    app.run()
```

Run your app locally to make sure everything works - `python app.py`

Test it out by adding a name after the URL - i.e., [http://localhost:5000/mike](http://localhost:5000/mike).

Now let's try out our changes on staging before we push them live to production. Make sure your changes are committed in git and then push your work up to the staging environment - `git push stage master`.

Now if you navigate to your staging environment, you'll be able to use the new URL - i.e., "/mike" and get "Hello NAME" based on what you put into the URL as the output in the browser. However, if you try the same thing on the production site you will get an error. **So we can build things and test them out in the staging environment and then when we're happy with the changes, we can push them live to production.**

Let's push our site to production now that we're happy with it - `git push pro master`

Now we have the same functionality live on our production site.

**This staging/production workflow allows us to make changes, show things to clients, experiment, etc. - all within a sandboxed server without causing any changes to the live production site that users are, well, using.**

## Config Settings

The last thing that we're going to do is set up different config environments for our app. Often there are things that are going to be different between your local, staging, and production setups. You’ll want to connect to different databases, have different AWS keys, etc. Let’s set up a config file to deal with the different environments.

Add a *config.py* file to your project root:

```sh
$ touch config.py
```

With our config file we're going to borrow a bit from how Django's config is set up. We'll have a base config class that the other config classes inherit from. Then we'll import the appropriate config class as needed.

Add the following to your newly created *config.py* file:

```python
import os
basedir = os.path.abspath(os.path.dirname(__file__))


class Config(object):
    DEBUG = False
    TESTING = False
    CSRF_ENABLED = True
    SECRET_KEY = 'this-really-needs-to-be-changed'


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

We imported `os` and then set the `basedir` variable as a relative path from any place we call it to this file. We then set up a base `Config()` class with some basic setup that our other config classes inherit from. Now we'll be able to import the appropriate config class based on the current environment. Thus, we can use environment variables to choose which settings we’re going to use based on the environment - e.g., local, staging, production.

### Local Settings

To set up our application with environment variables, we're going to use [autoenv](https://github.com/kennethreitz/autoenv). This program allows us to set commands that will run every time we cd into our directory. In order to use it, we will need to install it globally. First, exit out of your virtual environment in the terminal, install autoenv, then and add a *.env* file:

```sh
$ deactivate
$ pip install autoenv==1.0.0
$ touch .env
```

Next, in your *.env* file, add the following:

```sh
source env/bin/activate
export APP_SETTINGS="config.DevelopmentConfig"
```

Run the following to update then refresh your *.bashrc*:

```sh
$ echo "source `which activate.sh`" >> ~/.bashrc
$ source ~/.bashrc
```

Now, if you move up a directory and then cd back into it, the virtual environment will automatically be started and the `APP_SETTINGS` variable is declared.

### Heroku Settings

Similarly we’re going to set environment variables on Heroku.

For staging run the following command:

```sh
$ heroku config:set APP_SETTINGS=config.StagingConfig --remote stage
```

For production:

```sh
$ heroku config:set APP_SETTINGS=config.ProductionConfig --remote pro
```

To make sure that we use the right environment change *app.py*:

```python
import os
from flask import Flask


app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])


@app.route('/')
def hello():
    return "Hello World!"


@app.route('/<name>')
def hello_name(name):
    return "Hello {}!".format(name)

if __name__ == '__main__':
    app.run()
```

We imported `os` and used the `os.environ` method to import the appropriate `APP_SETTINGS` variables, depending on our environment. We then set up the config in our app with the `app.config.from_object` method.

Commit and push your changes to both staging and production (and Github if you have it setup).

Want to test the environment variables out to make sure it's detecting the right environment (sanity check!)? Add a print statement to *app.py*:

```python
print(os.environ['APP_SETTINGS'])
```

Now when you run the app, it will show which config settings it's importing:

**Local**:

```sh
$ python app.py
config.DevelopmentConfig
```

Commit and push again to staging and production. Now let's test it out...

**Staging**:

```sh
$ heroku run python app.py --app wordcount-stage
Running python app.py on wordcount-stage... up, run.7699
config.StagingConfig
```

**Production**:

```sh
$ heroku run python app.py --app wordcount-pro
Running python app.py on wordcount-pro... up, run.8934
config.ProductionConfig
```

Be sure to remove `print(os.environ['APP_SETTINGS'])` when done, commit, and push back up to your various environments.

## Conclusion

With the setup out of the way, we're going to start to build out the word counting functionality of this app. Along the way, we'll add a [task queue](https://www.fullstackpython.com/task-queues.html) to set up background processing for the word count portion, as well dig further into our Heroku setup by setting up the configuration and migrations for our database ([part2](/blog/python/flask-by-example-part-2-postgres-sqlalchemy-and-alembic)) which we'll use to store our word count results. Best!

Grab the code from the [repo](https://github.com/realpython/flask-by-example/releases).

<br>

<p style="font-size: 14px;">
  <em>This is a collaboration piece between Cam Linke, co-founder of <a href="http://startupedmonton.com/">Startup Edmonton</a>, and the folks at Real Python</em>
</p>