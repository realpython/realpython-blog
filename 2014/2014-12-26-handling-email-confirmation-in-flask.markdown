---
layout: post
title: "Handling Email Confirmation during Registration in Flask"
date: 2014-12-26 08:44:03 -0600
toc: true
comments: true
category_side_bar: true
categories: [python, flask]

keywords: "python, flask, email confirmation, confirmation email, registration, authentication"
description: "This tutorial details how to validate email addresses during user registration."
---

**This tutorial details how to validate email addresses during user registration.**

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask-email-confirmation/flask-email-confirmation.png" style="max-width: 100%;" alt="flask email confirmation">
</div>

<br>

**Updated 04/30/2015**: Added Python 3 support.

<hr>

In terms of workflow, after a user registers a new account, a confirmation email is sent. The user account is marked as "unconfirmed" until the user, well, "confirms" the account via the instructions in the email. This is a simple workflow that most web applications follow.

One important thing to take into account is what unconfirmed users are allowed to do. In other words, do they have full access to your application, limited/restricted access, or no access at all? For the application in this tutorial, unconfirmed users can log in but they are immediately redirected to a page reminding them that they need to confirm their account before they can access the application.

> Before beginning, most of the functionality that we will be adding is part of the [Flask-User](http://pythonhosted.org/Flask-User/) and [Flask-Security](https://pythonhosted.org/Flask-Security/) extensions - which begs the question why not just use the extensions? Well, first and foremost, this is an opportunity to learn. Also, both of those extensions have limitations, like the supported databases. What if you wanted to use [RethinkDB](http://www.rethinkdb.com/), for example?

Let's begin.

## Flask basic registration

We're going to start with a Flask boilerplate that includes basic user registration. Grab the code from the [repository](https://github.com/mjhea0/flask-basic-registration). Once you've created and activated a virtualenv, run the following commands to quickly get started:

```sh
$ pip install -r requirements.txt
$ export APP_SETTINGS="project.config.DevelopmentConfig"
$ python manage.py create_db
$ python manage.py db init
$ python manage.py db migrate
$ python manage.py create_admin
$ python manage.py runserver
```

> Check out the [readme](https://github.com/mjhea0/flask-basic-registration#quickstart) for more information.

With the app running, navigate to [http://localhost:5000/register](http://localhost:5000/register) and register a new user. Notice that after registration, the app automatically logs you in and redirects you to the main page. Take a look around, then run through the code - specifically the "user" blueprint.

Kill the server when done.

## Update the current app

### Models

First, let's add the `confirmed` field to our `User` model in *project/models.py*:

```python
class User(db.Model):

    __tablename__ = "users"

    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String, unique=True, nullable=False)
    password = db.Column(db.String, nullable=False)
    registered_on = db.Column(db.DateTime, nullable=False)
    admin = db.Column(db.Boolean, nullable=False, default=False)
    confirmed = db.Column(db.Boolean, nullable=False, default=False)
    confirmed_on = db.Column(db.DateTime, nullable=True)

    def __init__(self, email, password, confirmed,
                 paid=False, admin=False, confirmed_on=None):
        self.email = email
        self.password = bcrypt.generate_password_hash(password)
        self.registered_on = datetime.datetime.now()
        self.admin = admin
        self.confirmed = confirmed
        self.confirmed_on = confirmed_on
```

Notice how this field defaults to 'False'. We also added a `confirmed_on` field, which is a `datetime`. I like to include this field as well in order to analyze the difference between the `registered_on` and `confirmed_on` dates using [cohort analysis](http://mherman.org/blog/2012/11/16/the-benefits-of-performing-a-cohort-analysis-in-determining-engagement-over-time/#.VJS2tcADA).

Let's completely start over with our database and migrations. So, go ahead and delete the database, *dev.sqlite*, as well as the "migrations" folder.

### Manage command

Next, within *manage.py*, update the `create_admin` command to take the new database fields into account:

```python
@manager.command
def create_admin():
    """Creates the admin user."""
    db.session.add(User(
        email="ad@min.com",
        password="admin",
        admin=True,
        confirmed=True,
        confirmed_on=datetime.datetime.now())
    )
    db.session.commit()
```

Make sure to import `datetime`. Now, go ahead and run the following commands again:

```sh
$ python manage.py create_db
$ python manage.py db init
$ python manage.py db migrate
$ python manage.py create_admin
```

### `register()` view function

Finally, before we can register a user again, we need to make a quick change to the `register()` view function in *project/user/views.py*...


Change:

```python
user = User(
    email=form.email.data,
    password=form.password.data
)
```

To:

```python
user = User(
    email=form.email.data,
    password=form.password.data,
    confirmed=False
)
```

Make sense? Think about why we would want to default `confirmed` to `False`.

Okay. Run the app again. Navigate to [http://localhost:5000/register](http://localhost:5000/register) and register a new user again. If you open your SQLite database in the SQLite Browser, you should see:

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask-email-confirmation/user_registration.png" style="max-width: 100%;" alt="user registration">
</div>

<br>

So, the new user that I registered, `michael@realpython.com`, is not confirmed. Let's change that.

## Add email confirmation

### Generate confirmation token

The email confirmation should contain a unique URL that a user simply needs to click in order to confirm his/her account. Ideally, the URL should look something like this - `http://yourapp.com/confirm/<id>`. The key here is the `id`. We are going to encode the user email (along with a timestamp) in the `id` using the [itsdangerous](http://pythonhosted.org/itsdangerous/) package.

Create a file called *project/token.py* and add the following code:

```python
# project/token.py

from itsdangerous import URLSafeTimedSerializer

from project import app


def generate_confirmation_token(email):
    serializer = URLSafeTimedSerializer(app.config['SECRET_KEY'])
    return serializer.dumps(email, salt=app.config['SECURITY_PASSWORD_SALT'])


def confirm_token(token, expiration=3600):
    serializer = URLSafeTimedSerializer(app.config['SECRET_KEY'])
    try:
        email = serializer.loads(
            token,
            salt=app.config['SECURITY_PASSWORD_SALT'],
            max_age=expiration
        )
    except:
        return False
    return email
```

So, in the `generate_confirmation_token()` function we use the `URLSafeTimedSerializer` to generate a token using the email address obtained during user registration. The *actual* email is encoded in the token. Then to confirm the token, within the `confirm_token()` function, we can use the `loads()` method, which takes the token and expiration - valid for one hour (3,600 seconds) - as arguments. As long as the token has not expired, then it will return an email.

Be sure to add the `SECURITY_PASSWORD_SALT` to your app's config (`BaseConfig()`):

```python
SECURITY_PASSWORD_SALT = 'my_precious_two'
```

### Update `register()` view function

Now let's update the `register()` view function again from *project/user/views.py*:

```python
@user_blueprint.route('/register', methods=['GET', 'POST'])
def register():
    form = RegisterForm(request.form)
    if form.validate_on_submit():
        user = User(
            email=form.email.data,
            password=form.password.data,
            confirmed=False
        )
        db.session.add(user)
        db.session.commit()

        token = generate_confirmation_token(user.email)
```

Also, make sure to update the imports:

```python
from project.token import generate_confirmation_token, confirm_token
```

### Handle Email Confirmation

Next, let's add a new view to handle the email confirmation:

```python
@user_blueprint.route('/confirm/<token>')
@login_required
def confirm_email(token):
    try:
        email = confirm_token(token)
    except:
        flash('The confirmation link is invalid or has expired.', 'danger')
    user = User.query.filter_by(email=email).first_or_404()
    if user.confirmed:
        flash('Account already confirmed. Please login.', 'success')
    else:
        user.confirmed = True
        user.confirmed_on = datetime.datetime.now()
        db.session.add(user)
        db.session.commit()
        flash('You have confirmed your account. Thanks!', 'success')
    return redirect(url_for('main.home'))
```

Add this to *project/user/views.py*. Also, be sure to update the imports:

```python
import datetime
```

Here, we call the `confirm_token()` function, passing in the token. If successful, we update the user, changing the `email_confirmed` attribute to `True` and setting the `datetime` for when the confirmation took place. Also, in case the user already went through the confirmation process - and is confirmed - then we alert the user of this.

### Create the email template

Next, let's add a base email template:

{% raw %}
```html
<p>Welcome! Thanks for signing up. Please follow this link to activate your account:</p>
<p><a href="{{ confirm_url }}">{{ confirm_url }}</a></p>
<br>
<p>Cheers!</p>
```
{% endraw %}

Save this as *activate.html* in "project/templates/user". This take a single variable called `confirm_url`, which will be created in the `register()` view function.

### Send email

Let's create a basic function for sending emails with a little help from [Flask-Mail](https://pythonhosted.org/flask-mail/), which is already installed and setup in `project/__init__.py`.

Create a file called *email.py*:

```python
# project/email.py

from flask.ext.mail import Message

from project import app, mail


def send_email(to, subject, template):
    msg = Message(
        subject,
        recipients=[to],
        html=template,
        sender=app.config['MAIL_DEFAULT_SENDER']
    )
    mail.send(msg)
```

Save this in the "project" folder.

So, we simply need to pass a list of recipients, a subject, and a template. We'll deal with the mail configuration settings in a bit.


### Update `register()` view function in project/user/views.py (again!)

```python
@user_blueprint.route('/register', methods=['GET', 'POST'])
def register():
    form = RegisterForm(request.form)
    if form.validate_on_submit():
        user = User(
            email=form.email.data,
            password=form.password.data,
            confirmed=False
        )
        db.session.add(user)
        db.session.commit()

        token = generate_confirmation_token(user.email)
        confirm_url = url_for('user.confirm_email', token=token, _external=True)
        html = render_template('user/activate.html', confirm_url=confirm_url)
        subject = "Please confirm your email"
        send_email(user.email, subject, html)

        login_user(user)

        flash('A confirmation email has been sent via email.', 'success')
        return redirect(url_for("main.home"))

    return render_template('user/register.html', form=form)
```

Add the following import as well:

```python
from project.email import send_email
```

**Here, we are putting everything together. This function basically acts as a controller (either directly or indirectly) for the entire process:**

- Handle initial registration,
- Generate token and confirmation URL,
- Send confirmation email,
- Flash confirmation,
- Log in the user, and
- Redirect user.

Did you notice the `_external=True` argument? This adds the full absolute URL that includes the hostname and port ([http://localhost:5000](http://localhost:5000), in our case.)

Before we can test this out, we need to set up our mail settings.

### Mail

Start by updating the `BaseConfig()` in *project/config.py*:

```python
class BaseConfig(object):
    """Base configuration."""

    # main config
    SECRET_KEY = 'my_precious'
    SECURITY_PASSWORD_SALT = 'my_precious_two'
    DEBUG = False
    BCRYPT_LOG_ROUNDS = 13
    WTF_CSRF_ENABLED = True
    DEBUG_TB_ENABLED = False
    DEBUG_TB_INTERCEPT_REDIRECTS = False

    # mail settings
    MAIL_SERVER = 'smtp.googlemail.com'
    MAIL_PORT = 465
    MAIL_USE_TLS = False
    MAIL_USE_SSL = True

    # gmail authentication
    MAIL_USERNAME = os.environ['APP_MAIL_USERNAME']
    MAIL_PASSWORD = os.environ['APP_MAIL_PASSWORD']

    # mail accounts
    MAIL_DEFAULT_SENDER = 'from@example.com'
```

> Check out the [official Flask-Mail documentation](https://pythonhosted.org/flask-mail/#configuring-flask-mail) for more info.

If you already have a GMAIL account then you can use that or register a test GMAIL account. Then set the environment variables temporarily in the current shell session:

```sh
$ export APP_MAIL_USERNAME="foo"
$ export APP_MAIL_PASSWORD="bar"
```
> If your GMAIL account has [2-step authentication](https://support.google.com/accounts/topic/28786?hl=en&ref_topic=3382253), Google will block the attempt.

Now let's test!

## First test

Fire up the app, and navigate to [http://localhost:5000/register](http://localhost:5000/register). Then register with an email address that you have access to. If all went well, you should have an email in your inbox that looks something like this:

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask-email-confirmation/email_confirmation.png" style="max-width: 100%;" alt="email confirmation">
</div>

<br>

Click the URL and you should be taken to [http://localhost:5000/](http://localhost:5000/). Make sure that the user is in the database, the 'confirmed' field is `True`, and there is a `datetime` associated with the `confirmed_on` field.

Nice!

## Handle permissions

If you remember, at the beginning of this tutorial, we decided that "unconfirmed users can log in but they should be immediately redirected to a page - let's call the route `/unconfirmed` - reminding users that they need to confirm their account before they can access the application".

So, we need to-

1. Add the `/unconfirmed` route
1. Add an *unconfirmed.html* template
1. Update the `register()` view function
1. Create a decorator
1. Update *navigation.html* template

### Add `/unconfirmed` route

Add the following route to *project/user/views.py*:

```python
@user_blueprint.route('/unconfirmed')
@login_required
def unconfirmed():
    if current_user.confirmed:
        return redirect('main.home')
    flash('Please confirm your account!', 'warning')
    return render_template('user/unconfirmed.html')
```

You've seen similar code before, so let's move on.

### Add *unconfirmed.html* template

{% raw %}
```html
{% extends "_base.html" %}

{% block content %}

<h1>Welcome!</h1>
<br>
<p>You have not confirmed your account. Please check your inbox (and your spam folder) - you should have received an email with a confirmation link.</p>
<p>Didn't get the email? <a href="/">Resend</a>.</p>

{% endblock %}
```
{% endraw %}

Save this as *unconfirmed.html* in "project/templates/user". Again, this should all be straightforward. For now, we just added a dummy URL in for resending the confirmation email. We'll address this further down.

### Update the `register()` view function

Now simply change:

```python
return redirect(url_for("main.home"))
```

To:

```python
return redirect(url_for("user.unconfirmed"))
```

So, after the confirmation email is sent, the user is now redirected to the `/unconfirmed` route.

### Create a decorator

```python
# project/decorators.py


from functools import wraps

from flask import flash, redirect, url_for
from flask.ext.login import current_user


def check_confirmed(func):
    @wraps(func)
    def decorated_function(*args, **kwargs):
        if current_user.confirmed is False:
            flash('Please confirm your account!', 'warning')
            return redirect(url_for('user.unconfirmed'))
        return func(*args, **kwargs)

    return decorated_function
```

Here we have a basic function to check if a user is unconfirmed. If unconfirmed, the user is redirected to the `/unconfirmed` route. Save this as *decorators.py* in the "project" directory.

Now, decorate the `profile()` view function:

```python
@user_blueprint.route('/profile', methods=['GET', 'POST'])
@login_required
@check_confirmed
def profile():
    ... snip ...
```

Make sure to import the decorator:

```python
from project.decorators import check_confirmed
```

### Update *navigation.html* template

Finally, update the following part of the *navigation.html* template-

Change:

{% raw %}
```html
<ul class="nav navbar-nav">
  {% if current_user.is_authenticated() %}
    <li><a href="{{ url_for('user.profile') }}">Profile</a></li>
  {% endif %}
</ul>
```
{% endraw %}

To:

{% raw %}
```html
<ul class="nav navbar-nav">
  {% if current_user.confirmed and current_user.is_authenticated() %}
    <li><a href="{{ url_for('user.profile') }}">Profile</a></li>
  {% elif current_user.is_authenticated() %}
    <li><a href="{{ url_for('user.unconfirmed') }}">Confirm</a></li>
  {% endif %}
</ul>
```
{% endraw %}

Time to test again!

## Second test

Fire up the app, and register again with an email address that you have access to. (Feel free to delete the old user that you registered before first from the database to use again.) Now you should be redirected to [http://localhost:5000/unconfirmed](http://localhost:5000/unconfirmed) after registration.

Make sure to test the [http://localhost:5000/profile](http://localhost:5000/profile) route. This should redirect you to [http://localhost:5000/unconfirmed](http://localhost:5000/unconfirmed).

Go ahead and confirm the email, and you will have access to all pages. Boom!

## Resend email

Finally, let's get the resend link working. Add the following view function to *project/user/views.py*:

```python
@user_blueprint.route('/resend')
@login_required
def resend_confirmation():
    token = generate_confirmation_token(current_user.email)
    confirm_url = url_for('user.confirm_email', token=token, _external=True)
    html = render_template('user/activate.html', confirm_url=confirm_url)
    subject = "Please confirm your email"
    send_email(current_user.email, subject, html)
    flash('A new confirmation email has been sent.', 'success')
    return redirect(url_for('user.unconfirmed'))
```

Now update the *unconfirmed.html* template:

{% raw %}
```html
{% extends "_base.html" %}

{% block content %}

<h1>Welcome!</h1>
<br>
<p>You have not confirmed your account. Please check your inbox (and your spam folder) - you should have received an email with a confirmation link.</p>
<p>Didn't get the email? <a href="{{ url_for('user.resend_confirmation') }}">Resend</a>.</p>

{% endblock %}
```
{% endraw %}

## Third test

You know the drill. This time make sure to resend a new confirmation email and test the link. It should work.

Finally, what happens if you send yourself a few confirmation links? Are each valid? Test it out. Register a new user, and then send a few new confirmation emails. Try to confirm with the first email. Did it work? It should. Is this okay? Do you think those other emails should expire if a new one is sent?

Do some research on this. And test out other web applications that you use. How do they handle such behavior?

## Update test suite

Alright. So that's it for the main functionality. How about we update the current test suite since it's, well, broken.

Run the tests:

```sh
$ python manage.py test
```

You should see the following error:


```sh
TypeError: __init__() takes at least 4 arguments (3 given)
```

To correct this we just need to update the `setUp()` method in *project/util.py*:

```python
def setUp(self):
    db.create_all()
    user = User(email="ad@min.com", password="admin_user", confirmed=False)
    db.session.add(user)
    db.session.commit()
```

Now run the tests again. All should pass!

## Conclusion

There's clearly a lot more we can do:

1. Rich vs. plain text emails - We should be sending out both.
1. Reset password email - These should be sent out for users that have forgotten their passwords.
1. User management - We should allow users to update their emails and passwords, and when an email is changed, it should be confirmed again.
1. [Testing - We need to write more tests to cover the new features.](https://realpython.com/blog/python/the-minimum-viable-test-suite/)

<br>

Download the entire source code from the [Github repository](https://github.com/realpython/flask-registration). Comment below with questions. Check out [part 2](https://realpython.com/blog/python/the-minimum-viable-test-suite/).

Happy holidays!

<br>

<p style="font-size: 14px;">
  <em>Edits made by <a href="https://twitter.com/diek007">Derrick Kearney</a>. Thank you!</em>
</p>
