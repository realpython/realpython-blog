# Comparing Python Command-Line Parsing Libraries - Argparse, Docopt, and Click

About a year ago I began a job where building command-line applications was a common occurrence. At that time I had used [argparse](https://docs.python.org/3/library/argparse.html) quite a bit and wanted to explore what other options were available. I found that the most popular alternatives available were [click](http://click.pocoo.org/5/) and [docopt](http://docopt.org/). During my exploration I also found that other than each libraries "why use me" section there was not much available for a complete comparison of the three libraries. Now there is - this blog post!

<div class="center-text">
  <img class="no-border" src="/images/blog_images/command-line-tools/command-line-tools.png" style="max-width: 100%;" alt="python command line tools">
</div>

<br>

*This is a guest blog post by [Kyle Purdon](https://twitter.com/PurdonKyle)​, an Application Engineer at [Bitly](https://bitly.com/pages/careers) in Denver.*

If you want to, you can head directly to the [source](https://github.com/kpurdon/greeters) though it really won't do much good without the comparisons and step-by-step construction presented in this article.

This article uses the following versions of the libraries:

```sh
$ python --version
Python 3.4.3
# argparse is a Python core library

$ pip list | grep click
click (5.1)

$ pip list | grep docopt
docopt (0.6.2)

$ pip list | grep invoke
invoke (0.10.1)
# ignore this for now, it's a special surprise for later!
```

## Command-Line Example

The command-line application that we are creating will have the following interface:

`python [file].py [command] [options] NAME`

### Basic Usage

```sh
$ python [file].py hello Kyle
Hello, Kyle!

$ python [file].py goodbye Kyle
Goodbye, Kyle!
```

### Usage w/ Options (Flags)

```sh
$ python [file].py hello --greeting=Wazzup Kyle
Whazzup, Kyle!

$ python [file].py goodbye --greeting=Later Kyle
Later, Kyle!

$ python [file].py hello --caps Kyle
HELLO, KYLE!

$ python [file].py hello --greeting=Wazzup --caps Kyle
WAZZUP, KYLE!
```

This article will compare each libraries method for implementing the following features:

1. Commands (hello, goodbye)
2. Arguments (name)
3. Options/Flags (--greeting=<str\>, --caps)

Additional features:

1. Version Printing (`-v/--version`)
2. Automated Help Messages
3. Error Handling

As you would expect *argparse*, *docopt*, and *click* implement all of these features (as any complete command-line library would). This fact means that the actual implementation of these features is what we will compare. Each library takes a very different approach that lends to a very interesting comparison - *argparse=standard*, *docopt=docstrings*, *click=decorators*.

## Bonus Sections

1. I've been curious about using task-runner libraries like [fabric](https://fabric.readthedocs.org/en/latest/) and it's python3 replacement [invoke](https://invoke.readthedocs.org/en/latest/) to create simple command-line interfaces, so I will try and put the same interface together with *invoke*.
2. A few extra steps are needed when packaging command-line applications, so I'll cover those steps as well!

## Commands

Let's begin by setting up the basic skeleton (no arguments or options) for each library.

### Argparse

```python
import argparse

parser = argparse.ArgumentParser()
subparsers = parser.add_subparsers()

hello_parser = subparsers.add_parser('hello')
goodbye_parser = subparsers.add_parser('goodbye')

if __name__ == '__main__':
    args = parser.parse_args()
```

With this we now have two commands (`hello` and `goodbye`) and a built-in help message. Notice that the help message changes when run as an option on the command hello.

```sh
$ python argparse/commands.py --help
usage: commands.py [-h] {hello,goodbye} ...

positional arguments:
  {hello,goodbye}

optional arguments:
  -h, --help       show this help message and exit

$ python argparse/commands.py hello --help
usage: commands.py hello [-h]

optional arguments:
  -h, --help  show this help message and exit
```

### Docopt

```python
"""Greeter.

Usage:
  commands.py hello
  commands.py goodbye
  commands.py -h | --help

Options:
  -h --help     Show this screen.
"""
from docopt import docopt

if __name__ == '__main__':
    arguments = docopt(__doc__)
```

With this we, again, have two commands (`hello`, `goodbye`) and a built-in help message. Notice that the help message **DOES NOT** change when run as an option on the `hello` command. In addition we **do not** need to explicitly specify the `commands.py -h | --help` in the `Options` section to get a help command. However, if we don't they will not show up in the output help message as options.

```sh
$ python docopt/commands.py --help
Greeter.

Usage:
  commands.py hello
  commands.py goodbye
  commands.py -h | --help

Options:
  -h --help     Show this screen.

$ python docopt/commands.py hello --help
Greeter.

Usage:
  commands.py hello
  commands.py goodbye
  commands.py -h | --help

Options:
  -h --help     Show this screen.
```

### Click

```python
import click


@click.group()
def greet():
    pass


@greet.command()
def hello(**kwargs):
    pass


@greet.command()
def goodbye(**kwargs):
    pass

if __name__ == '__main__':
    greet()
```

With this we now have two commands (`hello`, `goodbye`) and a built-in help message. Notice that the help message changes when run as an option on the `hello` command.

```sh
$ python click/commands.py --help
Usage: commands.py [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  goodbye
  hello

$ python click/commands.py hello --help
Usage: commands.py hello [OPTIONS]

Options:
  --help  Show this message and exit.
```

Even at this point you can see that we have very different approaches to constructing a basic command-line application. Next let's add the *NAME* argument, and the logic to ouput the result from each tool.

## Arguments

In this section we will be adding new logic to the same code shown in the previous section. We'll add comments to new lines stating their purpose. Arguments (aka positional arguments) are required inputs to a command-line application. In this case we are adding a required `name` argument so that the tool can greet a specific person.

### Argparse

To add an argument to a subcommand we use the `add_argument` method. And in order to execute the correct logic, when a command is called, we use the `set_defaults` method to set a default function. Finally we execute the default function by calling `args.func(args)` after we parse the arguments at runtime.

```python
import argparse


def hello(args):
    print('Hello, {0}!'.format(args.name))


def goodbye(args):
    print('Goodbye, {0}!'.format(args.name))

parser = argparse.ArgumentParser()
subparsers = parser.add_subparsers()

hello_parser = subparsers.add_parser('hello')
hello_parser.add_argument('name')  # add the name argument
hello_parser.set_defaults(func=hello)  # set the default function to hello

goodbye_parser = subparsers.add_parser('goodbye')
goodbye_parser.add_argument('name')
goodbye_parser.set_defaults(func=goodbye)

if __name__ == '__main__':
    args = parser.parse_args()
    args.func(args)  # call the default function
```

```sh
$ python argparse/arguments.py hello Kyle
Hello, Kyle!

$ python argparse/arguments.py hello --help
usage: arguments.py hello [-h] name

positional arguments:
  name

optional arguments:
  -h, --help  show this help message and exit
```

### Docopt

In order to add an option, we add a `<name>` to the docstring. The `<>` are used to designate a positional argument. In order to execute the correct logic we must check if the command (treated as an argument) is `True` at runtime `if arguments['hello']:`, then call the correct function.

```python
"""Greeter.

Usage:
  basic.py hello <name>
  basic.py goodbye <name>
  basic.py (-h | --help)

Options:
  -h --help     Show this screen.

"""
from docopt import docopt


def hello(name):
    print('Hello, {0}'.format(name))


def goodbye(name):
    print('Goodbye, {0}'.format(name))

if __name__ == '__main__':
    arguments = docopt(__doc__)

    # if an argument called hello was passed, execute the hello logic.
    if arguments['hello']:
        hello(arguments['<name>'])
    elif arguments['goodbye']:
        goodbye(arguments['<name>'])
```

```sh
$ python docopt/arguments.py hello Kyle
Hello, Kyle

$ python docopt/arguments.py hello --help
Greeter.

Usage:
  basic.py hello <name>
  basic.py goodbye <name>
  basic.py (-h | --help)

Options:
  -h --help     Show this screen.
```

> Note that the help message is not specific to the subcommand, rather it is the entire docstring for the program.

### Click

In order to add an argument to a *click* command we use the `@click.argument` decorator. In this case we are just passing the argument name, but there are [many more options](http://click.pocoo.org/5/arguments/) some of which we'll use later. Since we are decorating the logic (function) with the argument we don't need to do anything to set or make a call to the correct logic.

```python
import click


@click.group()
def greet():
    pass


@greet.command()
@click.argument('name')  # add the name argument
def hello(**kwargs):
    print('Hello, {0}!'.format(kwargs['name']))


@greet.command()
@click.argument('name')
def goodbye(**kwargs):
    print('Goodbye, {0}!'.format(kwargs['name']))

if __name__ == '__main__':
    greet()
```

```sh
$ python click/arguments.py hello Kyle
Hello, Kyle!

$ python click/arguments.py hello --help
Usage: arguments.py hello [OPTIONS] NAME

Options:
  --help  Show this message and exit.
```

## Flags/Options

In this section we will again be adding new logic to the same code shown in the previous section. We'll add comments to new lines stating there purpose. Options are non-required inputs that can be given to alter the execution of a command-line application. Flags are a boolean only (`True`/`False`) subset of options. For example: `--foo=bar` will pass `bar` as the value for the `foo` option and `--baz` (if defined as a flag) will pass the value of `True` is the option is given, or `False` if not.

For this example we are going to add the `--greeting=[greeting]` option, and the `--caps` flag. The `greeting` option will have default values of `Hello` and `Goodbye` and allow the user to pass in a custom greeting. For example given `--greeting=Wazzup` the tool will respond with `Wazzup, [name]!`. The `--caps` flag will uppercase the entire response if given. For example given `--caps` the tool will respond with `HELLO, [NAME]!`.

### Argparse

```python
import argparse


# since we are now passing in the greeting
# the logic has been consolidated to a single greet function
def greet(args):
    output = '{0}, {1}!'.format(args.greeting, args.name)
    if args.caps:
        output = output.upper()
    print(output)

parser = argparse.ArgumentParser()
subparsers = parser.add_subparsers()

hello_parser = subparsers.add_parser('hello')
hello_parser.add_argument('name')
# add greeting option w/ default
hello_parser.add_argument('--greeting', default='Hello')
# add a flag (default=False)
hello_parser.add_argument('--caps', action='store_true')
hello_parser.set_defaults(func=greet)

goodbye_parser = subparsers.add_parser('goodbye')
goodbye_parser.add_argument('name')
goodbye_parser.add_argument('--greeting', default='Goodbye')
goodbye_parser.add_argument('--caps', action='store_true')
goodbye_parser.set_defaults(func=greet)

if __name__ == '__main__':
    args = parser.parse_args()
    args.func(args)
```

```sh
$ python argparse/options.py hello --greeting=Wazzup Kyle
Wazzup, Kyle!

$ python argparse/options.py hello --caps Kyle
HELLO, KYLE!

$ python argparse/options.py hello --greeting=Wazzup --caps Kyle
WAZZUP, KYLE!

$ python argparse/options.py hello --help
usage: options.py hello [-h] [--greeting GREETING] [--caps] name

positional arguments:
  name

optional arguments:
  -h, --help           show this help message and exit
  --greeting GREETING
  --caps
```

### Docopt

Once we hit the case of adding options with defaults, we hit a snag with the basic implementation of commands in *docopt*. Let's continue just to illustrate the issue.

```python
"""Greeter.

Usage:
  basic.py hello <name> [--caps] [--greeting=<str>]
  basic.py goodbye <name> [--caps] [--greeting=<str>]
  basic.py (-h | --help)

Options:
  -h --help         Show this screen.
  --caps            Uppercase the output.
  --greeting=<str>  Greeting to use [default: Hello].

"""
from docopt import docopt


def greet(args):
    output = '{0}, {1}!'.format(args['--greeting'],
                                args['<name>'])
    if args['--caps']:
        output = output.upper()
    print(output)


if __name__ == '__main__':
    arguments = docopt(__doc__)
    greet(arguments)
```

Now, see what happens when we run the following commands:

```sh
$ python docopt/options.py hello Kyle
Hello, Kyle!

$ python docopt/options.py goodbye Kyle
Hello, Kyle!
```

What?! Because we can only set a single default for the `--greeting` option both of our `Hello` and `Goodbye` commands now respond with `Hello, Kyle!`. In order for us to make this work we'll need to follow the [git example](https://github.com/docopt/docopt/tree/master/examples/git) docopt provides. The refactored code is shown below:

```python
"""usage: greet [--help] <command> [<args>...]

options:
  -h --help         Show this screen.

commands:
   hello       Say hello
   goodbye     Say goodbye

"""

from docopt import docopt

HELLO = """usage: basic.py hello [options] [<name>]

  -h --help         Show this screen.
  --caps            Uppercase the output.
  --greeting=<str>  Greeting to use [default: Hello].
"""

GOODBYE = """usage: basic.py goodbye [options] [<name>]

  -h --help         Show this screen.
  --caps            Uppercase the output.
  --greeting=<str>  Greeting to use [default: Goodbye].
"""


def greet(args):
    output = '{0}, {1}!'.format(args['--greeting'],
                                args['<name>'])
    if args['--caps']:
        output = output.upper()
    print(output)


if __name__ == '__main__':
    arguments = docopt(__doc__, options_first=True)

    if arguments['<command>'] == 'hello':
        greet(docopt(HELLO))
    elif arguments['<command>'] == 'goodbye':
        greet(docopt(GOODBYE))
    else:
        exit("{0} is not a command. \
          See 'options.py --help'.".format(arguments['<command>']))
```

As you can see the `hello`|`goodbye` subcommands are now there own docstrings tied to the variables `HELLO` and `GOODBYE`. When the tool is executed it uses a new argument, `command`, to decide which to parse. Not only does this correct the problem we had with only one default, but we now have subcommand specific help messages as well.

```sh
$ python docopt/options.py --help
usage: greet [--help] <command> [<args>...]

options:
  -h --help         Show this screen.

commands:
   hello       Say hello
   goodbye     Say goodbye

$ python docopt/options.py hello --help
usage: basic.py hello [options] [<name>]

  -h --help         Show this screen.
  --caps            Uppercase the output.
  --greeting=<str>  Greeting to use [default: Hello].

$ python docopt/options.py hello Kyle
Hello, Kyle!

$ python docopt/options.py goodbye Kyle
Goodbye, Kyle!
```

In addition all of our new options/flags are working as well:

```sh
$ python docopt/options.py hello --greeting=Wazzup Kyle
Wazzup, Kyle!

$ python docopt/options.py hello --caps Kyle
HELLO, KYLE!

$ python docopt/options.py hello --greeting=Wazzup --caps Kyle
WAZZUP, KYLE!
```

### Click

To add the `greeting` and `caps` options we use the `@click.option` decorator. Again, since we have default greetings now we have pulled the logic out into a single function (`def greeter(**kwargs):`).

```python
import click


def greeter(**kwargs):
    output = '{0}, {1}!'.format(kwargs['greeting'],
                                kwargs['name'])
    if kwargs['caps']:
        output = output.upper()
    print(output)


@click.group()
def greet():
    pass


@greet.command()
@click.argument('name')
# add an option with 'Hello' as the default
@click.option('--greeting', default='Hello')
# add a flag (is_flag=True)
@click.option('--caps', is_flag=True)
# the application logic has been refactored into a single function
def hello(**kwargs):
    greeter(**kwargs)


@greet.command()
@click.argument('name')
@click.option('--greeting', default='Goodbye')
@click.option('--caps', is_flag=True)
def goodbye(**kwargs):
    greeter(**kwargs)

if __name__ == '__main__':
    greet()
```

```sh
$ python click/options.py hello --greeting=Wazzup Kyle
Wazzup, Kyle!

$ python click/options.py hello --greeting=Wazzup --caps Kyle
WAZZUP, KYLE!

$ python click/options.py hello --caps Kyle
HELLO, KYLE!
```

## Version Option (--version)

In this section we'll be showing how to add a `--version` argument to each of our tools. For simplicity we'll just hardcode the version number to *1.0.0*. Keep in mind that in a production application, you will want to pull this from the installed application. One way to achieve this is with this simple process:

```sh
>>> import pkg_resources
>>> pkg_resources.get_distribution("click").version
>>> '5.1'
# replace click with the name of your tool
```

A second option for determining the version would be to have an automated version-bumping software change the version number defined in the file when a new version is released. This is possible with [bumpversion](https://pypi.python.org/pypi/bumpversion). But this approach is not recommended as it's easy to get out of sync. Generally, it's best practice to keep a version number in as few places as possible.

Since the implementation of adding a hard-coded version option is fairly simple we will use `...` to denote skipped sections of the code from the last section.

### Argparse

For *argparse* we again need to use the `add_argument` method, this time with the `action='version'` parameter and a value for `version` passed in. We apply this method to the root parser (instead of the `hello` or `goodbye` subparsers).

```python
...
parser = argparse.ArgumentParser()
parser.add_argument('--version', action='version', version='1.0.0')
...
```

```sh
$ python argparse/version.py --version
1.0.0
```

### Docopt

In order to add `--version` to *docopt* we add it as an option to our primary docstring. In addition we add the `version` parameter to our first call to docopt (parsing the primary docstring).

```python
"""usage: greet [--help] <command> [<args>...]

options:
  -h --help         Show this screen.
  --version         Show the version.

commands:
   hello       Say hello
   goodbye     Say goodbye

"""

from docopt import docopt

...

if __name__ == '__main__':
    arguments = docopt(__doc__, options_first=True, version='1.0.0')
    ...
```

```sh
$ python docopt/version.py --version
1.0.0
```

### Click

*Click* provides us with a convenient `@click.version_option` decorator. To add this we decorate our `greet` function (main `@click.group` function).

```python
...
@click.group()
@click.version_option(version='1.0.0')
def greet():
    ...
```

```sh
$ python click/version.py --version
version.py, version 1.0.0
```

## Improving Help (-h/--help)

The final step to completing our application is to improve the help documentation for each of the tools. We'll want to make sure that we can access help with both `-h` and `--help` and that each *argument* and *option* have some level of description.

### Argparse

By default *argparse* provides us with both `-h` and `--help` so we don't need to add anything for that. However our current help documentation for the subcommands is lacking information on what `--caps` and `--greeting` do and what the `name` argument is.

```sh
$ python argparse/version.py hello -h
usage: version.py hello [-h] [--greeting GREETING] [--caps] name

positional arguments:
  name

optional arguments:
  -h, --help           show this help message and exit
  --greeting GREETING
  --caps
```

In order to add more information we use the `help` parameter of the `add_argument` method.

```python
...

hello_parser = subparsers.add_parser('hello')
hello_parser.add_argument('name', help='name of the person to greet')
hello_parser.add_argument('--greeting', default='Hello', help='word to use for the greeting')
hello_parser.add_argument('--caps', action='store_true', help='uppercase the output')
hello_parser.set_defaults(func=greet)

goodbye_parser = subparsers.add_parser('goodbye')
goodbye_parser.add_argument('name', help='name of the person to greet')
goodbye_parser.add_argument('--greeting', default='Hello', help='word to use for the greeting')
goodbye_parser.add_argument('--caps', action='store_true', help='uppercase the output')

...
```

Now when we provide the help flag we get a much more complete result:

```sh
$ python argparse/help.py hello -h
usage: help.py hello [-h] [--greeting GREETING] [--caps] name

positional arguments:
  name                 name of the person to greet

optional arguments:
  -h, --help           show this help message and exit
  --greeting GREETING  word to use for the greeting
  --caps               uppercase the output
```

### Docopt

This section is where *docopt* shines. Because we wrote the documentation as the definition of the command-line interface itself, we already have the help documentation completed. In addition `-h` and `--help` are already provided.

```sh
$ python docopt/help.py hello -h
usage: basic.py hello [options] [<name>]

  -h --help         Show this screen.
  --caps            Uppercase the output.
  --greeting=<str>  Greeting to use [default: Hello].
```

### Click

Adding help documentation to *click* is very similar to *argparse*. We need to add the `help` parameter to all of our `@click.option` decorators.

```python
...

@greet.command()
@click.argument('name')
@click.option('--greeting', default='Hello', help='word to use for the greeting')
@click.option('--caps', is_flag=True, help='uppercase the output')
def hello(**kwargs):
    greeter(**kwargs)


@greet.command()
@click.argument('name')
@click.option('--greeting', default='Goodbye', help='word to use for the greeting')
@click.option('--caps', is_flag=True, help='uppercase the output')
def goodbye(**kwargs):
    greeter(**kwargs)

...
```

However, *click* **DOES NOT** provide us `-h` by default. We need to use the `context_settings` parameter to override the default `help_option_names`.

```python
import click

CONTEXT_SETTINGS = dict(help_option_names=['-h', '--help'])

...

@click.group(context_settings=CONTEXT_SETTINGS)
@click.version_option(version='1.0.0')
def greet():
    pass
```

Now the *click* help documentation is complete.

```sh
$ python click/help.py hello -h
Usage: help.py hello [OPTIONS] NAME

Options:
  --greeting TEXT  word to use for the greeting
  --caps           uppercase the output
  -h, --help       Show this message and exit.
```

## Error Handling

Error handling is an important part of any application. This section will explore the default error handling of each application and implement additional logic if needed. We'll explore three error cases:

1. Not enough required arguments given.
2. Invalid options/flags given.
3. A flag with a value is given.

### Argparse

```sh
$ python argparse/final.py hello
usage: final.py hello [-h] [--greeting GREETING] [--caps] name
final.py hello: error: the following arguments are required: name

$ python argparse/final.py --badoption hello Kyle
usage: final.py [-h] [--version] {hello,goodbye} ...
final.py: error: unrecognized arguments: --badoption

$ python argparse/final.py hello --caps=notanoption Kyle
usage: final.py hello [-h] [--greeting GREETING] [--caps] name
final.py hello: error: argument --caps: ignored explicit argument 'notanoption'
```

Not very exciting, as *argparse* handles all of our error cases out of the box.

### Docopt

```sh
$ python docopt/final.py hello
Hello, None!

$ python docopt/final.py hello --badoption Kyle
usage: basic.py hello [options] [<name>]
```

Unfortunatly, we have a bit of work to get *docopt* to an acceptable minimum level of error handling. The reccomended method for validation in *docopt* is the [schema](https://pypi.python.org/pypi/schema) module. *Make sure to install - `pip install schema`. In addition they provide a very basic [validation example](https://github.com/docopt/docopt/blob/master/examples/validation_example.py). The following is our application with schema validation:

```python
...
from schema import Schema, SchemaError, Optional
...
    schema = Schema({
        Optional('hello'): bool,
        Optional('goodbye'): bool,
        '<name>': str,
        Optional('--caps'): bool,
        Optional('--help'): bool,
        Optional('--greeting'): str
    })

    def validate(args):
        try:
            args = schema.validate(args)
            return args
        except SchemaError as e:
            exit(e)

    if arguments['<command>'] == 'hello':
        greet(validate(docopt(HELLO)))
    elif arguments['<command>'] == 'goodbye':
        greet(validate(docopt(GOODBYE)))
...
```

With this validation in place we now get some error messages.

```sh
$ python docopt/validation.py hello
None should be instance of <class 'str'>

$ python docopt/validation.py hello --greeting Kyle
None should be instance of <class 'str'>

$ python docopt/validation.py hello --caps=notanoption Kyle
--caps must not have an argument
usage: basic.py hello [options] [<name>]
```

While these messages are not very descriptive and may be hard to debug for larger applications, it's better than no validation at all. The schema module does provide other mechanisms for adding more descriptive error messages but we won't cover those here.

### Click

```sh
$ python click/final.py hello
Usage: final.py hello [OPTIONS] NAME

Error: Missing argument "name".

$ python click/final.py hello --badoption Kyle
Error: no such option: --badoption

$ python click/final.py hello --caps=notanoption Kyle
Error: --caps option does not take a value
```

Just like *argparse*, *click* handles error input by default.

<hr>

**With that, we have completed the construction of the command-line application we set out to build. Before we conclude let's take a look at another possible option.**

## Invoke

Can we use [invoke](https://invoke.readthedocs.org/en/latest/), a simple task running library, to build the greeter command-line application? Let's find out!

To start let's begin with the simplest version of the greeter:

*tasks.py*
```python
from invoke import task


@task
def hello(name):
    print('Hello, {0}!'. format(name))


@task
def goodbye(name):
    print('Goodbye, {0}!'.format(name))
```

With this very simple file we get a two tasks and very minimal help. From the same directory as *tasks.py* we get the following results:

```sh
$ invoke -l
Available tasks:

  goodbye
  hello

$ invoke hello Kyle
Hello, Kyle!

$ invoke goodbye Kyle
Goodbye, Kyle!
```

Now let's add in our options/flags - `--greeting` and `--caps`. In addition, we can pull out the greeting logic into it's own function, just as we did with the other tools.

```python
from invoke import task


def greet(name, greeting, caps):
    output = '{0}, {1}!'.format(greeting, name)
    if caps:
        output = output.upper()
    print(output)


@task
def hello(name, greeting='Hello', caps=False):
    greet(name, greeting, caps)


@task
def goodbye(name, greeting='Goodbye', caps=False):
    greet(name, greeting, caps)
```

Now we actually have the complete interface we designated in the beginning!

```sh
$ invoke hello Kyle
Hello, Kyle!

$ invoke hello --greeting=Wazzup Kyle
Wazzup, Kyle!

$ invoke hello --greeting=Wazzup --caps Kyle
WAZZUP, KYLE!

$ invoke hello --caps Kyle
HELLO, KYLE!
```

### Help Documentation

In order to compete with *argparse*, *docopt*, and *click*, we'll also need to be able to add complete help documentation. Luckily this is also available in *invoke* by using the `help` parameter of the `@task` decorator and adding docstrings to the decorated functions.

```python
...

HELP = {
    'name': 'name of the person to greet',
    'greeting': 'word to use for the greeting',
    'caps': 'uppercase the output'
}


@task(help=HELP)
def hello(name, greeting='Hello', caps=False):
    """
    Say hello.
    """
    greet(name, greeting, caps)


@task(help=HELP)
def goodbye(name, greeting='Goodbye', caps=False):
    """
    Say goodbye.
    """
    greet(name, greeting, caps)

```

```sh
$ invoke --help hello
Usage: inv[oke] [--core-opts] hello [--options] [other tasks here ...]

Docstring:
  Say hello.

Options:
  -c, --caps                     uppercase the output
  -g STRING, --greeting=STRING   word to use for the greeting
  -n STRING, --name=STRING       name of the person to greet
  -v, --version
```

### Version Option

Implementing a `--version` option is not quite as simple and comes with a caveat. The basics are that we will add `version=False` as an option to each of the tasks that calls a new `print_version` function if `True`. In order to make this work we cannot have any positional arguments without defaults or we get:

```sh
$ invoke hello --version
'hello' did not receive all required positional arguments!
```

Also note that we are calling `--version` on our commands `hello` and `goodbye` because *invoke* itself has a version command:

```sh
$ invoke --version
Invoke 0.10.1
```

The completed implementation of a version command follows:

```python
...

def print_version():
    print('1.0.0')
    exit(0)


@task(help=HELP)
def hello(name='', greeting='Hello', caps=False, version=False):
    """
    Say hello.
    """
    if version:
        print_version()
    greet(name, greeting, caps)

...
```

Now we are able to ask invoke for the version of our tool:

```sh
$ invoke hello --version
1.0.0
```

## Conclusion

To review, let's take a look at the final version of each of the tools we created.

### Argparse

```python
import argparse


def greet(args):
    output = '{0}, {1}!'.format(args.greeting, args.name)
    if args.caps:
        output = output.upper()
    print(output)

parser = argparse.ArgumentParser()
parser.add_argument('--version', action='version', version='1.0.0')
subparsers = parser.add_subparsers()

hello_parser = subparsers.add_parser('hello')
hello_parser.add_argument('name', help='name of the person to greet')
hello_parser.add_argument('--greeting', default='Hello', help='word to use for the greeting')
hello_parser.add_argument('--caps', action='store_true', help='uppercase the output')
hello_parser.set_defaults(func=greet)

goodbye_parser = subparsers.add_parser('goodbye')
goodbye_parser.add_argument('name', help='name of the person to greet')
goodbye_parser.add_argument('--greeting', default='Hello', help='word to use for the greeting')
goodbye_parser.add_argument('--caps', action='store_true', help='uppercase the output')
goodbye_parser.set_defaults(func=greet)

if __name__ == '__main__':
    args = parser.parse_args()
    args.func(args)
```

### Docopt

```python
"""usage: greet [--help] <command> [<args>...]

options:
  -h --help         Show this screen.
  --version         Show the version.

commands:
   hello       Say hello
   goodbye     Say goodbye

"""

from docopt import docopt
from schema import Schema, SchemaError, Optional

HELLO = """usage: basic.py hello [options] [<name>]

  -h --help         Show this screen.
  --caps            Uppercase the output.
  --greeting=<str>  Greeting to use [default: Hello].
"""

GOODBYE = """usage: basic.py goodbye [options] [<name>]

  -h --help         Show this screen.
  --caps            Uppercase the output.
  --greeting=<str>  Greeting to use [default: Goodbye].
"""


def greet(args):
    output = '{0}, {1}!'.format(args['--greeting'],
                                args['<name>'])
    if args['--caps']:
        output = output.upper()
    print(output)


if __name__ == '__main__':
    arguments = docopt(__doc__, options_first=True, version='1.0.0')

    schema = Schema({
        Optional('hello'): bool,
        Optional('goodbye'): bool,
        '<name>': str,
        Optional('--caps'): bool,
        Optional('--help'): bool,
        Optional('--greeting'): str
    })

    def validate(args):
        try:
            args = schema.validate(args)
            return args
        except SchemaError as e:
            exit(e)

    if arguments['<command>'] == 'hello':
        greet(validate(docopt(HELLO)))
    elif arguments['<command>'] == 'goodbye':
        greet(validate(docopt(GOODBYE)))
    else:
        exit("{0} is not a command. See 'options.py --help'.".format(arguments['<command>']))
```

### Click

```python
import click

CONTEXT_SETTINGS = dict(help_option_names=['-h', '--help'])


def greeter(**kwargs):
    output = '{0}, {1}!'.format(kwargs['greeting'],
                                kwargs['name'])
    if kwargs['caps']:
        output = output.upper()
    print(output)


@click.group(context_settings=CONTEXT_SETTINGS)
@click.version_option(version='1.0.0')
def greet():
    pass


@greet.command()
@click.argument('name')
@click.option('--greeting', default='Hello', help='word to use for the greeting')
@click.option('--caps', is_flag=True, help='uppercase the output')
def hello(**kwargs):
    greeter(**kwargs)


@greet.command()
@click.argument('name')
@click.option('--greeting', default='Goodbye', help='word to use for the greeting')
@click.option('--caps', is_flag=True, help='uppercase the output')
def goodbye(**kwargs):
    greeter(**kwargs)

if __name__ == '__main__':
    greet()
```

### Invoke

```python
from invoke import task


def greet(name, greeting, caps):
    output = '{0}, {1}!'.format(greeting, name)
    if caps:
        output = output.upper()
    print(output)


HELP = {
    'name': 'name of the person to greet',
    'greeting': 'word to use for the greeting',
    'caps': 'uppercase the output'
}


def print_version():
    print('1.0.0')
    exit(0)


@task(help=HELP)
def hello(name='', greeting='Hello', caps=False, version=False):
    """
    Say hello.
    """
    if version:
        print_version()
    greet(name, greeting, caps)


@task(help=HELP)
def goodbye(name='', greeting='Goodbye', caps=False, version=False):
    """
    Say goodbye.
    """
    if version:
        print_version()
    greet(name, greeting, caps)
```

### My Recomendation

Now, to get this out of the way, my personal go-to library is *click*. I have been using it on large, multi-command, complex interfaces for the last year. (Credit goes to [@kwbeam](https://twitter.com/kwbeam) for introducing me to *click*). I prefer the decorator approach and think it lends a very clean, composable interface. That being said, let's evaluate each option fairly.

### arparse

***Arparse* is the standard library (included with Python) for creating command-line utilities.** For that fact alone, it is arguably the most used of the tools examined here. *Argparse* is also very simple to use as lots of *magic* (implicit work that happens behind the scenes) is used to construct the interface. For example, both arguments and options are defined using the `add_arguments` method and *argparse* figures out which is which behind the scenes.

### docopt

**If you think writing documentation is great, *docopt* is for you!** In addition *docopt* has implementations for [many other languages](https://github.com/docopt) - meaning you can learn one library and use it across many languages. The downside of *docopt* is that it is very structured in the way you have to define your command-line interface. (Some might say this is a good thing!)

### click

I've already said that I really like *click* and have been using it in production for over a year. **I encourage you to read the very complete [Why Click?](http://click.pocoo.org/5/why/) documentation.** In fact, that documentation is what inspired this blog post! The decorator style implementation of *click* is very simple to use and since you are decorating the function you want executed, it makes it very easy to read the code and figure out what is going to be executed. In addition, *click* supports advanced features like callbacks, command nesting, and more. *Click* is based on a fork of the now deprecated [optparse](https://docs.python.org/2/library/optparse.html) library.

### invoke

**Invoke surprised me in this comparison.** I thought that a library designed for task execution might not be able to easily match full command-line libraries - but it did! That being said, I would not recommend using it for this type of work as you will certainly run into limitations for anything more complex than the example presented here.

## Bonus: Packaging

Since not everyone is packaging up there python source with [setuptools](https://pypi.python.org/pypi/setuptools) (or other solutions), we decided not to make this a core component of the article. In addition, we don't want to cover *packaging* as a complete topic. If you want to learn more about packaging with setuptools [go here](https://packaging.python.org/en/latest/) or with conda [go here](http://conda.pydata.org/docs/building/build.html) or you can read my previous [blog post](http://kylepurdon.com/blog/packaging-python-basics-with-continuum-analytics-conda.html) on conda packaging. **What we will cover here is how to use the *entry_points* option to make a command-line application an executable command on install**.

### Entry Point Basics

An [entry_point](https://packaging.python.org/en/latest/distributing.html?highlight=entry_points#entry-points) is essentially a map to a single function in your code that will be given a command on your system PATH. An entry_point has the form - `command = package.module:function`

The best way to explain this is to just look at our *click* example and add an entry point.

### Packaging Click Commands

*Click* makes packaging simple, as by default we are calling a single function when we execute our program:

```python
if __name__ == '__main__':
    greet()
```

In addition to the rest of the *setup.py* (not covered here), we would add the following to create an *entry_point* for our *click* application.

Assuming the following directory structure-

```sh
greeter/
├── greet
│   ├── __init__.py
│   └── cli.py       <-- the same as our final.py
└── setup.py
```

-we will create the following *entry_point*:

```python
entry_points={
    'console_scripts': [
        'greet=greet.cli:greet',  # command=package.module:function
    ],
},
```

When a user installs the package created with this *entry_point*, *setuptools* will create the following executable script (called `greet`) and place it on the PATH of the user's system.

```python
#!/usr/bin/python
if __name__ == '__main__':
    import sys
    from greet.cli import greet

    sys.exit(greet())
```

After installation the user will now be able to run the following:

```sh
$ greet --help
Usage: greet [OPTIONS] COMMAND [ARGS]...

Options:
  --version   Show the version and exit.
  -h, --help  Show this message and exit.

Commands:
  goodbye
  hello
```

### Packaging Argparse Commands

The only thing we need to do differently from *click* is to pull all of the application initialization into a single function that we can call in our *entry_point*.

This:

```python
if __name__ == '__main__':
    args = parser.parse_args()
    args.func(args)
```

Becomes:

```python

def greet():
    args = parser.parse_args()
    args.func(args)

if __name__ == '__main__':
    greet()

```

Now we can use the same pattern for the *entry_point* we defined for *click*.

### Packaging Docopt Commands

Packaging *docopt* commands requires the same process as *argparse*.

This:

```python
if __name__ == '__main__':
    arguments = docopt(__doc__, options_first=True, version='1.0.0')

    if arguments['<command>'] == 'hello':
        greet(docopt(HELLO))
    elif arguments['<command>'] == 'goodbye':
        greet(docopt(GOODBYE))
    else:
        exit("{0} is not a command. See 'options.py --help'.".format(arguments['<command>']))
```

Becomes:

```python
def greet():
    arguments = docopt(__doc__, options_first=True, version='1.0.0')

    if arguments['<command>'] == 'hello':
        greet(docopt(HELLO))
    elif arguments['<command>'] == 'goodbye':
        greet(docopt(GOODBYE))
    else:
        exit("{0} is not a command. See 'options.py --help'.".format(arguments['<command>']))

if __name__ == '__main__':
    greet()
```
