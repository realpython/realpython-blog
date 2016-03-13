# Flask by Example - Text Processing with Requests, BeautifulSoup, and NLTK

*The following is a collaboration piece between Cam Linke, co-founder of [Startup Edmonton](http://startupedmonton.com/), and the folks at Real Python.*

**Updates**:

1. 10/10/2014: Added a name element to the HTML form input.
1. 02/22/2015: Added Python 3 support.

<br>

**In this section, we're going to scrape the contents of a webpage that the user enters and then process the text to display the number of times a word occurs.**

Remember, here's what we're building: A Flask app that calculates word-frequency pairs based on the text from a given URL. This is a full-stack tutorial.

1. [Part One](http://www.realpython.com/blog/python/flask-by-example-part-1-project-setup): Setup a local development environment and then deploy both a staging environment and a production environment on Heroku.
1. [Part Two](http://www.realpython.com/blog/flask-by-example-part-2-postgres-sqlalchemy-and-alembic): Setup a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations.
1. **Part Three: Add in the back-end logic to scrape and then process the counting of words from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries. (current)**
1. [Part Four](https://realpython.com/blog/python/flask-by-example-implementing-a-redis-task-queue/): Implement a Redis task queue to handle the text processing.
1. [Part Five](https://realpython.com/blog/python/flask-by-example-integrating-flask-and-angularjs/): Setup Angular on the front-end to continuously poll the back-end to see if the request is done.
1. [Part Six](https://realpython.com/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis, detailing how to run two processes (web and worker) on a single Dyno.
1. [Part Seven](https://realpython.com/blog/python/flask-by-example-updating-the-ui/): Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Install Requirements

Tools we'll use the following tools:

- **requests** - [http://docs.python-requests.org/en/latest/](http://docs.python-requests.org/en/latest/)
- **BeautifulSoup** - [http://www.crummy.com/software/BeautifulSoup/](http://www.crummy.com/software/BeautifulSoup/)
- **NLTK** - [http://www.nltk.org/](http://www.nltk.org/)

```sh
$ cd wordcounts
$ pip install requests nltk beautifulsoup4
$ pip freeze > requirements.txt
```

## Refactor the Index Route

To get started let's get rid of the "hello world" part of the index route in our *app.py* file and set up the route to render a form to accept URLs. First, add a templates folder to hold our templates and add an *index.html* file to it.

```sh
$ mkdir templates
$ cd templates
$ touch index.html
```

Set up a very basic HTML page:

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
      <h1>Wordcount 3000</h1>
      <form role="form" method='POST' action='/'>
        <div class="form-group">
          <input type="text" name="url" class="form-control" id="url-box" placeholder="Enter URL..." style="max-width: 300px;">
        </div>
        <button type="submit" class="btn btn-default">Submit</button>
      </form>
      <br>
      {% for error in errors %}
        <h4>{{ error }}</h4>
      {% endfor %}
    </div>
    <script src="//code.jquery.com/jquery-1.11.0.min.js"></script>
    <script src="//netdna.bootstrapcdn.com/bootstrap/3.1.1/js/bootstrap.min.js"></script>
  </body>
</html>
```
{% endraw %}

We used [Bootstrap](http://getbootstrap.com/) to add a bit of style so our page isn't completely hideous. Then we added a form with a text box for users to enter a URL. Additionally, we utilized a Jinja `for` loop to iterate through a list of errors, displaying each one.

Update the *app.py* file to serve the template:

```python
from flask import Flask, render_template
from flask.ext.sqlalchemy import SQLAlchemy
import os

app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
db = SQLAlchemy(app)


@app.route('/', methods=['GET', 'POST'])
def index():
    return render_template('index.html')

if __name__ == '__main__':
    app.run()
```

Why both HTTP methods? Well, we will eventually use that same route for both GET and POST requests - to serve the *index.html* page and handle form submissions, respectively.

Fire up the app to test it out:

```sh
$ python manage.py runserver
```

Navigate to [http://localhost:5000/](http://localhost:5000/) and you should see the form staring back at you.

## Requests

Now let's use the [requests](http://docs.python-requests.org/en/latest/) library to grab the HTML page from the submitted URL.

Change your index route to the following:

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
from flask import Flask, render_template, request
from flask.ext.sqlalchemy import SQLAlchemy
import os
import requests
```

1. Here, we import the `requests` library as well as the `request` object from Flask. The former is used to scrape the specific user-provided URL, while the latter is used to handle GET and POST requests in Flask.
1. Next we add variables to capture both errors and results, which are passed into the template.
1. Within the view itself, we check if the request is a POST-

    - If POST: Grab the value from the form and assign it to the `url` variable. Then we add an exception to handle any errors and append a generic error message to the `errors` list. Finally, we render the template, including the `errors` list and `results` dictionary.
    - If GET: We simply render the template.

**Let's test this out:**

```
$ python manage.py runserver
```

You should be able to type in a web page and in the terminal you'll see the text of that webpage returned (as long as it's a valid page, of course).

## Text Processing

With the HTML in hand, let's now count the frequency of the words that are on the page and display them to the end user. Update your code in *app.py* to the following and we'll walk through what's happening:

```python
from flask import Flask, render_template, request
from flask.ext.sqlalchemy import SQLAlchemy
from stop_words import stops
from collections import Counter
from bs4 import BeautifulSoup
import operator
import os
import requests
import re
import nltk


#################
# configuration #
#################

app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
db = SQLAlchemy(app)

from models import Result


##########
# routes #
##########

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
    'i', 'me', 'my', 'myself', 'we', 'our', 'ours', 'ourselves', 'you',  'your',
    'yours', 'yourself', 'yourselves', 'he', 'him', 'his', 'himself', 'she',
    'her', 'hers', 'herself', 'it', 'its', 'itself', 'they', 'them', 'their',
    'theirs', 'themselves', 'what', 'which', 'who', 'whom', 'this', 'that',
    'these', 'those', 'am', 'is', 'are', 'was', 'were', 'be', 'been', 'being',
    'have', 'has', 'had',  'having', 'do', 'does', 'did', 'doing', 'a', 'an',
    'the', 'and', 'but', 'if', 'or', 'because', 'as', 'until', 'while', 'of',
    'at', 'by', 'for', 'with', 'about', 'against', 'between', 'into', 'through',
    'during', 'before', 'after', 'above', 'below', 'to', 'from', 'up', 'down',
    'in', 'out', 'on', 'off', 'over', 'under', 'again', 'further', 'then',
    'once', 'here', 'there', 'when', 'where', 'why', 'how', 'all', 'any',
    'both', 'each', 'few', 'more', 'most', 'other', 'some', 'such', 'no', 'nor',
    'not', 'only', 'own', 'same', 'so', 'than', 'too', 'very', 's', 't', 'can',
    'will', 'just', 'don', 'should', 'now', 'id', 'var', 'function', 'js', 'd',
    'script', '\'script', 'fjs', 'document'
]
```

### What's happening?

**Text Processing**

1. In our index route we use `beautifulsoup` to [clean](http://www.crummy.com/software/BeautifulSoup/bs4/doc/#get-text) the text, by removing the HTML tags, that we get back from the URL as well as `nltk` to-

    - Tokenize the raw text (break up the text into individual words), and
    - Turn the tokens into an nltk text object.

1. In order for nltk to work right, you need to download the [correct](http://www.nltk.org/api/nltk.tokenize.html#module-nltk.tokenize.punkt) tokenizers. First create a new directory - `mkdir nltk_data` - then run - `python -m nltk.downloader`.

    When the installation window appears, update the 'Download Directory' to *whatever_the_absolute_path_to_your_app_is/nltk_data/*.

    Then click the 'Models' tab and select 'punkt' from under the 'Identifier' column. Click 'Download'. Check the official [documentation](http://nltk.googlecode.com/svn/trunk/doc/howto/data.html) for more information if you need help.

**Remove Punctuation, Count Raw Words**

1. Since we don't want punctuation counted in the final results, we create a regular expression that matches anything not in the standard alphabet.
1. Then, using a list comprehension, we create a list of words without punctuation or numbers.
1. Finally, we count the number of times each word appears in the list using [Counter](http://pymotw.com/2/collections/counter.html) - which is a really useful tool for tallying the number of times something appears in a list.

**Stop Words**

Our current output contains a lot of words that we likely don't want to count - i.e., "I", "me", "the", and so forth. These are called stop words.

1. With the `stops` list, we again use a list comprehension to create a list of words that do not include those stop words.
1. Next, we create a dictionary with the words (as keys) and their associated counts (as values).
1. And finally we use the `sorted` tool to get a sorted representation of our dictionary. We then can use this to display the words with the highest count at the top of the list, which means that we won't have to do that sorting in our Jinja template.

> For a more robust stop word list, use the NLTK [stopwords corpus](http://www.nltk.org/book/ch02.html).

**Save the Results**

Finally we use a try/except to save the results of our search and counts to the database.

## Display Results

Now let's update *index.html* in order to display the results:

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

What if we wanted to display the first ten keywords from the dictionary? We can simply limit the dictionary to the first 10 results:

```python
results = sorted(
    no_stop_words_count.items(),
    key=operator.itemgetter(1),
    reverse=True
)[:10]
```

Test it out.

## Summary

Okay great. Given a URL we can count the words that are on the page. If you use a site without a massive amount of words, like [http://realpython.com](http://realpython.com) for instance, the processing should happen fairly quickly. What happens if the site has *a lot* of words, though? For example, try out [http://gutenberg.ca](http://gutenberg.ca/). You'll notice that this takes longer to process. If you have a number of users all hitting your site at once to get word counts, and some of them are trying to count larger pages, this can become a problem. Or perhaps you decide to change the functionality so that when a user inputs a URL, we recursively scrape the entire web site and calculate word frequencies based on each individual page. With enough traffic, this will significantly slow down the site.

What's the solution? Instead of counting the words after each user makes a request, we need to use a queue to process this in the backend - which is exactly where will start next time in [Part 4](https://realpython.com/blog/python/flask-by-example-implementing-a-redis-task-queue/). For now, commit your code and push it up to staging only since this new text processing feature is only half finished.

Before you push to Heroku, I recommend removing all language tokenizers except for English along with the zip file:

```sh
└── tokenizers
    └── punkt
        ├── PY3
        │   └── finnish.pickle
        └── english.pickle
```

This will significantly reduce the size of the commit. Keep in mind though that if you do process a non-English site, it will only process English words.


```sh
$ git push stage master
```

Check it out here - [http://wordcounts-stage.herokuapp.com/](http://wordcounts-stage.herokuapp.com/)

**Test it out on staging. Comment if you have questions. See you next time!**
