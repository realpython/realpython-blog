# Flask by Example - Implementing a Redis Task Queue

**This part of the tutorial details how to implement a Redis task queue to handle text processing.**

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask_by_example/flask-by-example-part4.png" style="max-width: 100%;" alt="flask by example part 4">
</div>

<br>

*Updates:*

  - 03/22/2016: Upgraded to Python version [3.5.1](https://www.python.org/downloads/release/python-351/) as well as the latest versions of Redis, Python Redis, and RQ. See [below](#install-requirements) for details.
  - 02/22/2015: Added Python 3 support.

<hr><br>

Remember: Here's what we're building - A Flask app that calculates word-frequency pairs based on the text from a given URL.

<br>

1. [Part One](/blog/python/flask-by-example-part-1-project-setup/): Set up a local development environment and then deploy both a staging and a production environment on Heroku.
1. [Part Two](/blog/python/flask-by-example-part-2-postgres-sqlalchemy-and-alembic): Set up a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations.
1. [Part Three](/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/): Add in the back-end logic to scrape and then process the word counts from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
1. **Part Four: Implement a Redis task queue to handle the text processing. (_current_)**
1. [Part Five](/blog/python/flask-by-example-integrating-flask-and-angularjs/): Set up Angular on the front-end to continuously poll the back-end to see if the request is done processing.
1. [Part Six](/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis and detailing how to run two processes (web and worker) on a single Dyno.
1. [Part Seven](/blog/python/flask-by-example-updating-the-ui/): Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Install Requirements

Tools used:

- Redis ([3.0.7](https://raw.githubusercontent.com/antirez/redis/3.0/00-RELEASENOTES))
- Python Redis ([2.10.5](https://pypi.python.org/pypi/redis/2.10.5))
- RQ ([0.5.6](https://pypi.python.org/pypi/rq/0.5.6)) - a simple library for creating a task queue

Start by downloading and installing Redis from either [the official site](http://redis.io/download) or via Homebrew (`brew install redis`). Once installed, start the Redis server:

```sh
$ redis-server
```

Next install Python Redis and RQ in a new terminal window:

```sh
$ cd flask-by-example
$ pip install redis==2.10.5 rq==0.5.6
$ pip freeze > requirements.txt
```

## Set up the Worker

Let's start by creating a worker process to listen for queued tasks. Create a new file *worker.py*, and add this code:

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

Here, we listened for a queue called `default` and established a connection to the Redis server on `localhost:6379`.

Fire this up in another terminal window:

```sh
$ cd flask-by-example
$ python worker.py
17:01:29 RQ worker started, version 0.5.6
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
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
db = SQLAlchemy(app)

q = Queue(connection=conn)

from models import *
```

`q = Queue(connection=conn)` set up a Redis connection and initialized a queue based on that connection.

Move the text processing functionality out of our index route and into a new function called `count_and_save_words()`. This function accepts one argument, a URL, which we will pass to it when we call it from our index route.

```python
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

Here we used the queue that we initialized earlier and called the `enqueue_call()` function. This added a new job to the queue and that job ran the `count_and_save_words()` function with the URL as the argument. The `result_ttl=5000` line argument tells RQ how long to hold on to the result of the job for - 5,000 seconds, in this case. Then we outputted the job id to the terminal. This id is needed to see if the job is done processing.

Let's setup a new route for that...

## Get Results

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

Fire up the server, navigate to [http://localhost:5000/](http://localhost:5000/), use the URL [http://realpython.com](http://realpython.com), and grab the job id from the terminal. Then use that id in the '/results/' endpoint - i.e., [http://localhost:5000/results/ef600206-3503-4b87-a436-ddd9438f2197](http://localhost:5000/results/ef600206-3503-4b87-a436-ddd9438f2197).

As long as less than 5,000 seconds have elapsed before you check the status, then you should see an id number, which is generated when we add the results to the database:

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

Test this out again. If all went well, you should see something similar to in your browser:

```sh
{
  Course: 5,
  Python: 19,
  Real: 11,
  course: 4,
  courses: 7,
  development: 7,
  product: 4,
  sample: 4,
  videos: 5,
  web: 12
}
```

## What's Next?

*In [Part 5](/blog/python/flask-by-example-integrating-flask-and-angularjs/) we'll bring the client and server together by adding Angular into the mix to create a [poller](https://en.wikipedia.org/wiki/Polling_%28computer_science%29), which will send a request every five seconds to the `/results/<job_key>` endpoint asking for updates. Once the data is available, we'll add it to the DOM.*

Cheers!

<br>

<p style="font-size: 14px;">
  <em>This is a collaboration piece between Cam Linke, co-founder of <a href="http://startupedmonton.com/">Startup Edmonton</a>, and the folks at Real Python</em>
</p>