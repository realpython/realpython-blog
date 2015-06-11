# Developing with Bottle - part 1

I love [bottle](http://bottlepy.org/docs/stable/). It's a simple, yet fast and powerful Python micro-framework, perfect for small web applications and rapid prototyping. It's also a perfect learning tool for those just getting starting with web development.

Let's look at a quick example.

> This tutorial assumes you are running a Unix-based environment - e.g, Mac OSX, straight Linux, or Linux VM through Windows. I will also be using Sublime 2 as my text editor.

## Starting up

First, let's create a directory to work in:

```sh
$ mkdir bottle
$ cd bottle
```

Next, you need to have pip, virtualenv, and git installed.

[virtualenv](https://pypi.python.org/pypi/virtualenv) is a Python tool that makes it easy to manage the Python modules needed for a particular project. It also keeps modules from conflicting with other projects. [pip](https://pypi.python.org/pypi/pip), meanwhile, is a package manager used for managing the installation of libraries and modules.

For help with installing pip (and it's dependencies) in a Unix environment, follow the instructions in this [Gist](https://gist.github.com/mjhea0/5692708). If you are on Windows, please view this [video](http://www.youtube.com/watch?v=MIHYflJwyLk) for help.

Once you have pip installed, run the following command to install virtualenv:

```sh
$ pip install virtualenv
```

Now we can easily setup our local environment by running:

```sh
$ virtualenv --no-site-packages testenv
$ source testenv/bin/activate
```

Install bottle:

```sh
$ pip install bottle
```

Now create the *requirements.txt* file, which allows you to install the exact same modules and dependencies in case you want to use this app elsewhere. Click [here](http://www.pip-installer.org/en/latest/requirements.html) to learn more.

```sh
pip freeze > requirements.txt
```

Finally, let's put our app under version control using Git. For more information on Git, please view this [site](http://git-scm.com/), which also includes installation instructions.

```sh
$ git init
$ git add .
$ git commit -m "initial commit"
```

## Writing your app

We're ready to write our bottle app. Create your application file, *app.py*, which will hold *all* of our first app:

{% raw %}
```python
import os
from bottle import route, run, template

index_html = '''My first web app! By {{ author }}'''

@route('/:anything')
def something(anything=''):
    return template(index_html, author=anything)

@route('/')
def index():
    return template(index_html, author='your name here')

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

You should be able to connect to [http://localhost:8080/abc](http://localhost:8080/abc) and see your application running! Change `abc` to your name. Refresh.

## What's going on?

{% raw %}
1. The `@route` decorator tells the app to interpret the path after the `/` as the variable `anything`.
2. This is passed to the function as an argument `def something(anything='')`.
3. We then pass this to the template-function as an argument (`author=anything`)
4. The template then renders the author variable with `{{ author }}`.
{% endraw %}


## Shell script

Want to get started quickly? Generate the starter app in a few seconds using this Shell script.

{% raw %}
```sh
mkdir bottle
cd bottle
pip install virtualenv
virtualenv --no-site-packages testenv
source testenv/bin/activate
pip install bottle
pip freeze > requirements.txt
git init
git add .
git commit -m "initial commit"

cat >app.py <<EOF
import os
from bottle import route, run, template

index_html = '''My first web app! By {{=<% %>=}}{{ author }}<%={{ }}=%>'''

@route('/:anything')
def something(anything=''):
   return template(index_html, author=anything)

@route('/')
def index():
   return template(index_html, author='your name here')

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

From this point, it is as easy as adding new `@route`-decorated functions() to create new pages, just as we did with those two pages.

Creating the HTML is simple: in the above app, we just inlined the HTML in the file itself. It is easy to modify this (using, for example, `open('index.html').read())` to read the template from a file.

Refer to the bottle [documentation](http://bottlepy.org/docs/dev/) for more info.

****

*We'll look at how to add additional pages and templates in subsequent posts. However, I urge you to try this on your own. Post any questions as comments below.*

### Check out [Part 2](http://www.realpython.com/blog/python/developing-with-bottle-part-2-plot-ly-api/)!
