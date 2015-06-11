# Asynchronous Testing with Django and PyVows

Welcome!

### Articles in this series:

- **Part 1: [Asynchronous Testing with Django and PyVows](http://www.realpython.com/blog/python/asynchronous-testing-with-django-and-pyvows/) (current article)**
- Part 2: [Unit Testing with pyVows and Django](http://www.realpython.com/blog/python/unit-testing-with-pyvows-and-django/)
- Part 3: [Integration testing with pyVows and Django](http://www.realpython.com/blog/python/integration-testing-with-pyvows-and-django/)

Repo: [TDD-Django](https://github.com/mjhea0/TDD-Django)

<br>


[PyVows](http://heynemann.github.io/pyvows/), is a port of the popular Vows behavior-driven development (or BDD) framework for Node.js. For this article I will focus on using PyVows and selenium to run GUI Tests against Django, and in the next article in this series, I will discuss unit testing with PyVows.

Before we start, take a minute to read [this](http://blog.codeship.io/2013/04/22/from-tdd-to-bdd.html) article that describes the differences between behavior-driven and test-driven development.

## Basic Setup

Let's work through a set of examples.  Start a new Django project (with [virtualenvwrapper](https://bitbucket.org/dhellmann/virtualenvwrapper), [django-pyVows](http://github.com/rafaelcaricio/django-pyvows), and [selenium](https://code.google.com/p/selenium/)):

```sh
$ mkvirtualenv TDD-Django --no-site-packages
$ workon TDD-Django
$ pip install pyVows django django-pyVows selenium lxml cssselect
$ django-admin.py startproject tddapp
```


This setup is all we need for now. Let's create our first GUI test with pyVows by adding `uitests.py` to `tddapp`, and then add the following code:

```python
from pyvows import Vows, expect
from django_pyvows.context import DjangoHTTPContext
from selenium import webdriver

@Vows.batch
class TDDDjangoApp(DjangoHTTPContext):

    def get_settings(self):
        return "tddapp.settings"

    def topic(self):
        self.start_server()
        browser = webdriver.Firefox()
        browser.get(self.get_url("/"))
        return browser

    def should_prompt_the_user_with_a_login_page(self, topic):
        expect(topic.title).to_include('Django')
```

### What's really going on here?

The start of the test imports all the necessary libraries including pyvows, django_pyvows and selenium:

```python
from pyvows import Vows, expect
from django_pyvows.context import DjangoHTTPContext
from selenium import webdriver
```

The next line creates what in pyVows is referred to as a test batch.  A test batch in pyVows is identified by the `@Vows.batch` decorator and is synonymous with `unittest.TestCase` in the Python standard library's unit test module.

Test batches are *contexts* which describe different components and states you want to test.  Generally a test batch will inherit from `Vows.Context` but in our case we are inheriting from `DjangoHTTPContext` as that class provides all the helper functionality baked into django_pyvows.  In addition to inheriting from `DjangoHTTPContext` you need to overwrite the function `get_settings()` and return the path to your `settings.py` file you want to use to run your tests:

```python
@Vows.batch
class TDDDjangoApp(DjangoHTTPContext):

    def get_settings(self):
        return "tddapp.settings"
```

Each *context* should contain a single topic, which is the object you want to test and a number of vows which perform the tests against said topic. For our topic we start the django server using the `start_server()` helper function, fire up Selenium and point Selenium to the root of our django application. Then, since this is a GUI test, our topic returns the Selenium webdriver so we can execute our tests.


> Note: `self.get_url("/")` translates the relative url to the absolute url that we need to access the site.

Next, our first vow/test  checks to see if the title of the page includes 'Django':

```python
def topic(self):
    self.start_server()
    browser = webdriver.Firefox()
    browser.get(self.get_url("/"))
    return browser

def should_prompt_the_user_with_a_django_page(self, topic):
    expect(topic.title).to_include('Django')
```


Running the test at this point should pass, because the default page in Django has a title of "Welcome to Django". This verifies that our basic setup is correct and we can go ahead and start coding something.  The output of running the test looks like this:

```sh
$ pyVows -vvv uitests.py

============
Vows Results
============

Tdd django app
✓  should prompt the user with a django page
✓ OK » 1 honored • 0 broken (2.489381s)

And a failed test would look like this:

$ pyVows -vvv uitests.py

============
Vows Results
============

Tdd django app
✗ should prompt the user with a django page
    Expected topic(u'Page Not Found') to include 'Login'

    ...snipping Traceback to save space...

✗ OK » 0 honored • 1 broken (2.489381s)
```

Pay attention to the naming of the tests in the report.  The outermost indented line is the context (TDDDjangoApp), with each vow/test in that context listed and indented underneath.  This makes reporting and finding defects really easy as it reads like a set of requirements.  Consider the following report:


```sh
$ pyVows -vvv uitests.py

============
Vows Results
============


Tdd django app
on Chrome
       ✓ should prompt the user with a django page
on Firefox
       ✗ should prompt the user with a django page

    ...snipping Traceback to save space...

✗ OK » 1 honored • 1 broken (3.119354s)
```

In the above fictitious example we can see that our setup is working for
Chrome and not Firefox.  Thus by using intuitive naming of the tests, pyVows
produces a simple yet highly effective report for pinpointing issues.

## Let's Run Things in Parallel

In addition we are now approaching Asynchronous Parallel testing on multiple browsers!  Let's have a look at the the test code that would have produced that report:

```python
@Vows.batch
class TDDDjangoApp(DjangoContext):

    class OnFirefox(DjangoHTTPContext):

        def get_settings(self):
            return "tddapp.settings"

        def topic(self):
            self.start_server(port=8888)
            browser = webdriver.Firefox()
            browser.get(self.get_url("/"))
            return browser

        def should_prompt_the_user_with_a_django_page(self, topic):
            expect(topic.title).to_include('string not in the title')

    class OnChrome(DjangoHTTPContext):

        def get_settings(self):
            return "tddapp.settings"

        def topic(self):
            self.start_server(port=8887)
            browser = webdriver.Chrome()
            browser.get(self.get_url("/"))
            return browser

        def should_prompt_the_user_with_a_django_page(self, topic):
            expect(topic.title).to_include('Django')
```

In pyVows, sibling contexts are executed in parallel and nested contexts are executed sequentially - which means TDDDjangoApp would be executed first, then its two nested contexts OnFirefox and OnChrome would be executed in parallel (because they are siblings of each other).  We can however remove a lot of the duplicate code by creating a standard set of tests and running those tests against multiple contexts.

```python
def onBrowser(webdriver, port):
    class BrowserTests(DjangoHTTPContext):

        def get_settings(self):
            return "tddapp.settings"

        def topic(self):
            self.start_server(port=port)
            browser = webdriver()
            browser.get(self.get_url("/"))
            return browser

        def should_prompt_the_user_with_a_django_page(self, topic):
            expect(topic.title).to_include('Django')

    return BrowserTests

@Vows.batch
class TDDDjangoApp(DjangoContext):

    class OnChrome(onBrowser(webdriver.Chrome, 8887)):
        pass

    class OnFirefox(onBrowser(webdriver.Firefox, 8888)):
        pass

    class OnPhantonJS(onBrowser(webdriver.PhantomJS, 8886)):
        pass
```

Functional programming to the rescue! (well kind of...)  One of the beauties of Python is you aren't tied to a particular programming paradigm.  So we can use a variety of solutions. In this case, we want to dynamically create a testing context for each browser and port we want to run tests against.  To do this we can write all of our tests once (in the BrowserTests class, which is returned by the OnBrowser function).  The nice thing about this technique is that the `topic()` function in the BrowserTests class returns whatever webdriver (aka browser) that is passed into the onBrowser function.

From here our test batch (denoted by `@Vows.batch`) is simply describing what will be ran /reported. The output of which would look like this:

```sh
 ============
 Vows Results
 ============


    Tdd django app
      On phantom js
      ✓ should prompt the user with a django page
      On chrome
      ✓ should prompt the user with a django page
      On firefox
      ✓ should prompt the user with a django page
  ✓ OK » 3 honored • 0 broken (4.068020s)
```

With this setup each test we add to the BrowserTests class will run against
each browser we specify.  For example if we added the following tests to the
BrowserTests class...

```python
    def heading_should_tell_the_user_it_worked(self,topic):
            heading_text = topic.find_element_by_css_selector("#summary h1").text
            expect(heading_text).to_equal("It worked!")

    def should_display_debug_message(self,topic):
            explain_text = topic.find_element_by_id("explanation").text
            expect(explain_text).to_include("DEBUG = True")
```

...then the report would show:


```sh
 ============
 Vows Results
 ============


    Tdd django app
      On firefox
      ✓ should prompt the user with a django page
      ✓ heading should tell the user it worked
      ✓ should display debug message
      On chrome
      ✓ heading should tell the user it worked
      ✓ should prompt the user with a django page
      ✓ should display debug message
      On phantom js
      ✓ should prompt the user with a django page
      ✓ should display debug message
      ✓ heading should tell the user it worked
    ✓ OK » 9 honored • 0 broken (4.238522s)
```

## An Important Issue to Consider

You may wonder why, however, we are calling `start_server` for every new context.  We actually don't have to do this, but we are working around a limitation of django_pyvows.  Unfortunately the `DjangoHTTPContext.start_server()` function creates a single thread, which decreases performance. For this set of tests, for example, execution time increases from roughly 5 seconds (as written above) to about 21 seconds if we only create one CherryPy server.

Ideally we would want to only create one server to test against and perhaps
make the server multi-threaded so we don't have to incur the performance
penalty.  So let's see if we can make that happen.  First off let's update the
test code to use pyVows notion of context inheritance so we only have to create
the server once.

```python
def onBrowser(webdriver):
    class BrowserTests(DjangoHTTPContext):

        def topic(self):
            browser = webdriver()
            browser.get(self.get_url("/"))
            return browser

        def should_prompt_the_user_with_a_login_page(self, topic):
            expect(topic.title).to_include('Django')

    return BrowserTests

@Vows.batch
class TDDDjangoApp(DjangoHTTPContext):

    def get_settings(self):
        return "tddapp.settings"

    def topic(self):
        self.start_server()

    class OnChrome(onBrowser(webdriver.Chrome)):
        pass

    class OnFirefox(onBrowser(webdriver.Firefox)):
        pass

    class OnPhantonJS(onBrowser(webdriver.PhantomJS)):
        pass
```

Notice how the TDDDjangoApp context now calls `start_server()` from its topic
function and we no longer need to pass in a port to our `onBrowser()` context
creation function. We now only instantiate one version of theCherryPy server and run all tests against that.  This has the advantage of using less resources
and it makes the code easier to maintain.

To deal with the performance issue, we can manually manage the threads to get tests running faster:

```python
def onBrowser(webdriver):
    class BrowserTests(DjangoHTTPContext):

        def topic(self):
            webdriver.get(self.get_url("/"))
            return webdriver

        def teardown(self):
            webdriver.quit()

        def should_prompt_the_user_with_a_login_page(self, topic):
            expect(topic.title).to_include('Django')

    return BrowserTests

@Vows.batch
class TDDDjangoApp(DjangoHTTPContext):

    def get_settings(self):
        return "tddapp.settings"

    def topic(self):
        self.start_server()
        #manual add some more threads to the CherryPy server
        self.server.thr.server.requests.grow(3)

    def teardown(self):
        #clean up the threads so we can exit cleanly
        self.server.thr.server.requests.stop(timeout=1)

    class OnChrome(onBrowser(webdriver.Chrome())):
        pass

    class OnFirefox(onBrowser(webdriver.Firefox())):
        pass

    class OnPhantonJS(onBrowser(webdriver.PhantomJS())):
        pass
```

We have done three things here.  First, in the `topic()` function we added three more
threads to CherryPy's thread pool:

```python
def topic(self):
    self.start_server()
    #manual add some more threads to the CherryPy server
    self.server.thr.server.requests.grow(3)
```

Second, we needed to make sure the threads were cleaned up or the test run
will never complete.

```python
def teardown(self):
    #clean up the threads so we can exit cleanly
    self.server.thr.server.requests.stop(timeout=1)
```

Finally, we ensured that webdriver cleans up after itself by adding a teardown function to our BrowserTest class:

```python
def teardown(self):
    webdriver.quit()
```

## Wrapping Up

Node isn't the only game in town that can do things asynchronously.  And withthe port of Vows.js to pyVows and a little bit of ingenuity we can get parallel testing with multiple browsers for Django up and running with very little effort. Taking this a step further, we could link up to something like BrowserMob or SauceLabs to test against a huge number of browsers and then things would really start to get interesting.  Also, with simple, easy to understand reporting and a very clean syntax, your GUI testing can become (relatively) painless and FAST!

The code used for this article can be found in the associated Github [repo](https://github.com/j1z0/TDD-Django).

Please let me know what you think in the comments below. Cheers! <3