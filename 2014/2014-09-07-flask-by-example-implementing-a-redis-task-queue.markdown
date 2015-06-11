---
layout: post
title: "Flask by Example - Implementing a Redis Task Queue"
date: 2014-09-08 05:52:29 -0500
toc: true
comments: true
category_side_bar: true
categories: [python, flask]

keywords: "python, web development, flask, heroku, requests, nltk, beautiful soup, text processing"
description: "This tutorial shows you how to process text and then setup a request queue with Flask. In part four, we're implementing a Redis task queue to handle the text processing."
---

*The following is a collaboration piece between Cam Linke, co-founder of [Startup Edmonton](http://startupedmonton.com/), and the folks at Real Python.*

**Updated 02/22/2015:** Added Python 3 support.

<br>

**In this section, we'll add a basic Redis task queue to handle the text processing.**

There are a number of tools to do this, such as [ReTask](http://retask.readthedocs.org/en/latest/) and [HotQueue](http://richardhenry.github.io/hotqueue/). We are going to use [Python RQ](http://python-rq.org/). It's a really simple library for creating a task queue built on top of Redis that's easy to set up and implement.

Remember, here's what we're building: A Flask app that calculates word-frequency pairs based on the text from a given URL. This is a full-stack tutorial.

1. [Part One](http://www.realpython.com/blog/python/flask-by-example-part-1-project-setup): Setup a local development environment and then deploy both a staging environment and a production environment on Heroku.
1. [Part Two](http://www.realpython.com/blog/flask-by-example-part-2-postgres-sqlalchemy-and-alembic): Setup a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations.
1. [Part Three](https://realpython.com/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/): Add in the back-end logic to scrape and then process the counting of words from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
1. **Part Four: Implement a Redis task queue to handle the text processing. (current)**
1. [Part Five](https://realpython.com/blog/python/flask-by-example-integrating-flask-and-angularjs/): Setup Angular on the front-end to continuously poll the back-end to see if the request is done.
1. [Part Six](https://realpython.com/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis, detailing how to run two processes (web and worker) on a single Dyno.
1. Part Seven: Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Install Requirements

Tools:

- **Redis** - [http://redis.io/](http://redis.io/)
- **RQ (Redis Queue)** - [http://python-rq.org/](http://python-rq.org/)

Start by downloading and installing Redis from [http://redis.io/](http://redis.io/) (or use Homebrew - `brew install redis`), then start the Redis server:

```sh
$ redis-server
```

Next install the Python Redis library as well as the RQ Library in a new terminal window:

```sh
$ workon wordcounts
$ pip install redis rq
$ pip freeze > requirements.txt
```

## Setup the Worker

Let's start by creating a worker process to listen for queued tasks:

```python
import os

import redis
from rq import Worker, Queue, Connection

listen = ['default']

redis_url = os.getenv('REDISTOGO_URL', 'redis://localhost:6379')

conn = redis.from_url(redis_url)

if __name__ == '__main__':
    with Connection(conn):
        worker = Worker(list(map(Queue, listen)))
        worker.work()
```

Save this as *worker.py*. Here, we listen for a queue called `default` and establish a connection to our Redis server on `localhost:6379`.

Fire this up in another terminal window:

```sh
$ workon wordcounts
$ python worker.py
17:01:29 RQ worker started, version 0.4.6
17:01:29
17:01:29 *** Listening on default...
```

Now we need to update our *app.py* to send jobs to the queue...

## Update *app.py*

Add the following imports to *app.py*:

```python
from rq import Queue
from rq.job import Job
from worker import conn
```

Then update the configuration section:

```python
app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
db = SQLAlchemy(app)

q = Queue(connection=conn)

from models import *
```

`q = Queue(connection=conn)` sets up a Redis connection and initializes a queue based on that connection.

Now let's move the text processing functionality out of our index route and into a new function called `count_and_save_words()`. This function accepts one argument, a URL, which we will pass to it when we call it from our index route.

```python
##########
# helper #
##########

def count_and_save_words(url):

    errors = []

    try:
        r = requests.get(url)
    except:
        errors.append(
            "Unable to get URL. Please make sure it's valid and try again."
        )
        return {"error": errors}

    # text processing
    raw = BeautifulSoup(r.text).get_text()
    nltk.data.path.append('./nltk_data/')  # set the path
    tokens = nltk.word_tokenize(raw)
    text = nltk.Text(tokens)

    # remove punctuation, count raw words
    nonPunct = re.compile('.*[A-Za-z].*')
    raw_words = [w for w in text if nonPunct.match(w)]
    raw_word_count = Counter(raw_words)

    # stop words
    no_stop_words = [w for w in raw_words if w.lower() not in stops]
    no_stop_words_count = Counter(no_stop_words)

    # save the results
    try:
        result = Result(
            url=url,
            result_all=raw_word_count,
            result_no_stop_words=no_stop_words_count
        )
        db.session.add(result)
        db.session.commit()
        return result.id
    except:
        errors.append("Unable to add item to database.")
        return {"error": errors}


##########
# routes #
##########

@app.route('/', methods=['GET', 'POST'])
def index():
    results = {}
    if request.method == "POST":
        # get url that the person has entered
        url = request.form['url']
        if 'http://' not in url[:7]:
            url = 'http://' + url
        job = q.enqueue_call(
            func=count_and_save_words, args=(url,), result_ttl=5000
        )
        print(job.get_id())

    return render_template('index.html', results=results)
```

Take note of the following code:

```python
job = q.enqueue_call(
    func=count_and_save_words, args=(url,), result_ttl=5000
)
print(job.get_id())
```

Here we use the queue that we initialized earlier and call the `enqueue_call()` function. This adds a new job to our queue and that job runs the `count_and_save_words()` function with the URL as the argument. The `result_ttl=5000` line argument tells RQ how long to hold on to the result of the job for - 5 seconds. Then we output the job id to the terminal. This id is needed to see if the job is done processing.

Let's setup a new route for that...

## New Route

```python
@app.route("/results/<job_key>", methods=['GET'])
def get_results(job_key):

    job = Job.fetch(job_key, connection=conn)

    if job.is_finished:
        return str(job.result), 200
    else:
        return "Nay!", 202
```

Let's test this out.

Fire up the server, navigate to [http://localhost:5000/](http://localhost:5000/), use the URL [http://realpython.com](http://realpython.com), and then grab the job id from the terminal. Then use that id in the '/results/' endpoint.

For example: [http://localhost:5000/results/ef600206-3503-4b87-a436-ddd9438f2197](http://localhost:5000/results/ef600206-3503-4b87-a436-ddd9438f2197).

As long as 5 seconds has elapsed before you check the status, then you should see an id number, which is generated when we add the results to the database:

```python
# save the results
try:
    from models import Result
    result = Result(
        url=url,
        result_all=raw_word_count,
        result_no_stop_words=no_stop_words_count
    )
    db.session.add(result)
    db.session.commit()
    return result.id
```

Now, let's refactor the route slightly to return the actual results from the database in JSON:

```python
@app.route("/results/<job_key>", methods=['GET'])
def get_results(job_key):

    job = Job.fetch(job_key, connection=conn)

    if job.is_finished:
        result = Result.query.filter_by(id=job.result).first()
        results = sorted(
            result.result_no_stop_words.items(),
            key=operator.itemgetter(1),
            reverse=True
        )[:10]
        return jsonify(results)
    else:
        return "Nay!", 202
```

Make sure to add the import:

```python
from flask import jsonify
```

Test this out again. If all went well, you should see the following results in your browser:

```javascript
{
    Course: 5,
    Download: 4,
    Python: 19,
    Real: 11,
    courses: 7,
    development: 7,
    return: 4,
    sample: 4,
    videos: 5,
    web: 12
}
```

## What's Next?

In order to bring everything together, we'll add AngularJS into the mix, in the next [part](https://realpython.com/blog/python/flask-by-example-integrating-flask-and-angularjs/), to create a basic means of polling the server side - sending a request every five seconds or so to the `/results/<job_key>` endpoint, asking for updates. Once the data is available, we'll add it to the DOM.

> Curious about Redis and what it's used for? Check out this excellent [presentation](http://www.slideshare.net/dvirsky/kicking-ass-with-redis).

Cheers!