---
layout: post
title: "Automating Django Deployments with Fabric and Ansible"
date: 2016-12-28 07:18:21 -0700
toc: true
comments: true
category_side_bar: true
categories: [python, devops, django]

keywords: "python, fedora, ansible, fabric, web development, deployment, django, digital ocean, cookiecutter, provision"
description: "In this tutorial we'll detail how to provision a server for a Django project with Ansible."
---

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-fabric-ansible/django-ansible.png" style="max-width: 100%;" alt="Django + Ansible">
</div>

<br>

In the [last](https://realpython.com/blog/python/development-and-deployment-of-cookiecutter-django-on-fedora/) post, we covered all the steps required to successfully develop and deploy a Django app on a single server. **In this tutorial we will automate the deployment process with [Fabric](http://www.fabfile.org/) (v[1.12.0](http://docs.fabfile.org/en/1.12/)) and [Ansible](https://github.com/ansible/ansible) (v[2.1.3](https://github.com/ansible/ansible/releases/tag/v2.1.3.0-1))** to address these issues:

1. **Scaling**: When it comes to scaling a web app to handle thousands of daily requests, relying on a single server is not a good approach. Put simply, as the server approaches maximum CPU utilization, it can cause slow load times which can eventually lead to server failure. To overcome this, the app must be scaled to run on more than one server so that the servers can *cumulatively* handle the incoming concurrent requests.
1. **Redundancy**: Deploying a web app manually to a new server means a lot of repeated work with more chances of human error. Automating the process is key.

Specifically, we will automate:

1. Adding a new, non-root user
1. Configuring the server
1. Pulling the Django app code from a GitHub repo
1. Installing the dependencies
1. Daemonizing the app

## Setup and Config

Start by spinning up a new [Digital Ocean](https://www.digitalocean.com/?refcode=d8f211a4b4c2) droplet, making sure to use the Fedora 25 image. Do not set up a pre-configured SSH key; we will be automating this process later via a Fabric script. Since the deployment process should be scalable, create a separate repository to house all the deployment scripts. Make a new project directory locally, and create and activate a virtualenv using Python 2.7x.

> Why Python 2.7? Fabric does NOT [support](http://www.fabfile.org/roadmap.html) Python 3. Don't worry: We'll be using Python 3.5 when we provision the server.

```sh
$ mkdir automated-deployments
$ cd automated-deployments
$ virtualenv env
$ source env/bin/activate
```

## Fabric Setup

Fabric is a tool used for automating routine shell commands over SSH, which we will be using to:

1. Set up the SSH keys
1. [Harden](<https://en.wikipedia.org/wiki/Hardening_(computing%29)>) user passwords
1. Install Ansible dependencies
1. Upgrade the server

Start by installing Fabric:

```sh
$ pip install fabric==1.12.0
```

Create a new folder called "prod", and add a new file called *fabfile.py* to it to hold all of the Fabric scripts:

```python
# prod/fabfile.py


import os
from fabric.contrib.files import sed
from fabric.api import env, local, run
from fabric.api import env

# initialize the base directory
abs_dir_path = os.path.dirname(
    os.path.dirname(os.path.abspath(__file__)))


# declare environment global variables

# root user
env.user = 'root'

# list of remote IP addresses
env.hosts = ['<remote-server-ip>']

# password for the remote server
env.password = '<remote-server-password>'

# full name of the user
env.full_name_user = '<your-name>'

# user group
env.user_group = 'deployers'

# user for the above group
env.user_name = 'deployer'

# ssh key path
env.ssh_keys_dir = os.path.join(abs_dir_path, 'ssh-keys')
```

Take note of the inline comments. Be sure to add you remote server's IP address to the `env.hosts` variable. Update `env.full_name_user` as well. Hold off on updating `env.password`; we will get to that shortly. Look over all the `env` variables - they are completely customizable based on your system setup.

### Set up the SSH keys

Add the following code to *fabfile.py*:

```python
def start_provision():
    """
    Start server provisioning
    """
    # Create a new directory for a new remote server
    env.ssh_keys_name = os.path.join(
        env.ssh_keys_dir, env.host_string + '_prod_key')
    local('ssh-keygen -t rsa -b 2048 -f {0}'.format(env.ssh_keys_name))
    local('cp {0} {1}/authorized_keys'.format(
        env.ssh_keys_name + '.pub', env.ssh_keys_dir))
    # Prevent root SSHing into the remote server
    sed('/etc/ssh/sshd_config', '^UsePAM yes', 'UsePAM no')
    sed('/etc/ssh/sshd_config', '^PermitRootLogin yes',
        'PermitRootLogin no')
    sed('/etc/ssh/sshd_config', '^#PasswordAuthentication yes',
        'PasswordAuthentication no')

    install_ansible_dependencies()
    create_deployer_group()
    create_deployer_user()
    upload_keys()
    set_selinux_permissive()
    run('service sshd reload')
    upgrade_server()
```

This function acts as the entry point for the Fabric script. Besides triggering a series of functions, each explained in further steps, it explicitly-

- Generates a new pair of SSH keys in the specified location within your local system
- Copies the contents of the public key to the *authorized_keys* file
- Makes changes to the remote *sshd_config* file to prevent root login and disable password-less auth

> Preventing SSH access for the root user is an optional step, but it is recommended as it ensures that no one has superuser rights.

Create a directory for your SSH keys in the project root:

```sh
├── prod
│   └── fabfile.py
└── ssh-keys
```

### Harden user passwords

This step includes the addition of three different functions, each executed serially to configure SSH password hardening...

#### Create deployer group

```python
def create_deployer_group():
    """
    Create a user group for all project developers
    """
    run('groupadd {}'.format(env.user_group))
    run('mv /etc/sudoers /etc/sudoers-backup')
    run('(cat /etc/sudoers-backup; echo "%' +
        env.user_group + ' ALL=(ALL) ALL") > /etc/sudoers')
    run('chmod 440 /etc/sudoers')
```

Here, we add a new group called `deployers` and grant sudo permissions to it so that users can carry out processes with root privileges.

#### Create user

```python
def create_deployer_user():
    """
    Create a user for the user group
    """
    run('adduser -c "{}" -m -g {} {}'.format(
        env.full_name_user, env.user_group, env.user_name))
    run('passwd {}'.format(env.user_name))
    run('usermod -a -G {} {}'.format(env.user_group, env.user_name))
    run('mkdir /home/{}/.ssh'.format(env.user_name))
    run('chown -R {} /home/{}/.ssh'.format(env.user_name, env.user_name))
    run('chgrp -R {} /home/{}/.ssh'.format(
        env.user_group, env.user_name))
```

This function-

- Adds a new user to the `deployers` user group, which we defined in the last function
- Sets up the SSH directory for keeping SSH key pairs and grants permission to the group and the user to access that directory

#### Upload SSH keys

```python
def upload_keys():
    """
    Upload the SSH public/private keys to the remote server via scp
    """
    scp_command = 'scp {} {}/authorized_keys {}@{}:~/.ssh'.format(
        env.ssh_keys_name + '.pub',
        env.ssh_keys_dir,
        env.user_name,
        env.host_string
    )
    local(scp_command)
```

Here, we-

- Upload the locally generated SSH keys to the remote server so that non-root users can log in via SSH without entering a password
- Copy the public key and the authorized keys to the remote server in the newly created *ssh-keys* directory

### Install Ansible dependencies

Add the following function to install the dependency packages for Ansible:

```python
def install_ansible_dependencies():
    """
    Install the python-dnf module so that Ansible
    can communicate with Fedora's Package Manager
    """
    run('dnf install -y python-dnf')
```

> Keep in mind that this is specific to the Fedora Linux distro, as we will be using the [DNF module](http://docs.ansible.com/ansible/dnf_module.html) for installing packages, but it could vary by distro.

### Set SELinux to permissive mode

The next function sets [SELinux](https://en.wikipedia.org/wiki/Security-Enhanced_Linux) to [permissive mode](https://wiki.gentoo.org/wiki/SELinux/Tutorials/Permissive_versus_enforcing#Permissive_versus_enforcing). This is done to overcome any potential Nginx 502 Bad Gateway [errors](https://asdqwe.net/blog/solutions-502-bad-gateway-error-on-nginx/).

```python
def set_selinux_permissive():
    """
    Set SELinux to Permissive/Disabled Mode
    """
    # for permissive
    run('sudo setenforce 0')
```

> Again, this is specific to the Fedora Linux distro.

### Upgrade the server

Finally, upgrade the server:

```python
def upgrade_server():
    """
    Upgrade the server as a root user
    """
    run('dnf upgrade -y')
    # optional command (necessary for Fedora 25)
    run('dnf install -y python')
    run('reboot')
```

## Sanity check

With that, we're done with the Fabric script. Before running it, make sure you SSH into the server as root and change the password:

```sh
$ ssh root@<server-ip-address>
You are required to change your password immediately (root enforced)
Changing password for root.
(current) UNIX password:
New password:
Retype new password:
```

Be sure to update `env.password` with the new password. Exit the server and return to the local terminal, then execute Fabric:

```sh
$ fab -f ./prod/fabfile.py start_provision
```

If all went well, new SSH keys will be generated, and you will be asked to create a password (make sure to do this!):

```sh
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

A number of tasks will run. After the `deployer` user is created, you will be prompted to add a password for the user-

```sh
[104.236.66.172] out: Changing password for user deployer.
```

-which you will then have to enter when the SSH keys are uploaded:

```sh
deployer@104.236.66.172s password:
```

After this script exits successfully, you will NO longer be able to log into the remote server as a root user. Instead, you will only be able to use the non-root user `deployer`.

Try it out:

```sh
$ ssh root@<server-ip-address>
Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

This is expected. Then, when you run-

```sh
$ ssh -i ./ssh-keys/104.236.66.172_prod_key deployer@104.236.66.172
```

-you should be able to log in just fine:

```sh
[deployer@fedora-512mb-nyc2-01 ~]$
```

## Ansible Primer

[Ansible](http://docs.ansible.com) is a configuration management and provisioning tool used to automate deployment tasks over SSH.

You can fire [individual Ansible tasks](http://docs.ansible.com/ansible/intro_adhoc.html) against the app servers from your shell remotely and execute tasks on the go. Tasks can also be combined into [Playbooks](http://docs.ansible.com/ansible/playbooks_intro.html) - a collection of multiple *plays*, where each play defines certain specific tasks that are required during the deployment process. They are executed against the app servers during the deployment process. Playbooks are written in [YAML](http://docs.ansible.com/ansible/YAMLSyntax.html).

### Playbooks

Playbooks consist of a modular architecture as follows:

1. [Hosts](http://docs.ansible.com/ansible/playbooks_intro.html#hosts-and-users) specify all the IP addresses or domain names of our remote servers that need to be orchestrated. Playbooks always run on a targeted group of hosts.
1. [Roles](http://docs.ansible.com/ansible/playbooks_roles.html) are divided into sub parts. Let's look at some sample roles:
    - Tasks are a collection of multiple tasks that need to be carried out during the deployment process.
    - Handlers provide a way to trigger a set of operations when a module makes a change to the remote server (best thought of as hooks).
    - Templates, in this context, are generally used for specifying some module-related configuration files - like nginx.
1. [Variables](http://docs.ansible.com/ansible/playbooks_variables.html) are simply a list of key-value pairs where every key (a variable) is mapped to a value. Such variables can be used in the Playbooks as placeholders.

### Sample Playbook

Now let's look at a sample single-file Playbook:

{% raw %}
```
---
# My Ansible playbook for configuring Nginx
- hosts: all

  vars:
    http_port: 80
    app_name: django_bootstrap

  tasks:
    - name: Install nginx
      dnf: name=nginx state=latest

    - name: Create nginx config file
      template: src=django_bootstrap.conf dest=/etc/nginx/conf.d/{{ app_name }}.conf
      become: yes
      notify:
        - restart nginx

  handlers:
    - name: Restart nginx
      service: name=nginx state=restarted enabled=yes
      become: yes
```
{% endraw %}

Here, we defined the-

- Hosts as `hosts: all`, which indicates that the Playbook will run on all of the servers that are listed in the *[inventory/hosts](http://docs.ansible.com/ansible/intro_inventory.html)* file
- Variables `http_port: 80` and `app_name: django_bootstrap` for use in a template
- Tasks in order to install nginx, set up the nginx config (`become` indicates that we need admin privileges), and trigger the restart handler
- Handler in order to restart the nginx service

## Playbook Setup

Now let's set up a Playbook for Django. Add a *deploy.yml* file to the "prod" directory:

```
##
# This playbook deploys the whole app stack
##
- name: apply common configuration to server
  hosts: all
  user: deployer
  roles:
    - common
```

The above snippet glues together the Ansible hosts, users, and roles.

### Hosts

Add a *hosts* (plain text format) file to the "prod" directory and list the servers under their respective role names. We are provisioning a single server here:

```
[common]
<server-ip-address>
```

In the above snippet, `common` refers to the role name. Under the roles we have a list of IP addresses that need to be configured. Make sure to add your remote server's IP address in place of `<server-ip-address>`.

### Variables

Now we define the variables that will be used by the roles. Add a new folder inside "prod" called "group_vars", then create a new file called *all* (plain text format) within that folder. Here, specify the following variables to start with:

```sh
# App Name
app_name: django_bootstrap

# Deployer User and Groups
deployer_user: deployer
deployer_group: deployers

# SSH Keys Directory
ssh_dir: <path-to-your-ssh-keys>
```

Make sure to update `<path-to-your-ssh-keys>`. To get the correct path, within the project root, run:

```sh
$ cd ssh-keys
$ pwd
/Users/michael.herman/repos/realpython/automated-deployments/ssh-keys
```

With these files in place, we are now ready to coordinate our deployment process with all the roles that need be carried out on the server.

## Playbook Roles

Again, Playbooks are simply a collection of different plays, and all these plays are run under specific roles. Create a new directory called "roles" within "prod".

> Did you catch the name of the role in the *deploy.yml* file?

Then within the "roles" directory add a new directory called "common" - the role. Roles consists of "tasks", "handlers", and "templates". Add a new directory for each.

Once done your file structure should look something like this:

```sh
├── prod
│   ├── deploy.yml
│   ├── fabfile.py
│   ├── group_vars
│   │   └── all
│   ├── hosts
│   └── roles
│       └── common
│           ├── handlers
│           ├── tasks
│           └── templates
└── ssh-keys
    ├── 104.236.66.172_prod_key
    ├── 104.236.66.172_prod_key.pub
    └── authorized_keys
```

All the plays are defined in a "tasks" directory, starting with a *main.yml* file. This file serves as the entry point for *all* Playbook tasks. It's simply a list of multiple YAML files that need to be executed in order.

Create that file now within the "tasks" directory, then add the following to it:

```
##
# Configure the server for the Django app
##
- include: 01_server.yml
- include: 02_git.yml
- include: 03_postgres.yml
- include: 04_dependencies.yml
- include: 05_migrations.yml
- include: 06_nginx.yml
- include: 07_gunicorn.yml
- include: 08_systemd.yml
# - include: 09_fix-502.yml
```

Now, let's create each task. Be sure to add a new file for each task to the "tasks" directory and add the accompanying code to each file. If you get lost, refer to the [repo](https://github.com/realpython/automated-deployments).


### *01_server.yml*

{% raw %}
```
##
# Update the DNF package cache and install packages as a root user
##
- name: Install required packages
  dnf: name={{item}} state=latest
  become: yes
  with_items:
    - vim
    - fail2ban
    - python3-devel
    - python-virtualenv
    - python3-virtualenv
    - python-devel
    - gcc
    - libselinux-python
    - redhat-rpm-config
    - libtiff-devel
    - libjpeg-devel
    - libzip-devel
    - freetype-devel
    - lcms2-devel
    - libwebp-devel
    - tcl-devel
    - tk-devel
    - policycoreutils-devel
```
{% endraw %}

Here, we list all the system packages that need to be installed.

### *02_git.yml*

{% raw %}
```
##
# Clone and pull the repo
##
- name: Set up git configuration
  dnf: name=git state=latest
  become: yes

- name: Clone or pull the latest code
  git: repo={{ code_repository_url }}
        dest={{ app_dir }}
```
{% endraw %}

Add the following variables to the *group_vars/all* file:

{% raw %}
```
# Github Code's Repo URL
code_repository_url: https://github.com/realpython/django-bootstrap

# App Directory
app_dir: /home/{{ deployer_user }}/{{app_name}}
```
{% endraw %}

Make sure to fork then clone the [django-bootstrap](https://github.com/realpython/django-bootstrap) repo, then update the `code_repository_url` variable to the URL of your fork.

### *03_postgres.yml*

{% raw %}
```
##
# Set up and configure postgres
##
- name: Install and configure db
  dnf: name={{item}} state=latest
  become: yes
  with_items:
    - postgresql-server
    - postgresql-contrib
    - postgresql-devel
    - python-psycopg2

- name: Run initdb command
  raw: postgresql-setup initdb
  become: yes

- name: Start and enable postgres
  service: name=postgresql enabled=yes state=started
  become: yes

- name: Create database
  postgresql_db: name={{ app_name }}
  become_user: postgres
  become: yes

- name: Configure a new postgresql user
  postgresql_user: db={{ app_name }}
                                name={{ db_user }}
                                password={{ db_password }}
                                priv=ALL
                                role_attr_flags=NOSUPERUSER
  become: yes
  become_user: postgres
  notify:
    - restart postgres
```
{% endraw %}

Update *group_vars/all* with the database configuration needed for the playbook:

{% raw %}
```
# DB Configuration
db_url: postgresql://{{deployer_user}}:{{db_password}}@localhost/{{app_name}}
db_password: thisissomeseucrepassword
db_name: "{{ app_name }}"
db_user: "{{ deployer_user }}"
```
{% endraw %}

Update the `db_password` variable with a secure password.

Did you notice that we restart the postgres service within the *main.yml* file in order to apply the changes after the database is configured? This is our first handler. Create a new file called *main.yml* in the "handlers" folder, then add the following:

```sh
- name: restart postgres
  service: name=postgresql state=restarted
  become: yes
```

### *04_dependencies.yml*

{% raw %}
```
##
# Set up all the dependencies in a virtualenv required by the Django app
##
- name: Create a virtualenv directory
  file: path={{ venv_dir }} state=directory

- name: Install dependencies
  pip:    requirements={{ app_dir }}/requirements.txt
          virtualenv={{ venv_dir }}
          virtualenv_python=python3.5

- name: Create the .env file for running ad-hoc python commands in our virtualenv
  template: src=env.j2 dest={{ app_dir }}/.env
  become: yes
```
{% endraw %}

Update *group_vars/all* like so:

{% raw %}
```
# Application Dependencies Setup
venv_dir: '/home/{{ deployer_user }}/envs/{{ app_name }}'
venv_python: '{{ venv_dir }}/bin/python3.5'
```
{% endraw %}

Add a template called *env.j2* to the "templates" folder, and add the following environment variables:

```
#!/bin/bash
export DEBUG="True"
export DATABASE_URL="postgresql://deployer:thisissomeseucrepassword@localhost/django_bootstrap"
export DJANGO_SECRET_KEY="changeme"
export DJANGO_SETTINGS_MODULE="config.settings.production"
```

> Be very careful with the environment variables and their values in *env.j2* since these are used to get the Django Project up and running.

### *05_migrations.yml*

{% raw %}
```
##
# Run db migrations and get all static files
##
- name: Make migrations
  shell: ". {{ app_dir }}/.env; {{ venv_python }} {{ app_dir }}/manage.py makemigrations "
  become: yes

- name: Migrate database
  django_manage: app_path={{ app_dir }}
                                 command=migrate
                                 virtualenv={{ venv_dir }}

- name: Get all static files
  django_manage: app_path={{ app_dir }}
                                 command=collectstatic
                                 virtualenv={{ venv_dir }}
  become: yes
```
{% endraw %}

### *06_nginx.yml*

{% raw %}
```
##
# Configure nginx web server
##
- name: Set up nginx config
  dnf: name=nginx state=latest
  become: yes

- name: Write nginx conf file
  template: src=django_bootstrap.conf dest=/etc/nginx/conf.d/{{ app_name }}.conf
  become: yes
  notify:
    - restart nginx
```
{% endraw %}

Add the following variable to *group_vars/all*:

```
# Remote Server Details
server_ip: <remote-server-ip>
wsgi_server_port: 8000
```

Don't forget to update `<remote-server-ip>`. Then add the handler to *handlers/main.yml*:

```
- name: restart nginx
  service: name=nginx state=restarted enabled=yes
  become: yes
```

Then we need to add the *django_bootstrap.conf* template. Create that file within the "templates" directory, then add the code:

{% raw %}
```
upstream app_server {
    server 127.0.0.1:{{ wsgi_server_port }} fail_timeout=0;
}

server {
    listen 80;
    server_name {{ server_ip }};
    access_log /var/log/nginx/{{ app_name }}-access.log;
    error_log /var/log/nginx/{{ app_name }}-error.log info;

    keepalive_timeout 5;

    # path for staticfiles
    location /static {
            autoindex on;
            alias {{ app_dir }}/staticfiles/;
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
{% endraw %}

### *07_gunicorn.yml*

{% raw %}
```
##
# Set up Gunicorn and configure systemd to execute gunicorn_start script
##
- name: Create a deploy directory
  file: path={{ deploy_dir }} state=directory
  become: yes

- name: Create the gunicorn_start script for running our app from systemd service
  template: src=gunicorn_start
                    dest={{ deploy_dir }}/gunicorn_start
  become: yes

- name: Make the gunicorn_start script executable
  raw: cd {{ deploy_dir }}; chmod +x gunicorn_start
  become: yes
```
{% endraw %}

Add more variables to *groups_vars/all*:

{% raw %}
```sh
# Deploy Dir in App Directory
deploy_dir: '{{ app_dir }}/deploy'

# WSGI Vars
django_wsgi_module: config.wsgi
django_settings_module: config.settings.production
django_secret_key: 'changeme'
database_url: '{{ db_url }}'
```
{% endraw %}

Add the *gunicorn_start* template:

{% raw %}
```
#!/bin/bash

### Define script variables

# Name of the app
NAME='{{ app_name }}'
# Path to virtualenv
VIRTUALENV='{{ venv_dir }}'
# Django Project Directory
DJANGODIR='{{ app_dir }}'
# The user to run as
USER={{ deployer_user }}
# The group to run as
GROUP={{deployer_group }}
# Number of worker processes Gunicorn should spawn
NUM_WORKERS=3
# Settings file that Gunicorn should use
DJANGO_SETTINGS_MODULE={{django_settings_module}}
# WSGI module name
DJANGO_WSGI_MODULE={{ django_wsgi_module }}


### Activate virtualenv and create environment variables

echo "Starting $NAME as `whoami`"
# Activate the virtual environment
cd $VIRTUALENV
source bin/activate
cd $DJANGODIR
# Defining the Environment Variables
export DJANGO_SECRET_KEY='{{ django_secret_key }}'
export DATABASE_URL='{{ db_url }}'
export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE
export PYTHONPATH=$DJANGODIR:$PYTHONPATH


### Start Gunicorn

exec gunicorn ${DJANGO_WSGI_MODULE}:application \
        --name $NAME \
        --workers $NUM_WORKERS \
        --user=$USER --group=$GROUP \
        --log-level=debug \
        --bind=127.0.0.1:8000
```
{% endraw %}

### *08_systemd.yml*

```
##
# Set up systemd for executing gunicorn_start script
##
- name: write a systemd service file
  template: src=django-bootstrap.service
                    dest=/etc/systemd/system
  become: yes
  notify:
    - restart app
    - restart nginx
```

Add the template - *django-bootstrap.service*:

{% raw %}
```
#!/bin/sh

[Unit]
Description=Django Web App
After=network.target

[Service]
PIDFile=/var/run/djangoBootstrap.pid
User={{ deployer_user }}
Group={{ deployer_group }}
ExecStart=/bin/sh {{ deploy_dir }}/gunicorn_start
Restart=on-abort

[Install]
WantedBy=multi-user.target
```
{% endraw %}

Add the following to the handlers:

```
- name: restart app
  service: name=django-bootstrap state=restarted enabled=yes
  become: yes
```

### *09_fix-502.yml*

```
##
# Fix the 502 nginx error post deployment
#
- name: Fix nginx 502 error
  raw: cd ~; cat /var/log/audit/audit.log | grep nginx | grep denied | audit2allow -M mynginx; semodule -i mynginx.pp
  become: yes
```

## Sanity Check (final)

With the virtualenv activated, install Ansible locally:

```sh
$ pip install ansible==2.1.3
```

Create a new file called *deploy_prod.sh* in the project root to run the playbook, making sure to update `<server-ip>`:

```sh
#!/bin/bash

ansible-playbook ./prod/deploy.yml --private-key=./ssh_keys<server-ip>_prod_key -K -u deployer -i ./prod/hosts -vvv
```

Then run following command to execute the playbook:

```sh
$ sh deploy_prod.sh
```

If any errors occur, consult the terminal for info on how to correct them. Once fixed, execute the deploy script again. When the script is done visit the server's IP Address to verify your Django web app is live and running!

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-fabric-ansible/django-live.png" style="max-width: 100%;" alt="live Django app">
</div>

Make sure to uncomment this line in *prod/roles/common/tasks/main.yml* if you see the 502 error, which indicates that there is a problem with communication between nginx and Gunicorn:

```
# - include: 09_fix-502.yml
```

Then execute the playbook again.

> If you execute the playbook more than once, make sure to comment out the `Run initdb command` found in *03_postgres.yml* since it needs run only once. Otherwise, it will throw errors when trying to reinitialize the DB server.

## Conclusion

This post provides a basic understanding of how you can automate the configuring of a server with Fabric and Ansible. Ansible Playbooks are particularly powerful since you can automate almost any task on the server via a YAML file. Hopefully, you can now start writing your own Playbooks and even use them in your workplace to configure production-ready servers.

Please add questions and comments below. The full code can be found in the [automated-deployments](https://github.com/realpython/automated-deployments) repository.
