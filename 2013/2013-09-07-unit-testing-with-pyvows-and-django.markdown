# Unit Testing with pyVows and Django

Welcome!

### Articles in this series:

- Part 1: [Asynchronous Testing with Django and PyVows](http://www.realpython.com/blog/python/asynchronous-testing-with-django-and-pyvows/)
- **Part 2: [Unit Testing with pyVows and Django](http://www.realpython.com/blog/python/unit-testing-with-pyvows-and-django/) (current article)**
- Part 3: [Integration testing with pyVows and Django](http://www.realpython.com/blog/python/integration-testing-with-pyvows-and-django/)

Repo: [TDD-Django](https://github.com/mjhea0/TDD-Django)

<br/>

In the last article, we talked about using the amazing [pyVows](http://heynemann.github.io/pyvows/) and selenium to speed up your GUI Tests. But GUI Tests are not the entire story. Unit Testing is equally - arguably, even more - important. And we can continue to use pyVows for our unit tests as well. No need to use a different tool.  Although, you won't see much speed improvement from the asynchronous nature of pyVows with a few tests, with hundreds of tests you will. Especially if your tests do a fair amount of I/O.

So lets have a look at using pyVows for unit testing. Imagine we wanted to create some functionality to manage user accounts, probably starting with a login form.  After activating your virtualenv, the first thing we will do is create an app called accounts (these days the django folks say everything should be in an app and who am I to argue). That is done pretty easily with this command:

```sh
$ python manage.py startapp accounts
```

That will create a new directory called account with a `models.py`, `tests.py`, and
`views.py` files:

```sh
├── README.mdown
├── requirements.txt
└── tddapp
    ├── accounts
    │   ├── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    ├── manage.py
    ├── tddapp
    │   ├── __init__.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    └── uitests.py
```


For our example the first thing we want to start building is a login page accessed from `/login`.  So we can add an entry in urls.py to map `/login` to our view function:

```python
urlpatterns = patterns('',
     url(r'^login/$', 'accounts.views.login',name='login'),
```

> Make sure to add `accounts` to your `INSTALLED_APPS` within `settings.py` file.

Also, the code used for this article can be found in the associated Github [repo](https://github.com/j1z0/TDD-Django).

## Testing URL Mappings

While the above entry in `urls.py` is pretty straight forward, its still a good idea to write a unit test to make sure we have everything wired up correctly.  Also in this case a test is good to have to make sure somebody doesn't accidentally clobber this url rule sometime down the road.  (Perhaps by including another url rule that overwrites this one.

So we can modify the `accounts/tests.py` file to look like this:

```python
from pyvows import Vows, expect
from django_pyvows.context import DjangoHTTPContext

@Vows.batch
class LoginPageVows(DjangoHTTPContext):

    def get_settings(self):
        return "tddapp.settings"

    def topic(self):
        self.start_server()
        return self.url("^login/$")

    def url_should_be_mapped_to_login_view(self, topic):
        from accounts.views import login
        expect(topic).to_match_view(login)
```

Here we are creating the same type of test case as we did in the [first
article](http://www.realpython.com/blog/python/asynchronous-testing-with-django-and-pyvows/). The important things to remember are:

1.  `@Vows.batch` marks the class as a test suite that pyVows will run.
2.  Inheriting from `DjangoHTTPContext` provides the pyVows support for testing
    Django Apps.
3.  Overwriting `get_settings` and returning the location (as if from an import
    statement) of the settings file you want to use is necessary because pyVows
    will automatically start up the Django server.  **Note: `get_settings` also
    makes it easy to use a different settings.py for testing as opposed to
    production, if for example you want to use a faster in memory database.**

The meat of the test is in the `url_should_be_mapped_to_login_view` function
which uses the `to_match_view` assertion that is built into `django-pyvows`.
The `to_match_view` assertion simply ensures that django will call the specified
function.  In this case `accounts.views.login` for the specified URL - which is what we returned as the topic of our tests:

```python
return self.url("^login/$")
```
Running the test should fail initially until we add the `accounts.views.login
function` to `tests.py` and then add the `login()` function to `views.py`:


```python
from django.http import HttpResponse

def login(request):
    pass
```

In order to run the test correctly we may need to provide a PYTHONPATH environment variable to pyVows so it doesn't get confused about where to locate our modules.  To do this we should always run pyVows from the project root directory with a command line like this:

```sh
$ env PYTHONPATH=$$PYTHONPATH:tddapp/ pyvows tddapp/accounts/tests.py
```

This basically says add the tddapp/ app directory to our existing python path
and then run pyvows for the tests in tddapp/accounts/tests.py.  This should
avoid any issues with not being able to find the `settings.py` file and ensure
all imports work as expected.

## Testing View functions

Now that we know our `urls.py` is mapping to the correct view the next logical
step is to test the view to ensure that it does what is expected. So update `views.py` with a very simple login view function like this:

```python
from django.http import HttpResponse

def login(request):
    return HttpResponse("this is a login")
```

Lets add the following tests:

```python
class LoginPageView(DjangoHTTPContext):

    def topic(self):
        return login(self.request())

    def should_return_valid_HTTP_Response(self,topic):
        expect(topic).to_be_http_response()

    def should_return_login_page(self, topic):
        expect(topic).to_have_contents_of("this is a login")
```

Notice that for our topic we are calling the login view function directly and passing in `self.request()`. `self.request()` is a helper function provided by `DjangoHTTPContext` that creates and returns a new `HTTPRequest` and thus makes it simple to test Django Views.

Once we have the return value from our `login view()` function we test two different characteristics:

1. That it returns a valid HTTPResponse object using the `to_be_http_response()` assertion.
2. That the contents of the response equals "this is a login".

Further to show that we can run these test in parallel we could refactor the tests a little bit to look like this:

```python
from accounts.views import login
from pyvows import Vows, expect
from django_pyvows.context import DjangoHTTPContext

@Vows.batch
class LoginPageVows(DjangoHTTPContext):

    def get_settings(self):
        return "tddapp.settings"

    def topic(self):
        self.start_server()

    class LoginPageURL(DjangoHTTPContext):

        def topic(self):
             return self.url("^login/$")

        def url_should_be_mapped_to_login_view(self, topic):
            expect(topic).to_match_view(login)

    class LoginPageView(DjangoHTTPContext):

        def topic(self):
            return login(self.request())

        def should_return_valid_HTTP_Response(self,topic):
            expect(topic).to_be_http_response()

        def should_return_login_page(self, topic):
            expect(topic).to_have_contents_of("this is a login")
```

Notice that we now instantiate the Django server in our outermost test class `LoginPageVows` and then the two child classes test the URLs `LoginPageURL` and the view `LoginPageView`.  If you recall from the previous article, sibling classes run in parallel.  So the URL tests and the View tests will run at the same time with the above class / test structure.

## Testing Templates

Arguably the above view function is too simple to ever be used in a real application.  More often than not in a real application you're going to want to use some sort of template to display your dynamic view.  Well fear not, we can test that with django-pyvows as well. <3

To illustrate that let's first create a template for our login view. We will create the file tddapp/accounts/templates/login.html. The code might look like:

```html
<html>
  <head>
    <title>Login</title>
  </head>
  <body>
    <h1>Please Login</h1>
    <form method="POST" action="#" id="login-form">
        <p>Username: <input type="text" id="username" /></p>
        <p>Password:<input type="password" id="password" /></p>
        <input type="submit" id="login" value="Login">
    </form>
  </body>
</html>
```

Once the template has been created, we just change the login view function to return the template like so:

```python
from django.shortcuts import render

def login(request):
    #return HttpResponse("this is a login")
    return render(request, 'login.html')
```

The render function tells the login function to return the `login.html` template wrapped in an HTTPResponse object.  So now in terms of testing, all we have to do is change our `should_return_login_page` to check to see if the login.html template was returned.  This can be done with the following changes:

```python
def should_return_login_page(self, topic):
    from django.template.loader import render_to_string
    loginTemplate = render_to_string("login.html")
    expect(topic.content.decode()).to_equal(loginTemplate)
```

The first line of the `should_return_login_page` function now imports the `render_to_string` function which takes a template as an argument (and an optional context) and returns the generated html as a string.  This will make it super easy to ensure that we are returning the login.html template.  We do that by generating the template using the `render_to_string` function (second line in the `should_return_login_page` function above).  Then we compare the returned `HTTPResponse` to the generated loginTemplate (on the third line) and your off to the races.

### A Note On Imports

Notice in the above function we put the import in the function. Some people prefer to put all imports at the  top of the file so they are easy to see and all get executed at once as soon as the module is loaded.  This is all well and good, but when unit testing Django with pyVows one must be aware of how the import works.

In  Python when you import a module or class, or function python will then execute all imports in the containing  module.  So for example the import `from accounts.views import login` will load the accounts.views module and import everything that is referenced there.  Which in this case is now `from django.shortcuts import render`. When that happens Django will want the server configuration (i.e. settings.py) to already be executed, but in pyVows that doesn't happen until we call `start_server`.  Which happens later, so an error will be raised.

One way to avoid this problem is to not import anything that relies on Django until after start_server is executed.  Which is what we did previously in the `should_return_login_page`.  However this can be tricky especially if you have a bunch of tests (which you likely will) that depend upon django in some way.

Have no fear.  django-pyVows offers a solution.  The `DjangoHTTPContext.start_environment` function.  You pass in the path to your settings file to that function and just call that function before you do any of the imports.  The start_environment function will get Django all setup and ready to go so you won't get strange import issues later.  Here is a look at how we would do that for our tests.py file:

```python
from pyvows import Vows, expect
from django_pyvows.context import DjangoHTTPContext

DjangoHTTPContext.start_environment("tddapp.settings")

from accounts.views import login
from django.template.loader import render_to_string

@Vows.batch
class LoginPageVows(DjangoHTTPContext):

    def topic(self):
        self.start_server()

    class LoginPageURL(DjangoHTTPContext):

        def topic(self):
             return self.url("^login/$")

        def url_should_be_mapped_to_login_view(self, topic):
            expect(topic).to_match_view(login)

    class LoginPageView(DjangoHTTPContext):

        def topic(self):
            return login(self.request())

        def should_return_valid_HTTP_Response(self,topic):
            expect(topic).to_be_http_response()

        def should_return_login_page(self, topic):
            loginTemplate = render_to_string("login.html")
            expect(topic.content.decode()).to_equal(loginTemplate)
```

Also we no longer need the `get_settings` function as we have already configured the settings, by calling the `DjangoHTTPContext.start_environment` function.

Go ahead and run the tests:

```sh
$ env PYTHONPATH=$$PYTHONPATH:tddapp/ pyvows tddapp/accounts/tests.py
```

They all should pass.


### Back to Templates

With the previous tests we can now prove that our view is calling the appropriate template and returning the html that is in our template. But how about testing the template itself?  Let's ensure that our template has a login form with a username and password field.

Django-pyvows has `CSSSelect` built in which is a library that allows you to query HTML using css selectors in much the same way you would if you were using jQuery.  This makes testing templates a synch.

First lets create a testing context using django-pyvows template class.

```python
class LoginPageTemplate(DjangoHTTPContext):

    def topic(self):
        return self.template("login.html", {})
```

The template `__init__` function takes two arguments, the name of the template (in this case `login.html`) and the context don't get confused, we are talking about a django template context here, not a pyvows testing context which in this case is empty but we will come back to template context later.

Now we can write the tests using our css selectors to verify what is in the template.  Here are some examples:

```python
def should_have_login_form(self, topic):
    expect(topic).to_contain("#login-form")

def should_have_username_field(self,topic):
    expect(topic).to_contain("#username")

def should_use_password_field(self,topic):
    expect(topic).to_contain("#password[type='password']")
```

All we have to do is use the `to_contain` assertion and pass in a css selector.  then if the selector is found our test will pass.  We can also explicitly test that a particular element is not found:

```python
def should_not_have_settings_link(self,topic):
    expect(topic).Not.to_contain("a#settings")
```

## Testing Django Template Context

For dynamic web pages our templates often generate dynamic html by reading variables from the template context. As an example let's assume we want to welcome users to our login page based upon the last page the visited (i.e. facebook, google, etc.).  We could update the template as follows:

{% raw %}
```html
<html>
  <head>
    <title>Login</title>
  </head>
  <body>
  <h1>Welcome {{ referrer }} user. Please login.</h1>
    <form method="POST" action="#" id="login-form">
        <p>Username: <input type="text" id="username"/></p>
        <p>Password:<input type="password" id="password"/></p>
        <input type="submit" id="login" value="Login"/>
    </form>
  </body>
</html>
```
{% endraw %}

Then when we create our template in the `LoginPageTemplate` test class we can pass in the appropriate context and verify that it is substituted correctly.  Here is the modify test class:

```python
class LoginPageTemplate(DjangoHTTPContext):

    def topic(self):
        return self.template("login.html", {"referrer":"Facebook"})

    def should_have_login_form(self, topic):
        expect(topic).to_contain("#login-form")

    ... removed some tests for brevity...

    class WelcomeMessage(DjangoHTTPContext):

        def topic(self, loginTemplate):
            return loginTemplate.get_text('h1')

        def should_welcome_user_from_referrer(self, topic):
            expect(topic).to_equal("Welcome Facebook user. Please login.")
```

A couple of things are important here:

First, our `WelcomeMessage` class, as a child of our `LoginPageTemplate` class has access to the topic defined in `LoginPageTemplate`.  That is why we can pass in an argument `loginTemplate` to our topic function in the `WelcomeMessage` class as illustrated here:

```python
def topic(self, loginTemplate):
    return loginTemplate.get_text('h1')
```

Second, calling the get_text function from a django-pyvows.template will find the element using a css selector and return the text.  From there its simple to add an assertion such as `to_equal` or `to_be_like` to ensure the element has the appropriate text. This technique should serve to test most of the dynamic changes in django templates.

Run the tests again:

```sh
$ env PYTHONPATH=$$PYTHONPATH:tddapp/ pyvows tddapp/accounts/tests.py
```

They all should pass. :)

## Conclusion

In this article we have covered many of the helpful features that django-pyVows provides to help with unit testing Django Views Templates and URL mappings.  What's more we can continue to use the same framework we did for our GUI testing and continue to get the benefits of parallel testing and take advantage of the numerous shortcuts that the library provides us.

Make sure to grab the code from the [repo](https://github.com/j1z0/TDD-Django).

Let me know what you guys think. Is anybody else using django-pyvows for unit testing?  Please share your thoughts in the comments below.