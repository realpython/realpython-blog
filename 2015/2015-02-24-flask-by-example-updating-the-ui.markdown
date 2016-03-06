# Flask by Example - Updating the UI

{% assign openTag = '{%' %}

*The following is a collaboration piece between Cam Linke, co-founder of [Startup Edmonton](http://startupedmonton.com/), and the folks at Real Python.*

<br>

**In part 7 of this series, we'll work on the user interface to make it, well, more user friendly.**

Remember, here's what we're building: A Flask app that calculates word-frequency pairs based on the text from a given URL. This is a full-stack tutorial.

1. [Part One](http://www.realpython.com/blog/python/flask-by-example-part-1-project-setup): Setup a local development environment and then deploy both a staging environment and a production environment on Heroku.
1. [Part Two](http://www.realpython.com/blog/flask-by-example-part-2-postgres-sqlalchemy-and-alembic): Setup a PostgreSQL database along with SQLAlchemy and Alembic to handle migrations.
1. [Part Three](https://realpython.com/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/): Add in the back-end logic to scrape and then process the counting of words from a webpage using the requests, BeautifulSoup, and Natural Language Toolkit (NLTK) libraries.
1. [Part Four](https://realpython.com/blog/python/flask-by-example-implementing-a-redis-task-queue/): Implement a Redis task queue to handle the text processing.
1. [Part Five](https://realpython.com/blog/python/flask-by-example-integrating-flask-and-angularjs/): Setup Angular on the front-end to continuously poll the back-end to see if the request is done.
1. [Part Six](https://realpython.com/blog/python/updating-the-staging-environment/): Push to the staging server on Heroku - setting up Redis, detailing how to run two processes (web and worker) on a single Dyno.
1. **Part Seven: Update the front-end to make it more user-friendly. (current)**
1. Part Eight: Add the D3 library into the mix to graph a frequency distribution and histogram.

> Need the code? Grab it from the [repo](https://github.com/realpython/flask-by-example/releases).

## Current User Interface

Let's look at the current user interface...

Start Redis in a terminal window:

```sh
$ redis-server
```

Then get your worker going in another window:

```sh
$ cd wordcounts
$ source env/bin/activate
$ export APP_SETTINGS="config.DevelopmentConfig"
$ export DATABASE_URL="postgresql://localhost/wordcount_dev"
$ python worker.py
17:11:39 RQ worker started, version 0.4.6
17:11:39
17:11:39 *** Listening on default...
```

Finally, in a third window, fire up the app:

```sh
$ cd wordcounts
$ source env/bin/activate
$ export APP_SETTINGS="config.DevelopmentConfig"
$ export DATABASE_URL="postgresql://localhost/wordcount_dev"
$ python manage.py runserver
```

Test the app out to make sure it still works. You should see something like:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask_by_example/current-ui.png" alt="current user interface">
</div>

<br>

Let's make some changes.

1. We'll start by disabling the submit button to prevent users from continually clicking while they are waiting for the submitted site to be counted.
1. Next, while our application is counting the words, we're going to display a [throbber](http://en.wikipedia.org/wiki/Throbber)/loading spinner where the wordcount list will go to show the user that there is activity happening in the back-end.
1. Finally, we're going to display an error if the domain is unable to be reached.

## Changing the button

Change the button in the HTML to the following:

```html
{{ openTag }} raw %}
{% raw %}<button type="submit" class="btn btn-primary" ng-disabled="loading">{{ submitButtonText }}</button>{% endraw %}
{{ openTag }} endraw %}
```

We added an `ng-disabled` [directive](https://docs.angularjs.org/api/ng/directive/ngDisabled) and attached that to `loading`. This will disable the button when `loading` evaluates to `true`. Next, we added a variable to display to the user called `submitButtonText`. This way we'll be able to change the text from `"Submit"` to `"Loading..."` so the user knows what's going on. We then wrapped the button in `{{ openTag }} raw %}` and `{{ openTag }} endraw %}` so that Jinja knows to evaluate this as raw HTML. If we don't do this, Flask will try to evaluate the {% raw %}`{{ submitButtonText }}`{% endraw %} as a Jinja variable and Angular won't get a chance to evaluate it.

The accompanying JavaScript is fairly simple.

At the top of the `WordcountController` in *main.js* add the following code:

```javascript
$scope.submitButtonText = "Submit";
$scope.loading = false;
```

This sets the initial value of `loading` to `false` so that the button will not be disabled. It also initializes the button's text as `"Submit"`.

Change the post call to:

```javascript
$http.post('/start', {"url": userInput}).
  success(function(results) {
    $log.log(results);
    getWordCount(results);
    $scope.wordcounts = null;
    $scope.loading = true;
    $scope.submitButtonText = "Loading...";
  }).
  error(function(error) {
    $log.log(error);
  });
```

We added three lines, which set...

1. `wordcounts` to `null` so that old values get cleared out.
1. `loading` to `true` so that the loading button will be disabled via the `ng-disabled` directive we added to our HTML.
1. `submitButtonText` to `"Loading..."` so that the user knows why the button is disabled.

Next update the `poller` function:

```javascript
var poller = function() {
  // fire another request
  $http.get('/results/'+jobID).
    success(function(data, status, headers, config) {
      if(status === 202) {
        $log.log(data, status)
      } else if (status === 200){
        $log.log(data);
        $scope.loading = false;
        $scope.submitButtonText = "Submit";
        $scope.wordcounts = data;
        $timeout.cancel(timeout);
        return false;
      }
      // continue to call the poller() function every 2 seconds
      // until the timeout is cancelled
      timeout = $timeout(poller, 2000);
    });
};
```

When the result is successful we set loading back to `false` so that the button is enabled again, and we change the button text back to `"Submit"` so the user knows they can submit a new URL.

Test it out!

## Adding a spinner

Next we want to add a spinner below our wordcount section so the user knows what's going on. This is accomplished by adding an animated gif below the results `div` as shown below:

```html
<div class="col-sm-5 col-sm-offset-1">
  <h2>Frequencies</h2>
  <br>
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
  {% raw %}<img class="col-sm-3 col-sm-offset-4" src="{{ url_for('static', filename='spinner.gif') }}" ng-show="loading">{% endraw %}
```

Be sure to grab *spinner.gif* from the [repo](https://github.com/realpython/flask-by-example/tree/master/static).

You can see that `ng-show` is attached to `loading` just like the button is. This way when `loading` is set to `true` the spinner gif is shown. When `loading` is set to `false` - e.g., when the wordcount is finished - the spinner disappears.

## Dealing with errors

Finally we want to deal with the case where the user submits a bad URL. Start by adding the following HTML below the form:

```html
<div class="alert alert-danger" role="alert" ng-show='urlError'>
  <span class="glyphicon glyphicon-exclamation-sign" aria-hidden="true"></span>
  <span class="sr-only">Error:</span>
  There was an error submitting your URL. Please check to make sure it is valid before trying again.
</div>
```

This uses Bootstrap's `alert` [class](http://getbootstrap.com/components/#alerts) to show a warning dialog if the user submits a bad URL. We use Angular's `ng-show` [directive](https://docs.angularjs.org/api/ng/directive/ngShow), similar to what we did with the spinner, to only show up when `urlError` is `true`.

Finally, in the `WordcountController` we want to initialize `$scope.urlError` to `false` so the warning doesn't show up at the start:

```javascript
$scope.urlError = false;
```

We'll catch the errors in the `poller` function:

```js
var poller = function() {
  // fire another request
  $http.get('/results/'+jobID).
    success(function(data, status, headers, config) {
      if(status === 202) {
        $log.log(data, status);
      } else if (status === 200){
        $log.log(data);
        $scope.loading = false;
        $scope.submitButtonText = "Submit";
        $scope.wordcounts = data;
        $timeout.cancel(timeout);
        return false;
      }
      // continue to call the poller() function every 2 seconds
      // until the timeout is cancelled
      timeout = $timeout(poller, 2000);
    }).
    error(function(error) {
      $log.log(error);
      $scope.loading = false;
      $scope.submitButtonText = "Submit";
      $scope.urlError = true;
    });
};
```

This logs the error to the console, changes `loading` to `false`, sets the submit button's text back to `"Submit"` so that the user can try again, and changes `urlError` to `true` so that the warning shows up.

Lastly, in our `success` function for the post call we want to set `urlError` to `false` so that it disappears when the user tries to submit a new url:

```javascript
$scope.urlError = false;
```

With that we've cleaned up the user interface a bit so that the user knows what is happening while we are running the wordcount functionality behind the scenes.

Test this out!

## Conclusion

What else could you add or change to better the user experience? Make the changes on your own or leave comments below. When done be sure to update both your [staging](http://wordcounts-stage.herokuapp.com/) and [production](http://wordcounts-pro.herokuapp.com/) environments.

See you next time!
