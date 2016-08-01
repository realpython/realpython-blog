# Scaffold a Flask Project

**Let's build a command-line utility for quickly generating a Flask boilerplate structure.**

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask-scaffold/flask_scaffold.png" style="max-width: 100%;" alt="flask scaffolding">
</div>

<br>

*This is a collaboration piece between [Depado](https://github.com/Depado) and the folks at Real Python.*

<hr>

Modeled after the [Flask-Skeleton](https://github.com/Depado/flask-skeleton) project, *this tool will automate a number of repetitive tasks so that you can quickly get a Flask project up and running with the structure, extensions, and configurations that you prefer*, step by step:

1. Set up the basic structure
1. Add a custom config file
1. Utilize [Bower](http://bower.io/) to manage front-end dependencies
1. Create a virtualenv
1. Initialize Git

Once done, you'll have a powerful scaffolding script that you can (and should) customize to meet your own development needs.

## Quickstart

To start, we need a basic Flask application. For simplicity, we'll use the [Real Python boilerplate Flask structure](https://github.com/realpython/flask-skeleton), so just clone it down to set the base structure:

```sh
$ mkdir flask-scaffold
$ cd flask-scaffold
$ git clone https://github.com/realpython/flask-skeleton skeleton
$ rm -rf skeleton/.git
$ rm skeleton/.gitignore
$ mkdir templates
$ pyvenv-3.5 env
$ source env/bin/activate
```

> Yes, this article utilizes Python 3.5; however, the final script is compatible with both Python 2 and 3.

## 1st Task - The Structure

Save a new Python file as *flask_skeleton.py* in the root directory. This file will be used to power the entire scaffolding utility. Open it up in you favorite text editor and add the following code:

```python
# -*- coding: utf-8 -*-

import sys
import os
import argparse
import shutil


# Globals #

cwd = os.getcwd()
script_dir = os.path.dirname(os.path.realpath(__file__))


def main(argv):

    # Arguments #

    parser = argparse.ArgumentParser(description='Scaffold a Flask Skeleton.')
    parser.add_argument('appname', help='The application name')
    parser.add_argument('-s', '--skeleton', help='The skeleton folder to use.')
    args = parser.parse_args()

    # Variables #

    appname = args.appname
    fullpath = os.path.join(cwd, appname)
    skeleton_dir = args.skeleton

    # Tasks #

    # Copy files and folders
    shutil.copytree(os.path.join(script_dir, skeleton_dir), fullpath)


if __name__ == '__main__':
    main(sys.argv)
```

Here, we're using [argparse](https://docs.python.org/3/library/argparse.html) to obtain an `appname` for the new project and then copying the *skeleton* directory (via [shutil](https://docs.python.org/3.4/library/shutil.html)), which contains the project boilerplate, to quickly recreate the project structure.

The `shutil.copytree()` method ([source](https://docs.python.org/3/library/shutil.html#shutil.copytree)) is used to recursively copy a source directory to a destination directory (as long as the destination directory does not already exist).

Test it out:

```sh
$ python flask_skeleton.py new_project -s skeleton
```

This should make a copy of the Real Python boilerplate Flask structure (source) to a new directory called "new_project" (destination). Did it work? If so, remove the new project since there's still much work to be done:

```sh
$ rm -rf new_project
```

### Handling Multiple Skeletons

What if you need an app with a MongoDB database or a payments blueprint? All apps have specific needs and you obviously can't create a skeleton for them all, but perhaps you need a NoSQL database about fifty percent of the time. You can add a new skeleton to the root to accomplish this. Then when you run the scaffold command, simply specify the name of the directory containing the skeleton app you wish to make a copy of.

## 2nd Task - Configuration

> Need the script up to this point? Grab it from the first [tag](https://github.com/realpython/flask-scaffold/releases/tag/first_tag). Or if you cloned the [repo](https://github.com/realpython/flask-scaffold), you can grab the tag like so - `git checkout tags/first_tag`.

**Note** Need to have an updated branch with code till now(if needed).

We now need to generate a custom *config.py* file for each skeleton. This script is going to do just that for us; let the code do the repetitive work! First, add a file called *config.jinja2* in the *templates* folder:

{% raw %}
```python
# config.jinja2

import os
basedir = os.path.abspath(os.path.dirname(__file__))


class BaseConfig(object):
    """Base configuration."""
    SECRET_KEY = '{{ secret_key }}'
    DEBUG = False
    BCRYPT_LOG_ROUNDS = 13
    WTF_CSRF_ENABLED = True
    DEBUG_TB_ENABLED = False
    DEBUG_TB_INTERCEPT_REDIRECTS = False


class DevelopmentConfig(BaseConfig):
    """Development configuration."""
    DEBUG = True
    BCRYPT_LOG_ROUNDS = 13
    WTF_CSRF_ENABLED = False
    SQLALCHEMY_DATABASE_URI = 'sqlite:///' + os.path.join(basedir, 'dev.sqlite')
    DEBUG_TB_ENABLED = True


class TestingConfig(BaseConfig):
    """Testing configuration."""
    DEBUG = True
    TESTING = True
    BCRYPT_LOG_ROUNDS = 13
    WTF_CSRF_ENABLED = False
    SQLALCHEMY_DATABASE_URI = 'sqlite:///'
    DEBUG_TB_ENABLED = False


class ProductionConfig(BaseConfig):
    """Production configuration."""
    SECRET_KEY = '{{ secret_key }}'
    DEBUG = False
    SQLALCHEMY_DATABASE_URI = 'postgresql://localhost/example'
    DEBUG_TB_ENABLED = False
```
{% endraw %}

At the start of the scaffold script, `flask_skeleton.py`, just before the `main()` function, we need to initialize `Jinja2` in order to render the config correctly.

```python
# Jinja2 environment
template_loader = jinja2.FileSystemLoader(searchpath=os.path.join(script_dir, "templates"))
template_env = jinja2.Environment(loader=template_loader)
```

Make sure to add the import as well:

```python
import jinja2
```

Install:

```sh
$ pip install jinja2
$ pip freeze > requirements.txt
```

Looking back at the template, *config.jinja2*, we have one variable that needs to be defined - {% raw %}`{{ secret_key }}`{% endraw %}.To do this, we can use the [codecs](https://docs.python.org/3/library/codecs.html) module.

To the imports of `flask_skeleton.py` add:

```python
import codecs
```

Add the following code to the bottom of the `main()` function:

```python
# Create config.py
secret_key = codecs.encode(os.urandom(32), 'hex').decode('utf-8')
template = template_env.get_template('config.jinja2')
template_var = {
    'secret_key': secret_key,
}
with open(os.path.join(fullpath, 'project', 'config.py'), 'w') as fd:
    fd.write(template.render(template_var))
```

> What if you manage several skeletons and need several configuration templates? Simple: You just have to check which skeleton is passed as an argument and use the appropriate config template. Keep in mind that `os.path.join(fullpath, 'project', 'config.py')` must represent the path where your configuration should be stored in your skeleton. If this is different for each skeleton, then you should specify the folder in which the config file is stored in as an additional argparse argument.

Ready to test?

```sh
$ python flask_skeleton.py new_project -s skeleton
```

Make sure the *config.py* file is present in the "new_project/project" folder and then remove the new project: `rm -rf new_project`

## 3rd Task - Bower

> Need the updated script? Grab it [here](https://github.com/realpython/flask-scaffold/releases/tag/second_tag).

That's right: We'll be using [bower](http://bower.io/) to download and manage static libraries. To add bower support to the scaffold script, start with adding another argument:

```python
parser.add_argument('-b', '--bower', help='Install dependencies via bower')
```

And to handle the running of bower, add the following code right below the config section of the scaffold script:

```python
# Add bower dependencies
if args.bower:
    bower = args.bower.split(',')
    bower_exe = which('bower')
    if bower_exe:
        os.chdir(os.path.join(fullpath, 'project', 'client', 'static'))
        for dependency in bower:
            output, error = subprocess.Popen(
                [bower_exe, 'install', dependency],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE
            ).communicate()
            # print(output)
            if error:
                print("An error occurred with Bower")
                print(error)
    else:
        print("Could not find bower. Ignoring.")
```

Don't forget to add the [subprocess](https://docs.python.org/3/library/subprocess.html) module to the import section of *flask_skeleton.py* - `import subprocess`.

Did you notice the `which()` method ([source](https://docs.python.org/dev/library/shutil.html#shutil.which))? This essentially uses the unix/linux [which](http://en.wikipedia.org/wiki/Which_%28Unix%29) tool in order to indicate where an executable is installed in the filesystem. So, in the above code, we're checking to see if `bower` is installed. If you're curious to know how this works, test it out in the Python 3 interpreter:

```sh
>>> import shutil
>>> shutil.which('bower')
'/usr/local/bin/bower'
```

Unfortunately, this method, `which()`, is new to Python 3.3, so, if you're using Python 2, then you need to install a separate package - [shutilwhich](https://github.com/mbr/shutilwhich):

```sh
$ pip install shutilwhich
$ pip freeze > requirements.txt
```

Update the imports:

```python
if sys.version_info < (3, 0):
    from shutilwhich import which
else:
    from shutil import which
```

Finally, turn you attention to the following lines of code:

```python
output, error = subprocess.Popen(
    [bower_exe, 'install', dependency],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
).communicate()
# print(output)
if error:
    print("An error occurred with Bower")
    print(error)
```

Start by looking at the official [subprocess](https://docs.python.org/3.4/library/subprocess.html) documentation. Put simply, it's used for invoking external shell commands. In the above code we are simply capturing the output from [stdout](http://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29) and [stderr](http://en.wikipedia.org/wiki/Standard_streams#Standard_error_.28stderr.29).

If you're curious to see what the output is, uncomment the print statement, `# print(output)`, and then run your code...

Before testing, this code assumes that in your skeleton folder(s), you have a "project" folder that contains a "static" folder. Classical Flask application. On the command line, you can now install multiple dependencies like so:

```sh
$ python flask_skeleton.py new_project -s skeleton -b 'angular, jquery, bootstrap'
```

## 4th Task - virtualenv

> Again, grab the updated [script](https://github.com/realpython/flask-scaffold/releases/tag/third_tag), if necessary.

Since the virtual environment is one of the most important parts of any Flask (err, Python) application, creating the virtualenv using the scaffold script will be really useful. As usual, start by adding the argument:

```python
parser.add_argument('-v', '--virtualenv', action='store_true')
```

And then add the following code below the bower section:

```python
# Add a virtualenv
virtualenv = args.virtualenv
if virtualenv:
    virtualenv_exe = which('pyvenv')
    if virtualenv_exe:
        output, error = subprocess.Popen(
            [virtualenv_exe, os.path.join(fullpath, 'env')],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        ).communicate()
        if error:
            with open('virtualenv_error.log', 'w') as fd:
                fd.write(error.decode('utf-8'))
                print("An error occurred with virtualenv")
                sys.exit(2)
        venv_bin = os.path.join(fullpath, 'env/bin')
        output, error = subprocess.Popen(
            [
                os.path.join(venv_bin, 'pip'),
                'install',
                '-r',
                os.path.join(fullpath, 'requirements.txt')
            ],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        ).communicate()
        if error:
            with open('pip_error.log', 'w') as fd:
                fd.write(error.decode('utf-8'))
                sys.exit(2)
    else:
        print("Could not find virtualenv executable. Ignoring")
```

This snippet assumes that there is a *requirements.txt* file in your "skeleton" folder in the root directory. If so, it will create a virtualenv and then install the dependencies.

## 5th task - Git Init

> Updated [script](https://github.com/realpython/flask-scaffold/releases/tag/fourth_tag).

Notice a pattern yet? Add the argument:

```python
parser.add_argument('-g', '--git', action='store_true')
```

Then add the code under the task for the virtualenv:

```python
# Git init
if args.git:
    output, error = subprocess.Popen(
        ['git', 'init', fullpath],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    ).communicate()
    if error:
        with open('git_error.log', 'w') as fd:
            fd.write(error.decode('utf-8'))
            print("Error with git init")
            sys.exit(2)
    shutil.copyfile(
        os.path.join(script_dir, 'templates', '.gitignore'),
        os.path.join(fullpath, '.gitignore')
    )
```

Now within the templates folder add a *.gitignore* file, and then add the files and folders that you'd like to ignore. Grab the [example](https://raw.githubusercontent.com/github/gitignore/master/Python.gitignore) from Github, if needed. Test again.

## Sum and confirm

> Updated [script](https://github.com/realpython/flask-scaffold/releases/tag/fifth_tag).

Finally, let's add a nice summary before the application is created and then ask for user confirmation before executing the script.

### Summary

Add a file called *brief.jinja2* to the "templates" folder:

```python

Welcome! The following settings will be used to create your application:

Python Version:     {{ pyversion }}
Project Name:       {{ appname }}
Project Path:       {{ path }}
Virtualenv:         {% if virtualenv %}Enabled{% else %}Disabled{% endif %}
Skeleton:           {{ skeleton }}
Git:                {% if git %}Yes{% else %}{{ disabled }}No{% endif %}
Bower:              {% if bower %}Enabled{% else %}Disabled{% endif %}
{% if bower %}Bower Dependencies: {% for dependency in bower %}{{ dependency }}{% endfor %}{% endif %}
```

Now we just need to catch every user-supplied argument and then render the template. First, add the import - `import platform` - to the import section and then the following code just below the "variables" section in the *flask_skeleton.py* script:

```python
# Summary #

def generate_brief(template_var):
    template = template_env.get_template('brief.jinja2')
    return template.render(template_var)

template_var = {
    'pyversion': platform.python_version(),
    'appname': appname,
    'bower': args.bower,
    'virtualenv': args.virtualenv,
    'skeleton': args.skeleton,
    'path': fullpath,
    'git': args.git
}

print(generate_brief(template_var))
```

Test this out:

```sh
$ python flask_skeleton.py new_project -s skeleton -b 'angular, jquery, bootstrap' -g -v

Welcome! The following settings will be used to create your application:

Python Version:     3.5.1
Project Name:       new_project
Project Path:       /Users/michael/repos/realpython/flask-scaffold/new_project
Virtualenv:         Enabled
Skeleton:           skeleton
Git:                Yes
Bower:              Enabled
Bower Dependencies: angular, jquery, bootstrap
```

Nice!

### Refactor

Now we need to refactor the script a bit to check for errors first. I suggest grabbing the code from the [refactor](https://github.com/realpython/flask-scaffold/releases/tag/refactor) tag and then comparing the diff between that script and the previous tag's [script](https://github.com/realpython/flask-scaffold/releases/tag/summary) since there are a number of small updates.

**Make sure you use the updated script from the [refactor](https://github.com/realpython/flask-scaffold/releases/tag/refactor) tag before moving on.**

### Confirm

Now let's add the user confirmation functionality by updating `if __name__ == '__main__':`:

```python
if __name__ == '__main__':
    arguments = get_arguments(sys.argv)
    print(generate_brief(arguments))
    if sys.version_info < (3, 0):
        input = raw_input
    proceed = input("\nProceed (yes/no)? ")
    valid = ["yes", "y", "no", "n"]
    while True:
        if proceed.lower() in valid:
            if proceed.lower() == "yes" or proceed.lower() == "y":
                main(arguments)
                print("Done!")
                break
            else:
                print("Goodbye!")
                break
        else:
            print("Please respond with 'yes' or 'no' (or 'y' or 'n').")
            proceed = input("\nProceed (yes/no)? ")
```

This should be fairly straightforward.

## Run!

If you use Linux or Mac you can make this script easier to run. Simply add the following alias to either *.bashrc* or *.zshrc*, customizing it to match your directory structure:

```sh
alias flaskcli="python /Users/michael/repos/realpython/flask-scaffold/flask_skeleton.py"
```

> **NOTE**: If you have both Python 2.7 and Python 3.4 installed you will have to specify the version you want to use - either `python` or `python3`.

Remove the new project (if necessary) - `rm -rf new_project` - and then test out the script one last time to confirm:

```sh
$ flaskcli new_project -s skeleton -b 'angular, jquery, bootstrap' -g -v
```

## Refactoring Requirements.txt

With our scaffolding tool up and running, we need to refactor the `requirements.txt` file and have only one in the root directory of our project.

```sh
rm -rf skeleton/requirements.txt
pip freeze > requirements.txt
```
Now with all the python dependencies at one place, we are ready to ship our Flask Scaffolding tool.

## Conclusion

What do you think? Did we miss anything? What other arguments would you add to `argparse` in order to customize your scaffold script even further? Comment below!

Grab the final code from the [repo](https://github.com/realpython/flask-scaffold).

### Author's Note

I (Depado) would like to thank [Antonin Lenfant](http://antonin-lenfant.com/) for his support toward the project and for the changes made to the article. He is a former coworker and a Python enthusiast. I'd also like to thank [Michael Herman](https://github.com/mjhea0), from Real Python, who spotted my project on Github and asked me to write this article.

Creating this tool has been an interesting challenge. I discovered how pleasant it was to work with a template system for an application that is not a web application.

<br>

<p style="font-size: 14px;">
  <em>Edits made by <a href="https://twitter.com/diek007">Derrick Kearney</a>. Thanks again, Derrick!</em>
</p>
