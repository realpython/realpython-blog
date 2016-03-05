# Flask by Example - Updating the Staging Environment

**In this part, we'll update the staging environment, with our latest code changes, by first setting up Redis on Heroku and then looking at how to run both our web and worker processes on a single dyno.**

**Updated 02/22/2015:** Added Python 3 support.

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask_by_example/heroku.png" alt="heroku logo">
</div>

<br>

Remember, here's what we're building: A Flask app that calculates word-frequency pairs based on the text from a given URL. This is a full-stack tutorial.

1. [Part One](http://www.realpython.com/blog/python/flask-by-example-part-1-project-setup): Setup a local development environment and then deploy both a staging environment and a production environment on Heroku.
1. [Part Two](http://www.realpython.com/blog/flask-by-example-part-2-postgres-sqlalchemy-and-alembic): Setup a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations.
1. [Part Three](https://realpython.com/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/): Add in the back-end logic to scrape and then process the counting of words from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
1. [Part Four](https://realpython.com/blog/python/flask-by-example-implementing-a-redis-task-queue/): Implement a Redis task queue to handle the text processing.
1. [Part Five](https://realpython.com/blog/python/flask-by-example-integrating-flask-and-angularjs/): Setup Angular on the front-end to continuously poll the back-end to see if the request is done.
1. **Part Six: Push to the staging server on Heroku - setting up Redis, detailing how to run two processes (web and worker) on a single Dyno. (current)**
1. [Part Seven](https://realpython.com/blog/python/flask-by-example-updating-the-ui/): Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Test Push

Let's start with pushing up the code in its current state and see what needs to be fixed.

```sh
$ cd wordcounts
$ source env/bin/activate
$ export APP_SETTINGS="config.DevelopmentConfig"
$ export DATABASE_URL="postgresql://localhost/wordcount_dev"
$ git add -A
$ git commit -m "added angular and the backend worker process"
$ git push stage master
$ heroku open --app wordcounts-stage
```

> Make sure to replace `wordcounts-stage` with the name of your app.

First off, if you open your app using the HTTP secure ('https://') protocol, notice how none of the external JavaScript or CSS files are loading. To fix that, remove the 'http:' from the links in the *index.html* file.

For example:

```html
<link href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css" rel="stylesheet" media="screen">
```

Update all of the files, then commit and push your changes back up to Heroku. Now regardless of whether you view the site from 'https' or 'http', the external files will load correctly.

Next, try to run a quick test to see if the wordcount feature works. Nothing should happen. Why? Well, If you open the "Network" tab in "Chrome Developer Tools", you'll see that the post request to the `/start` endpoint returned a 500 (Internal Server Error) status code.

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask_by_example/heroku_https.png">
</div>

<br>

Think about how we run this locally: We have a worker process and the Redis server running along with the Flask development server. The same needs to happen on Heroku.

## Redis

Start by adding Redis to the staging app:

```sh
$ heroku addons:add redistogo --app wordcounts-stage
```

You can test to make sure the `REDISTOGO_URL` has been set as an environment variable with the following command:

```
$ heroku config --app wordcounts-stage | grep REDISTOGO_URL
```

We need to make sure we're linking to the Redis URI in our code, which actually is already set up. Open up *worker.py* and find this code:

```python
redis_url = os.getenv('REDISTOGO_URL', 'redis://localhost:6379')
```

Here we first try to use the URI associated with the environment variable. And if that variable does not exist (like in our local environment), then we use the `redis://localhost:6379` URI. Perfect.

> Be sure to check out the [official Heroku documentation](https://devcenter.heroku.com/articles/redistogo) for more on working with Redis.

With Redis setup, we now just need to get our worker process running.

## Worker

Heroku allows you to run one free dyno. And you *should* run one process per dyno - meaning our web process should be in one dyno and the worker process should be in another. However, since we're working on a small project, there is a workaround that we can employ to run both processes on one dyno. Keep in mind that this method is **not** recommended for larger projects, as the process will not properly scale as traffic increases.

First, add a bash script called *heroku.sh* to the Project root directory:

```bash
#!/bin/bash
gunicorn app:app --daemon
python worker.py
```

Then, update the *Procfile*, replace the code in there with the following:

```
web: sh heroku.sh
```

Now both the web (demonized, in the background) and worker (in the foreground) processes are ran under the web process in the *Procfile*.

> Please note that there are other ways of running a web and worker for free on Heroku. We'll look at an alternative method in a future post (if there's interest).

Let's test this out locally before pushing to the staging server. In a new terminal window, run the Redis server - `redis-server`. Then run `foreman start`. Navigate to [http://localhost:5000/](http://localhost:5000/) and test the application out. It should work.

Commit your changes, and then push to Heroku. Test it out.

## Conclusion

Homework! Although we have much more to do, the application does work - so let's get an iteration out there for the world to see. Go ahead and update the production environment, using the same workflow.

**Links:**

1. [Repo](https://github.com/realpython/flask-by-example/releases)
1. [My Staging app](http://wordcounts-stage.herokuapp.com/)
1. [My Production app](http://wordcounts-pro.herokuapp.com/)

Leave questions and comments below.
