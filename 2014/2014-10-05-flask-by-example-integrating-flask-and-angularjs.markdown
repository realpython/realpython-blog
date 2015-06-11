---
layout: post
title: "Flask by Example - Integrating Flask and AngularJS"
date: 2014-10-06 06:48:03 -0700
toc: true
comments: true
category_side_bar: true
categories: [python, flask, front-end]

keywords: "python, web development, flask, angular, angularjs, polling"
description: "This tutorial shows you how to utilize AngularJS to create a simple polling service."
---

{% assign openTag = '{%' %}

**Welcome back. With the Redis task queue setup, let's use [AngularJS](https://angularjs.org/) to (a) poll the back-end to see if the task is complete and then (b) update the DOM once the data is made available.**

**Updated 02/22/2015:** Added Python 3 support.

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask_by_example/ngflask.png" alt="angular logo">
</div>

<br>

Remember, here's what we're building: A Flask app that calculates word-frequency pairs based on the text from a given URL. This is a full-stack tutorial.

1. [Part One](http://www.realpython.com/blog/python/flask-by-example-part-1-project-setup): Setup a local development environment and then deploy both a staging environment and a production environment on Heroku.
1. [Part Two](http://www.realpython.com/blog/flask-by-example-part-2-postgres-sqlalchemy-and-alembic): Setup a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations.
1. [Part Three](https://realpython.com/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/): Add in the back-end logic to scrape and then process the counting of words from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
1. [Part Four](https://realpython.com/blog/python/flask-by-example-implementing-a-redis-task-queue/): Implement a Redis task queue to handle the text processing.
1. **Part Five: Setup Angular on the front-end to continuously poll the back-end to see if the request is done. (current)**
1. [Part Six](https://realpython.com/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis, detailing how to run two processes (web and worker) on a single Dyno.
1. Part Seven: Update the front-end to make it more user-friendly.
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

**New to Angular?** Check out [this](https://github.com/mjhea0/thinkful-angular) tutorial as well as [this](https://www.youtube.com/watch?v=WuiHuZq_cg4) video to get up to speed.

## Current Functionality

Let's start by looking at the current state of our app.

Start Redis in one terminal window:

```sh
$ redis-server
```

In another window, navigate to your project directory and then run the worker:

```sh
$ workon wordcounts
$ python worker.py
20:38:04 RQ worker started, version 0.4.6
20:38:04
20:38:04 *** Listening on default...
```

Finally, open a third terminal window, navigate to your project directory, and fire up the main app:

```sh
$ workon wordcounts
$ python manage.py runserver
```

Open up [http://localhost:5000/](http://localhost:5000/) and test with the URL 'http://realpython.com'. In the terminal a job id should have outputted. Grab the id and navigate to this url:

[http://localhost:5000/results/add_the_job_id_here]( http://localhost:5000/results/add_the_job_id_here)

You should see a similar JSON response in your browser:

```sh
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

Now we're ready to add in Angular.

## Requirements

Add Angular to *index.html*:

```html
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.3.13/angular.min.js"></script>
```

## Update *index.html*

Add the following [directives](https://docs.angularjs.org/guide/directive) to *index.html*:

1. **`ng-app`**: `<html ng-app="WordcountApp">`
1. **`ng-controller`**: `<body ng-controller="WordcountController">`
1. **`ng-submit`**: `<form role="form" ng-submit="getResults()">`

So, we bootstrap Angular (which tells Angular to treat this page as an Angular application), add a controller, and then fire a function called `getResults()` when the form is submitted. For more on these directives, please consult the official Angular [documentation](https://angularjs.org/): [ngApp](https://docs.angularjs.org/api/ng/directive/ngApp), [ngController](https://docs.angularjs.org/api/ng/directive/ngController), and [ngSubmit](https://docs.angularjs.org/api/ng/directive/ngSubmit).

## Create the Angular Module

Create a "static" directory, and then add a file called *main.js* to that directory. Be sure to add the requirement to the *index.html* file: {% raw %}`<script src="{{ url_for('static', filename='main.js') }}"></script>`{% endraw %}

Let's start with this basic code:

```javascript
(function () {

  'use strict';

  angular.module('WordcountApp', [])

  .controller('WordcountController', ['$scope', '$log', function($scope, $log) {
    $scope.getResults = function() {
      $log.log("test");
    };
  }

  ]);

}());
```

Here, when the form is submitted, `getResults()` is called, which simply logs the text "test" to the JavaScript console in the browser. Be sure to test it out.

### Dependency Injection and $scope

In the above example, we are using a design pattern called [dependency injection](https://docs.angularjs.org/guide/di) to "inject" the `$scope` object and `$log` service. Stop here. It's very important that you understand `$scope`. Start with the Angular [documentation](https://docs.angularjs.org/guide/scope), then be sure to run through the [intro](https://github.com/mjhea0/thinkful-angular) tutorial if you haven't already. `$scope` may sound complicated but it really just provides a means of communication between the View and Controller. Both have access to it, and when you change a variable attached to `$scope` in one, the variable will automatically update in the other ([data binding](https://docs.angularjs.org/guide/databinding)). The same is true with dependency injection: It's much simpler than it sounds. Think of it as just a bit of magic for obtaining access to various services. So by injecting a service, we can now use it in our controller.

Back to our app...

If you test this out, you'll see that the form submission no longer sends a POST request to the back end. This is exactly what we want. Instead, we'll use the Angular `$http` service to handle this request asynchronously:

```javascript
.controller('WordcountController', ['$scope', '$log', '$http', function($scope, $log, $http) {
$scope.getResults = function() {

  $log.log("test");

  // get the URL from the input
  var userInput = $scope.input_url;
  // fire the API request
  $http.post('/start', {"url": userInput}).
    success(function(results) {
      $log.log(results);
    }).
    error(function(error) {
      $log.log(error);
    });

};
}

]);
```

Also, update the `input` in *index.html*:

```html
<input type="text" ng-model="input_url" name="url" class="form-control" id="url-box" placeholder="Enter URL..." style="max-width: 300px;">
```

We inject the `$http` service, grab the URL from the input box (via `ng-model="input_url"`), and then issue a POST request to the back end. The `success` and `error` callbacks handle the response. In the case of a 200 response, it will be handled by the `success` handler, which, in turn, logs the response to the console.

Before testing, let's refactor the back end, since the `/start` endpoint does not currently exist.

## Refactor *app.py*:

Refactor out the Redis job creation from the index view function, and then add it to a new view function called `get_counts()`:

```python
@app.route('/', methods=['GET', 'POST'])
def index():
    return render_template('index.html')


@app.route('/start', methods=['POST'])
def get_counts():
    # get url
    data = json.loads(request.data.decode())
    url = data["url"]
    # form URL, id necessary
    if 'http://' not in url[:7]:
        url = 'http://' + url
    # start job
    job = q.enqueue_call(
        func=count_and_save_words, args=(url,), result_ttl=5000
    )
    # return created job id
    return job.get_id()
```

Make sure to add the following import at the top as well:

```python
import json
```

These changes should be straightforward.

Now we test. Refresh your browser, submit a new URL. You should see the job id in your JavaScript console. Perfect. Now that Angular has the job id we can add in the polling functionality.

## Basic Polling

Update *main.js* by adding the following code to the controller:

```javascript
function getWordCount(jobID) {

  var timeout = "";

  var poller = function() {
    // fire another request
    $http.get('/results/'+jobID).
      success(function(data, status, headers, config) {
        if(status === 202) {
          $log.log(data, status);
        } else if (status === 200){
          $log.log(data);
          $timeout.cancel(timeout);
          return false;
        }
        // continue to call the poller() function every 2 seconds
        // until the timeout is cancelled
        timeout = $timeout(poller, 2000);
      });
  };
  poller();
}
```

Then update the success handler in the POST request:

```javascript
$http.post('/start', {"url": userInput}).
  success(function(results) {
    $log.log(results);
    getWordCount(results);

  }).
  error(function(error) {
    $log.log(error);
  });
```

Make sure to inject the `$timeout` service into the controller as well.

**What's happening here?**

1. A success results in the firing of the `getWordCount()` function.
1. Within the `poller()` function, we call the `/results/job_id` endpoint.
1. Using the `$timeout` service, this function continues to fire every 2 seconds until the timeout is cancelled when a 200 response is returned along with the word counts.

Check out the Angular [documentation](https://docs.angularjs.org/api/ng/service/$timeout) for an awesome description of how the `$timeout` service works.

When you test this out, be sure to open the JavaScript console. You should see something similar to this:

```
Nay! 202
Nay! 202
Nay! 202
Nay! 202
Nay! 202
Nay! 202
Object {Course: 5, Download: 4, Python: 19, Real: 11, courses: 7…}
```

So, in the above example, the `poller()` function is called seven times. The first six calls returned a 202, while the last call returned a 200 along with the word count object.

Perfect.

Now we need to append the word counts to the DOM.

## Updating the DOM

Update *index.html*:


```html
<div class="container">
  <div class="row">
    <div class="col-sm-5 col-sm-offset-1">
      <h1>Wordcount 3000</h1>
      <br>
      <form role="form" ng-submit="getResults()">
        <div class="form-group">
          <input type="text" name="url" class="form-control" id="url-box" placeholder="Enter URL..." style="max-width: 300px;" ng-model="input_url" required>
        </div>
        <button type="submit" class="btn btn-default">Submit</button>
      </form>
    </div>
    <div class="col-sm-5 col-sm-offset-1">
      <h2>Frequencies</h2>
      <br>
      {{ openTag }} raw %}
      <div id="results">
        {% raw %}
          {{wordcounts}}
        {% endraw %}
      </div>
      {{ openTag }} endraw %}
    </div>
  </div>
</div>
```

**What did we change?**

1. The `input` tag now has a `required` attribute, indicating that the input box must be filled out before the form can be submitted.
1. Say goodbye to the Jinja2 template tags. Jinja2 is served from the server side, and since the polling is handled completely on the client side, we need to use Angular tags. That said, since both Jinja2 and Angular template tags utilize double curly braces, {% raw %}`{{}}`{% endraw %}, we have to to escape the Jinja2 tags using `{{ openTag }} raw %}` and `{{ openTag }} endraw %}`. *If you need to use a number of Angular tags, it's a good idea to change the template tags AngularJS uses with the `$interpolateProvider`. For more, check out the Angular [docs](https://docs.angularjs.org/api/ng/provider/$interpolateProvider).*

Second, update the success handler in the `poller()` function:

```javascript
success(function(data, status, headers, config) {
  if(status === 202) {
    $log.log(data, status);
  } else if (status === 200){
    $log.log(data);
    $scope.wordcounts = data;
    $timeout.cancel(timeout);
    return false;
  }
```

Here, we attach the results to the `$scope` object so that it's available to the view.

Test that out. If all went well, you should see the object on the DOM. Not very pretty, but that's an easy fix with Bootstrap:

```html
<div id="results">
  <table class="table table-striped">
    <thead>
      <tr>
        <th>Word</th>
        <th>Count</th>
      </tr>
    </thead>
    <tbody>
    {{ openTag }} raw %}
      <tr ng-repeat="(key, val) in wordcounts">
        {% raw %}
        <td>{{key}}</td>
        <td>{{val}}</td>
        {% endraw %}
      </tr>
    {{ openTag }} endraw %}
    </tbody>
  </table>
</div>
```

## Conclusion and Next Steps

Before we move on to charting with D3, we still need to:

1. Add a loading spinner: Also know as a [throbber](http://en.wikipedia.org/wiki/Throbber), this will be displayed until the task is done so that the end user knows that something is happening.
1. Refactor the Angular Controller: Right now there's too much happening (logic) in the controller. We need to move the majority of the functionality into a [service](https://docs.angularjs.org/guide/providers). We'll discuss both the *why* and *how*.
1. Update Staging: We need to update our staging environment on Heroku - adding the code changes, our worker, and Redis.

See you next [time](https://realpython.com/blog/python/updating-the-staging-environment/). Best!
