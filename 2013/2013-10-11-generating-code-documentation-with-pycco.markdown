# Generating Code Documentation with Pycco

As developers we love to write code, and even though it makes sense to us at the time, we must think of our audience. Someone has to read, use, and/or maintain said code. It could be another developer, a customer, or our future selves, three months later. Even a beautiful language like Python can be difficult to understand at times. So as good programming citizens who care deeply about our fellow coders (or more likely because our boss wants us to) we should write some documentation.

But there are rules:

1.  Writing documentation must not suck (i.e. we cannot use MS Word).
2.  Writing documentation must be as effortless as possible.
3.  Writing documentation must not require us to leave our favorite text editor / IDE.
4.  Writing documentation must not require us to do any formatting or care at all about the final presentation.

This is where [Pycco](https://pypi.python.org/pypi/Pycco/0.1) comes in:

> "Pycco" is a Python port of Docco: the original quick-and-dirty, hundred-line-long, literate-programming-style documentation generator. It produces HTML that displays your comments alongside your code. Comments are passed through Markdown and SmartyPants, while code is passed through Pygments for syntax highlighting.

So basically that means that Pycco can auto-generate decent looking code documentation for us.  Pycco also adheres to the four rules of code documentation stated above. So what's not to love?  Let's jump into to some examples of how to use it.

For this article, I've started off with a simple TODO app written in Django, which you can grab from the [repo](https://github.com/mjhea0/django-todo). The app allows a user to add items to a list and delete them upon completion.  This app probably isn't going to win me a Webby, but it should serve the purpose for this article.

> Please note, the files in the repo already contain the final code, already documented. You can still create the seperate documented files though, so feel free to clone the repo and follow along with me.

## Project setup

Clone the repo:

```sh
$ git clone git@github.com:mjhea0/django-todo.git
```

Setup virtualenv:

```sh
cd django-todo
virtualenv --no-site-packages venv
source venv/bin/activate
```

Install the requirements:

```sh
pip install -r requirements.txt
```

Sync the database:

```sh
cd todo
python manage.py syncdb
```

## Generate some docs

Getting started is simple:

```sh
pip install pycco
```

Then to run it you can use a command like:

```sh
pycco todos/*.py
```

Note this way you can specify individual files or a directory of files.  Executing the above command on our TODO repo generates the following outcome:

```sh
pycco = todo/todos/__init__.py -> docs/__init__.html
pycco = todo/todos/models.py -> docs/models.html
pycco = todo/todos/tests.py -> docs/tests.html
pycco = todo/todos/views.py -> docs/views.html
```

In other words, it generates the html files (one for each python file) and dumps them all in the "docs" directory.  For larger projects you may want to use the -p option that will preserve the original file paths.

For example -

```sh
pyccoo todo/todos/*.py -p
```

-will generate:

```sh
pycco = todo/todos/__init__.py -> docs/todo/todos/__init__.html
pycco = todo/todos/models.py -> docs/todo/todos/models.html
pycco = todo/todos/tests.py -> docs/todo/todos/tests.html
pycco = todo/todos/views.py -> docs/todo/todos/views.html
```

> Notice that it has stored the documentation for the todos app under the "docs/todo/todos" subfolder.  By doing this it will be much easier to navigate the documentation for a large project as the documentation structure will match the code structure.

## Generated Docs

The intent of pycco is to generate a two-column document with comments on the left hand side, aligned with the subsequent code on the right hand side. So, calling pycco on our *models.py* class that has no comments yet will generate a page that looks like this:

![Simple Model Docs](https://raw.github.com/mjhea0/django-todo/master/img/simple_model_docs.png)

You'll notice that the left hand column (where the comments should be) is simply blank and the right hand side shows the code.  We can change *models.py* by adding a docstring to get more interesting documents by adding the following code to *models.py*.

```python
from django.db import models

# === Models for Todos app ===


class ListItem(models.Model):
    """
    The ListItem class defines the main storage point for todos.
    Each todo has two fields:
    text - stores the text of the todo
    is_visible - used to control if the todo is displayed on screen
    """

    text = models.CharField(max_length=300)
    is_visible = models.BooleanField()
```

With the code snippet above pycco finds the line -

```python
# === Models for Todos app ===
```

-then generates a header in the documentation.

And the docstring-

```python
"""
The ListItem class defines the main storage point for todos.
Each todo has two fields:
text - stores the text of the todo
is_visible - used to control if the todo is displayed on screen
"""
```

-will be used by pycco to generate the documentation.  The end result after running pycco again will be:

![Model Docs with Comments](https://raw.github.com/mjhea0/django-todo/master/img/model_with_comments_docs.png)

As you can see the documentation on the left is nicely aligned with the code on the right.  This makes it really easy to review the code and understand whats going on.

Pycco also recognizes single line comments in the code that start with a `#` to generate documentation.

## Make it fancy

But pycco doesn't stop there.  It also allows for custom formatting of comments using markdown. As an example let's put some comments in the *views.py* file.  For starters let's just put a docstring at the top of *views.py*:

```python
"""
All the views for our todos application
Currently we support the following 3 views:

1. **Home** - The main view for Todos
2. **Delete** - called to delete a todo
3. **Add** - called to add a new todo

"""

from django.http import HttpResponse
from django.shortcuts import render_to_response
from django.template import RequestContext

from todos.models import ListItem

def home(request):
    items = ListItem.objects.filter(is_visible=True)
    return render_to_response('home.html', {'items': items}, context_instance = RequestContext(request))

... code omitted for brevity ...
```

This will produce a report like this:

![View with Markdown Docs](https://raw.github.com/mjhea0/django-todo/master/img/view_with_markdown_docs.png)

We can also add links between files and even inside the same file. Links can be added by using `[[filename.py]]` or `[[filename.py#section]]` and they will render like a link with the file name. Let's update *views.py* to add some links at the end of each item in the list:

```python
"""
All the views for our todos application
Currently we support the following 3 views:

1. **Home** - The main view for Todos (jump to section in [[views.py#home]] )
2. **Delete** - called to delete a todo ( jump to section in [[views.py#delete]] )
3. **Add** - called to add a new todo (jump to section in [[views.py#add]])
"""

from django.http import HttpResponse
from django.shortcuts import render_to_response
from django.template import RequestContext

# defined in [[models.py]]
from todos.models import ListItem

# === home ===
def home(request):
    items = ListItem.objects.filter(is_visible=True)
    return render_to_response('home.html', {'items': items}, context_instance = RequestContext(request))

# === delete ===
def delete(request):

... code omitted for brevity ...
```

As you can tell there are two components to making the link. First, we have to define a section in our documentation. You can see we have defined the home section with the code `# === home ===`. Once the section is created we can link to it with the code `[[views.py#home]]`. Also we inserted a link to the models documentation file with the code:

```python
# defined in [[models.py]]
```

The end result is documentation that looks like this:

![View with links](https://raw.github.com/mjhea0/django-todo/master/img/view_with_links_docs.png)

Keep in mind that because pycco allows for markup syntax you can also use full blown html. So go crazy. :)

##Auto-generating documentation for an entire project

It may not be obvious how to generate documentation for your entire project using pycco, but it's actually quite straight forward if you are using bash or zsh or any sh that supports globing you can just run a command like this:

```sh
pycco todo/**/*.py -p
```

This will generate documents for all `.py` files in any / all subdirectories of todo.  If you're on windows you should be able to do this with cygwin, git bash or power sh.

## Documentation for non-python files

Pycco also supports several other file types, which is helpful for web projects that often use more than one file type.  The complete list of the supported file as of this writing is are:

* .coffee - coffee-script
* .pl - perl
* .sql - sql
* .c - C
* .cpp - C++
* .js - javascript
* .rb - Ruby
* .py - python
* .scm - scheme
* .lua - lua
* .erl - erlang
* .tcl - tcl
* .hs - haskell

That should cover your documentation needs.  So just remember write some comments and let pycco easily generate nice looking code documentation for you.

## How about Project Level Documentation?

Often projects have additional requirements other than just code documentation - like a readme, installation docs, deployment docs etc. It would be nice to use pycco to generate that documentation as well, so we can stick to one tool throughout. Well, at this point in time, pycco will only process files with the extensions in the list above.  But there is nothing stopping you from creating a *readme.py* or *installation.py* file and using pycco to generate the documentation. All you have to do is put your documentation in a docstring and then pycco will generate it and give you full markdown support as well. So imagine if at the base of your project directory you have a folder called `project_docs` which contains:

* install_docs.py
* project_readme.py
* deployment.py

Then you can run the following command-

```sh
pycco project_docs/*.py -p
```

-to add the appropriate html files in your "docs/project_docs directory".  Granted, this may be a bit of a hack but it does allow you to use one tool to generate all the documentation for your project.

## Regeration

Pycco has one more trick up it's sleeve: Automatic document regeneration.  In other words, you can ask pycco to watch your source files and automatically regenerate the necessary documentation each time you save the source file.  This is really useful if you're trying to put some custom markdown / html in your comments and want to make sure it renders correctly.  With automatic document regeneration running you just make the change, save your file and refresh your browser, no need to re-run the pycco command in-between.  To do this you need to have a watchdog installed:

```sh
pip install watchdog
```

Watchdog is the module that listens for file changes.  So once that is installed just execute a command like -

```sh
pycco todo/**/*.py -p --watch
```

-which will run pycco and keep it running until you stop it with `Ctrl-C`.  As long as it's running, it will regenerate the documentation on every change to the source file.  How's that for easy documentation?

<hr>

**Pycoo is by far the easiest and most straight forward code documentation tool for python that I have run across.  Anybody use anything else?  Hit us up with a comment and let us know your thoughts. Cheers!**
