---
layout: post
title: "Docker in Action - fitter, happier, more productive"
date: 2015-02-10 07:28:20 -0700
toc: true
comments: true
category_side_bar: true
categories: [python, devops, docker]

keywords: "python, docker, web development, flask, continuous integration, docker compose, fig"
description: "Let's look at how to set up your local development environment with Docker as well as continuous integration, step by step."
---

<div class="center-text">
  <img class="no-border" src="/images/blog_images/docker-in-action/docker-in-action.png" style="max-width: 100%;" alt="docker in action">
</div>

<br>

With Docker you can easily deploy a web application along with it's dependencies, environment variables, and configuration settings - everything you need to recreate your environment quickly and efficiently.

This tutorial looks at just that.

**Updated 02/28/2015**: Added [Docker Compose](https://docs.docker.com/compose/) and upgraded Docker and boot2docker to the latest versions.

<hr>

**We'll start by creating a Docker container for running a Python Flask application. From there, we'll look at a nice development workflow to manage the local development of an app as well as continuous integration and delivery, step by step ...**

> I ([Michael Herman](https://twitter.com/mikeherman)) originally presented this workflow at [ PyTennessee](https://www.pytennessee.org/) on February 8th, 2015. You can view the slides [here](http://realpython.github.io/fitter-happier-docker/), if interested.

## Workflow

1. Code locally on a feature branch
1. Open a pull request on Github against the master branch
1. Run automated tests against the Docker container
1. If the tests pass, manually merge the pull request into master
1. Once merged, the automated tests run again
1. If the second round of tests pass, a build is created on Docker Hub
1. Once the build is created, it's then automatically (err, automagically) deployed to production

<div class="center-text">
  <img class="no-border" src="/images/blog_images/docker-in-action/steps.jpg" style="max-width: 100%;" alt="docker in action workflow">
</div>

<br>

> This tutorial is meant for Mac OS X users, and we'll be utilizing the following tools/technologies - Python v2.7.9, Flask v0.10.1, Docker v1.5.0, Docker Compose, v1.1.0, boot2docker 1.5.0, Redis v2.8.19

Let's get to it...

<hr>

First, some Docker-specific terms:

- A *Dockerfile* is a file that contains a set of instructions used to create an *image*.
- An *image* is used to build and save snapshots (the state) of an environment.
- A *container* is an instantiated, live *image* that runs a collection of processes.

> Be sure to check out the Docker [documentation](https://docs.docker.com/) for more info on [Dockerfiles](https://docs.docker.com/reference/builder/), [images](https://docs.docker.com/terms/image/), and [containers](https://docs.docker.com/terms/container/).

## Why Docker?

You can truly mimic your production environment on your local machine. No more having to debug environment specific bugs or worrying that your app will perform differently in production.

1. Version control for infrastructure
1. Easily distribute/recreate your entire development environment
1. Build once, run anywhere - aka The Holy Grail!

## Docker Setup

Since Darwin (the kernel for OS X) does not have the Linux kernel features required to run Docker containers, we need to install [boot2docker](http://boot2docker.io/) - which is a *lightweight*t Linux distribution designed specifically to run Docker. In essence, it starts a small VM that's configured to run Docker containers.

Create a new directory called "fitter-happier-docker" to house your Flask project.

Next follow the instructions from the guide [Installing Docker on Mac OS X](https://docs.docker.com/installation/mac/) to install both Docker and the official boot2docker package.

Test the install:

```sh
$ boot2docker version
Boot2Docker-cli version: v1.5.0
Git commit: ccd9032
```

## Compose Up!

[Docker Compose](http://docs.docker.com/compose/) is an orchestration framework that handles the building and running of multiple services (via separate containers) using a simple *.yml* file. It makes it super easy to link services together running in different containers.

> Following along with me? Grab the code in a pre-Compose state from the [repository](https://github.com/realpython/fitter-happier-docker/releases/tag/pre-compose).

Start by installing the requirements via pip and then make sure Compose is installed:

```sh
$ pip install docker-compose
$ docker-compose --version
docker-compose 1.1.0
```

Now let's get our Flask application up and running along with Redis.

Add a new file called *docker-compose.yml* to the root directory:

```yaml
web:
    build: web
    volumes:
        - web:/code
    ports:
        - "80:5000"
    links:
        - redis
    command: python app.py
redis:
    image: redis:2.8.19
    ports:
        - "6379:6379"
```

Here we add the services that make up our stack:

1. **web**: First, we build the image from the "web" directory and then mount that directory to the "code" directory within the Docker container. The Flask app is ran via the `python app.py` command. This exposes port 5000 on the container, which is forwarded to port 80 on the host environment.
1. **redis**: Next, the Redis service is built from the Docker Hub "Redis" [image](https://registry.hub.docker.com/_/redis/). Port 6379 is exposed and forwarded.

Did you notice the Dockerfile in the "web" directory? This file is used to build our image, starting with an Ubuntu base, the required dependencies are installed and the app is built.

## Build and run

With one simple command we can build the image and run the container:

```sh
$ docker-compose up
```

<div class="center-text">
  <img class="no-border" src="/images/blog_images/docker-in-action/figup.png" alt="fig up" style="max-width: 100%;">
</div>

<br>

This command builds an image for our Flask app, pulls the Redis image, and then starts everything up.

Grab a cup of coffee. Or two. This will take some time the first time you build the container. That said, since Docker caches each step (or *[layer](https://docs.docker.com/terms/layer/)*) of the build process from the Dockerfile, rebuilding will happen *much* quicker because only the steps that have *changed* since the last build are rebuilt.

> If you do change a line/step/layer in your Dockerfile, it will recreate/rebuild everything in that line - so be mindful of this when you structure your Dockerfile.

Docker Compose brings each container up at once in parallel. Each container also has a unique name and each process within the stack trace/log is color-coded for readability.

Ready to test?

Open your web browser and navigate to the IP address associated with the `DOCKER_HOST` variable - i.e., http://192.168.59.103/, in this example. (Run `boot2docker ip` to get the address.)

You should see the text, "Hello! This page has been seen 1 times." in your browser:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/docker-in-action/test.png" style="border: 1px solid black" alt="test flask app running on docker">
</div>

<br>

Refresh. The page counter should have incremented.

Kill the processes (Ctrl-C), and then run the following command to run the process in the background.

```sh
$ docker-compose up -d
```

Want to view the currently running processes?

```sh
$ docker-compose ps
            Name                          Command             State              Ports
--------------------------------------------------------------------------------------------------
fitterhappierdocker_redis_1     /entrypoint.sh redis-server   Up      0.0.0.0:6379->6379/tcp
fitterhappierdocker_web_1       python app.py                 Up      0.0.0.0:80->5000/tcp, 80/tcp
```

> Both processes are running in a different container, connected via Docker Compose!

## Next Steps

Once done, kill the processes via `docker-compose stop`, then run `boot2docker down` to gracefully shutdown the VM. Commit your changes locally, and then push to Github.

So, what did we accomplish?

We set up our local environment, detailing the basic process of building an *image* from a *Dockerfile* and then creating an instance of the *image* called a *container*. We tied everything together with Docker Compose to build and connect different containers for both the Flask app and Redis process.

**Now, let's look at a nice continuous integration workflow powered by [CircleCI](https://circleci.com/)**.

> Still with me? You can grab the updated code from the [repository](https://github.com/realpython/fitter-happier-docker/releases/tag/added-compose).

## Docker Hub

Thus far we've worked with Dockerfiles, images, and containers (abstracted by Docker Compose, of course).

Are you familiar with the Git workflow? Images are like Git repositories while containers are similar to a cloned repository. Sticking with that metaphor, [Docker Hub](https://hub.docker.com/), which is repository of Docker images, is akin to Github.

1. Signup [here](https://hub.docker.com/account/signup/), using your Github credentials.
1. Then add a new automated build. And add your Github repo that you created and pushed to. Just accept all the default options, except for the "Dockerfile Location" - change this to "/web".

Once added, this will trigger an initial build. Make sure the build is successful.

### Docker Hub for CI

Docker Hub, in itself, acts as a continuous integration server since you can configure it to create an [automated build](https://docs.docker.com/userguide/dockerrepos/#automated-builds) every time you push a new commit to Github. In other words, it ensures you do not cause a regression that completely breaks the build process when the code base is updated.

> There are some drawbacks to this approach - namely that you cannot push (via `docker push`) updated images directly to Docker Hub. Docker Hub must pull in changes from your repo and create the images itself to ensure that their are no errors. Keep this in mind as you go through this workflow. The Docker documentation is not clear with regard to this matter.

Let's test this out. Add an assert to the test suite:

```python
self.assertNotEqual(four, 102)
```

Commit and push to Github to generate a new build on Docker Hub. Success?

**Bottom-line:** It's good to know that if a commit does cause a regression that Docker Hub will catch it, but since this is the last line of defense before deploying (to either staging or production) you ideally want to catch any breaks before generating a new build on Docker Hub. Plus, you also want to run your unit and integration tests from a *true* continuous integration server - which is exactly where CircleCI comes into play.

## CircleCI

<div class="center-text">
  <img class="no-border" src="/images/blog_images/docker-in-action/circleci.png" alt="circleci">
</div>

<br>

[CircleCI](https://circleci.com/) is a continuous integration and delivery platform that supports testing within Docker containers. Given a Dockerfile, CircleCI builds an image, starts a new container, and then runs tests inside that container.

Remember the workflow we want? [Link](https://realpython.com/blog/python/docker-in-action-fitter-happier-more-productive/#workflow).

Let's take a look at how to achieve just that...

### Setup

The best place to start is the excellent [Getting started with CircleCI](https://circleci.com/docs/getting-started) guide...

Sign up with your Github account, then add the Github repo to create a new project. This will automatically add a webhook to the repo so that anytime you push to Github a new build is triggered. You should receive an email once the hook is added.

Next we need to add a configuration file to the root folder of repo so that CircleCI can properly create the build.

### *circle.yml*

Add the following build commands/steps:

```yaml
machine:
  services:
    - docker

dependencies:
  override:
    - pip install -r requirements.txt

test:
  override:
    - docker-compose run -d --no-deps web
    - python web/tests.py
```
Essentially, we create a new image, run the container, then run your unit tests.

> Notice how we're using the command `docker-compose run -d --no-deps web`, to run the web process, instead of `docker-compose up`. This is because CircleCI already has Redis [running](https://circleci.com/docs/environment#databases) and available to us for our tests. So, we just need to run the web process.

With the *circle.yml* file created, push the changes to Github to trigger a new build. *Remember: this will also trigger a new build on Docker Hub.*

Success?

Before moving on, we need to change our workflow since we won't be pushing directly to the master branch anymore.

### Feature Branch Workflow

> For these unfamiliar with the Feature Branch workflow, check out [this](https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow) excellent introduction.

Let's run through a quick example...

### Create the feature branch

```sh
$ git checkout -b circle-test master
Switched to a new branch 'circle-test'
```

### Update the app

Add a new assert in *tests.py*:

```python
self.assertNotEqual(four, 60)
```

### Issue a Pull Request

```sh
$ git add web/tests.py
$ git commit -m "circle-test"
$ git push origin circle-test
```

Even before you create the actual pull request, CircleCI starts creating the build. Go ahead and create the pull request, then once the tests pass on CircleCI, press the Merge button. Once merged, the build is triggered on Docker Hub.

### Refactoring the workflow

If you jump back to the overall workflow at the [top of this post](https://realpython.com/blog/python/docker-in-action-fitter-happier-more-productive/#workflow), you'll see that we don't actually want to trigger a new build on Docker Hub until the tests pass against the master branch. So, let's make some quick changes to the workflow:

1. Open your repository on Docker Hub, and then under *Settings* click *Automated Build*.
1. Uncheck the Active box: "When active we will build when new pushes occur".
1. Save.
1. Click *Build Triggers* under *Settings*
1. Change the status to on.
1. Copy the example curl command - i.e., `$ curl --data "build=true" -X POST https://registry.hub.docker.com/u/mjhea0/fitter-happier-docker/trigger/84957124-2b85-410d-b602-b48193853b66/`

Now add the following code to the bottom of your *circle.yml* file:

```yaml
deployment:
  hub:
    branch: master
    commands:
      - $DEPLOY
```

Here we fire the `$DEPLOY` variable *after* we merge to master *and* the tests pass. We'll add the actual value of this variable as an environment variable on CircleCI:

1. Open up the *Project Settings*, and click *Environment variables*.
1. Add a new variable with the name "DEPLOY" and paste the example curl command (that you copied) from Docker Hub as the value.

Now let's test.

```sh
$ git add circle.yml
$ git commit -m "circle-test"
$ git push origin circle-test
```

Open a new pull request, and then once the tests pass on Circle CI, merge to master. Another build is trigged. Then once the tests pass again, a new build will be triggered on Docker Hub via the curl command. Nice.

> Remember how I said that I configured Docker Hub to pull updated code to create a new image? Well, you could also set it up to where you can push images directly to Docker Hub. So once these test pass, you could simply push the image to update Docker Hub and then deploy to staging or production, directly from CircleCI. Since I have it set up differently, I handle the push to production from Docker Hub, not CircleCI. There's positives and negatives to both approaches, as you will soon find out.

## Conclusion

So, we went over a nice development workflow that included setting up a local environment coupled with continuous integration via [CircleCI](https://circleci.com/) (steps 1 through 6):

1. Code locally on a feature branch
1. Open a pull request on Github against the master branch
1. Run automated tests against the Docker container
1. If the tests pass, manually merge the pull request into master
1. Once merged, the automated tests run again
1. If the second round of tests pass, a build is created on Docker Hub
1. Once the build is created, it's then automatically (err, automagically) deployed to production

What about the final piece - delivering this app to the production environment (step 7)? You can actually follow another one of my Docker blog [posts](https://blog.rainforestqa.com/2015-01-15-docker-in-action-from-deployment-to-delivery-part-3-continuous-delivery) to extend this workflow to include delivery.

Comment below if you have questions. Grab the final code [here](https://github.com/realpython/fitter-happier-docker/releases/tag/final-code). Cheers!

<hr>

**If you have a workflow of your own, please let us know. I am currently experimenting with [Salt](https://github.com/saltstack/salt) as well as [Tutum](https://www.tutum.co/) to better handle orchestration and delivery on Digital Ocean and Linode.**