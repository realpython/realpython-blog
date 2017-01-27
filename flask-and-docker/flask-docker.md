# Flask + Docker

**This tutorial details how to package and run a Flask application environment with [Docker](https://www.docker.com/). We'll also look at how deploy the application to Digital Ocean.**

MICHAEL: ADD IMAGE

## Objectives

MICHAEL: UPDATE OBJECTIVES

By the end of this tutorial, you will be able to...

1. Create a base Docker image to host a Flask app with a Dockerfile
1. Configure a database container for storing data
1. Link multiple Docker containers together with Docker Compose
1. Use Nginx to serve static assets

## Docker

Put simply, Docker is a powerful tool for packaging, shipping, and running applications within isolated environment containers. You can package together everything you need to run an application, including the code, language runtime, and system libraries.

If this is your first time using Docker, start with the [official tutorial](https://docs.docker.com/engine/getstarted/) to install Docker and learn the basic concepts and commands.  

This tutorial uses the following Docker versions:

```sh
$ docker -v
Docker version 1.13.0, build 49bf474
$ docker-compose -v
docker-compose version 1.10.0, build 4bd6f1a
$ docker-machine -v
docker-machine version 0.9.0, build 15fd4c7
```

## Project Setup

MICHAEL: UPDATE SETUP

Start by cloning down the base project:

```sh
$ foo bar
```

Your project folder should look like:

```sh
├── nginx
│   ├── Dockerfile
│   └── sites-enabled
│       └── flask_project
└── web
    ├── app.py
    ├── config.py
    ├── create_db.py
    ├── models.py
    ├── requirements.txt
    ├── static
    │   ├── css
    │   │   ├── bootstrap.min.css
    │   │   └── main.css
    │   ├── img
    │   └── js
    │       ├── bootstrap.min.js
    │       └── main.js
    └── templates
        ├── _base.html
        └── index.html
```

We're now ready package and test our app out locally with Docker.

## Docker Setup

MICHAEL: ADD BASIC DOCKER OVERVIEW

### Dockerfile

A [Dockerfile](https://docs.docker.com/engine/reference/builder/) is simply a blueprint that Docker uses to build a container [image](https://docs.docker.com/engine/getstarted/step_two/). Within the "web" directory, add a new file called *Dockerfile*:

```
FROM python:3.6

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

COPY requirements.txt /usr/src/app/
RUN pip install --no-cache-dir -r requirements.txt

VOLUME /usr/src/app
```

Here, the Python environment is setup, requirements are installed, and a [volume](https://docs.docker.com/engine/reference/builder/#/volume) is defined, which will be used to mount the source code into the container.

### Docker Compose

[Docker Compose](https://docs.docker.com/compose/) is used for spinning up a multi-container environment. Add *docker-compose.yml* to the project root:

```
version: '2'
services:
  web:
    restart: always
    build: ./web
    expose:
      - "8000"
    links:
      - postgres:postgres
    volumes:
      - ./web:/usr/src/app
    env_file: .env
    command: /usr/local/bin/gunicorn -w 2 -b :8000 app:app

  nginx:
    restart: always
    build: ./nginx/
    ports:
      - "80:80"
    volumes:
      - ./web/static:/www/static
    volumes_from:
      - web
    links:
      - web:web

  data:
    restart: always
    image: postgres:latest
    volumes:
      - /var/lib/postgresql
    command: "true"

  postgres:
    restart: always
    image: postgres:latest
    volumes_from:
      - data
    ports:
      - "5432:5432"

  pgweb:
    container_name: pgweb
    restart: always
    image: sosedoff/pgweb
    ports:
      - "8081:8081"
    links:
      - postgres:postgres
    environment:
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/postgres?sslmode=disable
    depends_on:
      - postgres
```

This will create and link five services - *web*, *nginx*, *postgres*, *data*, and *pgweb*:

1. *web* is built via the instructions in the *Dockerfile*. The "web" directory is mounted in the container, allowing us to modify the code without having to rebuild the image. The app itself runs on port 800, which is then forwarded to port 80 on the host environment. This service also adds environment variables to the container that are defined in the *.env* file, which we still need to add.
1. *nginx*: is used for reverse proxy to forward requests either to the Flask app or the static asset files. Review the "nginx" folder for more details.
1. *postgres* is built from the the official [PostgreSQL image](https://registry.hub.docker.com/_/postgres/) from [Docker Hub](https://hub.docker.com/), which installs Postgres and runs the server on the default port, 5432.
1. *data*: spins up a separate container that's used to store the database data. This helps ensure that the data persists even if the Postgres container is completely destroyed.

ADD INFO ABOUT PGWEB

Add a new file called *.env* to the root, to hold environment variables:

```
DEBUG=False
SECRET_KEY=changeme
DB_NAME=postgres
DB_USER=postgres
DB_PASS=postgres
DB_SERVICE=postgres
DB_PORT=5432
```

With that, we are now ready to test this configuration locally.

## Docker Machine

[Docker Machine](https://docs.docker.com/machine/) is used for creating Docker hosts both locally and in the cloud. To start a new machine, first make sure you're in the project root and then simply run:

```sh
$ docker-machine create -d virtualbox dev;
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env dev
```

The `create` command setup a "machine" (called *dev*) for Docker development. In essence, it downloaded boot2docker (which is necessary on a Mac environment) and started a VM with Docker running. Now just point the Docker client at the *dev* machine:

```sh
$ docker-machine env dev
$ eval "$(docker-machine env dev)"
```

Run the following command to view the currently running Machines:

```sh
$ docker-machine ls
NAME   ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER    ERRORS
dev    *        virtualbox   Running   tcp://192.168.99.100:2376           v1.13.0
```

Next, let's fire up the containers with Docker Compose.

## Docker Compose

To get the containers running, build the images and then start the services:

```sh
$ docker-compose build
$ docker-compose up -d
```

Grab a cup of coffee. Or two. Follow us [@RealPython](https://twitter.com/RealPython) on Twitter. This will take a while the first time you run the build.

We also need to create the database table:

```sh
$ docker-compose run web /usr/local/bin/python create_db.py
```

Open your browser and navigate to the IP address associated with Docker Machine (`docker-machine ip dev`):

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask-docker-compose-machine/flask_app_docker.png" style="max-width: 100%;" alt="flask running on docker">
</div>

<br>

Nice!

To see which environment variables are available to the *web* service, run:

```sh
$ docker-compose run web env
```

To view the logs:

```sh
$ docker-compose logs
```

You can also enter the Postgres Shell - since we forward the port to the host environment in the *docker-compose.yml* file - to add users/roles as well as databases via:

```sh
$ psql -h 192.168.99.100 -p 5432 -U postgres --password
```

Once done, stop the processes via `docker-compose stop`.
