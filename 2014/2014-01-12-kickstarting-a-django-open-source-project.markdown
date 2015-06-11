# Kickstarting a Django Open Source Project

This is a guest post by [Patrick Altman](http://paltman.com/), a passionate open source hacker, and VP of engineering at [Eldarion](http://eldarion.com/).

<hr>

In this article we will dissect a Django app boilerplate meant for quickly developing open source projects that adhere to widely shared conventions.

The layout (or pattern) that we will be looking at is based on [django-stripe-payments](https://github.com/eldarion/django-stripe-payments). This layout has proven successful for well over 100 different open source projects published by [Eldarion](http://github.com/eldarion/) and [Pinax](http://github.com/pinax/).

As you go through this article, keep in mind that your specific project may vary from this pattern; however, there are a number of items that are fairly common across Python projects, and to some degree open source projects in general. We will focus on these items.

## Project Layout

A good project layout helps a new user navigate your source code. There are certain conventions that are commonly accepted as well that are important to abide by. In addition, a good project layout helps with packaging.

Project layouts will differ a bit depending on if they are Python packages (like a reusable Django app) or something like a Django project. Using our example project, we’d like to highlight some aspects of the layout as well as files included in the top level of the project.

You should reserve the root of your project for metadata files that describe various aspects of your project such as `LICENSE`, `CONTRIBUTING.md`, `README.rst`, as well as any scripts for running tests and packaging. In addition, there should be a folder in this root level that is named what you want the Python package name to be. In our `django-stripe-payments` example it is `payments`. Lastly, you should store your documentation as a Sphinx based project in a folder called `docs`.

### Licensing

It is generally best to license your software in the permissive MIT or BSD licenses if your goal is the most widespread adoption possible. The greater the adoption, the more exposure in varied real world environments which increases opportunity for feedback and cooperation via Pull requests. Store the contents of the license in a `LICENSE` file at the root of your project.

### README

Every project should have a `README.rst` file in the project root. This document should briefly introduce the user to the project, describe what problem it solves, and provide a quick getting started guide.

Name it `README.rst` and put it at the root of your repo and GitHub will display it on your main project page for potential users to see and skim through to get a quick feel for how your software might help them.

> Using [reStructuredText](http://docutils.sourceforge.net/rst.html) instead of [Markdown](http://daringfireball.net/projects/markdown/) in the readme is recommended  so that it displays nicely on [PyPI](https://pypi.python.org/pypi/) if you publish your package.

### Contributing Guidelines

A `CONTRIBUTING.md` file discusses the code style guide, processes, and guidelines for people wishing to contribute code through Pull Requests to your project. This is helpful in lowering the barrier for people wishing to contribute code. First time contributors might be nervous about doing something wrong or out of convention and the more detailed this document it is the more they can check themselves without having to ask questions that they might be too shy to ask.

### setup.py

Good packaging helps your project’s distribution. By writing a `setup.py` script, you can leverage Python’s packaging tools to create and publish your project on [PyPI](https://pypi.python.org/pypi/).

This is a really simple script. For example, here is the core of the script for django-stripe-payments:

```python
PACKAGE = "payments"
NAME = "django-stripe-payments"
DESCRIPTION = "a payments Django app for Stripe"
AUTHOR = "Patrick Altman"
AUTHOR_EMAIL = "paltman@eldarion.com"
URL = "https://github.com/eldarion/django-stripe-payments"
VERSION = __import__(PACKAGE).__version__


setup(
    name=NAME,
    version=VERSION,
    description=DESCRIPTION,
    long_description=read("README.rst"),
    author=AUTHOR,
    author_email=AUTHOR_EMAIL,
    license="BSD",
    url=URL,
    packages=find_packages(exclude=["tests.*", "tests"]),
    package_data=find_package_data(PACKAGE, only_in_packages=False),
    classifiers=[
        "Development Status :: 3 - Alpha",
        "Environment :: Web Environment",
        "Intended Audience :: Developers",
        "License :: OSI Approved :: BSD License",
        "Operating System :: OS Independent",
        "Programming Language :: Python",
        "Framework :: Django",
        ],
    install_requires=[
        "django-jsonfield>=0.8",
        "stripe>=1.7.9",
        "django>=1.4",
        "pytz"
        ],
    zip_safe=False,
    )
```

There are several things going on here. Remember how we discussed making the `README.rst` file reStructuredText? That is because as you can see for the `long_description` we are using the contents of that file to populate the landing page on PyPI and that’s the markup language used there. The classifiers are a set of metadata that help put your project in the right categories on PyPI. Finally, the `install_requires` argument will make sure when your package is installed that these listed dependancies get installed or are already installed.

## GitHub

If your project is not on [GitHub](http://github.com) you are really missing out. Sure there are other web based DVCS (distributed version control system) sites that offer free open source hosting but none have done more for open source than GitHub.

### Handling Pull Requests

Part of building a great open source project is making it bigger than yourself. This involves not only increasing merely the user base but also the contributor base. GitHub (and git in general) has really transformed how this is done.

One key to increasing contributors is to be responsive in managing Pull requests. This doesn't mean accepting every contribution but also remain open minded and handle responses with the deference you would like to receive if you were contributing to another project.

Do not just close requests that you don't want but take the time to explain why you won't accept them, or if possible explain how they can be improved so they can be excepted. If the improvements are minor or you can otherwise improve upon them yourself, go ahead and accept it and then make the corrections you'd like. There is a fine line between asking for improvements versus just doing them yourself.

The guiding principal is to create a welcoming and grateful atmosphere. Remember your contributors are volunteering their time and energy to improve your project.

### Versioning, Branching, and Releases

Read and follow [Semantic Versioning](http://semver.org) when creating releases.

When you do major releases always document clearly backward incompatible changes. It's easiest if your document changes as you are committing them by updating a change log file as you work between releases. This file could simply be a CHANGELOG file at the root of your project, or part of your documentation in a file somewhere like `docs/changelog.rst`. This will enable you to create nice Release Notes with very little effort.

Keep the master stable. There is always the chance that people will use code on master instead of a packages release. Create feature branches for work and merge when it's tested and relatively stable.

## Documentation

No project is fully complete until there is some amount of documentation. Good documentation saves users from having to read source code to determine how to use your software. Good documentation communicates that you care about your users.

With [Read The Docs](https://readthedocs.org), you can have your documentation automatically rendered and hosted for free. It will automatically update on every commit to master which is super cool.

In order to use Read the Docs, you should create a [Sphinx](http://sphinx-doc.org/) based documentation project in the `docs` folder at the root of your project. This really is a pretty simple thing and consists of a `Makefile` and a `conf.py` file and then a collection of reStructuredText formatted files. You can do this manually by copying and pasting the `Makefile` and `conf.py` file from a previous project and modifying the values, or by running:

```sh
$ pip install Sphinx
$ sphinx-quickstart
```

## Automating Code Quality

There are a number of tools you can use to help keep quality in check on your projects. Linting, testing, and test coverage should all be used to help insure quality doesn’t drift over the life of the project.

Start with linting with something like `pylint` or `pep8` or `pyflakes`. They all have their pros and cons which are beyond the scope of this article to explore. Point being: consistent style is the first step in a high quality project. In addition, helping with style these linters can help identify some simple bugs quickly.

For example, for `django-stripe-payments`, we have a script that  combines running two different lint tools with customized exceptions for our project:

```sh
# lint.sh
pylint --rcfile=.pylintrc payments && pep8 --ignore=E501 payments
```

Take a look at the `.pylintrc` file in the `django-stripe-payments` repo for examples of some of the exceptions. One thing about `pylint` is that it's pretty aggressive and can be noisy with things are that are really not problematic. You need to decide for yourself to tune your own `.pylintrc` file but I recommend documenting the file so you know later why you are excluding certain rules.

Setting up a good testing infrastructure is important in proving that your code works. Furthermore, writing some of your tests first can help you think through your API. Even if you write tests last, the act of writing tests will expose weak spots in your API design and/or other usability issues that you can address before they are reported.

Part of your testing infrastructure should involve using coverage.py to keep an eye on a module’s coverage. This tool won’t tell you if code is tested, only you can do that, but it will help identify code that isn’t executed at all so you know what code is definitely not being tested.

Once you have linting, testing and coverage scripts integrated into your project you can setup automation so that these tools execute on every push in one more environments (e.g. different versions of Python, different versions of Django, or both in a test matrix).

Setting up a [Travis](https://travis-ci.org/) integration can automatically execute tests and linters. [Coveralls](https://coveralls.io/) can be added to this configuration to provide historical testing coverage when Travis builds run. Both have features that enable you to embed a badge in your README.md to show off latest build status and coverage.

## Collaboration versus cooperation

During DjangoCon 2011, [David Eaves](http://eaves.ca/) gave a [keynote address](https://www.youtube.com/watch?v=SzGi1DfbZMI) that eloquently put into words the notion that although collaboration and cooperation have similar definitions, there is a subtle difference:

"I would argue that collaboration, unlike cooperation, requires the parties involved in a project jointly solve problems."

Eaves goes on to devote an entire post specifically to how GitHub was the driving force for innovating how open source works—specifically, the aspect of community management. In "How GitHub Saved OpenSource" (see Resources), Eaves states:

"I believe open source projects work best when contributors are able to engage in low transaction cost cooperation and high transaction cost collaboration is minimized. The genius of open source is that it does not require a group to debate every issue and work on problems collectively, quite the opposite."

He goes on to talk about the value of forking and how it reduces the high costs of collaboration by enabling low-cost cooperation among people able to take projects forward without permission. This forking pushes off the need for coordination until solutions are ready to be merged in, enabling much more rapid and dynamic experimentation.

You can shape your project in similar ways, with the same goal of increasing low-cost cooperation while minimizing expensive collaboration throughout writing, maintaining, and supporting your project by following the conventions and patterns detailed in this post.

## Summary

A lot was covered here with little actual examples. The best way to learn about these things in detail is to browse the repositories on GitHub of projects that do a good job on these patterns. Pinax has their own boilerplate [here](https://github.com/pinax/pinax-starter-app), which can be used to quickly generate a Project based on the conventions and patterns found in this post. Keep in mind, that even if you go with our boilerplate or some other boilerplate, you will need to find a style in implementing these things that fits you and your project. All of these things are in addition to writing the actual code of your project&mdash;but they all help in growing a community of contributors.