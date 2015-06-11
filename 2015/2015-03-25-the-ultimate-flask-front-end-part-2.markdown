# The Ultimate Flask Front-End - part 2

<div class="center-text">
  <img class="no-border" src="/images/ultimate-flask-front-end/main-logo-part2.png" style="max-width: 100%;" alt="main logo">
</div>

<br>

Welcome to part 2!

*Again, here's what we're covering*: **Let's look at the small, yet powerful JavaScript UI library [ReactJS](http://facebook.github.io/react/) in action, as we build a basic web application.** This app is powered by Python 3 and the [Flask framework](http://flask.pocoo.org/), in the back-end, along with [gulp.js](http://gulpjs.com/) (task runner), [bower](http://bower.io/) (front-end package manager), and [Browserify](http://browserify.org/) (JavaScript dependency bundler), in the front.

- Part 1 - [getting started](/blog/python/the-ultimate-flask-front-end/)
- **Part 2 - developing a dynamic search tool (current)**

> **Note** This tutorial uses React v[0.12.2](http://facebook.github.io/react/blog/2014/12/18/react-v0.12.2.html).

## React - round two

Hello World is great and all, but let's build something a bit more fun - a dynamic search tool. This [Component](http://facebook.github.io/react/docs/reusable-components.html) is used to filter down a list of items based on user input.

> Need the code? Grab it from the [repo](https://github.com/realpython/ultimate-flask-front-end/tags). Download [v2](https://github.com/realpython/ultimate-flask-front-end/releases/tag/v2) to start where we left off from part 1.

Add the following code to *project/statics/scripts/jsx/main.js*:

```javascript
/** @jsx React.DOM */

var DynamicSearch = React.createClass({

  // sets initial state
  getInitialState: function(){
    return { searchString: '' };
  },

  // sets state, triggers render method
  handleChange: function(event){
    // grab value form input box
    this.setState({searchString:event.target.value});
    console.log("scope updated!")
  },

  render: function() {

    var countries = this.props.items;
    var searchString = this.state.searchString.trim().toLowerCase();

    // filter countries list by value from input box
    if(searchString.length > 0){
      countries = countries.filter(function(country){
        return country.name.toLowerCase().match( searchString );
      });
    }

    return (
      <div>
        <input type="text" value={this.state.searchString} onChange={this.handleChange} placeholder="Search!" />
        <ul>
          { countries.map(function(country){ return <li>{country.name} </li> }) }
        </ul>
      </div>
    )
  }

});

// list of countries, defined with JavaScript object literals
var countries = [
  {"name": "Sweden"}, {"name": "China"}, {"name": "Peru"}, {"name": "Czech Republic"},
  {"name": "Bolivia"}, {"name": "Latvia"}, {"name": "Samoa"}, {"name": "Armenia"},
  {"name": "Greenland"}, {"name": "Cuba"}, {"name": "Western Sahara"}, {"name": "Ethiopia"},
  {"name": "Malaysia"}, {"name": "Argentina"}, {"name": "Uganda"}, {"name": "Chile"},
  {"name": "Aruba"}, {"name": "Japan"}, {"name": "Trinidad and Tobago"}, {"name": "Italy"},
  {"name": "Cambodia"}, {"name": "Iceland"}, {"name": "Dominican Republic"}, {"name": "Turkey"},
  {"name": "Spain"}, {"name": "Poland"}, {"name": "Haiti"}
];

React.render(
  <DynamicSearch items={ countries } />,
  document.getElementById('main')
);
```

**What's going on?**

We created a Component called `DynamicSearch`, which updates the DOM when the value in the input box is changed. How does this work? Well, the `handleChange()` function is called when a value is added or removed from the input box, which, in turn, updates the state via `setState()`. This method then calls the `render()` function to re-render the Component.

*The key takeaway here is that state changes only occur within the Component.*

Test it out:

<iframe width="100%" height="300" src="//jsfiddle.net/mjhea0/4orhg6q9/embedded/result,html/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

<br><br>

Since we added our JSX code to an external file, we need to trigger the transform, from JSX to vanilla JavaScript, outside the browser with Gulp.

## Gulp

[Gulp](http://gulpjs.com/) is a powerful task runner/build tool that can be used to handle the transform process. We'll also use it to watch for changes to our code (*project/static/scripts/jsx/main.js*) and automatically create new builds based on those changes.

### Initialization

Like bower, you can install gulp with npm. Install it globally, and then add it to the *package.json* file:

```sh
$ npm install -g gulp
$ npm install --save-dev gulp
```

Add a *gulpfile.js* at the root directory of your project:

```javascript
'use strict';

// requirements
var gulp = require('gulp');

gulp.task('default', function() {
  console.log("hello!")
});
```

This file tells gulp what tasks to run and in what order to run them in. You can see that our `default` task logs the string `hello!` to the console. You can run this task by simply running `gulp`. You should see something like:

```sh
$ gulp
[08:54:47] Using gulpfile ~/gulpfile.js
[08:54:47] Starting 'default'...
hello!
[08:54:47] Finished 'default' after 148 μs
```

We need the following gulp plugins for this project-

- [gulp-clean](https://github.com/peter-vilja/gulp-clean)
- [gulp-size](https://github.com/sindresorhus/gulp-size)
- [gulp-browserify](https://github.com/deepak1556/gulp-browserify)

-as well as-

- [react](https://github.com/facebook/react)
- [reactify](https://github.com/andreypopp/reactify)

-which we can add to the *package.json* file. Update the file like so:

```json
{
  "name": "ultimate-flask-front-end",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "bower": "^1.3.12",
    "gulp": "^3.8.11",
    "gulp-browserify": "^0.5.1",
    "gulp-clean": "^0.3.1",
    "gulp-size": "^1.2.1",
    "react": "^0.12.2",
    "reactify": "^1.0.0"
  }
}
```

Now just run `npm install` to install these additional plugins.

> You can see these installed plugins within the "node_modules" directory. Make sure to include this directory in your *.gitignore* file.

Finally, update *gulpfile.js*:

```javascript
'use strict';

// requirements

var gulp = require('gulp'),
    browserify = require('gulp-browserify'),
    size = require('gulp-size'),
    clean = require('gulp-clean');


// tasks

gulp.task('transform', function () {
  // add task
});

gulp.task('clean', function () {
  // add task
});

gulp.task('default', function() {
  console.log("hello!")
});
```

Now let's add some tasks, starting with `transform` to handle, well, the transform process from JSX to JavaScript.

### First task - `transform`

```javascript
gulp.task('transform', function () {
  return gulp.src('./project/static/scripts/jsx/main.js')
    .pipe(browserify({transform: ['reactify']}))
    .pipe(gulp.dest('./project/static/scripts/js'))
    .pipe(size());
});
```

The `task()` function takes two arguments - the task's name and an anonymous function to run in order to tell/instruct the task what to do. Here we define the task, `transform`, and then within the function, we:

 - Specify the source directory,
 - Create the Browserify bundler functionality along with the transformer via [Reactify](https://github.com/andreypopp/reactify),
 - Specify the destination directory, and
 - Calculate the size of the created file.

Gulp utilizes pipes to [stream](http://nodejs.org/api/stream.html) data for processing. After grabbing the source file (*main.js*), the file is then "piped" to the `browserify()` function for transformation/bundling. This transformed and bundled code is then "piped" to the destination directory along with the `size()` function.

> Curious about pipes and streams? Check out [this](https://github.com/substack/stream-handbook) excellent resource.

Ready for a quick test? Update *index.html*:

{% raw %}
```html
<!DOCTYPE html>
<html>
  <head lang="en">
    <meta charset="UTF-8">
    <title>Flask React</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <!-- styles -->
    <link rel="stylesheet" type="text/css" href="{{ url_for('static', filename='bower_components/bootstrap/dist/css/bootstrap.min.css') }}">
    <link rel="stylesheet" type="text/css" href="{{ url_for('static', filename='css/style.css') }}">
  </head>
  <body>
    <div class="container">
      <h1>Flask React</h1>
      <br>
      <div id="main"></div>
    </div>
    <!-- scripts -->
    <script src="{{ url_for('static', filename='bower_components/react/react.min.js') }}"></script>
    <script src="{{ url_for('static', filename='bower_components/jquery/dist/jquery.min.js') }}"></script>
    <script src="{{ url_for('static', filename='bower_components/bootstrap/dist/js/bootstrap.min.js') }}"></script>
    <script src="{{ url_for('static', filename='scripts/js/main.js') }}"></script>
  </body>
</html>
```
{% endraw %}

Then update the `default` task:

```javascript
gulp.task('default', function () {
  gulp.start('transform');
});
```

Test:

```sh
$ gulp
[08:58:39] Using gulpfile /gulpfile.js
[08:58:39] Starting 'default'...
[08:58:39] Starting 'transform'...
[08:58:39] Finished 'default' after 12 ms
[08:58:40] all files 2.37 kB
[08:58:40] Finished 'transform' after 181 ms
```

Did you notice the second to last line? This is the result of the `size()` function. In other words, the newly created JavaScript file (after the transform), *project/static/scripts/js/main.js*, is 2.37 kB in size.

Fire up the Flask server, and navigate to [http://localhost:5000/](http://localhost:5000/hello). You should see all the countries along with the search box. Test out the functionality. Also, if you open the JavaScript console within Chrome Developer Tools, you'll see the string, `scope updated!`, logged every time there's a change in scope - which comes from the `handleChange()` function in the `DynamicSearch` Component.

### Second task - `clean`

```javascript
gulp.task('clean', function () {
  return gulp.src(['./project/static/scripts/js'], {read: false})
    .pipe(clean());
});
```

When this task is ran, we grab the source directory - the result of the `transform` task - and then run the `clean()` function to remove the directory and its contents. It's a good idea to run this before each new build to ensure you start fresh and clean.

Try running `gulp clean`. This should remove "project/static/scripts/js". Let's add it to our `default` task so that it automatically runs before the transformation:

```javascript
gulp.task('default', ['clean'], function () {
  gulp.start('transform');
});
```

Be sure to test this out before moving on.

### Third task -`watch`

Finally, update the `default` task one last time, adding the ability to automatically re-run the `transform` task anytime changes are made to the *project/static/scripts/jsx/main.js* file:

```javascript
gulp.task('default', ['clean'], function () {
  gulp.start('transform');
  gulp.watch('./project/static/scripts/jsx/main.js', ['transform']);
});
```

Open up a new terminal window and run `gulp` to generate a new build and activate the `watcher` functionality. In the other window run `sh run.sh` to run the Flask server. Your app should be running. Now if you comment out all the code in the *project/static/scripts/jsx/main.js* file, this will trigger the `transform` function. Refresh the browser to see the changes. Make sure to revert the changes when done.

Want to take this to the next level? Check out the [Livereload](https://github.com/vohof/gulp-livereload) plugin.

## Conclusion

Here's the end result after adding some custom styles to *project/static/css/style.css*:

<div class="center-text">
  <img class="no-border" src="/images/ultimate-flask-front-end/flask-react-dynamic-search.png" style="max-width: 100%;" alt="flask reactjs dynamic search">
</div>

<br>

Be sure to check out the official [documentation](http://facebook.github.io/react/) along with the excellent [tutorial](http://facebook.github.io/react/docs/tutorial.html) for more information on React.

Grab the code from the [repo](https://github.com/realpython/ultimate-flask-front-end). Comment below if you have any questions or spot any errors. Also, what else would you like to see? We may add a third part if people are interested. Comment below.


<br>

<p style="font-size: 14px;">
  <em>Edits made by <a href="https://twitter.com/diek007">Derrick Kearney</a>. Thanks again!</em>
</p>