# Developing with Bottle

<div class="center-text">
  <img class="no-border" src="/images/blog_images/bottle/bottle-logo.png" style="max-width: 100%;" alt="bottle logo">
</div>

<br>

I love [bottle](http://bottlepy.org/docs/stable/). It's a simple, yet fast and powerful Python micro-framework, perfect for small web applications and rapid prototyping. It's also an excellent learning tool for those just getting starting with web development.

Let's look at a quick example.

> **NOTE**: This tutorial assumes you are running a Unix-based environment - e.g, Mac OS X, a Linux flavor, or a Linux flavor powered through a virtual machine. We recommend using [Sublime Text 3](https://realpython.com/blog/python/setting-up-sublime-text-3-for-full-stack-python-development/) as your text editor.

**Updated 06/13/2015:** updated code examples and explanations

## Starting up

First, let's create a directory to work in:

```sh
$ mkdir bottle && cd bottle
```

Next, you need to have pip, virtualenv, and git installed.

[virtualenv](https://pypi.python.org/pypi/virtualenv) is a Python tool that makes it easy to manage the Python packages needed for a particular project; it keeps packages in one project from conflicting with packages in other projects. [pip](https://pypi.python.org/pypi/pip), meanwhile, is a package manager used for managing the installation of Python packages.

For help with installing pip (and it's dependencies) in a Unix environment, follow the instructions in this [Gist](https://gist.github.com/mjhea0/5692708). If you are on Windows environment, please view this [video](http://www.youtube.com/watch?v=MIHYflJwyLk) for help.

Once you have pip installed, run the following command to install virtualenv:

```sh
$ pip install virtualenv==12.0.7
```

Now we can easily setup our local environment:

```sh
$ virtualenv venv
$ source venv/bin/activate
```

Install bottle:

```sh
$ pip install bottle==0.12.8
$ pip freeze > requirements.txt
```

Finally, let's put our app under version control using Git. For more information on Git, please view this [site](http://git-scm.com/), which also includes installation instructions.

```sh
$ git init
$ git add .
$ git commit -m "initial commit"
```

## Writing your app

We're ready to write our bottle app. Create your application file, *app.py*, which will hold the *entirety* of our first app:

{% raw %}
```python
import os
from bottle import route, run, template

index_html = '''My first web app! By <strong>{{ author }}</strong>.'''


@route('/')
def index():
    return template(index_html, author='Real Python')


@route('/name/<name>')
def name(name):
    return template(index_html, author=name)


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    run(host='0.0.0.0', port=port, debug=True)
```
{% endraw %}

Save the file.

Now you can run your app locally:

```sh
$ python app.py
```

You should be able to connect to [http://localhost:8080/](http://localhost:8080/) and see your application running!

```
My first web app! By RealPython.
```

So, the `@route` decorator binds a function to the route. In the first route, `/`, the `index()` function is bound to the route, which renders the `index_html` template and passes in a variable, `author`, as a keyword argument. This variable is then accessible in the template.

Now navigate to the next route, making sure to add your name on the end of the route - i.e., [http://localhost:8080/name/Michael](http://localhost:8080/name/Michael). You should see something like:

```
My first web app! By Michael.
```

**What's going on?**

{% raw %}
1. Again, the `@route` decorator binds a function to the route. In this case, we're using a dynamic route that includes a wildcard of `<name>`.
1. This wildcard is then passed to the view function as an argument - `def name(name)`.
1. We then passed this to the template as a keyword argument - `author=name`
1. The template then renders the author variable - `{{ author }}`.
{% endraw %}


## Shell script

Want to get started quickly? Generate the starter app in a few seconds using this Shell script.

{% raw %}
```sh
mkdir bottle
cd bottle
pip install virtualenv==12.0.7
virtualenv venv
source venv/bin/activate
pip install bottle==0.12.8
pip freeze > requirements.txt
git init
git add .
git commit -m "initial commit"

cat >app.py <<EOF
import os
from bottle import route, run, template

index_html = '''My first web app! By <strong>{{ author }}</strong>.'''


@route('/')
def index():
    return template(index_html, author='Real Python')


@route('/name/<name>')
def name(name):
    return template(index_html, author=name)


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    run(host='0.0.0.0', port=port, debug=True)
EOF

chmod a+x app.py

git init
git add .
git commit -m "Updated"
```
{% endraw %}

Download this script from this [Gist](https://gist.github.com/mjhea0/5784132), and then run it using the following command:

```sh
$ bash bottle.sh
```

## Next steps

From this point, it is as easy as adding new `@route`-decorated functions to create new pages.

Creating the HTML is simple: In the above app, we just inlined the HTML in the file itself. It's easy to modify this to load the template from a file. For example:

```python
@route('/main')
def main(name):
    return template(main_template)
```

This will load the template file `main_template.tpl`, which must be placed in a `views` folder within your project structure, and render it to the end user.

Refer to the bottle [documentation](http://bottlepy.org/docs/dev/) for more info.

<hr>

*We'll look at how to add additional pages and templates in subsequent posts. However, I urge you to try this on your own. Post any questions as comments below.*

**Check out [Part 2](http://www.realpython.com/blog/python/developing-with-bottle-part-2-plot-ly-api/)!**