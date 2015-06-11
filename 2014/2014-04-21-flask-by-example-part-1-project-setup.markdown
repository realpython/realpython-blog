# Flask by Example - Project Setup

*The following is a collaboration piece between Cam Linke, co-founder of [Startup Edmonton](http://startupedmonton.com/), and the folks at Real Python.*

**Updated 02/22/2015:** Added Python 3 support.

<br>

**Welcome! Today we're going to start building a Flask app that calculates word-frequency pairs based on the text from a given URL.** This is a full-stack tutorial.

<br>

1. **Part One: Setup a local development environment and then deploy both a staging environment and a production environment on Heroku. (current)**
1. [Part Two](http://www.realpython.com/blog/flask-by-example-part-2-postgres-sqlalchemy-and-alembic): Setup a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations.
1. [Part Three](https://realpython.com/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/): Add in the back-end logic to scrape and then process the counting of words from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
1. [Part Four](https://realpython.com/blog/python/flask-by-example-implementing-a-redis-task-queue/): Implement a Redis task queue to handle the text processing.
1. [Part Five](https://realpython.com/blog/python/flask-by-example-integrating-flask-and-angularjs/): Setup Angular on the front-end to continuously poll the back-end to see if the request is done.
1. [Part Six](https://realpython.com/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis, detailing how to run two processes (web and worker) on a single Dyno.
1. Part Seven: Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Setup

We'll start with a *basic "Hello World" app on Heroku with staging (or pre-production) and production environments*.

For the initial setup, we're going to use Virtualenv and Virtualenvwrapper. This will give us a few extra tools to help us silo our environment. I'm going to assume for this tutorial you've used the following tools before:

- **Virtualenv** - [http://www.virtualenv.org/en/latest/](http://www.virtualenv.org/en/latest/)
- **Virtualenvwrapper** - [http://virtualenvwrapper.readthedocs.org/en/latest/](http://virtualenvwrapper.readthedocs.org/en/latest/)
- **Flask** - [http://flask.pocoo.org/](http://flask.pocoo.org/)
- **git/Github** - [http://try.github.io/levels/1/challenges/1](http://try.github.io/levels/1/challenges/1)
- **Heroku (basics)** - [https://devcenter.heroku.com/articles/getting-started-with-python](https://devcenter.heroku.com/articles/getting-started-with-python)

First things first, let's get a working directory set up:

```sh
$ mkdir flask-by-example
```

Initialize a new git repo within your working directory:

```sh
$ git init
```

Next, we're going to use Virtualenvwrapper to set up a new virtual environment with Python 3.

```sh
$ which python3
/usr/local/bin/python3

$ mkvirtualenv --python=/usr/local/bin/python3 wordcounts
Running virtualenv with interpreter /usr/local/bin/python3
Using base prefix '/usr/local/Cellar/python3/3.4.2_1/Frameworks/Python.framework/Versions/3.4'
New python executable in wordcounts/bin/python3.4
Also creating executable in wordcounts/bin/python
Installing setuptools, pip...done.

(wordcounts)$ python
Python 3.4.2 (default, Feb 10 2015, 03:38:22)
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.56)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> exit()
```

This creates a new virtualenv for us. Along with creating a new virtualenv, it creates some new options, including *[Postactivate](http://virtualenvwrapper.readthedocs.org/en/latest/scripts.html#scripts-postactivate)*, after you run the `workon` command to start your virtual environment. This is going to help us later when we are setting up some environment variables - but for now we're also going to use it to automatically jump to our project when we first start it.

Open up the *postactivate* file. The easiest way to do this is with VIM:

```sh
$ vi $VIRTUAL_ENV/bin/postactivate
```

Add the following line to your project:

```sh
cd ~/path/to/your/project
```

*Make sure to alter the above command for your environment. For example, my working directory is within my "repos" directory, so my path is: `cd ~/repos/realpython/flask-by-example`.*

> Within VIM, press "i" to enter the INSERT mode. Paste the line in the file, then press "escape" to exit INSERT mode. Finally press ":", then "w", and finally "q" to save and exit VIM.

Now open a new terminal window and run the following command:

```sh
$ workon wordcounts
```

If all is setup properly, it will activate your environment and move you directly to the project directory. A nice timesaver.

> If you are having problems with virtualenvwrapper, just use virtualenv.

Next we're going to get our basic structure for our app set up. Add the following files to your "flask-by-example" folder:

```sh
$ touch app.py .gitignore README.md requirements.txt
```

This will give you the following structure:

```sh
├── README.md
├── app.py
├── blog.md
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

And you should see your basic Hello world app in action on [http://localhost:5000/](http://localhost:5000/). Kill the server.

Next we're going to set up our Heroku environments for both our production and staging app.

## Setup Heroku

I'm going to assume you have the Heroku [Toolbelt](https://toolbelt.heroku.com/) installed. For more basic information about setting up your Heroku app for use with Python see the following [link](https://devcenter.heroku.com/articles/getting-started-with-python).

After you have Heroku setup on your machine create a Procfile:

```sh
$ touch Procfile
```

Add the following line to your newly created file

```python
web: gunicorn app:app
```

Make sure to add gunicorn to your *requirments.txt* file

```sh
$ pip install gunicorn
$ pip freeze > requirements.txt
```

We also need to specify a Python version so that Heroku uses the right one to run our app. Simply create a file called *runtime.txt* with the following code:

```
python-3.4.2
```

Commit your changes in git and optionally PUSH to Github, then create two new Heroku apps.

One for production:

```sh
$ heroku create wordcounts-pro
```

And one for staging:

```sh
$ heroku create wordcounts-stage
```

Add your new apps to your git remotes. Make sure to name one remote *pro* (for "production") and the other *stage* (for "staging"):

```sh
$ git remote add pro git@heroku.com:YOUR_APP_NAME.git
$ git remote add stage git@heroku.com:YOUR_APP_NAME.git
```

Now we can push both of our apps live to Heroku.

- For staging: `git push stage master`
- For production: `git push pro master`

Once both of those have been pushed, open the URLs up in your web browser and see that your app is working.

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

Test it out by adding a name after the URL. For example: [http://localhost:5000/mike](http://localhost:5000/mike).

Now let's try out our changes on staging before we push them live to production. Make sure your changes are committed in git and then push your work up to staging - `git push stage master`.

Now if you navigate to your staging environment, you'll be able to use the new URL - i.e., "/mike" and get "Hello NAME" based on what you put into the URL as the output in the browser. However, if you try the same thing on the production site you will get an error. **So we can build things and test them out on staging and then when we're happy, push them live to production.**

Let's push our site to production now that we're happy with it - `git push pro master`

Now we have the same functionality live on our production site.

**This staging/production workflow allows us to make changes, show things to clients, etc., all within a sandboxed server without causing any changes to the live production site that users are, well, using.**

## Config Settings

The last thing that we're going to do is set up different config environments for our app. Often there are things that are going to be different between your local, staging, and production setups. You’ll want to connect to different databases, have different AWS keys, etc. Let’s set up a config file to deal with the different environments.

Add a *config.py* file to your project:

```sh
$ touch config.py
```

With our config file we're going to borrow a bit from how Django's config is set up. We'll have a base config class that the other config classes inherit from. Then we'll import the appropriate config class as needed.

Add the following to your newly created *config.py* file:

```python
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

We set up a base `Config` class with some basic setup that our other config classes inherit from. Now we'll be able to import the appropriate config class based on the current environment. Thus, we can use environment variables to choose which settings we’re going to use based on the environment (e.g., local, staging, production).

### Local Settings

To set up our `APP_SETTINGS` variable locally, we can use our Virtualenvwrapper *postactivate* file again.

Add the following line to your *postactivate* file:

```sh
export APP_SETTINGS="config.DevelopmentConfig"
```

Reload your environment by running the `workon wordcounts` command again:

```
$ workon wordcounts
```

Now when you run your app it will import the configuration that you set up in your `DevelopmentConfig` class.

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

To make sure we use the right environment change *app.py*:

```python
from flask import Flask
import os

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
$ heroku run python app.py --app wordcounts-stage
Running `python app.py` attached to terminal... up, run.2830
config.StagingConfig
```

**Production**:

```sh
$ heroku run python app.py --app wordcounts-pro
Running `python app.py` attached to terminal... up, run.1360
config.ProductionConfig
```

Be sure to remove `print(os.environ['APP_SETTINGS'])` when done, commit, and push back up to your various environments.

## Conclusion

With the setup out of the way, we're going to start to build out the word counting functionality of this app. Along the way, we'll add a request queue to set up background processing for the word count portion, as well dig further into our Heroku setup by setting up the configuration and migrations for our database ([part2](http://www.realpython.com/blog/flask-by-example-part-2-postgres-sqlalchemy-and-alembic)) which we'll use to store our word count results.

Best!

> Grab the code from the [repo](https://github.com/realpython/flask-by-example/releases).