---
layout: post
title: "Caching External API Requests"
date: 2014-12-05 07:29:57 -0700
toc: true
comments: true
category_side_bar: true
categories: [python, flask, fundamentals, api]

keywords: "python, flask, request, cache, caching API requests, caching external API requests, cacheing api calls"
description: "Let's look at how to to cache external api calls."
---

<div class="center-text">
  <img class="no-border" src="/images/blog_images/requests-cache.png" alt="python requests cache">
</div>

Ever find yourself making the *exact* same request to an external API, using the *exact* same parameters and returning the *exact* same results? If so, then you should cache this request to limit the number of HTTP requests to help improve performance.

Let's look at an example using the [requests](http://docs.python-requests.org/en/latest/) package.

## Github API

Grab the code from the [Github repo](https://github.com/realpython/flask-single-page-app/tree/part5) (or download the [zip](https://github.com/realpython/flask-single-page-app/releases/tag/part5). Basically, we're searching the Github API over and over again to find similar developers by location and programming language:

```python
url = "https://api.github.com/search/users?q=location:{0}+language:{1}".format(first, second)
response_dict = requests.get(url).json()
```

Right now, after the initial search, if the user searches again (e.g., doesn't change the parameters), the application will perform the exact same search, hitting the Github API again and again and again. Since this is an expensive process, it slows down our application for the end user. Plus, by making several calls like this, we can quickly use up our rate limit.

Fortunately, there is an easy fix.

## Requests-cache

To implement caching, we can use a simple package called [Requests-cache](http://requests-cache.readthedocs.org/en/latest/index.html), which is a "transparent persistent cache for [requests](http://docs.python-requests.org/en/latest/)".

> Keep in mind that you can use this package with any Python Framework, not just Flask, or script as long as you couple it with the requests package.

Start by installing the package:

```sh
$ pip install --upgrade requests-cache
```

Then add the import to *app.py* as well as the `install_cache()` method:

```python
requests_cache.install_cache(cache_name='github_cache', backend='sqlite', expire_after=180)
```

Now whenever you use `requests`, the response will be cached automatically. Also, you can see that we are defining a few options. Take note of the `expire_after` option, which is set to 180 seconds. Since the Github API is frequently updated, we want to make sure we deliver the most up-to-date results. So, 180 seconds after the initial caching takes place, the request will re-fire and cache a new set of results, delivering updated results.

For more options, please check out the [official documentation](http://requests-cache.readthedocs.org/en/latest/api.html#requests_cache.core.install_cache).

So your *app.py* file should now look like this:

```python
import requests
import requests_cache

from flask import Flask, render_template, request, jsonify

app = Flask(__name__)

requests_cache.install_cache('github_cache', backend='sqlite', expire_after=180)


@app.route('/', methods=['GET', 'POST'])
def home():
    if request.method == 'POST':
        first = request.form.get('first')
        second = request.form.get('second')
        url = "https://api.github.com/search/users?q=location:{0}+language:{1}".format(first, second)
        response_dict = requests.get(url).json()
        return jsonify(response_dict)
    return render_template('index.html')


if __name__ == '__main__':
    app.run(debug=True)
```

## Test!

Fire up the app and search for a developer. Within the "app" directory a SQLite database, called *github_cache.sqlite*,  should be created. Now if you keep searching by the same location and programming language, `requests` will not actually make the call. Instead, it will use the cached response from the SQLite database.

Let's make sure that the cache actually expires. Update the `home()` view function like so:

```python
@app.route('/', methods=['GET', 'POST'])
def home():
    if request.method == 'POST':
        first = request.form.get('first')
        second = request.form.get('second')
        url = "https://api.github.com/search/users?q=location:{0}+language:{1}".format(first, second)
        now = time.ctime(int(time.time()))
        response = requests.get(url)
        print "Time: {0} / Used Cache: {1}".format(now, response.from_cache)
        return jsonify(response.json())
    return render_template('index.html')
```

So, here we're just using the `from_cache` attribute to see if the response came from cache. Let's test this out. Try a new search. Then open up your terminal:

```sh
Time: Fri Nov 28 13:34:25 2014 / Used Cache: False
```

So you can see we made the initial request to the Github API at 13:34:25, and since `False` was outputted to the screen, caching was not used. Try the search again.

```sh
Time: Fri Nov 28 13:35:28 2014 / Used Cache: True
```

Now you can see that caching is used. Try it a few more times.

```sh
Time: Fri Nov 28 13:36:10 2014 / Used Cache: True
Time: Fri Nov 28 13:37:59 2014 / Used Cache: False
Time: Fri Nov 28 13:39:09 2014 / Used Cache: True
```

So you can see that the cache expired, and we made a new API call at 13:37:59. After that, caching was used. Simple, right?

What happens when you change the parameters in your request? Try it. Enter a new location and programming language. What happened here? Well, since the parameters changed, Requests-cache treated it as a different request and did not use the cache.

## Balance - flush vs. performance

Again, in the above example, we expire the cache (commonly known as flushing) after 180 seconds in order to deliver the most up-to-date data to the end user. Think about that for a minute. Is it really necessary to flush it that regularly? Probably not. In this application, we could get away with changing this to five or ten minutes since it's not a huge issue if we miss a few new users added to the API every now and then. That said, you really want to pay close attention to flushing when the data is time-sensitive and paramount to your application's core functionality.

For example, if you were pulling data from an API that is updated several times a minute (like a [seismic activity API](http://www.programmableweb.com/api/seismic-data-portal)) and your end user must have the most updated data, then you would want to expire it every 30 or 60 seconds or so.

It's also important to balance the flushing frequency vs. the amount of time the call takes. If your API call is fairly expensive - perhaps it takes one to five seconds - then you want to increase the amount of time between flushing to improve performance.

## Conclusion

Caching is a powerful tool. In this case we improved our application's performance by limiting the number of external HTTP requests. We cut out the latency from the actual HTTP request itself. In many cases, you are not just making a request. You have to process the request as well, which could involve hitting a database, performing some sort of filtering, etc. Thus, caching can cut the latency from the processing of the request as well.

Cheers!

<hr>

Want the code? Grab it [here](https://github.com/realpython/flask-single-page-app/tree/part6).
