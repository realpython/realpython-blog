# Flask by Example - Text Processing with Requests, BeautifulSoup, and NLTK

**In this part of the series, we're going to scrape the contents of a webpage and then process the text to display word counts.**

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask_by_example/flask-by-example-part3.png" style="max-width: 100%;" alt="flask by example part 3">
</div>

<br>

*Updates:*

  - 03/22/2016: Upgraded to Python version [3.5.1](https://www.python.org/downloads/release/python-351/) as well as the latest versions of requests, BeautifulSoup, and nltk. See [below](#install-requirements) for details.
  - 02/22/2015: Added Python 3 support.

<hr><br>

Remember: Here's what we're building - A Flask app that calculates word-frequency pairs based on the text from a given URL.

<br>

1. [Part One](/blog/python/flask-by-example-part-1-project-setup/): Set up a local development environment and then deploy both a staging and a production environment on Heroku.
1. [Part Two](/blog/python/flask-by-example-part-2-postgres-sqlalchemy-and-alembic): Set up a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations.
1. **Part Three: Add in the back-end logic to scrape and then process the word counts from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries. (_current_)**
1. [Part Four](/blog/python/flask-by-example-implementing-a-redis-task-queue/): Implement a Redis task queue to handle the text processing.
1. [Part Five](/blog/python/flask-by-example-integrating-flask-and-angularjs/): Set up Angular on the front-end to continuously poll the back-end to see if the request is done processing.
1. [Part Six](/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis and detailing how to run two processes (web and worker) on a single Dyno.
1. [Part Seven](/blog/python/flask-by-example-updating-the-ui/): Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Install Requirements

Tools used:

- requests ([2.9.1](https://pypi.python.org/pypi/requests/2.9.1)) - a library for sending HTTP requests
- BeautifulSoup ([4.4.1](https://pypi.python.org/pypi/beautifulsoup4/4.4.1)) - a tool used for scraping and parsing documents from the web
- Natural Language Toolkit ([3.2](https://pypi.python.org/pypi/nltk/3.2)) - a natural language processing library

Navigate into the project directory to activate the virtual environment, via [autoenv](https://pypi.python.org/pypi/autoenv/1.0.0), and then install the requirements:

```sh
$ cd flask-by-example
$ pip install requests==2.9.1 beautifulsoup4==4.4.1 nltk==3.2
$ pip freeze > requirements.txt
```

## Refactor the Index Route

To get started, let's get rid of the "hello world" part of the index route in our *app.py* file and set up the route to render a form to accept URLs. First, add a templates folder to hold our templates and add an *index.html* file to it.

```sh
$ mkdir templates
$ touch templates/index.html
```

Set up a very basic HTML page:

{% raw %}
```html
<!DOCTYPE html>
<html>
  <head>
    <title>Wordcount</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="//netdna.bootstrapcdn.com/bootstrap/3.3.6/css/bootstrap.min.css" rel="stylesheet" media="screen">
    <style>
      .container {
        max-width: 1000px;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <h1>Wordcount 3000</h1>
      <form role="form" method='POST' action='/'>
        <div class="form-group">
          <input type="text" name="url" class="form-control" id="url-box" placeholder="Enter URL..." style="max-width: 300px;" autofocus required>
        </div>
        <button type="submit" class="btn btn-default">Submit</button>
      </form>
      <br>
      {% for error in errors %}
        <h4>{{ error }}</h4>
      {% endfor %}
    </div>
    <script src="//code.jquery.com/jquery-2.2.1.min.js"></script>
    <script src="//netdna.bootstrapcdn.com/bootstrap/3.3.6/js/bootstrap.min.js"></script>
  </body>
</html>
```
{% endraw %}

We used [Bootstrap](http://getbootstrap.com/) to add a bit of style so our page isn't completely hideous. Then we added a form with a text input box for users to enter a URL into. Additionally, we utilized a [Jinja](https://realpython.com/blog/python/primer-on-jinja-templating/) `for` loop to iterate through a list of errors, displaying each one.

Update *app.py* to serve the template:

```python
import os
from flask import Flask, render_template
from flask.ext.sqlalchemy import SQLAlchemy


app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

from models import Result


@app.route('/', methods=['GET', 'POST'])
def index():
    return render_template('index.html')


if __name__ == '__main__':
    app.run()
```

Why both HTTP methods, `methods=['GET', 'POST']`? Well, we will eventually use that same route for both GET and POST requests - to serve the *index.html* page and handle form submissions, respectively.

Fire up the app to test it out:

```sh
$ python manage.py runserver
```

Navigate to [http://localhost:5000/](http://localhost:5000/) and you should see the form staring back at you.

## Requests

Now let's use the [requests](http://docs.python-requests.org/en/v2.9.1/) library to grab the HTML page from the submitted URL.

Change your index route like so:

```python
@app.route('/', methods=['GET', 'POST'])
def index():
    errors = []
    results = {}
    if request.method == "POST":
        # get url that the user has entered
        try:
            url = request.form['url']
            r = requests.get(url)
            print(r.text)
        except:
            errors.append(
                "Unable to get URL. Please make sure it's valid and try again."
            )
    return render_template('index.html', errors=errors, results=results)
```

Make sure to update the imports as well:

```python
import os
import requests
from flask import Flask, render_template, request
from flask.ext.sqlalchemy import SQLAlchemy
```

1. Here, we imported the `requests` library as well as the `request` object from Flask. The former is used to send external HTTP GET requests to grab the specific user-provided URL, while the latter is used to handle GET and POST requests within the Flask app.
1. Next, we added variables to capture both errors and results, which are passed into the template.
1. Within the view itself, we checked if the request is a GET or POST-

    - If POST: We grabbed the value (URL) from the form and assigned it to the `url` variable. Then we added an exception to handle any errors and, if necessary, appended a generic error message to the `errors` list. Finally, we rendered the template, including the `errors` list and `results` dictionary.
    - If GET: We simply rendered the template.

**Let's test this out:**

```sh
$ python manage.py runserver
```

You should be able to type in a valid webpage and in the terminal you'll see the text of that page returned.

## Text Processing

With the HTML in hand, let's now count the frequency of the words that are on the page and display them to the end user. Update your code in *app.py* to the following and we'll walk through what's happening:

```python
import os
import requests
import operator
import re
import nltk
from flask import Flask, render_template, request
from flask.ext.sqlalchemy import SQLAlchemy
from stop_words import stops
from collections import Counter
from bs4 import BeautifulSoup


app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
db = SQLAlchemy(app)

from models import Result


@app.route('/', methods=['GET', 'POST'])
def index():
    errors = []
    results = {}
    if request.method == "POST":
        # get url that the person has entered
        try:
            url = request.form['url']
            r = requests.get(url)
        except:
            errors.append(
                "Unable to get URL. Please make sure it's valid and try again."
            )
            return render_template('index.html', errors=errors)
        if r:
            # text processing
            raw = BeautifulSoup(r.text, 'html.parser').get_text()
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
            results = sorted(
                no_stop_words_count.items(),
                key=operator.itemgetter(1),
                reverse=True
            )
            try:
                result = Result(
                    url=url,
                    result_all=raw_word_count,
                    result_no_stop_words=no_stop_words_count
                )
                db.session.add(result)
                db.session.commit()
            except:
                errors.append("Unable to add item to database.")
    return render_template('index.html', errors=errors, results=results)


if __name__ == '__main__':
    app.run()
```

Create a new file called *stop_words.py* and add the following list:

```python
stops = [
    'i', 'me', 'my', 'myself', 'we', 'our', 'ours', 'ourselves', 'you',
    'your', 'yours', 'yourself', 'yourselves', 'he', 'him', 'his',
    'himself', 'she', 'her', 'hers', 'herself', 'it', 'its', 'itself',
    'they', 'them', 'their', 'theirs', 'themselves', 'what', 'which',
    'who', 'whom', 'this', 'that', 'these', 'those', 'am', 'is', 'are',
    'was', 'were', 'be', 'been', 'being', 'have', 'has', 'had', 'having',
    'do', 'does', 'did', 'doing', 'a', 'an', 'the', 'and', 'but', 'if',
    'or', 'because', 'as', 'until', 'while', 'of', 'at', 'by', 'for',
    'with', 'about', 'against', 'between', 'into', 'through', 'during',
    'before', 'after', 'above', 'below', 'to', 'from', 'up', 'down', 'in',
    'out', 'on', 'off', 'over', 'under', 'again', 'further', 'then',
    'once', 'here', 'there', 'when', 'where', 'why', 'how', 'all', 'any',
    'both', 'each', 'few', 'more', 'most', 'other', 'some', 'such', 'no',
    'nor', 'not', 'only', 'own', 'same', 'so', 'than', 'too', 'very', 's',
    't', 'can', 'will', 'just', 'don', 'should', 'now', 'id', 'var',
    'function', 'js', 'd', 'script', '\'script', 'fjs', 'document', 'r',
    'b', 'g', 'e', '\'s', 'c', 'f', 'h', 'l', 'k'
]
```

### What's happening?

**Text Processing**

1. In our index route we used beautifulsoup to [clean](http://www.crummy.com/software/BeautifulSoup/bs4/doc/#get-text) the text, by removing the HTML tags, that we got back from the URL as well as nltk to-

    - [Tokenize](http://www.nltk.org/api/nltk.tokenize.html) the raw text (break up the text into individual words), and
    - Turn the tokens into an nltk [text object](http://www.nltk.org/_modules/nltk/text.html).

1. In order for nltk to work properly, you need to download the [correct](http://www.nltk.org/api/nltk.tokenize.html#module-nltk.tokenize.punkt) tokenizers. First, create a new directory - `mkdir nltk_data` - then run - `python -m nltk.downloader`.

    When the installation window appears, update the 'Download Directory' to *whatever_the_absolute_path_to_your_app_is/nltk_data/*.

    Then click the 'Models' tab and select 'punkt' under the 'Identifier' column. Click 'Download'. Check the official [documentation](http://www.nltk.org/data.html#command-line-installation) for more information.

**Remove Punctuation, Count Raw Words**

1. Since we don't want punctuation counted in the final results, we created a regular expression that matched anything not in the standard alphabet.
1. Then, using a list comprehension, we created a list of words without punctuation or numbers.
1. Finally, we tallied the number of times each word appeared in the list using [Counter](https://pymotw.com/3/collections/counter.html).

**Stop Words**

Our current output contains a lot of words that we likely don't want to count - i.e., "I", "me", "the", and so forth. These are called stop words.

1. With the `stops` list, we again used a list comprehension to create a final list of words that do not include those stop words.
1. Next, we created a dictionary with the words (as keys) and their associated counts (as values).
1. And finally we used the [sorted](https://docs.python.org/3/library/functions.html#sorted) method to get a sorted representation of our dictionary. Now we can use the sorted data to display the words with the highest count at the top of the list, which means that we won't have to do that sorting in our Jinja template.

> For a more robust stop word list, use the NLTK [stopwords corpus](http://www.nltk.org/book/ch02.html).

**Save the Results**

Finally, we used a try/except to save the results of our search and the subsequent counts to the database.

## Display Results

Let's update *index.html* in order to display the results:

{% raw %}
```html
<!DOCTYPE html>
<html>
  <head>
    <title>Wordcount</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css" rel="stylesheet" media="screen">
    <style>
      .container {
        max-width: 1000px;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <div class="row">
        <div class="col-sm-5 col-sm-offset-1">
          <h1>Wordcount 3000</h1>
          <br>
          <form role="form" method="POST" action="/">
            <div class="form-group">
              <input type="text" name="url" class="form-control" id="url-box" placeholder="Enter URL..." style="max-width: 300px;">
            </div>
            <button type="submit" class="btn btn-default">Submit</button>
          </form>
          <br>
          {% for error in errors %}
            <h4>{{ error }}</h4>
          {% endfor %}
          <br>
        </div>
        <div class="col-sm-5 col-sm-offset-1">
          {% if results %}
            <h2>Frequencies</h2>
            <br>
            <div id="results">
              <table class="table table-striped" style="max-width: 300px;">
                <thead>
                  <tr>
                    <th>Word</th>
                    <th>Count</th>
                  </tr>
                </thead>
                {% for result in results%}
                  <tr>
                    <td>{{ result[0] }}</td>
                    <td>{{ result[1] }}</td>
                  </tr>
                {% endfor %}
              </table>
            </div>
          {% endif %}
        </div>
      </div>
    </div>
    <br><br>
    <script src="//code.jquery.com/jquery-1.11.0.min.js"></script>
    <script src="//netdna.bootstrapcdn.com/bootstrap/3.1.1/js/bootstrap.min.js"></script>
  </body>
</html>
```
{% endraw %}

Here, we added an `if` statement to see if our `results` dictionary has anything in it and then added a `for` loop to iterate over the `results` and display them in a table. Run your app and you should be able to enter a URL and get back the count of the words on the page.

```sh
$ python manage.py runserver
```

What if we wanted to display only the first ten keywords?

```python
results = sorted(
    no_stop_words_count.items(),
    key=operator.itemgetter(1),
    reverse=True
)[:10]
```

Test it out.

## Summary

Okay great. Given a URL we can count the words that are on the page. If you use a site without a massive amount of words, like [http://realpython.com](http://realpython.com), the processing should happen fairly quickly. What happens if the site has *a lot* of words, though? For example, try out [http://gutenberg.ca](http://gutenberg.ca/). You'll notice that this takes longer to process.

If you have a number of users all hitting your site at once to get word counts, and some of them are trying to count larger pages, this can become a problem. Or perhaps you decide to change the functionality so that when a user inputs a URL, we recursively scrape the entire web site and calculate word frequencies based on each individual page. With enough traffic, this will significantly slow down the site.

What's the solution?

*Instead of counting the words after each user makes a request, we need to use a queue to process this in the backend - which is exactly where will start next time in [Part 4](/blog/python/flask-by-example-implementing-a-redis-task-queue/).*

For now, commit your code, but before you push to Heroku, you should remove all language tokenizers except for English along with the zip file. This will significantly reduce the size of the commit. Keep in mind though that if you do process a non-English site, it will only process English words.

```sh
└── nltk_data
    └── tokenizers
        └── punkt
            ├── PY3
            │   └── english.pickle
            └── english.pickle
```

Push it up to the staging environment only since this new text processing feature is only half finished:

```sh
$ git push stage master
```

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask_by_example/flask-by-example-part3-final.png" style="max-width: 100%;" alt="flask by example part3 final">
</div>

<br>

Test it out on staging. Comment if you have questions. See you next time!

<br>

<p style="font-size: 14px;">
  <em>This is a collaboration piece between Cam Linke, co-founder of <a href="http://startupedmonton.com/">Startup Edmonton</a>, and the folks at Real Python</em>
</p>