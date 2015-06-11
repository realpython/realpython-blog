# Rethink Flask - A Simple todo list powered by Flask and RethinkDB

After a number of requests for a basic [Flask](http://flask.pocoo.org/) and [RethinkDB](http://www.rethinkdb.com/) template, I decided to go ahead and write a blog post. This is that post.

> BTW: We always welcome requests. If you have something you'd like us to write about, or build, shoot us an email.

Today we'll be creating a *simple* todo list, which you'll be able to modify to meet your own needs. Before beginning, I highly suggest reading over [this](http://www.rethinkdb.com/docs/rethinkdb-vs-mongodb/) article, which details how RethinkDB differs from some of the other NoSQL databases.

## Set up RethinkDB

### Install

Navigate [here](http://www.rethinkdb.com/docs/install/) and download the appropriate package for your system. I used Homebrew - `$ brew install rethinkdb` - and it took almost twenty minutes to download and install the build:

```sh
==> Installing rethinkdb
==> Downloading http://download.rethinkdb.com/dist/rethinkdb-     1.11.2.tgz
######################################################################## 100.0%
==> ./configure --prefix=/usr/local/Cellar/rethinkdb/1.11.2 --  fetch v8 --fetch protobuf
==> make
==> make install-osx
==> Caveats
To have launchd start rethinkdb at login:
  ln -sfv /usr/local/opt/rethinkdb/*.plist   ~/Library/LaunchAgents
Then to load rethinkdb now:
  launchctl load   ~/Library/LaunchAgents/homebrew.mxcl.rethinkdb.plist
==> Summary
🍺  /usr/local/Cellar/rethinkdb/1.11.2: 174 files, 29M, built in   19.7 minutes
```

### Install the Python drivers globally:

```sh
$ sudo pip install rethinkdb
```

> Note: I installed Rethink globally (outside of a virtualenv) since I'll probably use the same version with a number of projects, with a number of different languages. We will be installing in within a virtualenv later on in this tutorial.

### Test

First, let's start the server with the following command:

```sh
$ rethinkdb
```

If all is installed correctly, you should see something similar to:

```sh
info: Creating directory /Users/michaelherman/rethinkdb_data
info: Creating a default database for your convenience. (This is because you ran 'rethinkdb' without 'create', 'serve', or '--join', and the directory '/Users/michaelherman/rethinkdb_data' did not already exist.)
info: Running rethinkdb 1.11.2 (CLANG 4.2 (clang-425.0.28))...
info: Running on Darwin 12.4.0 x86_64
info: Loading data from directory    /Users/michaelherman/rethinkdb_data
info: Listening for intracluster connections on port 29015
info: Listening for client driver connections on port 28015
info: Listening for administrative HTTP connections on port 8080
info: Listening on addresses: 127.0.0.1, ::1
info: To fully expose RethinkDB on the network, bind to all addresses
info: by running rethinkdb with the `--bind all` command line option.
info: Server ready
```

Then test the connection. Open a new window in your terminal and enter the following commands:

```sh
$ python
>>> import rethinkdb
>>> rethinkdb.connect('localhost', 28015).repl()
```

You should see:

```sh
<rethinkdb.net.Connection object at 0x101122410>
```

Exit the Python shell but leave the RethinkDB server running in the other terminal window.

## Set up a Basic Flask project

### Create a directory to store your project:

```sh
$ mkdir flask-rethink
$ cd flask-rethink
```

### Set up and activate a virtualenv:

```sh
$ virtualenv --no-site-packages env
$ source env/bin/activate
```

### Install Flask and Flask-WTF:

```sh
$ pip install flask
$ pip install flask-wtf
```

### Create a requirements:

```sh
$ pip freeze > requirements.txt
```

### Download the Flask boilerplate

Found in the template directory of [this](https://github.com/mjhea0/flask-rethink) repo.

### Your project structure should now look like this:

```sh
.
├── app
│   ├── __init__.py
│   ├── forms.py
│   ├── models.py
│   ├── templates
│   │   ├── base.html
│   │   └── index.html
│   └── views.py
├── readme.md
├── requirements.txt
└── run.py
```

### Run the app:

```sh
$ python run.py
```

Navigate to [http://localhost:5000/](http://localhost:5000/), and you should see:

![main](https://raw.github.com/mjhea0/flask-rethink/master/main.png)

Don't try to submit anything yet, because we need to get a database setup first. Let's get RethinkDB going.

## RethinkDB Config

###  Install RethinkDB:

```sh
pip install rethinkdb
```

### Add the following code to "views.py":

```python
# rethink imports
import rethinkdb as r
from rethinkdb.errors import RqlRuntimeError, RqlDriverError

# rethink config
RDB_HOST =  'localhost'
RDB_PORT = 28015
TODO_DB = 'todo'

# db setup; only run once
def dbSetup():
    connection = r.connect(host=RDB_HOST, port=RDB_PORT)
    try:
        r.db_create(TODO_DB).run(connection)
        r.db(TODO_DB).table_create('todos').run(connection)
        print 'Database setup completed'
    except RqlRuntimeError:
        print 'Database already exists.'
    finally:
        connection.close()
dbSetup()

# open connection before each request
@app.before_request
def before_request():
    try:
        g.rdb_conn = r.connect(host=RDB_HOST, port=RDB_PORT, db=TODO_DB)
    except RqlDriverError:
        abort(503, "Database connection could be established.")

# close the connection after each request
@app.teardown_request
def teardown_request(exception):
    try:
        g.rdb_conn.close()
    except AttributeError:
        pass
```

Check the comments for a brief explanation of what each of the functions do.

### Start your server again.

You should see the following alert in your terminal:

```sh
Database setup completed
```

> If you see this error `rethinkdb.errors.RqlDriverError: Could not connect to localhost:28015.` your RethinkDB server is not running. Open up a new terminal window and run `$ rethinkdb`.

So, we created a new database called "todo", which has a table called "todos".

You can verify this in the RethinkDB Admin. Navigate to [http://localhost:8080/](http://localhost:8080/). The admin should load. If you click "Tables", you should see the database and table we created:

![rethink-admin](https://raw.github.com/mjhea0/flask-rethink/master/rethink-admin.png)

### Display Todos

With the database setup, let's add code to display todos. Update the `index()` function in "views.py":

```python
@app.route("/")
def index():
    form = TaskForm()
    selection = list(r.table('todos').run(g.rdb_conn))
    return render_template('index.html', form=form, tasks=selection)
```

Here we're selecting the "todos" table, pulling all of the data, which is in JSON, and passing the entire table to the template.

### Add data manually

Before we can view any todos, we need to add some first. Let's go through the shell and add them in manually.

```sh
$ python
>>> import rethinkdb
>>> conn = rethinkdb.connect(db='todo')
>>> rethinkdb.table('todos').insert({'name':'sail to the moon'}).run(conn)
{u'errors': 0, u'deleted': 0, u'generated_keys': [u'c5562325-c5a1-4a78-8232-c0de4f500aff'], u'unchanged': 0, u'skipped': 0, u'replaced': 0, u'inserted': 1}
>>> rethinkdb.table('todos').insert({'name':'jump in the ocean'}).run(conn)
{u'errors': 0, u'deleted': 0, u'generated_keys': [u'0a3e3658-4513-48cb-bc68-5af247269ee4'], u'unchanged': 0, u'skipped': 0, u'replaced': 0, u'inserted': 1}
>>> rethinkdb.table('todos').insert({'name':'think of another todo'}).run(conn)
{u'errors': 0, u'deleted': 0, u'generated_keys': [u'b154a036-3c3b-47f4-89ec-cb9f4eff5f5a'], u'unchanged': 0, u'skipped': 0, u'replaced': 0, u'inserted': 1}
>>>
```

So, we connected to the database, then entered three new objects into the table within the database. *Check the API [docs](http://www.rethinkdb.com/api/python/) for more information.*

Fire up the server. You should now see the three tasks:

![tasks](https://raw.github.com/mjhea0/flask-rethink/master/tasks.png)

### Finalize the form

Update the `index()` function again to pull the data from the form and add it to the database:

```python
@app.route('/', methods = ['GET', 'POST'])
def index():
    form = TaskForm()
        if form.validate_on_submit():
            r.table('todos').insert({"name":form.label.data}).run(g.rdb_conn)
            return redirect(url_for('index'))
        selection = list(r.table('todos').run(g.rdb_conn))
        return render_template('index.html', form = form, tasks = selection)
```

Test this out. Add some todos. Go crazy.

## Challenges

The current app is functional, but there's a lot more we can do. Take this app to the next level.

Here's a few ideas:

1. Add a user login.
2. Create a more robust form, where you can add a due date for each todo, and then sort the todos by that date before rendering them to the DOM.
3. Add functional and unit tests.
4. Add the ability to create sub tasks for each task.
5. Read over the API reference [docs](http://www.rethinkdb.com/api/python/). Play around with various methods.
6. Moduralize the app.
7. Refactor the code. Show off your new code to RethinkDB.

What else would you like to see? Interested in seeing a part 2? How do you like RethinkDB in comparison to MongoDB? Share your thoughts below.

You can grab all the code from the [repo](https://github.com/mjhea0/flask-rethink). Cheers!