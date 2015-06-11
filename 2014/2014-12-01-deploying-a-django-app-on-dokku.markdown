# Deploying Django on Dokku

**Originally written for *[Gun.io](http://www.gun.io)*, this post details how to use Dokku as a Heroku replacement for deploying your Django App.**

## What is Dokku?

A few days ago I was pointed towards the [Dokku](https://github.com/progrium/dokku) project, which is a "Docker powered mini-Heroku" that you can deploy on your own server to serve as your own private PaaS.

**Why would you want your own mini-Heroku?**

- Well, [Heroku can cost *A Lot* of money](http://joshsymonds.com/blog/2012/06/03/my-love-slash-hate-relationship-with-heroku/); it's hosted in the cloud and you may not want your application to leave the room just yet; and you don't have 100% control over all aspects of the platform.
- Or maybe you're just the DIY kind of person.

Regardless, [Jeff Lindsay](http://progrium.com/blog/) hacked Dokku together in less than 100 lines of bash code!

Dokku ties [Docker](http://www.docker.io/) together with [Gitreceive](https://github.com/progrium/gitreceive) and [Buildstep](https://github.com/progrium/buildstep) into one package that is easy to deploy, fork/hack, and update.

## What You Need To Get Started

You could use anything from AWS to a computer on your own private network. I decided to use [Digital Ocean](https://www.digitalocean.com/) as my cloud hosting service for this little project.

The requirements for hosting Dokku are simple:

* Ubuntu 14.10 x64
* SSH Capabilities

> Digital Ocean has a [pre-configured Droplet](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-dokku-application) that you can use that's already provisioned for a Dokku environment. Feel free to use this. **We'll be using a fresh server so you can recreate this process on any server, not just one on Digital Ocean**.

Let's begin!

1. First, sign up for an account on [Digital Ocean](https://www.digitalocean.com), making sure to add a public-key to the account. *You can follow steps one through three in this [guide](https://www.digitalocean.com/community/articles/how-to-set-up-ssh-keys--2) to help get you set up if you need to create a new key. Step four will come in handy later.*
1. Next, create a "droplet" (spin up a node) by clicking "Create Droplet". Make sure you pick "Ubuntu 14.10 x64" as your image. *I initially picked the x32 version and Dokku wouldn't install (see [https://github.com/progrium/dokku/issues/51](https://github.com/progrium/dokku/issues/51)).* Add your ssh public-key to the droplet so you can ssh into the machine without having to enter a password every time you log in. Digital Ocean takes about a minute to spin up your machine.
1. After it's ready, Digital Ocean will either email you saying so and include the machine's IP address in the email or the machine will appear under your droplets panel. Use the IP address to SSH into the machine and follow step four in Digital Ocean's [ssh guide](https://www.digitalocean.com/community/articles/how-to-set-up-ssh-keys--2).

## Installing Dokku

Now that our host is all set up, it's time to install and configure Dokku. SSH back into your host machine and run the following command:

```sh
$ wget -qO- https://raw.github.com/progrium/dokku/v0.2.3/bootstrap.sh | sudo DOKKU_TAG=v0.2.3 bash
```

> Make sure you use `sudo` regardless of whether you are logged in as the root or not. For more info, see the "Wrapping Up" section below.

Installation could take anywhere from two to five minutes. Log out of your host machine when done.

Be sure to upload a public-key for the user using the following format:

```sh
$ cat ~/.ssh/id_rsa.pub | ssh root@<machine-address> "sudo sshcommand acl-add dokku <your-app-name> "
```

Replace `<machine-address>` with the IP address or the domain name of your host machine and `<your-app-name>` with the name of your Django Project.

For example:

```sh
$ cat ~/.ssh/id_rsa.pub | ssh root@104.236.38.176 "sudo sshcommand acl-add dokku hellodjango"
```

## Deploying a Django Application to Dokku

For this tutorial, let's follow the [Getting Started with Django on Heroku](https://devcenter.heroku.com/articles/django) guide to get an initial Django application set up.

> Again, Dokku uses [Buildstep](https://github.com/progrium/buildstep) which uses Heroku buildpacks to build your applications. It comes with the [Heroku Python Buildpack](https://github.com/heroku/heroku-buildpack-python) built in, which is enough to run a Django or Flask application off of, right out of the box. However if you'd like to add a custom buildpack [you can](https://github.com/progrium/buildstep#adding-buildpacks).

Create a Django Project and add a local Git repo:

```sh
$ mkdir hellodjango && cd hellodjango
$ virtualenv venv
$ source venv/bin/activate
$ pip install django-toolbelt
$ django-admin.py startproject hellodjango .
$ echo "web: gunicorn hellodjango.wsgi" > Procfile
$ pip freeze > requirements.txt
$ echo "venv" > .gitignore
$ git init
$ git add .
$ git commit -m "First Commit HelloDjango"
```

We have to add Dokku on our host machine as a Git remote:

```sh
$ git remote add production dokku@<machine-address>:hellodjango
```

Now we can PUSH our code:

```sh
$ git push production master
```

Replace `<machine-address>` with the address or domain name of your host machine. If all went well, you should see an application deployed message in your terminal:

```sh
=====> Application deployed:
       http://104.236.38.176:49153
```

Next, go and visit `http://<machine-address>:49153` and you should see the familiar "Welcome to Django" page. Now you can work on your application locally and then push it to your own mini-heroku!.

## Wrapping Up

Initially I installed Dokku without the 'sudo':

```sh
$ wget -qO- https://raw.github.com/progrium/dokku/v0.2.3/bootstrap.sh | DOKKU_TAG=v0.2.3 bash
```

When I tried to push to Dokku the python build pack would fail trying to download/build python. The solution to this was to uninstall Dokku and then reinstall it the right way, using sudo:

```sh
$ wget -qO- https://raw.github.com/progrium/dokku/v0.2.3/bootstrap.sh | sudo DOKKU_TAG=v0.2.3 bash
```

Unfortunately, Dokku isn't quite as far along as Heroku is.

For example, all commands need to run directly on the host server since Dokku does not have a client-side app like Heroku.

So, in order to run a command like this-

```sh
$ heroku run python manage.py syncdb
```

-you need to first SSH into the server. The easiest way to achieve this is to create a `dokku` command:

```sh
alias dokku="ssh -t root@<machine-address> dokku"
```

Now you can run the following command to sync the database:

```sh
$ dokku run hellodjango python manage.py syncdb
```

Dokku does allow you to configure environment variables separately for each application. Simply create/edit `/home/git/APP_NAME/ENV` and fill it with things like:

```sh
export DATABASE_URL=somethinghere
```

<hr>

Dokku is still a young platform so hopefully it keeps growing and becoming more useful. It's open source as well so if you want to contribute, make a pull request or open an issue on [Github](https://github.com/progrium/dokku).
