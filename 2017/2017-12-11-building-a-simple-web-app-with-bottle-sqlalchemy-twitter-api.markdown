# Building a Simple Web App with Bottle, SQLAlchemy, and the Twitter API

<div class="center-text">
  <img class="no-border" src="/images/blog_images/bottle-daily-python-tip/pytip-what-we-are-building.png" style="max-width: 100%;" alt="daily python tip">
</div>

*This is a guest blog post by Bob Belderbos. Bob is a driven Pythonista working as a software developer at Oracle. He is also co-founder of [PyBites](https://pybit.es/), a Python blog featuring code challenges, articles, and news. Bob is passionate about automation, data, web development, code quality, and mentoring other developers.*

<hr>

Last October we [challenged](https://pybit.es/codechallenge40.html) our PyBites' audience to make a web app to better navigate the [Daily Python Tip](https://twitter.com/python_tip) feed. In this article, I'll share what I built and learned along the way.

In this article you will learn:

1. How to clone the project repo and set up the app.
1. How to use the Twitter API via the Tweepy module to load in the tweets.
1. How to use SQLAlchemy to store and manage the data (tips and hashtags).
1. How to build a simple web app with Bottle, a micro web-framework similar to Flask.
1. How to use the pytest framework to add tests.
1. How Better Code Hub's guidance led to more maintainable code.

If you want to follow along, reading the code in detail (and possibly contribute), I suggest you fork [the repo](https://github.com/pybites/pytip). Let's get started.

## Project Setup

First, *Namespaces are one honking great idea* so let's do our work in a virtual environment. Using Anaconda I create it like so:

```sh
$ virtualenv -p <path-to-python-to-use> ~/virtualenvs/pytip
```

Create a production and a test database in Postgres:

```sh
$ psql
psql (9.6.5, server 9.6.2)
Type "help" for help.

# create database pytip;
CREATE DATABASE
# create database pytip_test;
CREATE DATABASE
```

We'll need credentials to connect to the the database and the Twitter API ([create a new app](https://apps.twitter.com/) first). As per best practice configuration should be stored in the environment, not the code. Put the following env variables at the end of *~/virtualenvs/pytip/bin/activate*, the script that handles activation / deactivation of your virtual environment, making sure to update the variables for your environment:

```
export DATABASE_URL='postgres://postgres:password@localhost:5432/pytip'
# twitter
export CONSUMER_KEY='xyz'
export CONSUMER_SECRET='xyz'
export ACCESS_TOKEN='xyz'
export ACCESS_SECRET='xyz'
# if deploying it set this to 'heroku'
export APP_LOCATION=local
```

In the deactivate function of the same script, I unset them so we keep things out of the shell scope when deactivating (leaving) the virtual environment:

```
unset DATABASE_URL
unset CONSUMER_KEY
unset CONSUMER_SECRET
unset ACCESS_TOKEN
unset ACCESS_SECRET
unset APP_LOCATION
```

Now is a good time to activate the virtual environment:

```sh
$ source ~/virtualenvs/pytip/bin/activate
```

Clone the repo and, with the virtual environment enabled, install the requirements:

```sh
$ git clone https://github.com/pybites/pytip && cd pytip
$ pip install -r requirements.txt
```

Next, we import the collection of tweets with:

```sh
$ python tasks/import_tweets.py
```

Then, verify that the tables were created and the tweets were added:

```sh
$ psql

\c pytip

pytip=# \dt
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | hashtags | table | postgres
 public | tips     | table | postgres
(2 rows)

pytip=# select count(*) from tips;
 count
-------
   222
(1 row)

pytip=# select count(*) from hashtags;
 count
-------
    27
(1 row)

pytip=# \q
```

Now let's run the tests:

```sh
$ pytest
========================== test session starts ==========================
platform darwin -- Python 3.6.2, pytest-3.2.3, py-1.4.34, pluggy-0.4.0
rootdir: realpython/pytip, inifile:
collected 5 items

tests/test_tasks.py .
tests/test_tips.py ....

========================== 5 passed in 0.61 seconds ==========================
```

And lastly run the Bottle app with:

```sh
$ python app.py
```

Browse to [http://localhost:8080](http://localhost:8080) and voilà: you should see the tips sorted descending on popularity. Clicking on a hashtag link at the left, or using the search box, you can easily filter them. Here we see the *pandas* tips for example:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/bottle-daily-python-tip/pytip-pandas.png" style="max-width: 100%;" alt="daily python tip">
</div>

> The design I made with [MUI](https://www.muicss.com/) - a lightweight CSS framework that follows Google's Material Design guidelines.

## Implementation Details

### The DB and SQLAlchemy

I used [SQLAlchemy](https://www.sqlalchemy.org/) to interface with the DB to prevent having to write a lot of (redundant) SQL.

In *[tips/models.py](https://github.com/pybites/pytip/blob/master/tips/models.py)*, we define our models - `Hashtag` and `Tip` - that SQLAlchemy will map to DB tables:

```python
from sqlalchemy import Column, Sequence, Integer, String, DateTime
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()


class Hashtag(Base):
    __tablename__ = 'hashtags'
    id = Column(Integer, Sequence('id_seq'), primary_key=True)
    name = Column(String(20))
    count = Column(Integer)

    def __repr__(self):
        return "<Hashtag('%s', '%d')>" % (self.name, self.count)


class Tip(Base):
    __tablename__ = 'tips'
    id = Column(Integer, Sequence('id_seq'), primary_key=True)
    tweetid = Column(String(22))
    text = Column(String(300))
    created = Column(DateTime)
    likes = Column(Integer)
    retweets = Column(Integer)

    def __repr__(self):
        return "<Tip('%d', '%s')>" % (self.id, self.text)
```

In *[tips/db.py](https://github.com/pybites/pytip/blob/master/tips/db.py)*, we import these models, and now it's easy to work with the DB, for example to interface with the `Hashtag` model:

```python
def get_hashtags():
    return session.query(Hashtag).order_by(Hashtag.name.asc()).all()
```

And:

```python
def add_hashtags(hashtags_cnt):
    for tag, count in hashtags_cnt.items():
        session.add(Hashtag(name=tag, count=count))
    session.commit()
```

### Query the Twitter API

We need to retrieve the data from Twitter. For that, I created *[tasks/import_tweets.py](https://github.com/pybites/pytip/blob/master/tasks/import_tweets.py)*. I packaged this under *tasks* because it should be run in a daily cronjob to look for new tips and update stats (number of likes and retweets) on existing tweets. For the sake of simplicity I have the tables recreated daily. If we start to rely on FK relations with other tables we should definitely choose update statements over delete+add.

We used this script in the Project Setup. Let's see what it does in more detail.

First, we create an API session object which we pass to [tweepy.Cursor](http://docs.tweepy.org/en/v3.5.0/cursor_tutorial.html). This feature of the API is really nice: it deals with pagination, iterating through the timeline. For the amount of tips - 222 at the time I write this - it's really fast. The `exclude_replies=True` and `include_rts=False` arguments are convenient because we only want Daily Python Tip's own tweets (not re-tweets).

Extracting hashtags from the tips requires very little code.

<hr>

First, I defined a regex for a tag:

```python
TAG = re.compile(r'#([a-z0-9]{3,})')
```

Then, I used `findall` to get all tags.

I passed them to [collections.Counter](https://docs.python.org/3.7/library/collections.html) which returns a dict like object with the tags as keys, and counts as values, ordered in descending order by values (most common). I excluded the too common python tag which would skew the results.

```python
def get_hashtag_counter(tips):
    blob = ' '.join(t.text.lower() for t in tips)
    cnt = Counter(TAG.findall(blob))

    if EXCLUDE_PYTHON_HASHTAG:
        cnt.pop('python', None)

    return cnt
```

Finally, the `import_*` functions in *[tasks/import_tweets.py](https://github.com/pybites/pytip/blob/master/tasks/import_tweets.py)* do the actual import of the tweets and hashtags, calling `add_*` DB methods of the *tips* directory/package.

### Make a Simple web app with Bottle

With this pre-work done, making a web app is surprisingly easy (or not so surprising if you used Flask before).

First of all meet [Bottle](https://bottlepy.org/docs/dev/):

*Bottle is a fast, simple and lightweight [WSGI](http://www.wsgi.org/) micro web-framework for [Python](https://www.python.org). It is distributed as a single file module and has no dependencies other than the [Python Standard Library](https://docs.python.org/3/library/).*

Nice. The resulting web app comprises of < 30 LOC and can be found in [app.py](https://github.com/pybites/pytip/blob/master/app.py).

For this simple app, a single method with an optional tag argument is all it takes. Similar to Flask, the routing is handled with decorators. If called with a tag it filters the tips on tag, else it shows them all. The view decorator defines the template to use. Like Flask (and Django) we return a dict for use in the template.

```python
@route('/')
@route('/<tag>')
@view('index')
def index(tag=None):
    tag = tag or request.query.get('tag') or None
    tags = get_hashtags()
    tips = get_tips(tag)

    return {'search_tag': tag or '',
            'tags': tags,
            'tips': tips}
```

As per [documentation](https://bottlepy.org/docs/dev/tutorial.html#static-files), to work with static files, you add this snippet at the top, after the imports:

```python
@route('/static/<filename:path>')
def send_static(filename):
    return static_file(filename, root='static')
```

Finally, we want to make sure we only run in [debug mode](https://bottlepy.org/docs/dev/tutorial.html#tutorial-debugging) on localhost, hence the `APP_LOCATION` env variable we defined in Project Setup:

```python
if os.environ.get('APP_LOCATION') == 'heroku':
    run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
else:
    run(host='localhost', port=8080, debug=True, reloader=True)
```

### Bottle Templates

Bottle comes with a fast, powerful and easy to learn built-in template engine called [SimpleTemplate](https://bottlepy.org/docs/dev/stpl.html).

In the [views](https://github.com/pybites/pytip/tree/master/views) subdirectory I defined a *header.tpl*, *index.tpl*, and *footer.tpl*. For the tag cloud, I used some simple inline CSS increasing tag size by count, see *[header.tpl](https://github.com/pybites/pytip/blob/master/views/header.tpl)*:

{% raw %}
```
% for tag in tags:
  <a style="font-size: {{ tag.count/10 + 1 }}em;" href="/{{ tag.name }}">#{{ tag.name }}</a>&nbsp;&nbsp;
% end
```
{% endraw %}

In *[index.tpl](https://github.com/pybites/pytip/blob/master/views/index.tpl)* we loop over the tips:

{% raw %}
```
% for tip in tips:
  <div class='tip'>
    <pre>{{ !tip.text }}</pre>
    <div class="mui--text-dark-secondary"><strong>{{ tip.likes }}</strong> Likes / <strong>{{ tip.retweets }}</strong> RTs / {{ tip.created }} / <a href="https://twitter.com/python_tip/status/{{ tip.tweetid }}" target="_blank">Share</a></div>
  </div>
% end
```
{% endraw %}

{% raw %}
If you are familiar with Flask and Jinja2 this should look very familiar. Embedding Python is even easier, with less typing - `(% ...` vs `{% ... %}`).
{% endraw %}

All css, images (and JS if we'd use it) go into the [static](https://github.com/pybites/pytip/tree/master/static) subfolder.

And that's all there is to making a basic web app with Bottle. Once you have the data layer properly defined it's pretty straightforward.

### Add tests with pytest

Now let's make this project a bit more robust by adding [some tests](https://github.com/pybites/pytip/tree/master/tests). Testing the DB required a bit more digging into the [pytest](https://docs.pytest.org/en/latest/) framework, but I ended up using the [pytest.fixture](https://docs.pytest.org/en/latest/fixture.html) decorator to set up and tear down a database with some test tweets.

Instead of calling the Twitter API, I used some static data provided in *[tweets.json](https://github.com/pybites/pytip/blob/master/tests/tweets.json)*.
And, rather than using the live DB, in *[tips/db.py](https://github.com/pybites/pytip/blob/master/tips/db.py)*, I check if pytest is the caller (`sys.argv[0]`). If so, I use the test DB. I probably will refactor this, because [Bottle supports working with config files](https://bottlepy.org/docs/dev/configuration.html).

The hashtag part was easier to test (`test_get_hashtag_counter`) because I could just add some hashtags to a multiline string. No fixtures needed.

### Code quality matters - Better Code Hub

[Better Code Hub](https://bettercodehub.com/) guides you in writing, well, better code. Before writing the tests the project scored a 7:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/bottle-daily-python-tip/bch1.png" style="max-width: 100%;" alt="better code hub">
</div>

Not bad, but we can do better:

1. I bumped it to a 9 by making the code more modular, taking the DB logic out of the app.py (web app), putting it in the tips folder/ package (refactorings [1](https://github.com/pybites/pytip/commit/3e9909284899fdf01bb15b01dea7db7e6ff0e7e8) and [2](https://github.com/pybites/pytip/commit/c09bb7a69deb992ed379599d36c23f49527ce693))

1. Then with the tests in place the project scored a 10:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/bottle-daily-python-tip/bch2.png" style="max-width: 100%;" alt="better code hub">
</div>


## Conclusion and Learning

Our [Code Challenge #40](https://pybit.es/codechallenge40_review.html) offered some good practice:

1. I built a useful app which can be expanded (I want to add an API).
1. I used some cool modules worth exploring: Tweepy, SQLAlchemy, and Bottle.
1. I learned some more pytest because I needed fixtures to test interaction with the DB.
1. Above all, having to make the code testable, the app became more modular which made it easier to maintain. Better Code Hub was of great help in this process.
1. I deployed the app to [Heroku](https://pytip.herokuapp.com/) using our [step-by-step guide](https://pybit.es/deploy-flask-heroku.html).

### We Challenge You

The best way to learn and improve your coding skills [is to practice](https://pybit.es/learn-by-doing.html). At PyBites we solidified this concept by organizing Python code challenges. Check out our [growing collection](https://pybit.es/pages/challenges.html), [fork the repo](https://github.com/pybites/challenges), and get coding!

Let us know if you build something cool by making a [Pull Request](https://github.com/pybites/challenges/blob/master/INSTALL.md) of your work. We have seen folks really stretching themselves through these challenges, and so did we.

Happy coding!  

## Contact Info

I am Bob Belderbos from [PyBites](https://pybit.es/), you can reach out to me by:

- Twitter: [https://twitter.com/pybites](https://twitter.com/pybites)
- GitHub: [https://github.com/pybites](https://github.com/pybites)
- Email: [pybitesblog@gmail.com](mailto:pybitesblog@gmail.com)
- Slack: upon request (send us an email).
