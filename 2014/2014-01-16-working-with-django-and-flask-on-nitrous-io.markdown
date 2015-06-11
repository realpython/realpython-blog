---
layout: post
title: "Working with Django and Flask on Nitrous.IO"
date: 2014-01-16 07:45:19 -0700
toc: true
comments: true
category_side_bar: true
categories: [python, django, flask]

keywords: "python, web development, django, flask, nitrious.io"
description: "In this article we will look at how to quickly set up a cloud-based development environment with Nitrious.IO."
---

This is a guest post by our friend Greg McKeever from [Nitrous.IO](http://www.nitrous.io?utm_source=realpython.com&utm_medium=blog&utm_content=nitrous_io_python_dev_environment).

<hr>

Nitrous.IO is a platform which allows you to spin up your own development environment quickly in the cloud. Here are a few key advantages of coding on [Nitrous.IO](http://www.nitrous.io?utm_source=realpython.com&utm_medium=blog&utm_content=nitrous_io_python_dev_environment):

- Save countless hours (or days) of setting up your Windows or Mac OS for development. Your Nitrous box comes pre-loaded with many [tools and interpreters](http://help.nitrous.io/box-interpreters-and-tools/?utm_source=realpython.com&utm_medium=blog&utm_content=nitrous_io_python_dev_environment), so you can start coding immediately.

- Your code is accessible from any computer or mobile device. Edit code via Web IDE, SSH, or [sync locally](http://www.nitrous.io/mac?utm_source=realpython.com&utm_medium=blog&utm_content=nitrous_io_python_dev_environment) and use your favorite text editor.

- Collaboration: Nitrous provides a way for you to share your development environment with any other user. If you are running into any issues with your project and need help, you can invite a friend to your Nitrous box to edit and run your code.

## Getting Started

To get started, sign up at [Nitrous.IO](http://www.nitrous.io?utm_source=realpython.com&utm_medium=blog&utm_content=nitrous_io_python_dev_environment). Once your account is confirmed, navigate to the [boxes page](https://www.nitrous.io/app#/boxes) and create a new Python/Django box.

![Create Python Box](https://realpython.com/images/blog_images/nitrous-create-python-box.png)

There are many [tools and interpreters](http://help.nitrous.io/box-interpreters-and-tools/?utm_source=realpython.com&utm_medium=blog&utm_content=nitrous_io_python_dev_environment) which are included with the Nitrous box, and at the time of writing this you will have Python 2.7.3 and Django 1.5.1 included with your dev environment. If this is what you are wanting to start working with then everything is ready to go!

If you are looking to use a different version of Python, Django, or utilize another framework such as Flask, keep reading.

## Setting up Python 2.7 with Virtualenv

Virtualenv allows you to create an isolated environment in order to install specific versions of Python, Django, and also install other frameworks such as Flask without requiring root access. Since the Nitrous boxes do not offer root at this time, this is the best route to go.

To view the available versions of Python available, run `ls /usr/bin/python*` in the console. Create a new environment with Python 2.7 by running the following command:

```sh
$ virtualenv -p /usr/bin/python2.7 py27env
```

You will now want to connect to this environment:

```sh
$ source py27env/bin/activate
```

![Virtualenv](https://realpython.com/images/blog_images/nitrous-virtual-env-python27.png)

>If you decide you want to disconnect from this environment at any point, type `deactivate` in the console.

Since you are in an isolated environment, you will need to install Django and any other dependencies that were available outside of your environment. You can check which modules are installed with `pip freeze`.

## Installing Django

To install the latest official version of Django, you will want to utilize pip:

```sh
$ pip install Django
```

## Installing Flask

Installing Flask is just as easy as installing Django with pip. Run the following command to install the latest official release:

```sh
$ pip install Flask
```

That's it! You can verify the installation by running the command `pip freeze`, and locating Flask in the list. You are now ready to start your course here at [RealPython](https://www.realpython.com/).

One thing to remember is that you can always disconnect from Virtualenv by running `deactivate` in the console. If you named your Virtualenv session 'py27env' as seen in this article, you can always reconnect by running `source py27env/bin/activate`.
