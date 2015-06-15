# The Minimum Viable Test Suite

<div class="center-text">
  <img class="no-border" src="/images/blog_images/minimum-viable-testing/minimum-viable-test.png" style="max-width: 100%;" alt="minimum viable testing">
</div>

<br>

In the last [post](https://realpython.com/blog/python/handling-email-confirmation-in-flask/) we detailed how to validate email addresses during user registration.

**This time, we'll add unit and integration tests (yay!) to our application using the [Flask-Testing](http://pythonhosted.org/Flask-Testing/) extension, covering the most important features. This type of testing is called Minimum Viable Testing (or [Risk-based Testing](http://en.wikipedia.org/wiki/Risk-based_testing)) and is designed to test the high-risk functionality, centered around the application's features.**

> Did you miss the first post? Grab the code from the [project repo](https://github.com/realpython/flask-registration/releases/tag/v1) to quickly get started.

## Unit and Integration Tests - defined

For those new to testing, it's vital to test your applications since "untested applications make it hard to improve existing code and developers of untested applications tend to become pretty paranoid. If an application has automated tests, you can safely make changes and instantly know if anything breaks" ([source](http://flask.pocoo.org/docs/0.10/testing/)).

Unit tests, by nature, test isolated units of code - i.e., individual functions - to ensure that the actual output is the same as the expected output. In many cases, since you often have to make external API calls or touch a database, unit tests can rely heavily on mocking fake data. By simulating the tests, they may run faster, but they can also be less effective and are harder to maintain. Because of this, we will not be using mocks unless we absolutely have to; instead we will read and write to the database as needed.

Keep in mind that when a database is touched in a specific test, it is technically an integration test since the test itself is not isolated to a specific unit. Also, if you run your tests through the Flask app, using the test helper test [client](http://werkzeug.pocoo.org/docs/0.10/test/#werkzeug.test.Client), they are considered integration tests as well.

## Getting Started

It's often difficult to determine how to start testing an application. One solution to this problem is to think about your app in terms of end user functionality:

1. Unregistered users must sign up before accessing the app.
1. After users register, a confirmation email is sent - and they are considered "unconfirmed" users.
1. Unconfirmed users can log in but they are immediately redirected to a page reminding them to confirm their account via email before they can access the app.
1. Once confirmed, users have full access to the site, where they can view the main page, update their password on the profile page, and logout.

Like stated in the beginning, we'll write just enough tests to cover this main functionality. Testing is hard; we are hyper aware of that, so if you're only keen on writing a few tests, test what matters the most. This, coupled with coverage testing via [coverage.py](http://nedbatchelder.com/code/coverage/), which we'll detail in the next article in this series, will make it much easier to structure a robust test suite.

## Setup

Activate your virtualenv, then make sure the following environment variables are set:

```sh
$ export APP_SETTINGS="project.config.DevelopmentConfig"
$ export APP_MAIL_USERNAME="foo"
$ export APP_MAIL_PASSWORD="bar"
```

Then run the current test suite:

```sh
$ python manage.py test
test_app_is_development (test_config.TestDevelopmentConfig) ... ok
test_app_is_production (test_config.TestProductionConfig) ... ok
test_app_is_testing (test_config.TestTestingConfig) ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.003s

OK
```

These tests simply test the configuration and environment variables. They should be fairly straightforward.

To expand the suite, we need to start with an organized structure to keep everything nice and neat. Since the app is already structured around blueprints, let's do the same for the test suite. So create two new test files in the "tests" directory - *test_main.py* and *test_user.py* - and add the following code to each:

```python
import unittest

from flask.ext.login import current_user

from project.util import BaseTestCase


# tests go here


if __name__ == '__main__':
    unittest.main()
```

> **NOTE**: You could also structure your tests around test type - unit, integration, functional, etc..

## Part 1 - Main Blueprint

Looking at the code in the *views.py* file (in the "project/main" folder), along with the end user workflow, we can see that we just need to test that the main route, `/`, requires the user to be logged in. So add the following code to *test_main.py*:

```python
def test_main_route_requires_login(self):
    # Ensure main route requires a logged in user.
    response = self.client.get('/', follow_redirects=True)
    self.assertTrue(response.status_code == 200)
    self.assertTemplateUsed('user/login.html')
```

Here, we're asserting that the response status code is `200` and that the correct template is used. Run the test suite. All 4 tests should pass.

## Part 2 - User Blueprint

There's quite a bit more going on in this blueprint, so the testing required is far more intensive. Essentially, we need to test the views and  - so, we'll break apart our test suite accordingly. Don't worry I will guide you through it. Let's create the 2 classes to ensure our tests are logically divided.

Add the following code to *test_user.py* so we can start testing the many functions required.

```python
class TestUserForms(BaseTestCase):
    pass


class TestUserViews(BaseTestCase):
    pass
```

### Forms

Having a user register is a core concept in a log in based program, without it we have an "open door" to trouble. This must work as designed. So, following the user workflow, let's start with the **registration form**. Add this code to the `TestUserForms()` class.

```python
def test_validate_success_register_form(self):
    # Ensure correct data validates.
    form = RegisterForm(
        email='new@test.test',
        password='example', confirm='example')
    self.assertTrue(form.validate())

def test_validate_invalid_password_format(self):
    # Ensure incorrect data does not validate.
    form = RegisterForm(
        email='new@test.test',
        password='example', confirm='')
    self.assertFalse(form.validate())

def test_validate_email_already_registered(self):
    # Ensure user can't register when a duplicate email is used
    form = RegisterForm(
        email='test@user.com',
        password='just_a_test_user',
        confirm='just_a_test_user'
    )
    self.assertFalse(form.validate())
```

In these tests, we're ensuring that the form either passes or fails validation based on the data entered. Compare this to the *forms.py* file in the "project/user" folder. In the last test, we're simply registering the same user from the `setUpClass()` method from our `BaseTestCase` in the *util.py* file.

While we're testing the forms, let's go ahead and test the **login form** as well:

```python
def test_validate_success_login_form(self):
    # Ensure correct data validates.
    form = LoginForm(email='test@user.com', password='just_a_test_user')
    self.assertTrue(form.validate())

def test_validate_invalid_email_format(self):
    # Ensure invalid email format throws error.
    form = LoginForm(email='unknown', password='example')
    self.assertFalse(form.validate())
```

Finally, let's test the change password form:

```python
def test_validate_success_change_password_form(self):
    # Ensure correct data validates.
    form = ChangePasswordForm(password='update', confirm='update')
    self.assertTrue(form.validate())

def test_validate_invalid_change_password(self):
    # Ensure passwords must match.
    form = ChangePasswordForm(password='update', confirm='unknown')
    self.assertFalse(form.validate())

def test_validate_invalid_change_password_format(self):
    # Ensure invalid email format throws error.
    form = ChangePasswordForm(password='123', confirm='123')
    self.assertFalse(form.validate())
```

Make sure to add the required imports:

```python
from project.user.forms import RegisterForm, \
    LoginForm, ChangePasswordForm
```

And then run the tests!

```sh
$ python manage.py test
test_app_is_development (test_config.TestDevelopmentConfig) ... ok
test_app_is_production (test_config.TestProductionConfig) ... ok
test_app_is_testing (test_config.TestTestingConfig) ... ok
test_main_route_requires_login (test_main.TestMainViews) ... ok
test_validate_email_already_registered (test_user.TestUserForms) ... ok
test_validate_invalid_change_password (test_user.TestUserForms) ... ok
test_validate_invalid_change_password_format (test_user.TestUserForms) ... ok
test_validate_invalid_email_format (test_user.TestUserForms) ... ok
test_validate_invalid_password_format (test_user.TestUserForms) ... ok
test_validate_success_change_password_form (test_user.TestUserForms) ... ok
test_validate_success_login_form (test_user.TestUserForms) ... ok
test_validate_success_register_form (test_user.TestUserForms) ... ok

----------------------------------------------------------------------
Ran 12 tests in 1.656s
```

For the form tests, we basically just instantiated the form and called the validate function which will trigger all validation, including our custom validation and return a boolean indicating if the form data is indeed valid or not.

With our forms tested, let's move on to the Views...

### Views

Logging in and viewing the profile are critical parts of security so we want to make certain this is thoroughly tested.

**`login`**

```python
def test_correct_login(self):
    # Ensure login behaves correctly with correct credentials.
    with self.client:
        response = self.client.post(
            '/login',
            data=dict(email="test@user.com", password="just_a_test_user"),
            follow_redirects=True
        )
        self.assertTrue(response.status_code == 200)
        self.assertTrue(current_user.email == "test@user.com")
        self.assertTrue(current_user.is_active())
        self.assertTrue(current_user.is_authenticated())
        self.assertTemplateUsed('main/index.html')

def test_incorrect_login(self):
    # Ensure login behaves correctly with incorrect credentials.
    with self.client:
        response = self.client.post(
            '/login',
            data=dict(email="not@correct.com", password="incorrect"),
            follow_redirects=True
        )
        self.assertTrue(response.status_code == 200)
        self.assertIn(b'Invalid email and/or password.', response.data)
        self.assertFalse(current_user.is_active())
        self.assertFalse(current_user.is_authenticated())
        self.assertTemplateUsed('user/login.html')
```

**`profile`**

```python
def test_profile_route_requires_login(self):
    # Ensure profile route requires logged in user.
    self.client.get('/profile', follow_redirects=True)
    self.assertTemplateUsed('user/login.html')
```

Add the required imports as well:

```python
from project import db
from project.models import User
```

**`register`** and **`resend_confirmation`**

Before writing tests to cover the `register` and `resend_confirmation` views, take a look at the [code](https://github.com/realpython/flask-registration/blob/v1/project/user/views.py). Notice how we're utilizing the `send_email()` function from the *email.py* file, which sends the confirmation email. Do we really want to send this email or should we fake it using a mocking library? Even if we do send it, it's very difficult to assert that an actual email shows up in a dummy inbox without utilizing Selenium to pull up the actual inbox in the browser. So, let's mock the sending of the email, which we'll handle in a subsequent article.

**`confirm/<token>`**

```python
def test_confirm_token_route_requires_login(self):
    # Ensure confirm/<token> route requires logged in user.
    self.client.get('/confirm/blah', follow_redirects=True)
    self.assertTemplateUsed('user/login.html')
```

Like the last two views, the remaining parts of this view could be mocked since a confirmation token needs to be generated. However, we can just generate a token using the utility function from the *token.py* file, `generate_confirmation_token()`:

```python
def test_confirm_token_route_valid_token(self):
    # Ensure user can confirm account with valid token.
    with self.client:
        self.client.post('/login', data=dict(
            email='test@user.com', password='just_a_test_user'
        ), follow_redirects=True)
        token = generate_confirmation_token('test@user.com')
        response = self.client.get('/confirm/'+token, follow_redirects=True)
        self.assertIn(b'You have confirmed your account. Thanks!', response.data)
        self.assertTemplateUsed('main/index.html')
        user = User.query.filter_by(email='test@user.com').first_or_404()
        self.assertIsInstance(user.confirmed_on, datetime.datetime)
        self.assertTrue(user.confirmed)

def test_confirm_token_route_invalid_token(self):
    # Ensure user cannot confirm account with invalid token.
    token = generate_confirmation_token('test@test1.com')
    with self.client:
        self.client.post('/login', data=dict(
            email='test@user.com', password='just_a_test_user'
        ), follow_redirects=True)
        response = self.client.get('/confirm/'+token, follow_redirects=True)
        self.assertIn(
            b'The confirmation link is invalid or has expired.',
            response.data
        )
```

Add the imports:

```sh
import datetime
from project.token import generate_confirmation_token, confirm_token
```

And then run the tests. One should fail:

```sh
Ran 18 tests in 4.666s

FAILED (failures=1)
```

This test failed - `test_confirm_token_route_invalid_token()`. Why? Because there's an error in the view:

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

What's wrong?

Right now the `flash` call - e.g., `flash('The confirmation link is invalid or has expired.', 'danger')` - does not cause the function to exit, so it will fall through to the if/else and confirm the user even if the token is invalid. *This is why you write tests.*

Let's rewrite the function:

```python
@user_blueprint.route('/confirm/<token>')
@login_required
def confirm_email(token):
    if current_user.confirmed:
        flash('Account already confirmed. Please login.', 'success')
        return redirect(url_for('main.home'))
    email = confirm_token(token)
    user = User.query.filter_by(email=current_user.email).first_or_404()
    if user.email == email:
        user.confirmed = True
        user.confirmed_on = datetime.datetime.now()
        db.session.add(user)
        db.session.commit()
        flash('You have confirmed your account. Thanks!', 'success')
    else:
        flash('The confirmation link is invalid or has expired.', 'danger')
    return redirect(url_for('main.home'))
```

Run the tests again. All 18 should pass.

What happens if a token expires? Write a test.

```python
def test_confirm_token_route_expired_token(self):
    # Ensure user cannot confirm account with expired token.
    user = User(email='test@test1.com', password='test1', confirmed=False)
    db.session.add(user)
    db.session.commit()
    token = generate_confirmation_token('test@test1.com')
    self.assertFalse(confirm_token(token, -1))
```

Run the tests again:

```python
$ python manage.py test
test_app_is_development (test_config.TestDevelopmentConfig) ... ok
test_app_is_production (test_config.TestProductionConfig) ... ok
test_app_is_testing (test_config.TestTestingConfig) ... ok
test_main_route_requires_login (test_main.TestMainViews) ... ok
test_validate_email_already_registered (test_user.TestUserForms) ... ok
test_validate_invalid_change_password (test_user.TestUserForms) ... ok
test_validate_invalid_change_password_format (test_user.TestUserForms) ... ok
test_validate_invalid_email_format (test_user.TestUserForms) ... ok
test_validate_invalid_password_format (test_user.TestUserForms) ... ok
test_validate_success_change_password_form (test_user.TestUserForms) ... ok
test_validate_success_login_form (test_user.TestUserForms) ... ok
test_validate_success_register_form (test_user.TestUserForms) ... ok
test_confirm_token_route_expired_token (test_user.TestUserViews) ... ok
test_confirm_token_route_invalid_token (test_user.TestUserViews) ... ok
test_confirm_token_route_requires_login (test_user.TestUserViews) ... ok
test_confirm_token_route_valid_token (test_user.TestUserViews) ... ok
test_correct_login (test_user.TestUserViews) ... ok
test_incorrect_login (test_user.TestUserViews) ... ok
test_profile_route_requires_login (test_user.TestUserViews) ... ok

----------------------------------------------------------------------
Ran 19 tests in 5.306s

OK
```

## Reflection

This is probably a good time to stop and reflect, especially since we are focusing on minimal testing. Remember our core features?

1. Unregistered users must sign up before accessing the app.
1. After users register, a confirmation email is sent - and they are considered "unconfirmed" users.
1. Unconfirmed users can log in but they are immediately redirected to a page reminding them to confirm their account via email before they can access the app.
1. Once confirmed, users have full access to the site, where they can view the main page, update their password on the profile page, and logout.

Are we covering each of these? Let's look:

**Unregistered users must sign up before accessing the app**

- `test_main_route_requires_login`
- `test_validate_email_already_registered`
- `test_validate_invalid_email_format`
- `test_validate_invalid_password_format`
- `test_validate_success_register_form`

**After users register, a confirmation email is sent - and they are considered "unconfirmed" users.** and **Unconfirmed users can log in but they are immediately redirected to a page reminding them to confirm their account via email before they can access the app.**

- `test_validate_success_login_form`
- `test_confirm_token_route_expired_token`
- `test_confirm_token_route_invalid_token`
- `test_confirm_token_route_requires_login`
- `test_confirm_token_route_valid_token`
- `test_correct_login`
- `test_incorrect_login`
- `test_profile_route_requires_login`

**Once confirmed, users have full access to the site, where they can view the main page, update their password on the profile page, and logout.**

- `test_validate_invalid_change_password`
- `test_validate_invalid_change_password_format`
- `test_validate_success_change_password_form`

In the above tests we tested the forms directly, and then *also* created tests for the views (which exercise *much* of the same code as in the form tests). What are the tradeoffs of this type of approach? We'll address this when we tie in coverage testing.

## Next Time

That's it for this post. In the next few posts we'll-

1. Mock all or parts of the following functions from the `user` blueprint to finalize unit/integration testing - `register()` and `resend_confirmation()`
1. Add coverage testing via [coverage.py](http://nedbatchelder.com/code/coverage/) to help ensure that our code base is adequately being tested.
1. Expand the test suite by adding functional tests with Selenium.

Happy testing!

<br>

<p style="font-size: 14px;">
  <em>Edits made by <a href="https://twitter.com/diek007">Derrick Kearney</a>. Thank you!</em>
</p>
