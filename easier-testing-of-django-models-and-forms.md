# Easier Testing of Django Models and Forms

Originally posted on [fRui Apps](https://web.archive.org/web/20150108125559/http://blog.fruiapps.com/2012/10/Easier-Testing-of-django-models-and-forms).

<hr>

In this post let's talk about making lives easier while testing django models and forms. This is the fourth article in our [Test Driven Development Series](http://blog.fruiapps.com/tag/test-driven-development). Until now, we have already talked about [intro to TDD](http://blog.fruiapps.com/2012/09/An-intro-tutorial-to-test-driven-development-in-python-django), [basic testing django views and models](http://blog.fruiapps.com/2012/09/An-introduction-to-unit-testing-django-views-and-models) and [functional testing with selenium](http://blog.fruiapps.com/2012/08/Functional-Testing-with-Selenium-WebDriver-and-Selector-in-Python).

### Testing Django Models

By default when Django runs against a SQLite back-end it creates a new in-memory database for a test, based on the fixtures which you have kept. Application fixtures are a great way to test, but these are limited to the cases where you have a constant data model. But the projects we deal with are much more complicated than just having simple models, and manually managed fixtures are an inflexible solution for providing the test case data. Test fixtures are hard to maintain, when trying to reproduce complex referential associations. Doing it manually or monkey patching gets us into more trouble as we fail, more often than not in maintaining the integrity of the schema.

While writing tests, a large portion of our test code mainly prepares the data that we provide to our tests. When this system becomes even slightly flawed, then reading, writing, and/or maintaining tests becomes a cumbersome task. The trouble escalates when data mutations lead to bigger failures across the system, and then you have to take an extra amount of pain to systematically refactor.

The above problem leads us to think about managing our fixtures. Utilizing APIs to create data in concise readable formats makes it much easier to handle schema association and updation and use them inside our tests. They help us to avoid code repetition and streamline the process of creating and managing test data. Amongst the most famous ones are [factory-boy](https://github.com/dnerdy/factory_boy) along [with](http://djangopackages.com/grids/g/fixtures). Personally, for me managing the factories becomes tiring, and I rely on [django-dynamic-fixture](https://github.com/paulocheque/django-dynamic-fixture), which we will utilize in this post.

The library has got decent documentation explaining why it should be used, what led to its creation, its features, and how it compares to other fixture tools. To easily get started you can install it with pip - `pip install django-dynamic-fixture`. Since this article is about testing models and not about choosing a fixture API, let's get back to our polls model, and see how we can easily test them:

```python
from django.test import TestCase
from django_dynamic_fixture import G

class PollModelTest(TestCase):

    def test_poll_creation(self):
        "Test the creation and saving of a new poll"
        poll1 = G(Poll)
        poll2 = G(Poll)
        polls_in_db = Poll.objects.all()
        self.assertEquals(polls_in_db.count(), 2)

        # check the values stored
        poll_zero = polls_in_db[0]
        self.assertEquals(poll_zero.question, poll1.question)
        self.assertEquals(poll_zero.pub_date, poll1.pub_date)
```

Please compare this, with the [previous](blog.fruiapps.com/2012/09/An-introduction-to-unit-testing-django-views-and-models) test case we wrote. In a single line we can create a fixture, and get rid of maintaining the separate JSON file, with a proper schema, which adapts to changes.

### Testing Django Forms

The first thing to keep in mind when testing Django forms is, again, that the Django form handling should not be tested - meaning that the core behavior should not be tested. What needs to be verified is if the application produces correct responses when a form is submitted/edited. Superficially, a form is just a class with few member functions. A form goes through the following three stages:

1. Instantiating an instance.
1. Populating some data.
1. Cleaning the form and validating the data.

Let's look at an example. Assume we have to test a basic registeration form. The steps would be as follows:

#### 1\. create a user

```python
User.objects.create('bob', 'bob@example.com', 'letmein')
```

#### 2\. create invalid data dictionaries

```python
invalid_data_dict = [
  {'data': {'username':'foobar',
            'email': 'foo@example.com',
            'password1': 'foo',
            'password2':'foo'},
   'error': ('password', [u"The two password fields didn't match."])
  }
]
```

This is one example of an invalid data (the two data passwords did not match). There are many ther cases - like an invalid username, an already existing username, undesired password strength, etc.

#### 3\. assert that the form fails for all invalid data and that the correct error message is displayed.

```python
for invalid_data in invalid_data_dict:
  form = forms.RegisterationForm(data=invalid_dict['data'])
  self.failIf(form.is_valid())
  self.assertEqual(form.errors[invalid_dict]['error'][0],
                   invalid_dict['error'][1])
```

The above was a very basic but an apt example of testing a django form. For further cases when you have custom validations or save methods, things can be made handier with the use of [django-webtest](http://pypi.python.org/pypi/django-webtest). Install via `pip install django-webtest` and then look at this simple example of testing with webtest:

```python
from django_webtest import WebTest

class DemoTestCase(WebTest):

def test_my_form(self)
    form = self.app.get(reverse('url-name')).form
    self.assertEqual(form[form['fieldname1'].value, 'some value'])
    form['field2'] = "some other value"
    response = form.submit().follow()
    self.assertEqual(response.context['field2'].value, 'some other value')
```

Having looked at webtest this forces us to make a choice: Do we chuck selenium/twill and use only django-webtest?. Well there is again no answer to it. Having said that. the thoughts that come are as follows:

*Selenium is primarily built to be a server agnostic testing framework, whereas django-webtest is built to work specifically on Django websites and is actually a WSGI only framework.*

**Pros for WebTest:**

1. Being Python only, WebTest can interact deeply with, and with a higher level of awareness of, how things are built inside. It provides access to Django internals and native API stuff - `self.assertFormError`, `response.templates`, etc.
1. WebTest is faster, since it hooks to the WSGI. This also makes for a smaller memory footprint.
1.  No server is required for running WebTest.

**Cons for WebTest:**

1. WebTest can't test JavaScript and reveal browser issues.
1. Macros can't be recorded and run as with Selenium.
1. No browser extensions exist.

*Bottomline? Choose one and stick with it, depending on your needs!*