# The Ultimate Flask Front-End

<div class="center-text">
  <img class="no-border" src="/images/ultimate-flask-front-end/main-logo.png" style="max-width: 100%;" alt="main logo">
</div>

<br>

**Let's look at the small, yet powerful JavaScript UI library [ReactJS](http://facebook.github.io/react/) in action, as we build a basic web application.** This app is powered by Python 3 and the [Flask framework](http://flask.pocoo.org/) in the back-end and [React](https://facebook.github.io/react/) in the front. In addition, we will use [gulp.js](http://gulpjs.com/) (task runner), [bower](http://bower.io/) (front-end package manager), and [Browserify](http://browserify.org/) (JavaScript dependency bundler).

- **Part 1 - getting started (current)**
- Part 2 - [developing a dynamic search tool](/blog/python/the-ultimate-flask-front-end-part-2/)

<br>

*Updates:*

  - 05/22/2016: Upgraded to the latest version of React (v[15.0.1](http://facebook.github.io/react/blog/2016/04/08/react-v15.0.1.html)).

## React Explained

React is a library, not a framework. Unlike client-side MVC frameworks, like Backbone, Ember, and AngularJS, it makes no
assumptions about your tech stack so you can easily integrate it into a new or legacy code base. It's often used to manage *specific* areas of an application's UI, rather than the *entire* UI.

React's only concern is with the user interface (the 'V' in MVC), which is defined by a hierarchy of modular view [Components](http://facebook.github.io/react/docs/reusable-components.html) that couple static markup with dynamic JavaScript. If you're familiar with Angular, these Components are similar to [Directives](https://docs.angularjs.org/guide/directive). Components use an XML-like syntax called [JSX](http://facebook.github.io/jsx/) that transcompiles down to vanilla JavaScript.

Since Components are defined in a hierarchical order, you don't have to re-render the entire DOM when a state changes. Instead, it uses a [Virtual DOM](http://facebook.github.io/react/docs/glossary.html) that only re-renders the individual Components after the state has changed, at blazingly fast speeds!

Be sure to review the [Getting Started](http://facebook.github.io/react/docs/getting-started.html) guide and the excellent [Why did we build React?](http://facebook.github.io/react/blog/2013/06/05/why-react.html) blog post from the official [React](http://facebook.github.io/react/) documentation.

## Project Setup

Let's start with what we know: Flask.

Download the boilerplate code from [the repository](https://github.com/realpython/ultimate-flask-front-end/releases/tag/v2.1), extract the files, create then activate a virtualenv, and install the requirements:
`pip install -r requirements.txt`


Finally, let's run the app and start the show:

```sh
$ sh run.sh
```

## React - round one

Let’s look at a simple Component to get our feet wet.

### The Component: Moving from Static to React
We are going to add this JSX script to our `hello.html`. Take a minute to check it out.

```javascript
<script type="text/jsx">

  /*** @jsx React.DOM */

  var realPython = React.createClass({
    render: function() {
      return (<h2>Greetings, from Real Python!</h2>);
    }
  });

  ReactDOM.render(
    React.createElement(realPython, null),
    document.getElementById('content')
  );

</script>
```

**What's going on?**

1. We created a Component by calling `createClass()`, then assigned it to the variable `realPython`. `React.createClass()` takes a single argument, an object.
1. Inside this object we added a `render()` function that declaratively updates the DOM when called.
1. Next comes a return value of `<h2>Greetings, from Real Python!</h2>`, in JSX, which represents the *actual* HTML element that will be added to the DOM.
1. Finally, `ReactDOM.render()` instantiates the `realPython` Component and injects the markup into a DOM element with an `ID` selector of `content`.

> Refer to the [official docs](http://facebook.github.io/react/docs/tutorial.html#jsx-syntax) for more info.

### The Transformation

What's next? Well, we need to "transform", or transcompile, the JSX to vanilla JavaScript. This is easy. Update *hello.html* like so:

```html
<!DOCTYPE html>
<html>
  <head lang="en">
    <meta charset="UTF-8">
    <title>Flask React</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <!-- styles -->
  </head>
  <body>
    <div class="container">
      <h1>Flask React</h1>
      <br>
      <div id="content"></div>
    </div>
    <!-- scripts -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.1.0/react.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.1.0/react-dom.min.js"></script>
    <script src="http://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/JSXTransformer.js"></script>
    <script type="text/jsx">

      /*** @jsx React.DOM */

      var realPython = React.createClass({
        render: function() {
          return (<h2>Greetings, from Real Python!</h2>);
        }
      });

      ReactDOM.render(
        React.createElement(realPython, null),
        document.getElementById('content')
      );

    </script>
  </body>
</html>
```

Here we addded the `helloWorld` Component to the template along with the following scripts-

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.1.0/react.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.1.0/react-dom.min.js"></script>
<script src="http://cdnjs.cloudflare.com/ajax/libs/react/0.13.3/JSXTransformer.js"></script>
```

-the latter of which, `JSXTransformer.js`, searches for `<script>` tags with `type="text/jsx"` and then "transforms" the JSX syntax into vanilla JavaScript within the browser. *Please note that this tool is [deprecated](https://facebook.github.io/react/blog/2015/06/12/deprecating-jstransform-and-react-tools.html#jsxtransformer) and has been replaced by [Browserify](http://browserify.org/), which we will set up later on.*

> Notice how we did not add jQuery since it is **not** required for React.

That's it. Run the Flask development server and check out the results in the browser at [http://localhost:5000/hello](http://localhost:5000/hello).

<div class="center-text">
  <img class="no-border" src="/images/ultimate-flask-front-end/flask-react-hello-world.png" style="max-width: 100%;" alt="flask reactjs hello world">
</div>

## Bower

Instead of using the pre-built JavaScript files, from the CDN, let's use [bower](http://bower.io/) to better ([IMHO](http://htmlcheats.com/cdn-2/6-reasons-use-cdn/)) manage those dependencies. Bower is a powerful package manager for front-end dependencies - i.e., jQuery, Bootstrap, React, Angular, Backbone.

> Make sure you have [Node and npm](http://nodejs.org/download/) installed before moving on.

### Initialization

Install bower with npm:

```
$ npm install -g bower
```

[npm](https://www.npmjs.org/) is another package manager used to manage Node modules. Unlike PyPI/pip, the default behavior of npm is to install dependencies at the local level. The `-g` flag is used to override that behavior to install bower globally since you will probably use bower for a number of projects.

### *bower.json*

Bower uses a file called *bower.json* to define project dependencies, which is similar to a *requirements.txt* file. Run the following command to interactively create this file:

```sh
$ bower init
```

Just accept the defaults for now. Once done, your *bower.json* file should look something like this:

```json
{
  "name": "ultimate-flask-front-end",
  "homepage": "https://github.com/realpython/ultimate-flask-front-end",
  "authors": [
    "Michael Herman michael@realpython.com"
  ],
  "description": "",
  "main": "",
  "license": "MIT",
  "ignore": [
    "**/.*",
    "node_modules",
    "bower_components",
    "test",
    "tests"
  ]
}
```

> For more on the *bower.json* and the `init` command, check out the official [documentation](http://bower.io/docs/creating-packages/#bowerjson).

### npm

Like the *bower.json* file, npm utilizes a similar file, called *package.json* to define project-specific dependencies. You can also create it interactively:

```sh
$ npm init
```

Againm, accept the defaults:

```json
{
  "name": "ultimate-flask-front-end",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/realpython/ultimate-flask-front-end.git"
  },
  "author": "",
  "license": "ISC",
  "bugs": {
    "url": "https://github.com/realpython/ultimate-flask-front-end/issues"
  },
  "homepage": "https://github.com/realpython/ultimate-flask-front-end#readme"
}
```

Now, let's add bower to the npm dependency file:

```sh
$ npm install --save-dev bower
```

### Configuration

Along with the *bower.json* file, we can define [configuration settings](http://bower.io/docs/config/) in a file called *.bowerrc*. Create the file now within the project root. Your project structure should now look like this:

```sh
├── .bowerrc
├── .gitignore
├── bower.json
├── package.json
├── project
│   ├── app.py
│   ├── static
│   │   └── css
│   │       └── style.css
│   └── templates
│       ├── hello.html
│       └── index.html
├── requirements.txt
└── run.sh
```

The standard behavior is for bower to install packages in a directory called "bower_components" in the project root. We need to override this behavior since Flask needs access to the packages within the *static* directory. Thus, add the following JSON to the file so that bower automatically installs the file in the correct directory:

```json
{
  "directory": "./project/static/bower_components"
}
```

### Installation

Let's install Bootstrap and React. This can be done in one of two ways:

1. Run `bower install <package_name> --save` for each package (the `--save` flag adds the dependencies (name and version) to the *bower.json* file.).
1. Update the *bower.json* file directly with each dependency (again, name and version) and then run `bower install` to install all dependencies from the file.

Since we (err, *I*) know the versions already, let's use the second method. Update the *bower.json* file like so-

```json
{
  "name": "ultimate-flask-front-end",
  "homepage": "https://github.com/realpython/ultimate-flask-front-end",
  "authors": [
    "Michael Herman michael@realpython.com"
  ],
  "description": "",
  "main": "",
  "license": "MIT",
  "ignore": [
    "**/.*",
    "node_modules",
    "bower_components",
    "test",
    "tests"
  ],
  "dependencies": {
    "bootstrap": "^3.3.6",
    "react": "^15.1.0"
  }
}
```

-and then run `bower install`:

```sh
$ bower install
bower cached        https://github.com/twbs/bootstrap.git#3.3.6
bower validate      3.3.6 against https://github.com/twbs/bootstrap.git#^3.3.6
bower cached        https://github.com/facebook/react-bower.git#15.1.0
bower validate      15.1.0 against https://github.com/facebook/react-bower.git#^15.1.0
bower cached        https://github.com/jquery/jquery-dist.git#2.2.4
bower validate      2.2.4 against https://github.com/jquery/jquery-dist.git#1.9.1 - 2
bower install       react#15.1.0
bower install       bootstrap#3.3.6
bower install       jquery#2.2.4

react#15.1.0 project/static/bower_components/react

bootstrap#3.3.6 project/static/bower_components/bootstrap
└── jquery#2.2.4

jquery#2.2.4 project/static/bower_components/jquery
```

You should now see the "project/static/bower_components" directory.

Now you can pull in all the required dependencies with pip, npm, and bower after cloning the [repo](https://github.com/realpython/ultimate-flask-front-end):

```sh
$ pip install -r requirements.txt
$ npm install
$ bower install
```

### Test

Update the React scripts in *hello.html*:

{% raw %}
```html
    <script src="{{ url_for('static', filename='bower_components/react/react.min.js') }}"></script>
    <script src="{{ url_for('static', filename='bower_components/react/react-dom.min.js') }}"></script>
```
{% endraw %}

Test out the app to make sure it still works.

## Next Steps

With the tools set up, we'll move back to React and develop a more robust app in the [second part](/blog/python/the-ultimate-flask-front-end-part-2/). Cheers!