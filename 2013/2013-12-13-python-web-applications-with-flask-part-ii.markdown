# Python Web Applications With Flask - Part II

Please note: This is a collaboration piece between Michael Herman, from Real Python, and Sean Vieira, a Python developer from [De Deo Designs](http://dedeodesigns.com/).

<hr>

### Articles in this series:

1. Part I: [Application setup](http://www.realpython.com/blog/python/python-web-applications-with-flask-part-i/#.Uu6GOHddUp8)
2. **Part II: Setup user accounts, Templates, Static files <-- *CURRENT ARTICLE***
3. Part III: [Testing (unit and integration), Debugging, and Error handling](http://www.realpython.com/blog/python/python-web-applications-with-flask-part-iii/#.Uu-xVnddUp9)


Welcome back to the Flask-Tracking development series! For those of you who are just joining us, we are implementing a web analytics application that conforms to [this napkin specification](http://www.realpython.com/blog/python/python-web-applications-with-flask-part-i/#toc_1).  For all those of you following along at home, you may check out today's code with:

```sh
$ git checkout v0.2
```

or you may download it from the [releases page on Github][repository:releases]. Those of you who are just joining us may wish to read [a note on the repository structure][series:faq:repo] as well.

## Housekeeping

To quickly review, in our [last article][series:part-1] we set up a bare-bones application which enabled sites to be added and visits recorded against them via a simple web interface or over HTTP.

Today we will add users, access control, and enable users to add visits from their own websites using tracking beacons. We will also be diving into some best practices for writing templates, keeping our models and forms in sync, and handling static files.

### From single to multi-package.

When last we left our application, the directory structure looked something like this:

```text
flask-tracking/
    flask_tracking/
        templates/    # Holds Jinja templates
        __init__.py   # General application setup
        forms.py      # User data to domain data mappers and validators
        models.py     # Domain models
        views.py      # well ... controllers, really.
    config.py         # Configuration, just like it says on the cover
    README.md
    requirements.txt
    run.py            # `python run.py` to bring the application up locally.
```

To keep things clear, let's move the existing `forms`, `models`, and `views` into a `tracking` sub-package and create another sub-package for our `User`-specific functionality which we will call `users`:

```text
flask_tracking/
    templates/
    tracking/         # This is the code from Part 1
        __init__.py   # Create this file - it should be empty.
        forms.py
        models.py
        views.py
    users/            # Where we are working today
        __init__.py
    __init__.py       # This is also code from Part 1
```

This means that we will need to change our import in `flask_tracking/__init__.py` from `from .views import tracking` to `from .tracking.views import tracking`.

Then there is the database setup in `tracking.models`. This we will move out into the parent package (`flask_tracking`) since the database manager will be shared between packages. Let's call that module `data`:

```python
# flask_tracking/data.py
from flask.ext.sqlalchemy import SQLAlchemy

db = SQLAlchemy()


def query_to_list(query, include_field_names=True):
    """Turns a SQLAlchemy query into a list of data values."""
    column_names = []
    for i, obj in enumerate(query.all()):
        if i == 0:
            column_names = [c.name for c in obj.__table__.columns]
            if include_field_names:
                yield column_names
        yield obj_to_list(obj, column_names)


def obj_to_list(sa_obj, field_order):
    """Takes a SQLAlchemy object - returns a list of all its data"""
    return [getattr(sa_obj, field_name, None) for field_name in field_order]
```

Then we can update `tracking.models` to use `from flask_tracking.data import db` and `tracking.views` to use `from flask_tracking.data import db, query_to_list` and we should now have a working *multi-package* application.

## Users

Now that we have split up our application into separate packages of related functionality, let's start working on the `users` package. Users need to be able to sign up for an account, manage their account, and log in and out. There could be more user-related functionality (especially around permissions) but to keep things clear we will stick with these basics.

### Enlisting help

We have a [rule][series:faq:deps] for taking on dependencies - each dependency we add must solve at least one difficult problem well.  Maintaining user sessions has several interesting edge-cases which makes it an excellent candidate for a dependency. Fortunately, there is one readily available for this use case - [Flask-Login][flask-login]. However, there is one thing that Flask-Login does not handle at all - authentication.  We can use any authentication scheme we want to - from "just provide a username" to distributed authentication schemes like Persona.  Let's keep it simple and go with username and password. This means that we need to store a user's password, which we will want to hash. Since properly hashing passwords is *also* a hard problem we will take on another dependency, [`backports.pbkdf2`][pbkdf2] to ensure our passwords are securely hashed. (We picked pbdkdf2 because it is [considered secure][security] as of this writing and is included in Python 3.3+ - we only need it while we are running on Python 2.)

Let's go ahead and add:

```text
Flask-Login==0.2.7
backports.pbkdf2==0.1
```

to our `requirements.txt` file and then (making sure our virtual environment is activated) we can run `pip install -r requirements.txt` again to install them. (You may get some errors compiling the C speedups for pbkdf2 - you can ignore them).  We will integrate it with our application in a moment - first we need to set up our Users so Flask-Login has something to work with.

### Models

We will set up our `User` SQLAlchemy class in `users.models`. We will only store a user's name, email address, and (salted and hashed) password:

```python
from random import SystemRandom

from backports.pbkdf2 import pbkdf2_hmac, compare_digest
from flask.ext.login import UserMixin
from sqlalchemy.ext.hybrid import hybrid_property

from flask_tracking.data import db


class User(UserMixin, db.Model):
    __tablename__ = 'users_user'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50))
    email = db.Column(db.String(120), unique=True)
    _password = db.Column(db.LargeBinary(120))
    _salt = db.Column(db.String(120))
    sites = db.relationship('Site', backref='owner', lazy='dynamic')

    @hybrid_property
    def password(self):
        return self._password

    # In order to ensure that passwords are always stored
    # hashed and salted in our database we use a descriptor
    # here which will automatically hash our password
    # when we provide it (i. e. user.password = "12345")
    @password.setter
    def password(self, value):
        # When a user is first created, give them a salt
        if self._salt is None:
            self._salt = bytes(SystemRandom().getrandbits(128))
        self._password = self._hash_password(value)

    def is_valid_password(self, password):
        """Ensure that the provided password is valid.

        We are using this instead of a ``sqlalchemy.types.TypeDecorator``
        (which would let us write ``User.password == password`` and have the incoming
        ``password`` be automatically hashed in a SQLAlchemy query)
        because ``compare_digest`` properly compares **all***
        the characters of the hash even when they do not match in order to
        avoid timing oracle side-channel attacks."""
        new_hash = self._hash_password(password)
        return compare_digest(new_hash, self._password)

    def _hash_password(self, password):
        pwd = password.encode("utf-8")
        salt = bytes(self._salt)
        buff = pbkdf2_hmac("sha512", pwd, salt, iterations=100000)
        return bytes(buff)

    def __repr__(self):
        return "<User #{:d}>".format(self.id)
```

*Phew* - almost half of this code is for the password! Even worse, by the time you are reading this our implementation of `_hash_password` is likely to be considered imperfect (such is the ever-changing nature of cryptography) but it does cover all of the basic best practices:

* Always use a salt unique to each user.
* Use a key-stretching algorithm with a tunable unit of work.
* Compare hashes using a *constant time* algorithm.

In non-password related notes, we are making a [one-to-many][sqa:1-to-many] relationship between `User`s and `Site`s (`sites = db.relationship('Site', backref='owner', lazy='dynamic')`) so that we can have users who manage multiple sites.

In addition, we are subclassing Flask-Login's `UserMixin` class.  Flask-Login requires that the `User` class implement [certain methods][login:methods] (`get_id`, `is_authenticated`, etc.) so that it can do its work.  `UserMixin` provides default versions of those methods that work quite well for our purposes.

### Integrating Flask-Login

Now that we have a `User` we can integrate with Flask-Login. In order to avoid circular imports we are going to setup the extension in its own top-level module named `auth`:

```python
# flask_tracking/auth.py
from flask.ext.login import LoginManager

from flask_tracking.users.models import User

login_manager = LoginManager()

login_manager.login_view = "users.login"
# We have not created the users.login view yet
# but that is the name that we will use for our
# login view, so we will set it now.


@login_manager.user_loader
def load_user(user_id):
    return User.query.get(user_id)
```

`@login_manager.user_loader` registers our `load_user` function with Flask-Login so that when a user returns after logging in Flask-Login can load the user from the user_id that it stores in Flask's `session`.

Finally, we import `login_manager` into `flask_tracking/__init__.py` and register it with our application object:

```python
from .auth import login_manager

# ...

login_manager.init_app(app)
```

### Views

Next let's set up our view and controller functions for Users to enable register/log in/log out functionality.  First, we will set up our forms:

```python
# flask_tracking/users/forms.py
from flask.ext.wtf import Form
from sqlalchemy.orm.exc import MultipleResultsFound, NoResultFound
from wtforms import fields
from wtforms.validators import Email, InputRequired, ValidationError

from .models import User


class LoginForm(Form):
    email = fields.StringField(validators=[InputRequired(), Email()])
    password = fields.StringField(validators=[InputRequired()])

    # WTForms supports "inline" validators
    # which are methods of our `Form` subclass
    # with names in the form `validate_[fieldname]`.
    # This validator will run after all the
    # other validators have passed.
    def validate_password(form, field):
        try:
            user = User.query.filter(User.email == form.email.data).one()
        except (MultipleResultsFound, NoResultFound):
            raise ValidationError("Invalid user")
        if user is None:
            raise ValidationError("Invalid user")
        if not user.is_valid_password(form.password.data):
            raise ValidationError("Invalid password")

        # Make the current user available
        # to calling code.
        form.user = user


class RegistrationForm(Form):
    name = fields.StringField("Display Name")
    email = fields.StringField(validators=[InputRequired(), Email()])
    password = fields.StringField(validators=[InputRequired()])

    def validate_email(form, field):
        user = User.query.filter(User.email == field.data).first()
        if user is not None:
            raise ValidationError("A user with that email already exists")
```

Again, a decent amount of code, this time mostly around validating user input.  One thing to note is that for our login form, when the user is authenticated, we expose the `User` instance on the form as `form.user` (so we do not have to make the same query in two places - even though SQLAlchemy will do the right thing here and only hit the database once).

Finally, we can set up our views:

```python
# flask_tracking/users/views.py
from flask import Blueprint, flash, redirect, render_template, request, url_for
from flask.ext.login import login_required, login_user, logout_user

from flask_tracking.data import db
from .forms import LoginForm, RegistrationForm
from .models import User

users = Blueprint('users', __name__)


@users.route('/login/', methods=('GET', 'POST'))
def login():
    form = LoginForm()
    if form.validate_on_submit():
        # Let Flask-Login know that this user
        # has been authenticated and should be
        # associated with the current session.
        login_user(form.user)
        flash("Logged in successfully.")
        return redirect(request.args.get("next") or url_for("tracking.index"))
    return render_template('users/login.html', form=form)


@users.route('/register/', methods=('GET', 'POST'))
def register():
    form = RegistrationForm()
    if form.validate_on_submit():
        user = User()
        form.populate_obj(user)
        db.session.add(user)
        db.session.commit()
        login_user(user)
        return redirect(url_for('tracking.index'))
    return render_template('users/register.html', form=form)


@users.route('/logout/')
@login_required
def logout():
    # Tell Flask-Login to destroy the
    # session->User connection for this session.
    logout_user()
    return redirect(url_for('tracking.index'))
```

And import and register them with our application object:

```python
# flask_tracking/__init__.py
from .users.views import users

# ...

app.register_blueprint(users)
```

Notice the call to `load_user` inside of our `login` view. `Flask-Login` requires us to call this function in order to activate our user's session (which it will manage for us).

One last thing to look at is our `users/login.html` template:

{% raw %}
```python
{% extends "layout.html" %}
{% import "helpers/forms.html" as forms %}
{% block title %}Log into Flask Tracking!{% endblock %}
{% block content %}
{{super()}}
<form action="{{ url_for('users.login', ext=request.args.get('next', '')) }}" method="POST">
{{ forms.render(form) }}
<p><input type="Submit" value="Sign In"></p>
</form>
{% endblock content %}
```
{% endraw %}

We will cover the `layout.html` and `forms` macros in a little while - the key thing to note is that for our form's `action` we are explicitly passing in the value of the `next` parameter:

```python
url_for('users.login', next=request.args.get('next', ''))
```

This ensures that when the user submits the form to `users.login` the `next` parameter is available for our redirect code:

```python
login_user(form.user)
flash("Logged in successfully.")
return redirect(request.args.get("next") or url_for("tracking.index"))
```

There's a subtle security hole in this code, which we will be fixing in our next article (but points to you if you have already spotted it).

### Fighting Duplication

But wait! Did you see the pattern that we just repeated for the third time? (We actually repeated *at least* two patterns, but we are only going to remove the duplication from one of them today).  This part of the `register` code:

```python
user = User()
form.populate_obj(user)
db.session.add(user)
db.session.commit()
```

is also repeated multiple times in the `tracking` code.  Let's pull that database session behavior out using a custom mixin which we can borrow from [Flask-Kit][flask-kit]. Open `flask_tracking/data` and add the following code:

```python
class CRUDMixin(object):
    __table_args__ = {'extend_existing': True}

    id = db.Column(db.Integer, primary_key=True)

    @classmethod
    def create(cls, commit=True, **kwargs):
        instance = cls(**kwargs)
        return instance.save(commit=commit)

    @classmethod
    def get(cls, id):
        return cls.query.get(id)

    # We will also proxy Flask-SqlAlchemy's get_or_44
    # for symmetry
    @classmethod
    def get_or_404(cls, id):
        return cls.query.get_or_404(id)

    def update(self, commit=True, **kwargs):
        for attr, value in kwargs.iteritems():
            setattr(self, attr, value)
        return commit and self.save() or self

    def save(self, commit=True):
        db.session.add(self)
        if commit:
            db.session.commit()
        return self

    def delete(self, commit=True):
        db.session.delete(self)
        return commit and db.session.commit()
```

`CRUDMixin` provides us with an easier way of handling the four most common model operations (Create, Read, Update, and Delete):

```python
def create(cls, commit=True, **kwargs): pass

def get(cls, id): pass

def update(self, commit=True, **kwargs): pass

def delete(self, commit=True): pass
```

Now, if we update our `User` class to also subclass `CRUDMixin`:

```python
from flask_tracking.data import CRUDMixin, db

class User(UserMixin, CRUDMixin, db.Model):
```

we can then use the much clearer:

```python
user = User.create(**form.data)
```

call in our views. This makes it easier to reason about what our code is doing and makes it much easier to refactor (since each piece of code deals with fewer concerns).  We can also update our `tracking` package's code to make use of the same methods.

### Templates

In Part I, we skipped reviewing our templates to save time.  Let's take a couple of minutes now and review the more interesting parts of what we are using to render our HTML.

Later on, we might break these all up into a RESTful interface. Instead of having Python/Flask/Jinja serve up a pre-formatted page we could use a JavaScript MVC framework to handle the front-end and make requests to the backend to fetch the necessary data. The client would then send requests to the server to make/register new sites and be in charge of updating the views when new sites and visits are created. The views would then be responsible for the REST interface.

That said, since we are focusing on Flask, we will use Jinja to serve up the page for now.

### Layout

First, look at [`layout.html`][repo:part-2:layout] (I am leaving the majority of the code out of this article to save space, but I am providing links to the full code):

{% raw %}
```html
<title>{% block title %}{{ title }}{% endblock %}</title>
<!-- ... snip ... -->
<h1>{{ self.title() }}</h1>
```
{% endraw %}

This snippet showcases two of my favorite tricks - first, we have a block (`title`) that contains a variable so we can set this value from our `render_template` calls (so we don't need to create a whole new template just to change a title).  Second, we are *re-using* the contents of the block for our header with the special `self` variable.  This means, when we set `title` (either in a child template or via a keyword argument to `render_template`) the text we provide will show up *both* in the browser's title bar and in the `h1` tag.

### Form management

The other piece of our templating structure that merits a look is [our macros][repo:part-2:macros]. For those of you coming from a Django background, Jinja's macros are Django's `tag`s on steroids.  Our `form.render` macro, for example, makes it incredibly easy to add a form to one of our templates:

{% raw %}
```html
{% macro render(form) %}
<dl>
{% for field in form if field.type not in ["HiddenField", "CSRFTokenField"] %}
<dt>{{ field.label }}</dt>
<dd>{{ field }}
{% if field.errors %}
<ul class="errors">
{% for error in field.errors %}
<li>{{error}}</li>
{% endfor %}
</ul>
{% endif %}</dd>
{% endfor %}
</dl>
{{ form.hidden_tag() }}
{% endmacro %}
```
{% endraw %}

Using it is as simple as:

{% raw %}
```html
{% import "helpers/forms.html" as forms %}
<!-- ... snip ... -->
<form action="{{url_for('users.register')}}" method="POST">
{{ forms.render(form) }}
<p><input type="Submit" value="Learn more about your visitors"></p>
</form>
```
{% endraw %}

Instead of writing the same form HTML over and over again we can just use `form.render` to automatically generate the boilerplate HTML for each field in our forms. This way all of our forms will look and function in the same way and if we ever have to change them we only have to do it in once place. **Don't Repeat Yourself** makes for very clean code.

## Refactoring the tracking application

Now that we have all that set up properly, let's go back and refactor the meat of the application: *the request tracking*.

In Part I, we built the skeleton of a request tracker.  Sites were created on the index page and anyone could view *all* the available sites. As long as the end user sent all the information themselves, Flask-Tracking would store it happily. Now, we have users, so we want to filter the list of sites.  Additionally, it would be good if our application could derive some of the data from the visitor, rather than asking the end user of our application to derive it all for themselves.

### Filtering sites

Let's start with site list:

```python
# flask_tracking/tracking/views.py
@tracking.route("/sites", methods=("GET", "POST"))
@login_required
def view_sites():
    form = SiteForm()

    if form.validate_on_submit():
        Site.create(owner=current_user, **form.data)
        flash("Added site")
        return redirect(url_for(".view_sites"))

    query = Site.query.filter(Site.user_id == current_user.id)
    data = query_to_list(query)
    results = []

    try:
        # The header row should not be linked
        results = [next(data)]
        for row in data:
            row = [_make_link(cell) if i == 0 else cell
                   for i, cell in enumerate(row)]
            results.append(row)
    except StopIteration:
        # This happens when a user has no sites registered yet
        # Since it is expected, we ignore it and carry on.
        pass

    return render_template("tracking/sites.html", sites=results, form=form)


_LINK = Markup('<a href="{url}">{name}</a>')


def _make_link(site_id):
    url = url_for(".view_site_visits", site_id=site_id)
    return _LINK.format(url=url, name=site_id)
```

Starting from the top, the `@login_required` decorator is provided by `Flask-Login`. Anyone who isn't logged in who tries to go to `/sites/` will be redirected to the login page. Next, we are checking to see if the user is currently adding a new site (`form.validate_on_submit` checks to see if `request.method` is POST and validates the form - if either of the preconditions fails, the method returns `False`, otherwise it returns `True`).  If the user is creating a new site, we create a new site (using the method defined by our `CRUDMixin`, so if you are making changes to the code yourself, you will want to make sure that `Site` and `Visit` both inherit from `CRUDMixin`) and redirect back to the same page. We redirect back to ourselves after saving the new site to prevent a page refresh causing the user to attempt to add the site twice. (This is called the Post-Redirect-Get pattern).

If you are not sure what I mean by that, try commenting out the `return redirect(url_for(".view_sites"))`, then submit the "Add a Site" form and when the page reloads push `F5` to refresh your browser.  Try that same exercise after restoring the redirect. (When the redirect is removed the browser will ask if you really want to submit the form data again - the last request that the browser made is the POST that created the new site.  With the redirect, the last request that the browser made is the GET request that reloaded the `view_sites` page).

Continuing on, if the user is not creating a new site (or if the provided data has errors) we are querying our database to look up all of the sites that were created by the currently logged in user. We then slightly transform our list, turning the database ID into an HTML link for each of our non-header rows. This use of an "inline" template is good for fast prototyping, when you do not yet have a good idea of whether the template pattern is worth "macro-izing".  In our case, this is the only view we have with a table with an action link, so we use the inline template technique to demonstrate another way of doing things.

It is worth noting that we have elected to use `sites_view` for both displaying sites and their visits and for registering sites. It's really up to you how you want to break up your application.  Having a `view_sites` and an `add_site` view, where the former is only accessible to GET requests and the latter to POST is also a valid technique.  Whichever technique feels clearer to you is the one you should prefer - just make sure you are consistent.

### Deriving data from visitors

`add_visit`, meanwhile, is now a bit more complex (although it is mostly mapping code):

```python
from flask import request

from .geodata import get_geodata

# ... snip ...

@tracking.route("/sites/<int:site_id>/visit", methods=("GET", "POST"))
def add_visit(site_id=None):
    site = Site.get_or_404(site_id)

    browser = request.headers.get("User-Agent")
    url = request.values.get("url") or request.headers.get("Referer")
    event = request.values.get("event")
    ip_address = request.access_route[0] or request.remote_addr
    geodata = get_geodata(ip_address)
    location = "{}, {}".format(geodata.get("city"),
                               geodata.get("zipcode"))

    # WTForms does not coerce obj or keyword arguments
    # (otherwise, we could just pass in `site=site_id`)
    # CSRF is disabled in this case because we will *want*
    # users to be able to hit the /sites/{id}  endpoint from other sites.
    form = VisitForm(csrf_enabled=False,
                     site=site,
                     browser=browser,
                     url=url,
                     ip_address=ip_address,
                     latitude=geodata.get("latitude"),
                     longitude=geodata.get("longitude"),
                     location=location,
                     event=event)

    if form.validate():
        Visit.create(**form.data)
        # No need to send anything back to the client
        # Just indicate success with the response code
        # (204 is "Your request succeeded; I have nothing else to say.")
        return '', 204

    return jsonify(errors=form.errors), 400
```

We have removed the ability for users to manually add visits from our website via a form (and so we have also removed the second route on `add_visit`).  We now do explicit mapping for data that we can derive on the server (the browser, the IP Address) and then we construct our `VisitForm` passing in those mapped values directly.  The IP address we pull from `access_route` in case we are behind a proxy since then `remote_addr` will contain the IP address of the last proxy, which is not what we want at all. We disable CSRF protection because we actually *want* users to be able to make requests to this endpoint from elsewhere. Finally, we know what site this request is for because of the `<int:site_id>` parameter that we have set to the URL.

This is not a perfect implementation of this idea. We do not have any way of verifying that the request is a licit request from our tracking beacons. Someone could modify the JavaScript code or submit modified requests from another server entirely and we would happily save it. This is simple and it easy to implement. But you probably should not use this code in a production environment.

`get_geodata(ip_address)` queries `http://freegeoip.net/` so we can get a rough idea of where the requests are coming from:

```python
from json import loads
from re import compile, VERBOSE
from urllib import urlopen

FREE_GEOIP_URL = "http://freegeoip.net/json/{}"
VALID_IP = compile(r"""
\b
(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)
\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)
\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)
\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)
\b
""", VERBOSE)


def get_geodata(ip):
    """
    Search for geolocation information using http://freegeoip.net/
    """
    if not VALID_IP.match(ip):
        raise ValueError('Invalid IPv4 format')

    url = FREE_GEOIP_URL.format(ip)
    data = {}

    try:
        response = urlopen(url).read()
        data = loads(response)
    except Exception:
        pass

    return data
```

Save this as `geodata.py` in the `tracking` directory.

Return to the view, all this view is doing is copying info from the request down and storing it in the database. It responds to the request with an HTTP 204 (No Content) response. This tells the browser that the request succeeded, but we do not have to spend any extra time generating content that the end-user will not see.

### Seeing the visits

We also add authentication to the Visits view for each individual site:

```python
@tracking.route("/sites/<int:site_id>")
@login_required
def view_site_visits(site_id=None):
    site = Site.get_or_404(site_id)
    if not site.user_id == current_user.id:
        abort(401)

    query = Visit.query.filter(Visit.site_id == site_id)
    data = query_to_list(query)
    return render_template("tracking/site.html", visits=data, site=site)
```

The only real change here is that if the user is logged in, but does not own the site, they will see an authorization error page, rather than being able to view the visits for the site.

### Providing a means of tracking visitors

Finally, we want to provide users a snippet of code that they can place on their website that will automatically record visits:

{% raw %}
```html
{# flask_tracking/templates/tracking/site.html #}
{% block content %}
{{ super() }}
<p>To track visits to this site, simple add the following snippet to the pages that you wish to track:</p>
<code><pre>
&lt;script>
(function() {
  var img = new Image();
  img.src = "{{ url_for('tracking.add_visit', site_id=site.id, event='PageLoad', _external=true) }}";
})();
&lt;/script>
&lt;noscript>
&lt;img src="{{ url_for('tracking.add_visit', site_id=site.id, event='PageLoad', _external=true) }}" width="1" height="1" />
&lt;/noscript>
</pre></code>
<h2>Visits for {{ site.base_url }}</h2>
<table>
{{ tables.render(visits) }}
</table>
{% endblock content %}
```
{% endraw %}

Our snippet is very simple - when the page loads we create a new image and set its source to be our tracking URL.  The browser will *immediately* load the image specified (which will be nothing at all) and we will record a tracking hit in our application.  We also have a `<noscript>` block for those people who are visiting us without JavaScript enabled.  (If we really wanted to keep up with the times, we could also update our server-side code to check for the `Do Not Track` header and only record the visit if the user has opted into tracking.)

## Wrapping Up

That's it for this post. We now have user accounts, and the beginnings of an easy to use client-side API for tracking. We still need to finalize our client-side API, style the application and add reports.

The code for the application can be found [here][repository].

Your app should now look like this:

![flask-tracking-3](https://raw.github.com/mjhea0/flask-tracking/master/screenshots/flask-tracking-3.png)

**Looking ahead:**

In Part III we'll explore writing tests for our application, logging, and debugging errors.

In Part IV we'll do some Test Driven Development to enable our application to accept payments and display simple reports.

In Part V we will write a RESTful JSON API for others to consume.

In Part VI we will cover automating deployments (on Heroku) with Fabric and basic A/B Feature Testing.

Finally, in Part VII we will cover preserving your application for the future with documentation, code coverage and quality metric tools.

  [flask]: http://flask.pocoo.org/
  [repository]: https://github.com/mjhea0/flask-tracking
  [repository:releases]: https://github.com/mjhea0/flask-tracking/releases
  [repository:part-0]: https://github.com/mjhea0/flask-tracking/tree/part-0
  [repository:part-1]: https://github.com/mjhea0/flask-tracking/tree/part-1
  [app]: http://localhost:5000
  [series:part-1]: http://www.realpython.com/blog/python/python-web-applications-with-flask-part-i/
  [series:faq:what]: http://www.realpython.com/blog/python/python-web-applications-with-flask-part-i/#toc_0
  [series:faq:repo]: http://www.realpython.com/blog/python/python-web-applications-with-flask-part-i/#toc_5
  [series:faq:deps]: http://www.realpython.com/blog/python/python-web-applications-with-flask-part-i/#toc_7
  [flask-login]: http://flask-login.readthedocs.org/en/latest/
  [pbkdf2]: https://pypi.python.org/pypi/backports.pbkdf2/
  [login:methods]: https://flask-login.readthedocs.org/en/latest/#your-user-class
  [sqa:1-to-many]: http://pythonhosted.org/Flask-SQLAlchemy/models.html#one-to-many-relationships
  [flask-kit]: https://github.com/semirook/flask-kit
  [jinja]: http://jinja.pocoo.org/
  [repo:part-2:layout]: https://github.com/mjhea0/flask-tracking/blob/part-2/flask_tracking/templates/layout.html
  [repo:part-2:macros]: https://github.com/mjhea0/flask-tracking/tree/part-2/flask_tracking/templates/helpers
  [security]: http://security.stackexchange.com/a/6415/3231

