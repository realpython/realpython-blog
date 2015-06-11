---
layout: post
title: "Lyricize: A Flask app to create lyrics using Markov chains"
date: 2014-01-06 17:20:19 -0700
toc: true
comments: true
category_side_bar: true
categories: [python, flask]

keywords: "python, web development, flask, markov chains"
description: "In this post, we'll work through the process of launching a MVP, from the initial conception of the idea to a shareable prototype utilizing Flask. We'll create an app to generate lyrics using Markov chains."
---

New coders [are](http://www.reddit.com/r/learnpython/comments/xjlsh/i_just_finished_codeacademys_python_course/) [always](http://www.reddit.com/r/learnpython/comments/zc2c2/done_learning_python_what_next/) [looking](http://www.reddit.com/r/learnpython/comments/ul1b8/any_good_projects_for_beginners/) [for](http://www.reddit.com/r/learnpython/comments/1a9yie/worked_with_python_for_almost_a_year_now_how_to/) [new](http://www.reddit.com/r/learnpython/comments/1iqv3c/please_help_me_prepare_a_roadmap_as_to_what_i/) [projects](http://www.reddit.com/r/learnpython/comments/1czbfw/kinda_lostwhat_to_do_next/) - as well they should be! Not only is making your own side project the best way to get hands-on experience, but if you're looking to make the move from a hobby to a profession, then side projects are a great way to start building up a portfolio of work.

## From Idea to MVP

In this post, we'll work through the process of launching a (bare minimum) MVP, from the initial conception of the idea to a shareable prototype. By the end, you'll have created your own version of [Lyricize](http://lyricize.herokuapp.com), a small app that uses an artist or band's lyrics to generate "new" similar-sounding lyrics based on probabilities. Instead of presenting the typical "here's how to replicate all this code" tutorial, we'll go through the process step-by-step to show what's *actually* involved in the thought process and creation along the way.

> Note that this isn't necessarily about building the next killer startup; we're just looking to find a project that can be 1) a fun learning opportunity and 2) shareable with others.

**Before we start take a look at the sample [app](http://lyricize.herokuapp.com) to see what you'll be creating.** Essentially, you can generate new lyrics based on a particular artist's lyrics using Markov chains. For example, try searching for "Bob Dylan" and change the number of lines to three. Pretty cool, right? I just executed the same search, which resulted in:

**Yonder stands your promise all and boats** <br>
**I'm ready for the gorge** <br>
**I'm a hook**

So deep. Anyway, let's begin ...

<hr>

## Find a topic of interest

So, step 1: Find a topic you're interested in learning more about. The following app was inspired by an old [college assignment](http://www.cs.princeton.edu/courses/archive/fall13/cos126/assignments/markov.html) (admittedly not the most common source of inspiration) that uses Markov chains to generate "real-looking" text given a body of sample text. Markov models crop up in [all sorts of scenarios](http://en.wikipedia.org/wiki/Markov_chain#Applications). (We'll dive into what a Markov model is shortly.) I found the idea of probability-based text generation particularly interesting; specifically, I wondered what would happen if you used song lyrics as sample text to generate "new" lyrics...

To the internet! A quick web search shows a few Markov-based lyric generator sites, but nothing quite like what I have in mind. Besides, plugging away at someone else's completed code isn't a very effective way to learn how Markov generators actually work; let's build our own.

So... how do Markov generators works? Basically, a Markov chain is generated from some text based on the frequency of certain patterns occurring. As an example, consider the following string as our sample text:

```
bobby
```

We'll build the simplest possible Markov model out of this text, which is a Markov model of *order 0*, as a way to predict the likelihood of any particular letter occurring. This is a straight-forward frequency table:

```
b: 3/5
o: 1/5
y: 1/5
```

However, this is a pretty bad language model; besides how frequently letters occur overall, we also want to look at how frequently a particular letter occurs *given the previous letter*. Since we're depending on one previous letter, this is a Markov model of order 1:

```
given "b":
  "b" is next: 1/3
  "o" is next: 1/3
  "y" is next: 1/3
given "o":
  "b" is next: 1
given "y":
  [terminates]: 1
```

From here, you could imagine Markov models of higher order; an order 2 model would start by measuring the frequency of each letter that occurs after the two-letter string "bo", etc. By increasing the order, we get a model that starts looking more like real language; for instance, an order 5 Markov model that had been given lots of sample input including the word "python" would be very likely to follow the string "pytho" with an "n", whereas a much lower order model might have come up with some creative words.

## Start Developing

How would we go about building a rough approximation of a Markov model? Essentially, the structure we've outlined above with the higher-order models is a dictionary of dictionaries. You could imagine a `model` dictionary with various word fragments (i.e., "bo") as keys. Each of these fragments would then point to a dictionary in turn, with those inner dictionaries holding the individual next letters ("y") as keys with their respective frequencies as values.

Let's start by making a `generateModel()` method that takes in some sample text and a Markov model order, then returns this dictionary of dictionaries:

```python
def generateModel(text, order):
    model = {}
    for i in range(0, len(text) - order):
        fragment = text[i:i+order]
        next_letter = text[i+order]
        if fragment not in model:
            model[fragment] = {}
        if next_letter not in model[fragment]:
            model[fragment][next_letter] = 1
        else:
            model[fragment][next_letter] += 1
    return model
```

We looped through all the available text, going up until the last available full fragment + next letter so as not to run off the end of the string, adding our `fragment` dictionaries to the `model` with each `fragment` holding a dictionary of total `next_letter` frequencies.

Copy this function into a Python shell and try it out:

```sh
>>> generateModel("bobby", 1)
{'b': {'y': 1, 'b': 1, 'o': 1}, 'o': {'b': 1}}
```

That'll do! We have counts of frequencies instead of relative probabilities, but we can work with that; there's no reason we need to normalize each dictionary to add to probabilities of 100%.

Now let's use this `model` in a `getNextCharacter()` method that will, given a model and a fragment, decide on an appropriate next letter given the model's probabilities:

```python
from random import choice
def getNextCharacter(model, fragment):
    letters = []
    for letter in model[fragment].keys():
        for times in range(0, model[fragment][letter]):
            letters.append(letter)
    return choice(letters)
```

It's not the most efficient setup, but it's simple to build and works for now. We simply built a list of letters, given their total frequencies of occurrence after the fragment, and chose randomly from that list.

All that remains is to use these two methods in a third method that will actually generate text of some specified length. To do this, we'll need to keep track of the current text fragment we're building while adding on new characters:

```python
def generateText(text, order, length):
    model = generateModel(text, order)

    currentFragment = text[0:order]
    output = ""
    for i in range(0, length-order):
        newCharacter = getNextCharacter(model, currentFragment)
        output += newCharacter
        currentFragment = currentFragment[1:] + newCharacter
    print output
```

Let's make this into a full runnable script that takes a Markov order and output text length as arguments:

```python
from random import choice
import sys

def generateModel(text, order):
    model = {}
    for i in range(0, len(text) - order):
        fragment = text[i:i+order]
        next_letter = text[i+order]
        if fragment not in model:
            model[fragment] = {}
        if next_letter not in model[fragment]:
            model[fragment][next_letter] = 1
        else:
            model[fragment][next_letter] += 1
    return model

def getNextCharacter(model, fragment):
    letters = []
    for letter in model[fragment].keys():
        for times in range(0, model[fragment][letter]):
            letters.append(letter)
    return choice(letters)

def generateText(text, order, length):
    model = generateModel(text, order)
    currentFragment = text[0:order]
    output = ""
    for i in range(0, length-order):
        newCharacter = getNextCharacter(model, currentFragment)
        output += newCharacter
        currentFragment = currentFragment[1:] + newCharacter
    print output

text = "some sample text"
if __name__ == "__main__":
    generateText(text, int(sys.argv[1]), int(sys.argv[2]))
```

For now, we'll generate sample text via the very scientific method of throwing a string directly into the code based on some copied & pasted Alanis Morisette lyrics.

## Test

Save the script and give it a whirl:

```sh
$ python markov.py 2 100
I wounts
You ho's humortel whime
 mateend I wass
How by Lover
```

```sh
$ python markov.py 4 100
stress you to cosmic tears
All they've cracked you (honestly) at the filler in to like raise
$ python markov.py 6 100
tress you place the wheel from me
Please be philosophical
Please be tapped into my house
```

Well, that was just precious. The last two trials are decently representative of her lyrics (although the first sample of order 2 looks more like Björk). These results are encouraging enough for a quick code sketch, so let's turn this thing into a real project.

## Next Iteration

First hurdle: how are we going to automate getting lots of lyrics? One option would be to selectively scrape content from a lyrics site, but that sounds like a lot of effort for probably low-quality results, plus a potential legal gray area given the shadiness of most lyrics aggregators and the draconianism of the music industry. Instead, let's see if there are any open APIs. Heading over to search through [programmableweb.com](http://www.programmableweb.com/search/lyrics), we actually find *14* different lyrics APIs listed. These listings aren't always the most up-to-date, though, so let's search through by the most recently listed.

[LYRICSnMUSIC](http://www.lyricsnmusic.com/api) offers a free, RESTful API using JSON to return up to 150 characters of song lyrics. This sounds perfect for our use-case, especially given the repetition of most songs; there's no need to gather full lyrics when just a sample will do. Go grab a [new key](http://www.lyricsnmusic.com/api_keys/new) so that you can access their API.

Let's try their API out before we settle on this source for good. Based on their documentation, we can make a sample request like so:

```
http://api.lyricsnmusic.com/songs?api_key=[YOUR_API_KEY_HERE]&artist=coldplay
```

The JSON results it spits back in a browser are a bit hard to read; through them in a [formatter](http://jsonformatter.curiousconcept.com/) to take a better look. It looks like we're successfully getting back a list of dictionaries based on Coldplay songs:

```json
[
  {
     "title":"Don't Panic",
     "url":"http://www.lyricsnmusic.com/coldplay/don-t-panic-lyrics/4294612",
     "snippet":"Bones sinking like stones \r\nAll that we've fought for \r\nHomes, places we've grown \r\nAll of us are done for \r\n\r\nWe live in a beautiful world \r\nYeah we ...",
     "context":null,
     "viewable":true,
     "instrumental":false,
     "artist":{
        "name":"Coldplay",
        "url":"http://www.lyricsnmusic.com/coldplay"
     }
  },
  {
     "title":"Shiver",
     "url":"http://www.lyricsnmusic.com/coldplay/shiver-lyrics/4294613",
     "snippet":"So I look in your direction\r\nBut you pay me no attention, do you\r\nI know you don't listen to me\r\n'Cause you say you see straight through me, don't you...",
     "context":null,
     "viewable":true,
     "instrumental":false,
     "artist":{
        "name":"Coldplay",
        "url":"http://www.lyricsnmusic.com/coldplay"
     }
  },
  ...
]
```

There's no way to limit the response, but we're only interested in each "snippet" provided, which looks just fine for this project.

Our preliminary experiments with Markov generators were educational, but our current model isn't the best suited to the task of generating lyrics. For one thing, we should probably use individual words as our tokens rather than taking things character by character; it's fun trying to mock language itself, but for generating fake lyrics, we'll want to stick to real English. This sounds trickier, though, and we've come a long way to understanding how Markov chains operate, which was the initial goal with that exercise. At this point, we reach a crossroads: reinvent the metaphorical wheel for the sake of more learning (it could be great coding practice), or see what else others have already created.

I chose the lazy way out and headed back to search the innerwebs. A kind soul on GitHub has already [implemented](https://github.com/MaxWagner/PyMarkovChain) a basic single-word-based Markov chain and even uploaded it to [PyPI](https://pypi.python.org/pypi/PyMarkovChain/). Taking a quick stroll through the [code](https://github.com/ketaro/markov-cranberries/blob/master/markov.py), it appears that this model is only of order 0. This probably would have been quick enough to build on our own, while a higher-order model might be significantly more work. For now, let's go with someone else's pre-packaged wheel; at least an order 0 model won't end up sounding like Björk if we're using whole words.

Since we want to easily share our creation with friends and family, it makes sense to turn it into a web application. Now, to choose a web framework. Personally, I'm by far the most familiar with Django, but that seems like overkill here; after all, we won't even need a database of our own. Let's try out Flask.

## Add Flask

Per the usual routine, fire up a virtual environment - if you haven't already! If this isn't a familiar process, take a look through some of our [previous posts](http://www.realpython.com/blog/python/python-web-applications-with-flask-part-i/#toc_5) to learn how to get set up.

```sh
$ mkdir lyricize
$ cd lyricize
$ virtualenv --no-site-packages venv
$ source venv/bin/activate
```

Also as per usual, install the necessary requirements and throw them in a requirements.txt file:

```sh
$ pip install PyMarkovChain flask requests
$ pip freeze > requirements.txt
```

We've added in the [requests](http://docs.python-requests.org/en/latest/) library as well so that we can make web requests to the lyrics API.

Now, to make the app. For the sake of simplicity, let's split it up into two pages: the main page will present a basic form to the user for choosing an artist name and a number of lines of lyrics to generate, while a second "lyrics" page will present the results. Let's start with a barebones Flask application named **app.py** that uses an **index.html** template:

```python
from flask import Flask, render_template

app = Flask(__name__)
app.debug = True

@app.route('/', methods=['GET'])
def index():
    return render_template('index.html')

if __name__ == '__main__':
    app.run()
```

*All* this app will do so far is load the contents of an index.html template. Let's make it a basic form:

{% raw %}
```html
<html>
 <body>
  <form action="#" method="post" class="lyrics">
    Artist or band name: <input name="artist" type="text" /><br />
    Number of lines:
    <select name="lines">
      {% for n in range(1,11) %}
        <option value="{{n}}">{{n}}</option>
      {% endfor %}
    </select>
    <br /><br />
    <input class="button" type="submit" value="Lyricize">
  </form>
 </body>
</html>
```
{% endraw %}

Save this index.html in a separate folder named *templates* so that Flask can find it. Here we're using Flask's Jinja2 templating to create a "selection" dropdown based on a loop covering the numbers 1 through 10. Before we add anything else, fire up this page to make sure we're set up correctly:

```sh
$ python app.py
* Running on http://127.0.0.1:5000/
```

You should now be able to visit http://127.0.0.1:5000/ in a browser and see the lovely form.

Now let's decide what we want to show on the results page, so that we know what we'll need to pass to it:

{% raw %}
```html
<html>
 <body>
  <div align="center" style="padding-top:20px;">
   <h2>
   {% for line in result %}
     {{ line }}<br />
   {% endfor %}
   </h2>
   <h3>{{ artist }}</h3>
   <br />
   <form action="{{ url_for('index') }}">
    <input type="submit" value="Do it again!" />
   </form>
  </div>
 </body>
</html>
```
{% endraw %}

Here we looped through a **result** array, line by line, displaying each line separately. Below that, we show the **artist** selected and link back to the homepage. Save this as *lyrics.html* in your /templates directory.

We also need to update the form action of index.html to point to this results page:

{% raw %}
```html
<form action="{{ url_for('lyrics') }}" method="post" class="lyrics">
```
{% endraw %}

Now to write a route for the resulting lyrics page:

```python
@app.route('/lyrics', methods=['POST'])
def lyrics():
    artist = request.form['artist']
    lines = int(request.form['lines'])

    if not artist:
        return redirect(url_for('index'))

    return render_template('lyrics.html', result=['hello', 'world'], artist=artist)
```

This page takes a POST request from the form, parsing out the provided **artist** and number of **lines** - we aren't generating any lyrics yet, just giving the template a dummy list of results. We'll also need to add the necessary Flask functionality - `url_for` and `redirect` - that we've relied on:

```python
from flask import Flask, render_template, url_for, redirect
```

Test it out to make sure nothing's broken yet:

```sh
$ python app.py
```

Great, now for the real meat of the project. Within lyrics(), let's get a response back from LYRICSnMUSIC based on our passed-in artist parameter:

```python
# Get a response of sample lyrics from the provided artist
uri = "http://api.lyricsnmusic.com/songs"
params = {
    'api_key': API_KEY,
    'artist': artist,
}
response = requests.get(uri, params=params)
lyric_list = response.json()
```

Using requests, we fetch a specific URL that includes a dictionary of parameters: the provided artist name, and our API key. This private API key should *not* appear in your code; after all, you'll want to share this code with others. Instead, let's make a separate file to hold this value as a variable:

```sh
$ echo "API_KEY=[youractualapikeygoeshere]" > .env
```

We've created a special "environment" file that Flask can now read in if we just add the following to the top of our app:

```python
import os
API_KEY = os.environ.get('API_KEY')
```

And finally, let's add in the Markov chain functionality. Now that we're using someone else's package, this ends up being fairly trivial. First, add the import at the top:

```python
from pymarkovchain import MarkovChain
```

And then, after we've received a **lyrics** response from the API, we simply create a MarkovChain, load in the lyrics data, and generate a list of sentences:

```python
mc = MarkovChain()
mc.generateDatabase(lyrics)

result = []
for line in range(0, lines):
    result.append(mc.generateString())
```

In total, then, app.py should now look something like this:

```python
from flask import Flask, url_for, redirect, request, render_template
import requests
from pymarkovchain import MarkovChain
import os

API_KEY = os.environ.get('API_KEY')

app = Flask(__name__)
app.debug = True

@app.route('/', methods=['GET'])
def index():
    return render_template('index.html')

@app.route('/lyrics', methods=['POST'])
def lyrics():
    artist = request.form['artist']
    lines = int(request.form['lines'])

    if not artist:
        return redirect(url_for('index'))

    # Get a response of sample lyrics from the artist
    uri = "http://api.lyricsnmusic.com/songs"
    params = {
        'api_key': API_KEY,
        'artist': artist,
    }
    response = requests.get(uri, params=params)
    lyric_list = response.json()

    # Parse results into a long string of lyrics
    lyrics = ''
    for lyric_dict in lyric_list:
        lyrics += lyric_dict['snippet'].replace('...', '') + ' '

    # Generate a Markov model
    mc = MarkovChain()
    mc.generateDatabase(lyrics)

    # Add lines of lyrics
    result = []
    for line in range(0, lines):
        result.append(mc.generateString())

    return render_template('lyrics.html', result=result, artist=artist)

if __name__ == '__main__':
    app.run()
```

Try it out! Everything should be working locally. Now, to share it with the world...

## Deploy to Heroku

Let's host on Heroku, since (for these minimal requirements) we can do that for free. To do so, we'll need to make a few minor tweaks to the code. First, add a [Procfile](https://devcenter.heroku.com/articles/procfile) that will tell Heroku how to serve the app:

```
$ echo "web: python app.py" > Procfile
```

Next, since Heroku specifies a random port on which to run the application, you'll need to pass a port number in at the top:

```python
PORT = int(os.environ.get('PORT', 5000))

app = Flask(__name__)
app.config.from_object(__name__)
```

And when the app is run, make sure to pass this port in

```python
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=PORT)
```

We also had to specify the host of '0.0.0.0' because Flask by default runs privately on the local computer, while we want the app to run on Heroku on a publicly available IP.

Finally, remove `app.debug=True` from your code so that users don't get to see your full stacktrace errors if something goes wrong.

Initialize a git repository (if you haven't already), create a new Heroku app, and push your code to it!

```sh
$ git init
$ git add .
$ git commit -m "First commit"
$ heroku create
$ git push heroku master
```

See the [Heroku docs](https://devcenter.heroku.com/articles/getting-started-with-python) for a more thorough rundown of this deployment process. Be sure to add your API_KEY variable on Heroku:

```sh
$ heroku config:set API_KEY=[youractualapikeygoeshere]
```

And we're all set! Time to share your creation with the world - or keep hacking at it :)

## Conclusion

If you liked this content, you might be interested in our [current courses](http://RealPython.com) for learning web development or our newest [Kickstarter](http://kck.st/1b0tz6I) that covers more advanced techniques. Or - just play around with the app [here](http://lyricize.herokuapp.com).

**Possible next steps:**

- This HTML looks like it was written in the early 90s; use [Bootstrap](https://github.com/mbr/flask-bootstrap) or just some basic CSS for styling
- Add some comments to the code before you forget what it does! (This is left as an exercise for the reader :o)
- Abstract the code in the lyrics route to be individual methods (ie, a method to return responses from the lyrics API and another separate method for generating the Markov model); this will make the code more easily maintainable and testable as it grows in size and complexity
- Create a Markov generator that is able to use higher orders
- Use [Flask-WTF](https://flask-wtf.readthedocs.org/en/latest/) to improve the forms and form validation
- Speaking of which: make it more secure! Right now, someone could potentially send unreasonable POST requests, inject their own code into the page or DoS the site with many quickly repeated requests; add some solid input validation and basic rate limiting
- Add better error handling; what if an API call takes too long or fails for some reason?
- Throw the results into a [text-to-speech](http://tts-api.com/) engine, learn to vary the pitch patterns using another Markov model, and set to a beat; soon you'll be at the top of the charts!

