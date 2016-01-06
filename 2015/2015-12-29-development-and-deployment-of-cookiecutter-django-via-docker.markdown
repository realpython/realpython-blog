# Development and Deployment of Cookiecutter-Django via Docker

**Let's look at how to bootstrap a Django Project pre-loaded with the basic requirements needed in order to quickly get a project up and running.** Further, beyond the project structure, most bootstrapped projects also take care of setting up the development and production environment settings, without troubling the user much - so we'll look at that as well.

<div class="center-text">
  <img class="no-border" src="/images/blog_images/cookiecutter-django-docker.png" style="max-width: 100%;" alt="cookiecutter django docker">
</div>

<br>

We'll be using the popular [cookiecutter-django](https://github.com/pydanny/cookiecutter-django) as the bootstrapper for our Django Project along with [Docker](https://www.docker.com/) to manage our application environment.

Let’s begin!

## Local Setup

Start by installing [cookiecutter](https://github.com/audreyr/cookiecutter) globally:

```sh
$ pip install cookiecutter==1.3.0
```

Now execute the following command to generate a bootstrapped django project:

```sh
$ cookiecutter https://github.com/pydanny/cookiecutter-django.git
```

This command runs cookiecutter with the [cookiecutter-django](https://github.com/pydanny/cookiecutter-django) repo, allowing us to enter project-specific details:

```sh
Cloning into 'cookiecutter-django'...
remote: Counting objects: 4810, done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 4810 (delta 0), reused 0 (delta 0), pack-reused 4803
Receiving objects: 100% (4810/4810), 844.79 KiB | 582.00 KiB/s, done.
Resolving deltas: 100% (3039/3039), done.
Checking connectivity... done.
project_name [project_name]: django_cookiecutter_docker
repo_name [django_cookiecutter_docker]: django_cookiecutter_docker
author_name [Your Name]: Michael Herman
email [Your email]: michael@realpython.com
description [A short description of the project.]: Tutorial on bootstrapping django projects
domain_name [example.com]: example.com
version [0.1.0]: 0.1.0
timezone [UTC]: UTC
now [2015/11/04]: 2015/11/04
year [2015]: 2015
use_whitenoise [y]: y
use_celery [n]: n
use_mailhog [n]: n
use_sentry [n]: n
use_newrelic [n]: n
use_opbeat [n]: n
windows [n]: n
use_python2 [n]: n
```

### Project Structure

Take a quick look at the generated project structure, taking specific note of the following directories:

1. "config" includes all the settings for our local and production environments.
1. "requirements" contains all the requirement files - *base.txt*, *local.txt*, *production.txt*, *test.txt* - which you can make changes to and then install via `pip install -r file_name`.
1. "django_cookiecutter_docker" is the main project directory which consists of the "static", "contrib" and "templates" directories along with the `users` app containing the models and boilerplate code associated with user authentication.

## Docker Setup

Start by downloading and then installing the [Docker Toolbox](https://www.docker.com/docker-toolbox) (v[1.9.1f](https://github.com/docker/toolbox/releases/tag/v1.9.1f)) to obtain virtualbox and the required Docker components:

- docker 1.9.1
- docker-machine 0.5.4
- docker-compose 1.5.2

### Docker Machine

Once installed, create a new Docker host within the root of the newly created Django Project:

```sh
$ docker-machine create -d virtualbox dev
$ eval "$(docker-machine env dev)"
```

> **NOTE**: `dev` can be named anything you want. For example, if you have more than one development environment, you could name them `djangodev1`, `djangodev2`, and so forth.

To view all Machines, run:

```sh
$ docker-machine ls
```

You can also view the IP of the `dev` Machine by running:

```sh
$ docker-machine ip dev
```

Finally, let's create a "/data" partition within the VM itself so that changes are persistent:

```sh
$ docker-machine ssh dev
$ sudo su
$ echo 'ln -sfn /mnt/sda1/data /data' >> /var/lib/boot2docker/bootlocal.sh
```

### Docker Compose

Now we can fire everything up - e.g., Django and Postgres - via Docker Compose:

```sh
$ docker-compose -f dev.yml build
$ docker-compose -f dev.yml up -d
```

> Running Windows? Hit this error - `Interactive mode is not yet supported on Windows`? See [this comment](https://realpython.com/blog/python/development-and-deployment-of-cookiecutter-django-via-docker/#comment-2442262433).

The first build will take a while. Due to [caching](https://docs.docker.com/engine/articles/dockerfile_best-practices/#build-cache), subsequent builds will run much faster.

## Sanity Check

Now we can test our Django Project by applying the migrations and then running the server:

```sh
$ docker-compose -f dev.yml run django python manage.py makemigrations
$ docker-compose -f dev.yml run django python manage.py migrate
$ docker-compose -f dev.yml run django python manage.py createsuperuser
```

Navigate to the `dev` IP (port 8000) in your browser to view the Project quick start page with debugging mode on and many more development environment oriented features installed and running.

Kill the server, initialize a new git repo, commit, and PUSH to GitHub.

## Deployment Setup

So, we have successfully set up our Django Project locally using cookiecutter-django and served it up using the traditional *manage.py* command line utility via Docker.

In this section, we move on to the deployment part, where the role of a web server comes into play. We will be setting up a Docker Machine on a [Digital Ocean](https://www.digitalocean.com/?refcode=d8f211a4b4c2) droplet, with Postgres as our database and [Nginx](http://nginx.org/) as our web server.

Along with this, we will be making use of [gunicorn](http://gunicorn.org/) instead of Django’s single-threaded development server to run the server process.

### Why Nginx?

Apart from being a high-performance HTTP server, which almost every good web server out in the market is, Nginx has some really good features that make it stand out from the rest - namely that it:

- Can couple as a [reverse proxy server](https://en.wikipedia.org/wiki/Reverse_proxy),
- Can host more than one site,
- Has an asynchronous way of handling web requests, which means that since it doesn’t rely on threads to handle web requests, it has a higher performance while handling multiple requests.

### Why Gunicorn?

[Gunicorn](http://gunicorn.org/) is a Python WSGI HTTP server that can be easily customized and provides better performance in terms of reliability than Django's single-threaded development server within production environments.

### Digital Ocean Setup

We will be using a [Digital Ocean](https://www.digitalocean.com/?refcode=d8f211a4b4c2) server for this tutorial. After you [sign up](https://www.digitalocean.com/?refcode=d8f211a4b4c2) (if necessary), [generate](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2) a Personal Access Token, and then run the following command:

```sh
$ docker-machine create \
-d digitalocean \
--digitalocean-access-token=ADD_YOUR_TOKEN_HERE \
prod
```

This should only take few minutes to provision the Digital Ocean droplet and set up a new Docker Machine called `prod`. While you wait, navigate to the [Digital Ocean Control Panel](https://cloud.digitalocean.com); you should see a new droplet being created, again, called `prod`.

Once done, there should now be two machines running, one locally (`dev`) and one on Digital Ocean (`prod`). Run `docker-machine ls` to confirm:

```sh
NAME   ACTIVE   DRIVER         STATE     URL                         SWARM   DOCKER   ERRORS
dev    *        virtualbox     Running   tcp://192.168.99.100:2376           v1.9.1
prod   -        digitalocean   Running   tcp://159.203.77.132:2376           v1.9.1
```

Set `prod` as the active machine and then load the Docker environment into the shell:

```sh
$ eval "$(docker-machine env prod)"
```

### Docker Compose (take 2)

Start by renaming *env.example* to *.env*. Update the `DJANGO_ALLOWED_HOSTS` variable to match the Digital Ocean IP address - i.e., `DJANGO_ALLOWED_HOSTS=159.203.77.132`. Keep the remaining defaults for now.

Now we can create the build and then fire up the services in the cloud:

```sh
$ docker-compose build
$ docker-compose up -d
```

### Sanity Check (take 2)

Apply all the migrations:

```sh
$ docker-compose run django python manage.py makemigrations
$ docker-compose run django python manage.py migrate
```

That's it!

Now just visit your server's IP address, associated with the Digital Ocean droplet, and view it in the browser.

You should be good to go.

<hr>

For further reference just grab the code from the [repository](https://github.com/realpython/django_cookiecutter_deploy). Thanks a lot for reading! Looking forward to your questions.
