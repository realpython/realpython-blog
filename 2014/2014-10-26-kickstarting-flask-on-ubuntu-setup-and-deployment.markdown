# Kickstarting Flask on Ubuntu - setup and deployment

**This tutorial details how to setup a Flask application on a server running Ubuntu.**

Since this process can be difficult, as there are a number of moving pieces, we'll look at this in multiple parts, starting with the most basic configuration and working our way up:

1. Part 1: Setting up the basic configuration
1. Part 2: Adding Supervisor
1. Part 3: Simplifying deployment with Git Hooks
1. Part 4: Automating with Fabric (*with an example video!*)

*Updates:*

- 10/28/2014: Added info on how to add a new user on Ubuntu
- 04/16/2015: Updated nginx configuration

<hr>

We'll specifically be using:

1. Ubuntu 14.04
1. nginx 1.4.6
1. gunicorn 19.1.1
1. Python 2.7.8
1. Pip 1.5.4
1. virtualenv 1.11.4
1. Flask 0.10.1
1. Supervisor 3.0b2

Assuming you already have a VPS running an Ubuntu operating system, we need to set up a web server on top of the operating system to serve static files - like stylesheets, JavaScript files, and images - to end users. We'll use [nginx](http://nginx.org/) as our web server. Since a web server cannot communicate directly with Flask (err Python), we'll use [gunicorn](http://gunicorn.org/) to act as a medium between the server and Python/Flask.

Wait, why do we need _two_ servers? Think if Gunicorn as the _application_ web server that will be running behind nginx - the front facing web server. Gunicorn is WSGI compatible. It can talk to other applications that support WSGI, like Flask or Django.

![nginx+gunicorn](https://raw.githubusercontent.com/realpython/flask-deploy/master/images/localhost2.jpg)

> Need access to a web server? Check out [Digital Ocean](https://www.digitalocean.com/), [Linode](https://www.linode.com/), or [Amazon EC2](http://aws.amazon.com/ec2/). Alternatively, you can use [Vagrant](https://www.vagrantup.com/) to emulate a Linux environment. This setup was tested on both Digital Ocean and Linode.

The end goal: HTTP requests are routed from the web server to Flask, which Flask handles appropriately, and the responses are then sent right back to the web server and, finally, back to the end user. Properly implementing a Web Server Gateway Interface (WSGI) will be paramount to our success.

![wsgi](https://raw.githubusercontent.com/realpython/flask-deploy/master/images/wsgi.jpg)

Let's get to it.

## Part 1 - Setup

Let's get the basic configuration setup.

### Add a new User

After SSH'ing into the server as the 'root' user, run-

```sh
$ adduser newuser
$ adduser newuser sudo
```

-to create a new user with 'sudo' privileges.

### Install the Requirements

SSH into the server with the new user, and then install the following packages:

```sh
$ sudo apt-get update
$ sudo apt-get install -y python python-pip python-virtualenv nginx gunicorn
```

### Set up Flask

Start by creating a new directory, "/home/www", to store the project:

```sh
$ sudo mkdir /home/www && cd /home/www
```

Then create and activate a virtualenv:

```sh
$ sudo virtualenv env
$ source env/bin/activate
```

Install the requirements:

```sh
$ sudo pip install Flask==0.10.1
```

Now set up your project:

```sh
$ sudo mkdir flask_project && cd flask_project
$ sudo vim app.py
```

Add the following code to *app.py*:

```python
from flask import Flask, jsonify

app = Flask(__name__)


@app.route('/')
def index():
    return 'Flask is running!'


@app.route('/data')
def names():
    data = {"names": ["John", "Jacob", "Julie", "Jennifer"]}
    return jsonify(data)


if __name__ == '__main__':
    app.run()
```

> Within VIM, press “i” to enter the INSERT mode. Add the code, then press “escape” to leave INSERT mode to go into COMMAND mode. Finally type “:wq” to save and exit VIM.

Set up a static directory-

```sh
$ sudo mkdir static
```

-and then add an *index.html* (`sudo vim static/index.html`) file with the following html:

```html
<h1>Test!</h1>
```

### Configure nginx

Start nginx:

```sh
$ sudo /etc/init.d/nginx start
```

Then:

```sh
$ sudo rm /etc/nginx/sites-enabled/default
$ sudo touch /etc/nginx/sites-available/flask_project
$ sudo ln -s /etc/nginx/sites-available/flask_project /etc/nginx/sites-enabled/flask_project
```

Here, we remove the default nginx configuration, create a new config file (called *flask_project*), and, finally, set up a symlink to the config file we just created so that nginx loads it on startup.

Now, let's add the config settings to *flask_project*:

```sh
$ sudo vim /etc/nginx/sites-enabled/flask_project
```

Add:

```
server {
    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    location /static {
        alias  /home/www/flask_project/static/;
    }
}
```

So, HTTP requests that hit the `/` endpoint will be '[reverse proxied](http://en.wikipedia.org/wiki/Reverse_proxy)' to port 8000 on `127.0.0.1` (or the "loopback ip" or "localhost"). This is same IP and port that gunicorn will use.

We also indicate that we want nginx to directly serve the static files from the "/home/www/flask_project/static/" directory rather than routing the requests through gunicorn/WSGI. This will speed up our site's load time since nginx knows to serve that directory directly.

Restart nginx:

```
$ sudo /etc/init.d/nginx restart
```

### Profit!

```sh
$ cd /home/www/flask_project/
$ gunicorn app:app -b localhost:8000
```

The latter command manually runs gunicorn on localhost port 8000.

Open your browser and navigate to [http://your_domain_name_or_ip_address](http://your_domain_name_or_ip_address).

Again, you should see the "Flask is running!" message. Test out the other URL, `/data`, as well. If you navigate to [http://your_domain_name_or_ip_address/static](http://your_domain_name_or_ip_address/static), you should see "Test!", indicating that we're serving static files correctly.

## Part 2 - Supervisor

So, we have a working Flask app; however, there's one problem: We have to manually (re)start gunicorn each time we make changes to our app. We can automate this with [Supervisor](http://supervisord.org/).

### Configure Supervisor

SSH into your server, and then install Supervisor:

```sh
$ sudo apt-get install -y supervisor
```

Now create a configuration file:

```sh
$ sudo vim /etc/supervisor/conf.d/flask_project.conf
```

Add:

```
[program:flask_project]
command = gunicorn app:app -b localhost:8000
directory = /home/www/flask_project
user = newuser
```

### Profit!

Stop gunicorn:

```sh
$ sudo pkill gunicorn
```

Start gunicorn with supervisor:

```sh
$ sudo supervisorctl reread
$ sudo supervisorctl update
$ sudo supervisorctl start flask_project
```

Make sure your app is still running at [http://your_domain_name_or_ip_address](http://your_domain_name_or_ip_address). Check out the Supervisor [documentation](http://supervisord.org/index.html) for custom configuration info.

## Part 3 - Deployment

In this final part, we'll look at using a post-receive [Git Hook](http://git-scm.com/book/en/Customizing-Git-Git-Hooks) along with Git, of course, to simplify the deployment process.

![githooks](https://raw.githubusercontent.com/realpython/flask-deploy/master/images/githooks2.jpg)

### Configure Git

Again, SSH into the remote server. And then install Git:

```sh
$ sudo apt-get install -y git
```

Now run the following commands to set up a bare Git repo that we can push to:

```sh
$ sudo mkdir /home/git && cd /home/git
$ sudo mkdir flask_project.git && cd flask_project.git
$ sudo git init --bare
```

> Quick tip: displaying the git branch on your prompt will help remind you of where you are at while trekking through the terminal.

> Consider adding this to your bash profile on production:

>     parse_git_branch() {
>         git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ (\1)/'
>     }
>
>     export PS1="\u@\h \W\[\033[32m\]\$(parse_git_branch)\[\033[00m\] $ "

### Configure the Post-Receive Hook

```sh
$ sudo vim hooks/post-receive
```

Add:

```
#!/bin/sh
GIT_WORK_TREE=/home/www/flask_project git checkout -f
```

Now on every push, the new files will copy over to the app directory, "/home/www/flask_project".

Then make the file executable:

```sh
$ sudo chmod +x hooks/post-receive
```

### Profit!

Back in your local Flask directory ("flask_project") add a new Git repo along with the following remote:

```sh
$ git init
$ git remote add production root@<your_ip_or_domain>:/home/git/flask_project.git
```

> Make sure to update the IP or domain name.

Make some changes to your code in the *app.py* file:

```python
@app.route('/data')
def names():
    data = {
        "first_names": ["John", "Jacob", "Julie", "Jennifer"],
        "last_names": ["Connor", "Johnson", "Cloud", "Ray"]
    }
    return jsonify(data)
```

Commit your local changes, then push:

```sh
$ git add -A
$ git commit -am "initial"
$ git push production master
```

SSH into your server and restart gunicorn via Supervisor:

```sh
$ sudo supervisorctl restart flask_project
```

Check out your changes at [http://your_domain_name_or_ip_address/data](http://your_domain_name_or_ip_address/data).

## Part 4 - Automating

Do you really want to manually configure a server? Sure, it's great for learning, but it's super tedious, as you can tell. Fortunately, we've automated the process with [Fabric](http://www.fabfile.org/). Along with setting up nginx, gunicorn, Supervisor, and Git, the script creates a basic Flask app, which is specific to the project that we've been working with. You can easily customize this to meet your own specific needs.

> You should remove the username and password from the file and either place them in a separate config file that stays out of version control or set up SSH keys on the remote server so that you do not need a password to login. Also, be sure to update the `env.hosts` variable to your IP or domain name.

### Setup

To test this script (*fabfile.py*) clone the [repo](https://github.com/realpython/flask-deploy) and start with a clean, freshly provisioned server with Ubuntu 14.04. Then navigate to the "flask-deploy" directory. To set up the basic configuration on the remote server as well as your app, run the following command:

```sh
$ fab create
```

Your app should now be live. Test this out in your browser.

### Deployment

Want to set up deployment with Git Hooks? Initialize a local repo within your Flask project directory (if necessary). Then make some local changes to your Flask app, and run the following command to deploy:

```sh
$ fab deploy
```

Check your app again in the browser. Make sure your changes show up.

### Status Check

Finally, you can check that the Supervisor process is running correctly, to ensure that your app is live, with the following command:

```sh
$ fab status
```

Again, this script is specific to your project at hand. You can customize this to your own needs by updating the config section and altering the tasks as necessary.

### Rollback

To err is human...

Things are bound to go wrong from time to time once you have code on production. Everything may work well on your local development environment only to crash in production. So it's important to have a strategy in place to quickly revert a Git commit. Take a quick look at the `rollback` task from the *fabfile.py*, which allows you to revert changes quickly to get a working app back up.

Test it out by purposely breaking your code and then deploying to production, and then run:

```sh
$ fab rollback
```

You can then update your code locally to fix the bugs, and then re-deploy.

## Example Video

Just press play...

{% youtube VmcGuKPpWH8 %}

## Conclusion and Next Steps

Want to take this workflow to the next level?

1. Add a pre-production (staging) server into the mix for testing purposes. Deploy to this server first, which should be an exact copy of your production environment, to test before deploying to production.
1. Utilize Continuous Integration and Delivery to further eliminate bugs and regressions with automated testing and decrease the time it takes to deploy your app.

Please leave questions and comments below. Be sure to grab the code from the [repo](https://github.com/realpython/flask-deploy).
