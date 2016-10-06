# Development and Deployment of Django on Fedora

Last [time](https://realpython.com/blog/python/development-and-deployment-of-cookiecutter-django-via-docker/) we set up a Django Project using Cookiecutter, managed the application environment via Docker, and then deployed the app to Digital Ocean. **In this tutorial, we'll shift away from Docker and detail a development to deployment workflow of a Cookiecutter-Django Project on Fedora 24.**

<div class="center-text">
  <img class="no-border" src="/images/blog_images/cookiecutter-django-fedora.png" style="max-width: 100%;" alt="cookiecutter django fedora">
</div>

**Updates**:

- *10/06/2016*: Updated to the latest versions of Fedora (v[24](https://fedoraproject.org/wiki/Fedora_24_Final_Release_Criteria)), cookiecutter (v[1.4.0](https://github.com/audreyr/cookiecutter/releases/tag/1.4.0)), cookiecutter-django, and Django (v[1.10.1](https://docs.djangoproject.com/en/1.10/releases/1.10.1/)).

## Development

Install [cookiecutter](https://github.com/audreyr/cookiecutter) globally and then generate a bootstrapped Django project:

```sh
$ pip install cookiecutter==1.4.0
$ cookiecutter https://github.com/pydanny/cookiecutter-django.git
```

This command runs cookiecutter with the [cookiecutter-django](https://github.com/pydanny/cookiecutter-django) repo, allowing us to enter project-specific details. (We named the project and the repo *django_cookiecutter_fedora*.)

> **NOTE**: Check out the [Local Setup](https://realpython.com/blog/python/development-and-deployment-of-cookiecutter-django-via-docker/#local-setup) section from the previous post for more information on this command as well as the generated project structure.

Before we can start our Django Project, we are still left with few steps...

### Database Setup

First, we need to set up Postgres since cookiecutter-django uses it as its default database (see *[django_cookiecutter_fedora/config/settings/common.py](https://github.com/realpython/django_cookiecutter_fedora/blob/master/config/settings/common.py)* for more info). Follow the steps to set up a Postgres database server, from any [good resource](https://wiki.postgresql.org/wiki/Detailed_installation_guides) on the Internet or just scroll down to the [Deployment Section](#deployment) for setting it up on [Fedora](https://getfedora.org/).

> **NOTE**: If you're on a Mac, check out [Postgres.app](http://postgresapp.com/).

Once the Postgres server is running, create a new database from [psql](http://www.postgresql.org/docs/9.4/static/app-psql.html) that shares the same name as your project name:

```sh
$ create database django_cookiecutter_fedora;
CREATE DATABASE
```
> **NOTE**: There may be some variation in the above command for creating a database based upon your version of Postgres. You can check for the correct command in Postgres' latest documentation found [here](http://www.postgresql.org/docs/9.5/interactive/manage-ag-createdb.html).

### Dependency Setup

Next, in order to get your Django Project in a state ready for development, navigate to the root directory, create/activate a virtual environment, and then install the dependencies:

```sh
$ cd django_cookiecutter_fedora
$ pyvenv-3.5 .venv
$ source .venv/bin/activate
$ ./utility/install_python_dependencies.sh
```

## Sanity Check

Apply the migrations and then run the local development server:

```sh
$ python manage.py makemigrations
$ python manage.py migrate
$ python manage.py runserver
```

Ensure all is well by navigating to [http://localhost:8000/](http://localhost:8000/) in your browser to view the Project quick start page. Once done, kill the development server, initialize a new Git repo, commit, and PUSH to Github.

## Deployment

With the Project setup and running locally, we can now move on to deployment where we will utilize the following tools:

- [Fedora 24](https://fedoraproject.org/wiki/Fedora_24_Final_Release_Criteria)
- Postgres
- [Nginx](http://nginx.org/)
- [gunicorn](http://gunicorn.org/)

### Fedora 24 Setup

Set up a quick [Digital Ocean](https://www.digitalocean.com/?refcode=d8f211a4b4c2) droplet, making sure to use a Fedora 24 image. For help, follow [this](https://www.digitalocean.com/community/tutorials/how-to-create-your-first-digitalocean-droplet-virtual-server) tutorial. Make sure you [set up an SSH key](https://www.digitalocean.com/community/tutorials/how-to-use-ssh-keys-with-digitalocean-droplets) for secure login.

Now let's [update](https://fedoraproject.org/wiki/DNF_system_upgrade) our server. SSH into the server as root, and then fire the update process:

```sh
$ ssh root@SERVER_IP_ADDRESS
# dnf upgrade
```

### Non-Root User

Next, let's set up a non-root user so that applications are not run under administrative privileges, which makes the system more secure.

As a root user, follow these commands to set up a non-root user:

```sh
# adduser <name-of-user>
# passwd <name-of-user>
Changing password for user <name-of-user>.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

In the above snippet we created the new, non-root user user then specified a password for said user. We now need to [add this user to the administrative group](http://fedoraproject.org/wiki/Configuring_Sudo) so that they can run commands that require admin privileges with `sudo`:

```sh
# usermod <name-of-user> -a -G wheel
```

Exit from the server and log back in again as the non-root user. Did you notice that the shell prompt changed from a `#` (pound-sign) to `$` (dollar-sign)? This indicates that we are logged in as a non-root user.

### Required Packages

While logged in as the non-root user, download and install the following packages:

> **NOTE**: Below we are giving our non-root user root rights (recommended!). If by any chance you wish to not give the non-root user root rights explicitly, you will have to prepend `sudo` keyword with every command you execute in the terminal.

```sh
$ sudo su
# dnf install postgresql-server postgresql-contrib postgresql-devel
# dnf install python3-devel python-devel gcc nginx git
```

### Postgres Setup

With the dependencies downloaded and installed, we just need to set up our Postgres server and create a database.

Initialize Postgres and then manually start the server:

```sh
# sudo postgresql-setup initdb
# sudo systemctl start postgresql
```

Then log in to the Postgres server by switching (`su`) to the `postgres` user:

```sh
# sudo su - postgres
$ psql
postgres=#
```

Now create a Postgres user and database required for our project, making sure that the username matches the name of the non-root user:

```sh
postgres=# CREATE USER <user-name> WITH PASSWORD '<password-for-user>';
CREATE ROLE
postgres=# CREATE DATABASE django_cookiecutter_fedora;
CREATE DATABASE
postgres=# GRANT ALL ON DATABASE django_cookiecutter_fedora TO <user-name>;
GRANT
```

> **NOTE**: If you need help, please follow the [official guide](https://fedoraproject.org/wiki/PostgreSQL) for setting up Postgres on Fedora.

Exit psql and return to your non-root user's shell session:

```sh
postgres=# \q
$ exit
```

When you exit from the postgres session, you will return back to your non-root user prompt. `[username@django-cookiecutter-deploy ~]#`.

> **NOTE**: Did you notice the `#` sign at the prompt? This is now appearing because we gave our non-root user the root rights before starting off with setting up our server. If you do not see this, you either need to run `sudo su` again or prefix each command with `sudo`.

Configure Postgres so that it starts when the server boots/reboots:

```sh
# sudo systemctl enable postgresql
# sudo systemctl restart postgresql
```

### Project Setup

Clone the project structure from your GitHub repo to the */opt* directory:

```sh
# sudo git clone <github-repo-url> /opt/<name of the repo>
```

> **NOTE**: Want to use the repo associated with this tutorial? Simply run:
>
    sudo git clone https://github.com/realpython/django_cookiecutter_fedora /opt/django_cookiecutter_fedora

### Dependency Setup

Next, in order to get your Django Project in a state ready for deployment, create and active a virtualenv inside your project's root directory:

```sh
# cd /opt/django_cookiecutter_fedora/
# sudo pip3 install virtualenv
# sudo pyvenv-3.5 .venv
```

Before activating the virtualenv, give the current, non-root user admin rights (if not given):

```sh
$ sudo su
# source venv/bin/activate
```

Unlike with the set up of the development environment from above, before installing the dependencies we need to install all of [Pillow](https://pillow.readthedocs.org/en/3.0.x/index.html)'s external libraries. Check out this [resource](https://pillow.readthedocs.org/en/3.0.x/installation.html#basic-installation) for more info.

```sh
# dnf install libtiff-devel libjpeg-devel libzip-devel freetype-devel
# dnf install lcms2-devel libwebp-devel tcl-devel tk-devel
```

Next, we need to install one more packages in order to ensure that we do not get conflicts while installing dependencies in our virtualenv.

```sh
# dnf install redhat-rpm-config
```

Then run:

```sh
# ./utility/install_python_dependencies.sh
```

Again, this will install all the base, local, and production requirements. This is just for a quick sanity check to ensure all is working. Since this is technically the production environment, we will change the environment shortly.

> **NOTE**: Deactivate the virtualenv. Now if you issue an exit command - e.g., `exit` - the non-root user will no longer have root rights. Notice the change in prompt. That said, the user can still activate the virtualenv. Try it!

## Sanity Check (take 2)

Apply all the migrations:

```sh
$ python manage.py makemigrations
$ python manage.py migrate
```

Now run the server:

```sh
$ python manage.py runserver 0.0.0.0:8000
```

To ensure things are working fine, just visit the server's IP address in your browser - e.g., <ip-address or hostname>:8000.

## Gunicorn Setup

Before setting up Gunicorn, we need to make some changes to the production settings, in the */config/settings/production.py* module.

Take a look at the production settings [here](https://github.com/realpython/django_cookiecutter_fedora/blob/master/config/settings/production.py), which are the minimal settings needed for deploying our Django Project to a production server.

To update these, open the file in VI:

```sh
$ sudo vi config/settings/production.py
```

First, select all and delete:

```sh
:%d
```

And then copy the new settings and paste them into the now empty file by entering INSERT mode and then pasting. Make sure to update the `ALLOWED_HOSTS` variable as well (very, very important!) with your server's IP address or hostname. Exit INSERT mode and then save and exit:

```sh
:wq
```

Once done, we need to add some environment variables to the *.bashrc* file, since the majority of the config settings come from environment variables within the *production.py* file.

Again, use VI to edit this file:

```sh
$ vi ~/.bashrc
```

Enter INSERT mode and add the following:

```sh
# Environment Variables
export DJANGO_SETTINGS_MODULE='config.settings.production'

export DJANGO_SECRET_KEY='CHANGEME!!!m_-0-_)3&-(+hn$!1i3$*$ujru4yw4@!u7048_(#1a*y_g2v3r'

export DATABASE_URL='postgres:///django_cookiecutter_fedora'
```

Two things to note:

1. Notice how we updated the `DJANGO_SETTINGS_MODULE` variable to use the production settings.
1. It's a good [practice](https://www.quora.com/What-is-the-SECRET_KEY-in-Django-actually-used-for) to change your `DJANGO_SECRET_KEY` to a [more complex](http://stackoverflow.com/questions/15170637/effects-of-changing-djangos-secret-key) string. Do this now if you want.

Again, exit INSERT mode, and then save and exit VI.

Now just reload the *.bashrc* file:

```sh
$ source ~/.bashrc
```

Ready to test?! Inside the root directory, with the virtualenv activated, execute the gunicorn server:

```sh
$ gunicorn --bind <ip-address or hostname>:8000 config.wsgi:application
```

This will make our web application, again, serve on <ip-address or hostname>:8000.

Keep in mind that as soon as we log out of our server this command will stop and hence we would no longer be able to serve our web app. So, we have to make our gunicorn server execute as a service so that it can be started, stopped, and monitored.

## Nginx Config

Follow these steps for adding a configuration file for making our Django Project serve via Nginx:

```sh
$ cd /etc/nginx/conf.d
$ sudo vi django_cookiecutter_fedora.conf
```

Add the following, making sure to update the `server`, `server_name`, and `location`:

```sh
upstream app_server {
    server 192.241.166.11:8000 fail_timeout=0;
}

server {
    listen 80;
    server_name 192.241.166.11;
    access_log /var/log/nginx/django_project-access.log;
    error_log /var/log/nginx/django_project-error.log info;

    keepalive_timeout 5;

    # path for staticfiles
    location /static {
            autoindex on;
            alias /opt/django_cookiecutter_fedora/staticfiles/;
    }

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;

        if (!-f $request_filename) {
            proxy_pass http://app_server;
            break;
        }
    }
}
```

Save and exit VI, and then restart the Nginx server:

```sh
$ sudo systemctl restart nginx.service
```

That's it!

## Gunicorn Start Script

Now let's create a Gunicorn start script which will run as an executable, making our bootstrapped Django web application run via the Gunicorn server, routed through Nginx.

Within the project root, run:

```sh
$ sudo mkdir deploy log
$ cd deploy
$ sudo vi gunicorn_start
```

The contents of the *gunicorn_start* script can be found [here](https://github.com/realpython/django_cookiecutter_fedora/blob/master/deploy/gunicorn_start). It is divided into 3 significant parts, which are, for the most part, self-explanatory. For any questions, please comment below.

> **NOTE**: Make sure the `USER` and `GROUP` variables match the same user and group for the non-root user.

Paste the contents into VI, and then save and exit.

Finally, let's make it executable:

```sh
$ sudo chmod +x gunicorn_start
```

Start the server:

```sh
$ sudo ./gunicorn_start
```

Once again visit the your server's IP address in the browser and you will see your Django web app running!

Did you get a 502 Bad gateway error? Just follow these steps and it will probably be enough to make your application work...

### Modifying SELinux Policy Rules

```sh
$ sudo dnf install policycoreutils-devel
$ sudo cat /var/log/audit/audit.log | grep nginx | grep denied | audit2allow -M mynginx
$ sudo semodule -i mynginx.pp
```

Once done, make sure to restart your server via Digital Ocean's dashboard utility.

## Systemd

To make our *gunicorn_start* script run as a system service so that, even if we are no longer logged into the server, it still serves our Django web application, we need to create a [systemd](https://fedoraproject.org/wiki/Systemd) service.

Just change the working directory to */etc/systemd/system*, and then create a service file:

```sh
$ cd /etc/systemd/system
$ sudo vi django-bootstrap.service
```

Add the following:


```sh
#!/bin/sh

[Unit]
Description=Django Web App
After=network.target

[Service]
PIDFile=/var/run/cric.pid
ExecStart=/opt/django_cookiecutter_fedora/deploy/gunicorn_start
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

Save and exit, and then start the service and enable it:

```sh
$ sudo systemctl start django-bootstrap.service
```

> **NOTE**: If you encounter an error, run `journalctl -xe` to see more details.

Finally, enable the service so that it runs forever and restarts on arbitrary shut downs:

```sh
$ sudo systemctl enable django-bootstrap.service
```

## Sanity Check (final!)

Check for the status of the service:

```sh
$ sudo systemctl status django-bootstrap.service
```

Now just visit your server's IP address (or hostname) and you will see a Django error page. To fix this, run the following command in your project's root directory (with your virtualenv activated):

```sh
$ python manage.py collectstatic
```

Now you are good to go, and you will see the Django web app running on the web browser with all the static files (HTML/CSS/JS) files working.

<hr>

For further reference grab the code from the [repository](https://github.com/realpython/django_cookiecutter_fedora/). Add your questions/comments/concerns below. Cheers!
