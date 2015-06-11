---
layout: post
title: "Using Flask-Login for User Management with Flask"
date: 2014-04-07 09:46:55 -0700
toc: true
comments: true
category_side_bar: true
categories: [python, flask]

keywords: "python, web development, flask, flask-login"
description: "This post details how to add an anaytics platform to the digital goods payment service 'bull' by Jeff Knupp."
---

*The following is a guest post by [Jeff Knupp](http://www.jeffknupp.com/), author of [Writing Idiomatic Python](http://www.jeffknupp.com/writing-idiomatic-python-ebook/). Jeff currently has a [Kickstarter](https://www.kickstarter.com/projects/1219760486/a-writing-idiomatic-python-video-series-watch-and) campaign running to turn the book into a video series - check it out!*

<hr>

A few months ago, I grew tired of the digital goods payment service I used to sell my book and decided to write my own. Two hours later, [bull](http://www.github.com/jeffknupp/bull/) was born.  It was a little application written using Flask and Python, which turned out to be an excellent choice for implementation. It started with bare bones functionality: A customer could enter their details in a Stripe JavaScript pop-up, `bull` would record their email address and create a unique id for the purchase, then associate the user with the content they purchased.

It worked fantastically well. Whereas before a potential customer had to not only, enter their full name and address (both of which I had no use for), they also had to *create an account on my payment processor's site*. I'm not sure how many sales I lost due to the convoluted checkout process, but I'm sure it was a good deal. With bull, the time between clicking the "Buy Now" button on the book sales page to actually reading the book was about 10 seconds. Customers loved it.

I loved it too, but for a slightly different reason: since `bull` was running on my web server, I could get a much richer set of analytics than if I had to send customers to a third-party site for payment. This opened the door to a host of new possibilities: A/B testing, analytics reports, custom sales reports. I was stoked.

## Adding Users

I decided that, at a minimum, I wanted `bull` to be able to display a "Sales Overview" page that contained basic sales data: transaction information, graphs of sales over time, etc. To do that (in a secure manner), I needed to add authentication and authorization to my little Flask app. Helpfully, though, I only needed to support a *single*, "admin" user who was authorized to view
reports.

Luckily, as is usually the case, a third-party package already existed to handle this. [Flask-login](https://flask-login.readthedocs.org/en/latest/) is a Flask extension that enables user authentication. All that's required is a `User` model and a few simple functions. Let's take a look at what was required.

### The `User` Model

`bull` was already using [Flask-sqlalchemy](http://pythonhosted.org/Flask-SQLAlchemy/) to create `purchase` and `product` models which captured the information about a sale and a product, respectively. Flask-login requires a `User` model with the following
properties:

* has an `is_authenticated()` method that returns `True` if the user has provided
  valid credentials
* has an `is_active()` method that returns `True` if the user's account is
  active
* has an `is_anonymous()` method that returns `True` if the current user is an
  anonymous user
* has a `get_id()` method which, given a `User` instance, returns the unique ID
  for that object

While Flask-login helpfully provides a `UserMixin` class that provides default implementations of all of these, I just defined everything required like so:

```python
#!py
from flask.ext.sqlalchemy import SQLAlchemy

db = SQLAlchemy()

#...

class User(db.Model):
    """An admin user capable of viewing reports.

    :param str email: email address of user
    :param str password: encrypted password for the user

    """
    __tablename__ = 'user'

    email = db.Column(db.String, primary_key=True)
    password = db.Column(db.String)
    authenticated = db.Column(db.Boolean, default=False)

    def is_active(self):
        """True, as all users are active."""
        return True

    def get_id(self):
        """Return the email address to satisfy Flask-Login's requirements."""
        return self.email

    def is_authenticated(self):
        """Return True if the user is authenticated."""
        return self.authenticated

    def is_anonymous(self):
        """False, as anonymous users aren't supported."""
        return False
```

### The `user_loader`

Flask-login also requires you to define a "user_loader" function which, given a user ID, returns the associated user object.

**Simple:**

```python
#!py
@login_manager.user_loader
def user_loader(user_id):
    """Given *user_id*, return the associated User object.

    :param unicode user_id: user_id (email) user to retrieve
    """
    return User.query.get(user_id)
```

The `@login_manager.user_loader` piece tells Flask-login how to load users given an id. I put this function in the file with all my routes defined, as that's where it's used.

### The `/reports` Endpoint

Now, I could create a `/reports` endpoint that required authentication. Here's what the code for that endpoint looks like:

```python
#!py
@bull.route('/reports')
@login_required
def reports():
    """Run and display various analytics reports."""
    products = Product.query.all()
    purchases = Purchase.query.all()
    purchases_by_day = dict()
    for purchase in purchases:
        purchase_date = purchase.sold_at.date().strftime('%m-%d')
        if purchase_date not in purchases_by_day:
            purchases_by_day[purchase_date] = {'units': 0, 'sales': 0.0}
        purchases_by_day[purchase_date]['units'] += 1
        purchases_by_day[purchase_date]['sales'] += purchase.product.price
    purchase_days = sorted(purchases_by_day.keys())
    units = len(purchases)
    total_sales = sum([p.product.price for p in purchases])

    return render_template(
            'reports.html',
            products=products,
            purchase_days=purchase_days,
            purchases=purchases,
            purchases_by_day=purchases_by_day,
            units=units,
            total_sales=total_sales)
```

You'll notice that most of the code is unrelated to authentication, which is just how it should be. The function assumes, due to the decorator, the user has already been authenticated and is thus authorized to see this data. There's only one problem: How does a user become "authenticated"?

### Login and Logout

Through a `/login` endpoint, of course! Both `/login` and `/logout` are straightforward and can be pulled almost verbatim from the Flask-login documentation:

```python
#!py
@bull.route("/login", methods=["GET", "POST"])
def login():
    """For GET requests, display the login form. For POSTS, login the current user
    by processing the form."""
    print db
    form = LoginForm()
    if form.validate_on_submit():
        user = User.query.get(form.email.data)
        if user:
            if bcrypt.check_password_hash(user.password, form.password.data):
                user.authenticated = True
                db.session.add(user)
                db.session.commit()
                login_user(user, remember=True)
                return redirect(url_for("bull.reports"))
    return render_template("login.html", form=form)

@bull.route("/logout", methods=["GET"])
@login_required
def logout():
    """Logout the current user."""
    user = current_user
    user.authenticated = False
    db.session.add(user)
    db.session.commit()
    logout_user()
    return render_template("logout.html")
```

You'll notice we update the `User` object in the database once they are authenticated. This is due to the fact that, from one request to the next, a new instance of the `User` object is created each time, so we need a place to store the information that the user already authenticated. Ditto for logging out.

### Creating an Admin User

Much like Django's `manage.py`, I needed a way to create a single admin user with the proper login credentials. I couldn't just add a row to the database manually, since the password is stored as a salted hash (rather than plain-text, which would be stupid from a security standpoint). I created the following script, `create_user.py`, to do just that:

```python
#!/usr/bin/env python
"""Create a new admin user able to view the /reports endpoint."""
from getpass import getpass
import sys

from flask import current_app
from bull import app, bcrypt
from bull.models import User, db

def main():
    """Main entry point for script."""
    with app.app_context():
        db.metadata.create_all(db.engine)
        if User.query.all():
            print 'A user already exists! Create another? (y/n):',
            create = raw_input()
            if create == 'n':
                return

        print 'Enter email address: ',
        email = raw_input()
        password = getpass()
        assert password == getpass('Password (again):')

        user = User(email=email, password=bcrypt.generate_password_hash(password))
        db.session.add(user)
        db.session.commit()
        print 'User added.'



if __name__ == '__main__':
    sys.exit(main())
```

Finally, I had a way to cordon off a section of the sales site to display sales data for an admin user. In my case, I only needed a single user, but Flask-login obviously supports many users at once.

## The Flask Ecosystem

My ability to quickly add this functionality to the site speaks to the rich ecosystem of Flask extensions that exist. Recently, I was looking to create a web application that had, among other things, a forum. Django has all sorts of complex forum applications you can get working with a good deal of effort, but none of them work well with the authentication application I had chosen; the two applications had no reason to be coupled together, but were.

Flask, on the other hand, makes it easy to compose orthogonal applications into a larger, more complex one in much the same way functions are composed in functional languages. Take, for example, [flask-forum](https://github.com/akprasad/flask-forum).
It uses the following Flask extensions in creating the forum:

* Flask-Admin for database management
* Flask-Assets for asset management
* Flask-DebugToolbar for debugging and profiling.
* Flask-Markdown for forum posts
* Flask-Script for basic commands
* Flask-Security for authentication
* Flask-SQLAlchemy for database queries
* Flask-WTF for forms

With a list that long, it's almost surprising that all of the applications are able to work together without creating dependencies on one another (or, rather, it's amazing if you're coming from Django), but Flask extensions usually follow the Unix philosophy of "do one thing well". I've yet to encounter a Flask extension that I would consider "bloated".

## Wrapping Up

While my use case and implementation were rather simple, the great part about Flask is that it makes simple things simple. If something seems like it should be easy to do and not take much time, with Flask, this is usually true. I was able to add an authenticated admin section of my payment processor in under an hour, and there was no magic involved. I knew how everything worked and fit together.

And that's a powerful concept: no magic. While many web frameworks pride themselves on how little the developer has to do to create an application, they fail to realize that the developer understands only what he writes and uses, and they might be doing developers a disservice by hiding so much. Flask puts it all in the open, and in the process, allows the functional composition of applications that Django set out to create.