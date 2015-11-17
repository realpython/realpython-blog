# Dockerizing Flask with Compose and Machine - from localhost to the cloud

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask-docker-compose-machine/flask-docker.png" style="max-width: 100%;" alt="main logo">
</div>

<br>

[Docker](https://www.docker.com/) is a powerful tool for spinning up isolated, reproducible application environment *containers*. **This piece looks at just that - how to containerize a Flask app for local development along with delivering the application to a cloud hosting provider via Docker Compose and Docker Machine.**

<br>

*Updates:*

- 11/16/2015: Updated to the latest versions of Docker - Docker client (v1.9.0), Docker compose (v1.5.0), and Docker Machine (v0.5.0)
- 04/25/2015: Fixed small typo, and updated the *docker-compose.yml* file to properly copy static files.
- 04/19/2015: Added a shell script for copying static files.

> Interested in creating a similar environment for Django? Check out [this](https://realpython.com/blog/python/django-development-with-docker-compose-and-machine/) blog post.

## Local Setup

Along with Docker (v1.9.0) we'll be using -

- *[Docker Compose](https://docs.docker.com/compose/)* (v1.5.0) - previously known as fig - for orchestrating a multi-container application into a single app, and
- *[Docker Machine](https://docs.docker.com/machine/)* (v0.5.0) for creating Docker hosts both locally and in the cloud.

Follow the directions [here](http://docs.docker.com/compose/install/) and [here](https://docs.docker.com/machine/#installation) to install Docker Compose and Machine, respectively.

> Running either Mac OS x or Windows, then your best bet is to install [Docker Toolbox](https://www.docker.com/docker-toolbox).

Test out the installs:

```sh
$ docker-machine --version
docker-machine version 0.5.0 (04cfa58)
$ docker-compose --version
docker-compose version: 1.5.0
```

Next clone the project from the [repository](https://github.com/realpython/orchestrating-docker) or create your own project based on the project structure found on the repo:

```sh
├── docker-compose.yml
├── nginx
│   ├── Dockerfile
│   └── sites-enabled
│       └── flask_project
└── web
    ├── Dockerfile
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

We're now ready to get the containers up and running. Enter Docker Machine.

## Docker Machine

To start Docker Machine, first make sure you're in the project root and then simply run:

```sh
$ docker-machine create -d virtualbox dev;
Running pre-create checks...
Creating machine...
Waiting for machine to be running, this may take a few minutes...
Machine is running, waiting for SSH to be available...
Detecting operating system of created instance...
Provisioning created instance...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
To see how to connect Docker to this machine, run: docker-machine env dev
```

The `create` command setup a "machine" (called *dev*) for Docker development. In essence, it downloaded boot2docker and started a VM with Docker running. Now just point the Docker client at the *dev* machine via:

```sh
$ eval "$(docker-machine env dev)"
```

Run the following command to view the currently running Machines:

```sh
$ docker-machine ls
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM
dev       *        virtualbox   Running   tcp://192.168.99.100:2376
```

Next, let's fire up the containers with Docker Compose and get the Flask app and Postgres database up and running.

## Docker Compose

Take a look at the *docker-compose.yml* file:

```yaml
web:
  restart: always
  build: ./web
  expose:
    - "8000"
  links:
    - postgres:postgres
  volumes:
    - /usr/src/app/static
  env_file: .env
  command: /usr/local/bin/gunicorn -w 2 -b :8000 app:app

nginx:
  restart: always
  build: ./nginx/
  ports:
    - "80:80"
  volumes:
    - /www/static
  volumes_from:
    - web
  links:
    - web:web

data:
  restart: no
  image: postgres:latest
  volumes:
    - /var/lib/postgresql
  command: true

postgres:
  restart: always
  image: postgres:latest
  volumes_from:
    - data
  ports:
    - "5432:5432"
```

Here, we're defining four services - *web*, *nginx*, *postgres*, and *data*.

1. First, the *web* service is built via the instructions in the *Dockerfile* within the "web" directory - where the Python environment is setup, requirements are installed, and the Flask app is fired up on port 8000. That port is then forwarded to port 80 on the host environment - e.g., the Docker Machine. This service also adds environment variables to the container that are defined in the *.env* file.
1. The *nginx* service is used for reverse proxy to forward requests either to the Flask app or the static files.
1. Next, the *postgres* service is built from the the official [PostgreSQL image](https://registry.hub.docker.com/_/postgres/) from [Docker Hub](https://hub.docker.com/), which install Postgres and runs the server on the default port 5432.
1. Finally, notice how there is a separate [volume](https://docs.docker.com/userguide/dockervolumes/) container that's used to store the database data, *data*. This helps ensure that the data persists even if the Postgres container is completely destroyed.

Now, to get the containers running, build the images and then start the services:

```sh
$ docker-compose build
$ docker-compose up -d
```

Grab a cup of coffee. Or two. Check out the [Real Python courses](https://realpython.com/courses/). This will take a while the first time you run it.

We also need to create the database table:

```sh
$ docker-compose run web /usr/local/bin/python create_db.py
```

Open your browser and navigate to the IP address associated with Docker Machine (`docker-machine ip`):

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

## Deployment

So, with our app running locally, we can now push this exact same environment to a cloud hosting provider with Docker Machine. Let's deploy to a [Digital Ocean](https://www.digitalocean.com/?refcode=d8f211a4b4c2) droplet.

After you [sign up](https://www.digitalocean.com/?refcode=d8f211a4b4c2) for Digital Ocean, generate a [Personal Access Token](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2), and then run the following command:

```sh
$ docker-machine create \
-d digitalocean \
--digitalocean-access-token=ADD_YOUR_TOKEN_HERE \
production
```

a626d7b46e92ef3e6796245ef5a357e71edcc9369b2ca193cbda695d2e8d0ddf

This will take a few minutes to provision the droplet and setup a new Docker Machine called *production*:

```sh
Running pre-create checks...
Creating machine...
Waiting for machine to be running, this may take a few minutes...
Machine is running, waiting for SSH to be available...
Detecting operating system of created instance...
Provisioning created instance...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
To see how to connect Docker to this machine, run: docker-machine env production
```

Now we have two Machines running, one locally and one on Digital Ocean:

```sh
$ docker-machine ls
NAME         ACTIVE   DRIVER         STATE     URL                         SWARM
dev          *        virtualbox     Running   tcp://192.168.99.100:2376
production   -        digitalocean   Running   tcp://104.131.93.156:2376
```

Then set *production* as the active machine and load the Docker environment into the shell:

```sh
$ eval "$(docker-machine env production)"
```

Finally, let's build the Flask app again in the cloud:

```sh
$ docker-compose build
$ docker-compose up -d
$ docker-compose run web /usr/local/bin/python create_db.py
```

Grab the IP address associated with that Digital Ocean account from the [control panel](https://cloud.digitalocean.com/droplets) and view it in the browser. If all went well, you should see your app running.

## Conclusion

Cheers!

- Grab the code from the [Github repo](https://github.com/realpython/orchestrating-docker) (star it too!).
- Comment below with questions.
- Next time we'll extend this workflow to include two more Docker containers running the Flask app and incorporate load balancing into the mix. Stay tuned!
