# Django 1.6 Test Driven Development

### Last Updated:

1. 01/29/2014 - updated code for the form (thanks vic!)
2. 12/29/2013 - restructured entire blog post

<hr>

Test Driven Development (TDD) is an iterative development cycle that emphasizes writing automated tests before writing the actual code.

The process is simple:

1. Write your tests first.
2. Watch them fail.
3. Write just enough code to make those tests pass.
4. Test again.
5. Refactor.
6. Repeat.

![tdd-process](https://raw.github.com/mjhea0/flaskr-tdd/master/static/tdd.png)

## Why TDD?

With TDD, you'll learn to break code up into logical, easily understandable pieces, helping ensure the correctness of code.

This is important because it's hard to-

1. Solve complex problems all at once in our heads;
2. Know when and where to start working on a problem;
3. Increase the complexity of a codebase without introducing errors and bugs; and
4. Recognize when code breaks occur.

TDD helps address these issues. It in no way guarantees that your code will be error free; however, you will write better code, which results in a better understanding of the code. This in itself will help with eliminating errors and at the very least, you will be able to address errors much easier.

TDD is practically an industry standard as well.

Enough talk. Let's get to the code.

For this tutorial, we'll be creating an app to store user contacts.

> Please note: This tutorial assumes you are running a Unix-based environment - e.g, Mac OSX, straight Linux, or Linux VM through Windows. I will also be using Sublime 2 as my text editor. Also, make sure you you've completed the official Django [tutorial](https://docs.djangoproject.com/en/1.6/intro/tutorial01/) and have a basic understanding of the Python language. **Also, in this first post, we will not yet get into some of the new tools available in Django 1.6. This post sets the foundation for subsequent posts that deal with different forms of testing.**

## First Test

Before we do anything we need to first setup a test. For this test we just want to make that Django is properly setup. We're going to be using a functional test for this - which we'll be explained further down.

Create a new directory to hold your project:

```sh
$ mkdir django-tdd
$ cd django-tdd
```

Now setup a new directory to hold your functional tests:

```sh
$ mkdir ft
$ cd ft
```

Create a new file called "tests.py" and add the following code:

```python
from selenium import webdriver

browser = webdriver.Firefox()
browser.get('http://localhost:8000/')

body = browser.find_element_by_tag_name('body')
assert 'Django' in body.text

browser.quit()
```

Now run the test:

```sh
$ python tests.py
```

> Make sure you have selenium installed - `pip install selenium`

You should see FireFox pop up and attempt to navigate to http://localhost:8000/. In your terminal you should see:

```sh
Traceback (most recent call last):
File "tests.py", line 7, in <module>
  assert 'Django' in body.text
AssertionError
```

Congrats! You wrote your first failing test.

Now let's write just enough code to make it pass, which simply amounts to setting up a Django development environment.

## Setup Django

Activate a virtualenv:

```sh
$ cd ..
$ virtualenv --no-site-packages env
$ source env/bin/activate
```

Install Django and setup a Project:

```sh
$ pip install django==1.6.1
$ django-admin.py startproject contacts
```

Your current project structure should look like this:

```sh
├── contacts
│   ├── contacts
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   └── manage.py
└── ft
    └── tests.py
```

Install Selenium:

```sh
pip install selenium==2.39.0
```

Run the server:

```sh
$ cd contacts
$ python manage.py runserver
```

Next, open up a new window in your terminal, navigate to the "ft" directory, then run the test again:

```sh
$ python tests.py
```

You should see the FireFox window navigate to http://localhost:8000/ again. This time there should be no errors. Nice. You just passed your first test!

Now, let's finish setting up our dev environment.

## Version Control

First, add a ".gitignore" and include the following code in the file:

```sh
.Python
env
bin
lib
include
.DS_Store
.pyc
```

Now create a Git repository and commit:

```sh
$ git init
$ git add .
$ git commit -am "initial"
```

With the project setup, let's take a step back and discuss functional tests.

## Functional Tests

We approached the first test through Selenium via Functional tests. Such tests let us drive the web browser as if we were the end user, to see how the application actually *functions*. Since these tests follow the behavior of the end user - also called a User Story - it involves the testing of a number of features, rather than just a single function - which is more appropriate for Unit tests. **It's important to note that when testing code you have not written, you should begin with functional tests.** Since we are essentially testing Django code, functional tests are the right way to go.

> Another way to think about functional vs unit tests is that Functional tests focus on testing the app from the outside, from the user's perspective, while unit tests focus on the app from the inside, from the developer's perspective.

This will make much more sense in practice.

Before moving on, let's restructure our testing environment to make testing easier.

First, let's re-write the first test in the "tests.py" file:

```python
from selenium import webdriver
from selenium.webdriver.common.keys import Keys

from django.test import LiveServerTestCase

class AdminTest(LiveServerTestCase):

    def setUp(self):
        self.browser = webdriver.Firefox()

    def tearDown(self):
        self.browser.quit()

    def test_admin_site(self):
        # user opens web browser, navigates to admin page
        self.browser.get(self.live_server_url + '/admin/')
        body = self.browser.find_element_by_tag_name('body')
        self.assertIn('Django administration', body.text)
```

Then run it:

```sh
$ python manage.py test ft
```

It should pass:

```sh
----------------------------------------------------------------------
Ran 1 test in 3.304s

OK
```

Congrats!

 Before moving on, let's see what's going on here. If all went well it should pass. You should also see FireFox open and go through the process we indicated in the test with the `setUp()` and `tearDown()` functions. The test itself is simply testing whether the "/admin" (`self.browser.get(self.live_server_url + '/admin/'`) page can be found and that the words "Django administration" are present in the body tag.

 Let's confirm this.

Run the server:

```sh
$ python manage.py runserver
```

Then navigate to [http://localhost:8000/admin/](http://localhost:8000/admin/) in your browser and you should see:

![admin-page](https://raw.github.com/mjhea0/django-tdd/master/admin.png)

We can confirm that the test is working correctly by simply testing for the wrong thing. Update the last line in the test to:

```python
self.assertIn('administration Django', body.text)
```

Run it again. You should see the following error (which is expected, of course):

```sh
AssertionError: 'administration Django' not found in u'Django administration\nUsername:\nPassword:\n '
```

Correct the test. Test it again. Commit the code.

Finally, did you notice that we started the function name for the actual test began with `test_`. This is so that the Django test runner can find the test. In other words, any function that begins with `test_` will be treated as a test by the test runner.

### Admin Login

Next, let's test to make sure the user can login to the admin site.

Update `test_admin_site` the function in "tests.py":

```python
def test_admin_site(self):
    # user opens web browser, navigates to admin page
    self.browser.get(self.live_server_url + '/admin/')
    body = self.browser.find_element_by_tag_name('body')
    self.assertIn('Django administration', body.text)
    # users types in username and passwords and presses enter
    username_field = self.browser.find_element_by_name('username')
    username_field.send_keys('admin')
    password_field = self.browser.find_element_by_name('password')
    password_field.send_keys('admin')
    password_field.send_keys(Keys.RETURN)
    # login credentials are correct, and the user is redirected to the main admin page
    body = self.browser.find_element_by_tag_name('body')
    self.assertIn('Site administration', body.text)
```

So -

- `find_element_by_name` - is used for locating the input fields
- `send_keys` - sends keystrokes

Run the test. You should see this error:

```sh
AssertionError: 'Site administration' not found in u'Django administration\nPlease enter the correct username and password for a staff account. Note that both fields may be case-sensitive.\nUsername:\nPassword:\n '
```

This failed because we don't have an admin user setup. This is an expected failure, which is good. In other words, we knew it would fail - which makes it much easier to fix.


Sync the database:

```sh
$ python manage.py syncdb
```

Setup an admin user.

Test again. It should fail again. Why? Django creates a copy of our database when tests are ran so that way tests do not affect the production database.

We need to setup a Fixture, which is a file containing data we want loaded into the test database: the login credentials. To do that, run these commands to dump the admin user info from the database to the Fixture:

 ```sh
 $ mkdir ft/fixtures
 $ python manage.py dumpdata auth.User --indent=2 > ft/fixtures/admin.json
 ```

 Now update the `AdminTest` class:

```python
class AdminTest(LiveServerTestCase):

    # load fixtures
    fixtures = ['admin.json']

    def setUp(self):
        self.browser = webdriver.Firefox()

    def tearDown(self):
        self.browser.quit()

    def test_admin_site(self):
        # user opens web browser, navigates to admin page
        self.browser.get(self.live_server_url + '/admin/')
        body = self.browser.find_element_by_tag_name('body')
        self.assertIn('Django administration', body.text)
        # users types in username and passwords and presses enter
        username_field = self.browser.find_element_by_name('username')
        username_field.send_keys('admin')
        password_field = self.browser.find_element_by_name('password')
        password_field.send_keys('admin')
        password_field.send_keys(Keys.RETURN)
        # login credentials are correct, and the user is redirected to the main admin page
        body = self.browser.find_element_by_tag_name('body')
        self.assertIn('Site administration', body.text)
```

Run the test. It should pass.

> Each time a test is ran, Django dumps the test database. Then all the Fixtures specified in the "test.py" file are loaded into the database.

Let's add one more assert. Update the test again:

```python
def test_admin_site(self):
    # user opens web browser, navigates to admin page
    self.browser.get(self.live_server_url + '/admin/')
    body = self.browser.find_element_by_tag_name('body')
    self.assertIn('Django administration', body.text)
    # users types in username and passwords and presses enter
    username_field = self.browser.find_element_by_name('username')
    username_field.send_keys('admin')
    password_field = self.browser.find_element_by_name('password')
    password_field.send_keys('admin')
    password_field.send_keys(Keys.RETURN)
    # login credentials are correct, and the user is redirected to the main admin page
    body = self.browser.find_element_by_tag_name('body')
    self.assertIn('Site administration', body.text)
    # user clicks on the Users link
    user_link = self.browser.find_elements_by_link_text('Users')
    user_link[0].click()
    # user verifies that user live@forever.com is present
    body = self.browser.find_element_by_tag_name('body')
    self.assertIn('live@forever.com', body.text)
```

Run it. It should fail, because we need to and another user to the fixture file:

```javascript
[
{
  "pk": 1,
  "model": "auth.user",
  "fields": {
    "username": "admin",
    "first_name": "",
    "last_name": "",
    "is_active": true,
    "is_superuser": true,
    "is_staff": true,
    "last_login": "2013-12-29T03:49:13.545Z",
    "groups": [],
    "user_permissions": [],
    "password": "pbkdf2_sha256$12000$VtsgwjQ1BZ6u$zwnG+5E5cl8zOnghahArLHiMC6wGk06HXrlAijFFpSA=",
    "email": "ad@min.com",
    "date_joined": "2013-12-29T03:49:13.545Z"
  }
},
{
  "pk": 2,
  "model": "auth.user",
  "fields": {
    "username": "live",
    "first_name": "",
    "last_name": "",
    "is_active": true,
    "is_superuser": false,
    "is_staff": false,
    "last_login": "2013-12-29T03:49:13.545Z",
    "groups": [],
    "user_permissions": [],
    "password": "pbkdf2_sha256$12000$VtsgwjQ1BZ6u$zwnG+5E5cl8zOnghahArLHiMC6wGk06HXrlAijFFpSA=",
    "email": "live@forever.com",
    "date_joined": "2013-12-29T03:49:13.545Z"
  }
}
]
```

Run it again. Make sure it passes. Refactor the test if needed. Now think about what else you could test. Perhaps you could test to make sure the admin user can add a user in the Admin panel. Or perhaps a test to ensure that someone without admin access cannot access the Admin panel. Write a few more tests. Update your code. Test again. Refactor if necessary.

Next, we're going to add the app for adding contacts. Don't forget to commit!

### Setup the Contacts App

Start with a test. Add the following function:

```python
def test_create_contact_admin(self):
    self.browser.get(self.live_server_url + '/admin/')
    username_field = self.browser.find_element_by_name('username')
    username_field.send_keys('admin')
    password_field = self.browser.find_element_by_name('password')
    password_field.send_keys('admin')
    password_field.send_keys(Keys.RETURN)
    # user verifies that user_contacts is present
    body = self.browser.find_element_by_tag_name('body')
    self.assertIn('User_Contacts', body.text)
```

Run the test suite again. You should see the following error-

```sh
AssertionError: 'User_Contacts' not found in u'Django administration\nWelcome, admin. Change password / Log out\nSite administration\nAuth\nGroups\nAdd\nChange\nUsers\nAdd\nChange\nRecent Actions\nMy Actions\nNone available'
```

 -which is expected.

 Now, write just enough code for this to pass.

Create the App:

```sh
$ python manage.py startapp user_contacts
```

Add it to the "settings.py" file:

```python
INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'ft',
    'user_contacts',
)
```

Within the "admin.py" file in the `user_contacts` directory add the following code:

```python
from user_contacts.models import Person, Phone
from django.contrib import admin

admin.site.register(Person)
admin.site.register(Phone)
```

Your project structure should now look like this:

```sh
.
├── user_contacts
│   ├── __init__.py
│   ├── admin.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── contacts
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── ft
│   ├── __init__.py
│   ├── fixtures
│   │   └── admin.json
│   └── tests.py
└── manage.py
```

Update "models.py":

```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length = 30)
    last_name = models.CharField(max_length = 30)
    email = models.EmailField(null = True, blank = True)
    address = models.TextField(null = True, blank = True)
    city = models.CharField(max_length = 15, null = True,blank = True)
    state = models.CharField(max_length = 15, null = True, blank = True)
    country = models.CharField(max_length = 15, null = True, blank = True)

    def __unicode__(self):
        return self.last_name +", "+ self.first_name

class Phone(models.Model):
    person = models.ForeignKey('Person')
    number = models.CharField(max_length=10)

    def __unicode__(self):
        return self.number
```

Run the test again now. You should now see:

```sh
Ran 2 tests in 11.730s

OK
```

Let's go ahead and add to the test to make sure the admin can add data:

```sh
# user clicks on the Persons link
persons_links = self.browser.find_elements_by_link_text('Persons')
persons_links[0].click()
# user clicks on the Add person link
add_person_link = self.browser.find_element_by_link_text('Add person')
add_person_link.click()
# user fills out the form
self.browser.find_element_by_name('first_name').send_keys("Michael")
self.browser.find_element_by_name('last_name').send_keys("Herman")
self.browser.find_element_by_name('email').send_keys("michael@realpython.com")
self.browser.find_element_by_name('address').send_keys("2227 Lexington Ave")
self.browser.find_element_by_name('city').send_keys("San Francisco")
self.browser.find_element_by_name('state').send_keys("CA")
self.browser.find_element_by_name('country').send_keys("United States")
# user clicks the save button
self.browser.find_element_by_css_selector("input[value='Save']").click()
# the Person has been added
body = self.browser.find_element_by_tag_name('body')
self.assertIn('Herman, Michael', body.text)
# user returns to the main admin screen
home_link = self.browser.find_element_by_link_text('Home')
home_link.click()
# user clicks on the Phones link
persons_links = self.browser.find_elements_by_link_text('Phones')
persons_links[0].click()
# user clicks on the Add phone link
add_person_link = self.browser.find_element_by_link_text('Add phone')
add_person_link.click()
# user finds the person in the dropdown
el = self.browser.find_element_by_name("person")
for option in el.find_elements_by_tag_name('option'):
    if option.text == 'Herman, Michael':
        option.click()
# user adds the phone numbers
self.browser.find_element_by_name('number').send_keys("4158888888")
# user clicks the save button
self.browser.find_element_by_css_selector("input[value='Save']").click()
# the Phone has been added
body = self.browser.find_element_by_tag_name('body')
self.assertIn('4158888888', body.text)
# user logs out
self.browser.find_element_by_link_text('Log out').click()
body = self.browser.find_element_by_tag_name('body')
self.assertIn('Thanks for spending some quality time with the Web site today.', body.text)
```

That's it for the admin functionality. Let's switch gears and focus on the application, `user_contacts`, itself. Did you forget to commit? If so, do it now.

## Unit Tests

Think about the features we have written thus far. We've just defined our model and allowed admins to alter the model. Based on that, and the overall goal of our Project, focus on the remaining user functionalities.

Users should be able to-

1. View all contacts.
2. Add new contacts.

Try to formulate the remaining Functional test(s) based on those requirements. Before we write the functional tests, though, we should define the behavior of the code through [unit tests](http://docs.python.org/2/library/unittest.html) - which will help you write good, clean code, making it easier to write the Functional tests.

> **Remember**: Functional tests are the ultimate indicator of whether your Project works or not, while Unit tests are the means to help you reach that end. This will all make sense soon.

Let's pause for a minute and talk about some conventions.

Although the basics of TDD (or ends) - test, code, refactor - are universal, many developers approach the means differently. For example, I like to write my unit tests first, to ensure that my code works at a granular level, then write the functional tests. Others write functional tests first, watch them fail, then write unit tests, watch them fail, then write code to first satisfy the unit tests, which should ultimately satisfy the functional tests. There's no right or wrong answer here. Do what feels most comfortable - but continue to test first, then write code, and finally refactor.

### Views

First, check to make sure all the views are setup correctly.

### Main View

As always, start with a test:

```python
from django.template.loader import render_to_string
from django.test import TestCase, Client
from user_contacts.models import Person, Phone
from user_contacts.views import *

class ViewTest(TestCase):

    def setUp(self):
        self.client_stub = Client()

    def test_view_home_route(self):
        response = self.client_stub.get('/')
        self.assertEquals(response.status_code, 200)
```

Name this test `test_views.py` and save it in the `user_contacts/tests` directory. Also add an `__init__.py` file to the directory and delete the "tests.py" file in the main `user_contacts` directory.

Run it:

```sh
$ python manage.py test user_contacts
```

It should fail - `AssertionError: 404 != 200` - because the URL, View, and the Template do not exist. If you're unfamiliar with how Django handles the MVC architecture, please read the short article [here](https://docs.djangoproject.com/en/dev/faq/general/#django-appears-to-be-a-mvc-framework-but-you-call-the-controller-the-view-and-the-view-the-template-how-come-you-don-t-use-the-standard-names).

The test is simple. We first GET the url "/" using the Client, which is part of Django’s `TestCase`. The response is stored, then we check to make sure the returned status code is equal to 200.

Add the following route to "contacts/urls.py":

```python
url(r'^', include('user_contacts.urls')),
```

Update "user_contacts/urls.py":

```python
from django.conf.urls import patterns, url

from user_contacts.views import *

urlpatterns = patterns('',
      url(r'^$', home),
)
```

Update "views.py":

```python
from django.http import HttpResponse, HttpResponseRedirect
from django.shortcuts import render_to_response, render
from django.template import RequestContext
from user_contacts.models import Phone, Person
# from user_contacts.new_contact_form import ContactForm

def home(request):
    return render_to_response('index.html')
```

Add an "index.html" template to the templates directory:

```html
<!DOCTYPE html>
  <head>
    <title>Welcome.</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="http://netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" media="screen">
    <style>
        .container {
            padding: 50px;
        }
    </style>
  </head>
  <body>
    <div class="container">
        <h1>What would you like to do?</h1>
        <ul>
            <li><a href="/all">View Contacts</a></li>
            <li><a href="/add">Add Contact</a></li>
        </ul>
    <div>
    <script src="http://code.jquery.com/jquery-1.10.2.min.js"></script>
    <script src="http://netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  </body>
</html>
```

Run the test again. It should pass just fine.

### All Contacts View

The test for this view is nearly identical to our last test. Try it on your own before looking at my answer.

Write the test first by adding the following function to the `ViewTest` class:

```python
def test_view_contacts_route(self):
    response = self.client_stub.get('/all/')
    self.assertEquals(response.status_code, 200)
```

When ran, you should see the same error: `AssertionError: 404 != 200`.

Update "user_contacts/urls.py" with the following route:

```python
url(r'^all/$', all_contacts),
```

Update "views.py":

```python
def all_contacts(request):
    contacts = Phone.objects.all()
    return render_to_response('all.html', {'contacts':contacts})
```

Add an "all.html" template to the templates directory:

{% raw %}
```html
<!DOCTYPE html>
<html>
<head>
  <title>All Contacts.</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link href="http://netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" media="screen">
  <style>
    .container {
      padding: 50px;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>All Contacts</h1>
    <table border="1" cellpadding="5">
      <tr>
        <th>First Name</th>
        <th>Last Name</th>
        <th>Address</th>
        <th>City</th>
        <th>State</th>
        <th>Country</th>
        <th>Phone Number</th>
        <th>Email</th>
      </tr>
      {% for contact in contacts %}
        <tr>
          <td>{{contact.person.first\_name}}</td>
          <td>{{contact.person.last\_name}}</td>
          <td>{{contact.person.address}}</td>
          <td>{{contact.person.city}}</td>
          <td>{{contact.person.state}}</td>
          <td>{{contact.person.country}}</td>
          <td>{{contact.number}}</td>
          <td>{{contact.person.email}}</td>
        </tr>
      {% endfor %}
    </table>
    <br>
    <a href="/">Return Home</a>
  </div>
  <script src="http://code.jquery.com/jquery-1.10.2.min.js"></script>
  <script src="http://netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
</body>
</html>
```
{% endraw %}

This should pass as well.

### Add Contact View

This test is slightly different from the previous two, so please follow along closely.

Add the test to the test suite:

```python
def test_add_contact_route(self):
    response = self.client_stub.get('/add/')
    self.assertEqual(response.status_code, 200)
```

You should see this error when ran: `AssertionError: 404 != 200`

Update "urls.py":

```python
url(r'^add/$', add),
```

Update "views.py":

```python
def add(request):
    person_form = ContactForm()
    return render(request, 'add.html', {'person_form' : person_form}, context_instance = RequestContext(request))
```

Make sure to add the following import:

```python
from user_contacts.new_contact_form import ContactForm
```

Create a new file called `new_contact_form.py` and add the following code:

```python
import re
from django import forms
from django.core.exceptions import ValidationError
from user_contacts.models import Person, Phone

class ContactForm(forms.Form):
    first_name = forms.CharField(max_length=30)
    last_name = forms.CharField(max_length=30)
    email = forms.EmailField(required=False)
    address = forms.CharField(widget=forms.Textarea, required=False)
    city = forms.CharField(required=False)
    state = forms.CharField(required=False)
    country = forms.CharField(required=False)
    number = forms.CharField(max_length=10)

    def save(self):
        if self.is_valid():
            data = self.cleaned_data
            person = Person.objects.create(first_name=data.get('first_name'), last_name=data.get('last_name'),
                email=data.get('email'), address=data.get('address'), city=data.get('city'), state=data.get('state'),
                country=data.get('country'))
            phone = Phone.objects.create(person=person, number=data.get('number'))
            return phone
```

Add "add.html" to the templates directory:

{% raw %}
```html
<!DOCTYPE html>
<html>
<head>
  <title>Welcome.</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link href="http://netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" media="screen">
  <style>
    .container {
      padding: 50px;
    }
  </style>
</head>
  <body>
    <div class="container">
    <h1>Add Contact</h1>
    <br>
    <form action="/create" method ="POST" role="form">
        {% csrf\_token %}
        {{ person_\_form.as\_p }}
        {{ phone\_form.as\_p }}
        <input type ="submit" name ="Submit" class="btn btn-default" value ="Add">
    </form>
      <br>
      <a href="/">Return Home</a>
    </div>
  <script src="http://code.jquery.com/jquery-1.10.2.min.js"></script>
  <script src="http://netdna.bootstrapcdn.com/bootstrap/3.0.0/js/bootstrap.min.js"></script>
  </body>
</html>
```
{% endraw %}

Does it pass? It should. If not, refactor.

### Validation

Now that we're done testing the Views, let's add validation to the form. But first we need to write a test. Surprise!

Create a new file called "test_validator.py" within the "tests" directory and add the following code:

```python
from django.core.exceptions import ValidationError
      from django.test import TestCase
      from user_contacts.validators import validate_number, validate_string

      class ValidatorTest(TestCase):
          def test_string_is_invalid_if_contains_numbers_or_special_characters(self):
              with self.assertRaises(ValidationError):
                  validate_string('@test')
                  validate_string('tester#')
          def test_number_is_invalid_if_contains_any_character_except_digits(self):
              with self.assertRaises(ValidationError):
                  validate_number('123ABC')
                  validate_number('75431#')
```

Before running the test suite, can you guess what might happen? *Hint: Pay close attention to the imports in the above code.* You should get the following error because we don't have a "validators.py" file:

 ```sh
 ImportError: cannot import name validate_string
 ```

In other words, we are testing the logic in a validation file that does not exist yet.

Add a new file called "validators.py" to the `user_contacts` directory:

```python
import re
from django.core.exceptions import ValidationError

def validate_string(string):
    if re.search('^[A-Za-z]+$', string) is None:
        raise ValidationError('Invalid')

def validate_number(value):
    if re.search('^[0-9]+$', value) is None:
        raise ValidationError('Invalid')
```

Run the test suite again. Five should now pass:

```python
Ran 5 tests in 0.019s

OK
```

### Create Contact

Since we added validation, we we want to test to ensure that the validators work in the admin area, so update "test_views.py":

```python
from django.template.loader import render_to_string
from django.test import TestCase, Client
from user_contacts.models import Person, Phone
from user_contacts.views import *

class ViewTest(TestCase):

    def setUp(self):
        self.client_stub = Client()
        self.person = Person(first_name = 'TestFirst',last_name = 'TestLast')
        self.person.save()
        self.phone = Phone(person = self.person,number = '7778889999')
        self.phone.save()

    def test_view_home_route(self):
        response = self.client_stub.get('/')
        self.assertEquals(response.status_code, 200)

    def test_view_contacts_route(self):
        response = self.client_stub.get('/all/')
        self.assertEquals(response.status_code, 200)

    def test_add_contact_route(self):
        response = self.client_stub.get('/add/')
        self.assertEqual(response.status_code, 200)

    def test_create_contact_successful_route(self):
        response = self.client_stub.post('/create',data = {'first_name' : 'testFirst', 'last_name':'tester', 'email':'test@tester.com', 'address':'1234 nowhere', 'city':'far away', 'state':'CO', 'country':'USA', 'number':'987654321'})
        self.assertEqual(response.status_code, 302)

    def test_create_contact_unsuccessful_route(self):
        response = self.client_stub.post('/create',data = {'first_name' : 'tester_first_n@me', 'last_name':'test', 'email':'tester@test.com', 'address':'5678 everywhere', 'city':'far from here', 'state':'CA', 'country':'USA', 'number':'987654321'})
        self.assertEqual(response.status_code, 200)

    def tearDown(self):
        self.phone.delete()
        self.person.delete()
```

Two tests should fail.

What needs to be done in order to get this test to pass? Well, we first need to add a function to the views for adding data to the database.

Add route:

```python
url(r'^create$', create),
```

Update "views.py":

```python
def create(request):
    form = ContactForm(request.POST)
    if form.is_valid():
        form.save()
        return HttpResponseRedirect('all/')
    return render(request, 'add.html', {'person_form' : form}, context_instance = RequestContext(request))
```

Test again:

```
$ python manage.py test user_contacts
```

This time only one test should fail - `AssertionError: 302 != 200` - because we tried to add data that should not have passed the validators but did. In other words, we need to update the "models.py" file as well as the form to take those validators into account.

Update "models.py":

```python
from django.db import models
from user_contacts.validators import validate_string, validate_number

class Person(models.Model):
    first_name = models.CharField(max_length = 30, validators = [validate_string])
    last_name = models.CharField(max_length = 30, validators = [validate_string])
    email = models.EmailField(null = True, blank = True)
    address = models.TextField(null = True, blank = True)
    city = models.CharField(max_length = 15, null = True,blank = True)
    state = models.CharField(max_length = 15, null = True, blank = True, validators = [validate_string])
    country = models.CharField(max_length = 15, null = True, blank = True)

    def __unicode__(self):
        return self.last_name +", "+ self.first_name

class Phone(models.Model):
    person = models.ForeignKey('Person')
    number = models.CharField(max_length=10, validators = [validate_number])

    def __unicode__(self):
        return self.number
```

Delete the current database, "db.sqlite3", and re-sync the database:

```sh
$ python manage.py syncdb
```

Setup an admin user again.

Update `new_contact_form.py` by adding validation:

```python
import re
from django import forms
from django.core.exceptions import ValidationError
from user_contacts.models import Person, Phone
from user_contacts.validators import validate_string, validate_number

class ContactForm(forms.Form):
    first_name = forms.CharField(max_length=30, validators = [validate_string])
    last_name = forms.CharField(max_length=30, validators = [validate_string])
    email = forms.EmailField(required=False)
    address = forms.CharField(widget=forms.Textarea, required=False)
    city = forms.CharField(required=False)
    state = forms.CharField(required=False, validators = [validate_string])
    country = forms.CharField(required=False)
    number = forms.CharField(max_length=10, validators = [validate_number])

    def save(self):
        if self.is_valid():
            data = self.cleaned_data
            person = Person.objects.create(first_name=data.get('first_name'), last_name=data.get('last_name'),
            email=data.get('email'), address=data.get('address'), city=data.get('city'), state=data.get('state'),
            country=data.get('country'))
            phone = Phone.objects.create(person=person, number=data.get('number'))
            return phone
```

Run the tests again. 7 should pass.

Now, deviating from TDD for a minute, I want to add an additional test to test validation on the client side. So add `test_contact_form.py`:

```python
from django.test import TestCase
from user_contacts.models import Person
from user_contacts.new_contact_form import ContactForm

class TestContactForm(TestCase):
    def test_if_valid_contact_is_saved(self):
        form = ContactForm({'first_name':'test', 'last_name':'test','number':'9999900000'})
        contact = form.save()
        self.assertEqual(contact.person.first_name, 'test')
    def test_if_invalid_contact_is_not_saved(self):
        form = ContactForm({'first_name':'tes&t', 'last_name':'test','number':'9999900000'})
        contact = form.save()
        self.assertEqual(contact, None)
```

Run the test suite. All 9 tests should now pass. Yay! Now commit.

## Functional Tests Redux

With the Unit tests done, we can now add a Functional test to ensure that the app runs correctly. Hopefully, with the Unit tests passing, we should have no problems with the Functional test.

Add a new class to the "tests.py" file:

```python
class UserContactTest(LiveServerTestCase):

    def setUp(self):
        self.browser = webdriver.Firefox()
        self.browser.implicitly_wait(3)

    def tearDown(self):
        self.browser.quit()

    def test_create_contact(self):
        # user opens web browser, navigates to home page
        self.browser.get(self.live_server_url + '/')
        # user clicks on the Persons link
        add_link = self.browser.find_elements_by_link_text('Add Contact')
        add_link[0].click()
        # user fills out the form
        self.browser.find_element_by_name('first_name').send_keys("Michael")
        self.browser.find_element_by_name('last_name').send_keys("Herman")
        self.browser.find_element_by_name('email').send_keys("michael@realpython.com")
        self.browser.find_element_by_name('address').send_keys("2227 Lexington Ave")
        self.browser.find_element_by_name('city').send_keys("San Francisco")
        self.browser.find_element_by_name('state').send_keys("CA")
        self.browser.find_element_by_name('country').send_keys("United States")
        self.browser.find_element_by_name('number').send_keys("4158888888")
        # user clicks the save button
        self.browser.find_element_by_css_selector("input[value='Add']").click()
        # the Person has been added
        body = self.browser.find_element_by_tag_name('body')
        self.assertIn('michael@realpython.com', body.text)

    def test_create_contact_error(self):
        # user opens web browser, navigates to home page
        self.browser.get(self.live_server_url + '/')
        # user clicks on the Persons link
        add_link = self.browser.find_elements_by_link_text('Add Contact')
        add_link[0].click()
        # user fills out the form
        self.browser.find_element_by_name('first_name').send_keys("test@")
        self.browser.find_element_by_name('last_name').send_keys("tester")
        self.browser.find_element_by_name('email').send_keys("test@tester.com")
        self.browser.find_element_by_name('address').send_keys("2227 Tester Ave")
        self.browser.find_element_by_name('city').send_keys("Tester City")
        self.browser.find_element_by_name('state').send_keys("TC")
        self.browser.find_element_by_name('country').send_keys("TCA")
        self.browser.find_element_by_name('number').send_keys("4158888888")
        # user clicks the save button
        self.browser.find_element_by_css_selector("input[value='Add']").click()
        body = self.browser.find_element_by_tag_name('body')
        self.assertIn('Invalid', body.text)
```

Run the Functional tests:

```sh
$ python manage.py test ft
```

Here we're just testing the code we wrote and already tested with Unit tests from the end user's perspective. All four tests should pass.

Finally, let's ensure that the validation we put into place applies to the Admin panel by adding the following function to the `AdminTest` class:

```python
def test_create_contact_admin_raise_error(self):
    # # user opens web browser, navigates to admin page, and logs in
    self.browser.get(self.live_server_url + '/admin/')
    username_field = self.browser.find_element_by_name('username')
    username_field.send_keys('admin')
    password_field = self.browser.find_element_by_name('password')
    password_field.send_keys('admin')
    password_field.send_keys(Keys.RETURN)
    # user clicks on the Persons link
    persons_links = self.browser.find_elements_by_link_text('Persons')
    persons_links[0].click()
    # user clicks on the Add person link
    add_person_link = self.browser.find_element_by_link_text('Add person')
    add_person_link.click()
    # user fills out the form
    self.browser.find_element_by_name('first_name').send_keys("test@")
    self.browser.find_element_by_name('last_name').send_keys("tester")
    self.browser.find_element_by_name('email').send_keys("test@tester.com")
    self.browser.find_element_by_name('address').send_keys("2227 Tester Ave")
    self.browser.find_element_by_name('city').send_keys("Tester City")
    self.browser.find_element_by_name('state').send_keys("TC")
    self.browser.find_element_by_name('country').send_keys("TCA")
    # user clicks the save button
    self.browser.find_element_by_css_selector("input[value='Save']").click()
    body = self.browser.find_element_by_tag_name('body')
    self.assertIn('Invalid', body.text)
```

Run it. Five tests should pass. Commit and let's call it a day.

## Test Structure

TDD is a powerful tool and an integral part of the development cycle, helping developers break programs into small, readable portions. Such portions are much easier to write now and change later. Further, having a comprehensive test suite, covering every feature of your codebase, helps ensure that new feature implementations will not break existing code.

Within the process, **Functional tests** are high-level tests, focused on the *features* that the end users interact with.

Meanwhile, **Unit tests** support Functional tests in that they test each feature of the code. Keep in mind that Unit tests are much easier to write, generally provide better coverage, and are easier to debug because they test only one feature at a time. They also run much quicker, so be sure to test your unit tests more often than your functional tests.

Let's take a look at our testing structure to see how our unit tests support the functional tests:

![test-structure](https://raw.github.com/mjhea0/django-tdd/master/test-structure.png)

## Conclusion

Congrats. You make it through. What's next?

First, you may have noticed that I did not 100% follow the TDD process. That's okay. Most developers engaged in TDD don't always adhere to it in every single situation. There are times that you must deviate from it in order to just get things done - which is perfectly fine. If you'd like to refactor some of the code/process to fully adhere to the TDD process, you can. In fact, it may be a good practice.

Second, think about the tests I missed. Determining what and when to test is difficult. It takes time and much practice to get good at testing in general. I've left many blanks that I intend to revel in my next post. See if you can find those and add tests.

Finally, remember the last step in the TDD process? Refactoring. This step is vital as it helps create readable, maintainable code that you not only understand now - but in the future as well. When you look back at your code, think about tests you can combine. Also, which tests should you add to ensure that all written code is tested? You could test for null values and/or server side authentication, for example. You should refactor your code before moving on to writing any new code - which I did not do for time's sake. Perhaps another blog post? Think about how bad code can pollute the entire process?

Thanks for reading. Grab the final code in the repo [here](https://github.com/mjhea0/django-tdd). Please comment below with any questions.
