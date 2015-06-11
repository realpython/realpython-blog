# Integration testing with pyVows and Django

Welcome back!

### Articles in this series:

- Part 1: [Asynchronous Testing with Django and PyVows](http://www.realpython.com/blog/python/asynchronous-testing-with-django-and-pyvows/)
- Part 2: [Unit Testing with pyVows and Django](http://www.realpython.com/blog/python/unit-testing-with-pyvows-and-django/)
- **Part 3: [Integration testing with pyVows and Django](http://www.realpython.com/blog/python/integration-testing-with-pyvows-and-django/) (current article)**

Repo: [TDD-Django](https://github.com/mjhea0/TDD-Django)

<br/>

If you've been following this series - [part 1](http://www.realpython.com/blog/python/asynchronous-testing-with-django-and-pyvows) and [part 2](http://www.realpython.com/blog/python/unit-testing-with-pyvows-and-django/) - you should now have a basic understanding of what pyVows is all about and how it can be used for asynchronous test execution for a Django project. We will continue in this final article of the three part series to explore how pyVows can be used for integration testing django and testing of django models.

There is much debate across the web about _unit testing_ as it pertains to models.  According to the most rigid definition a unit test shouldn't rely on a database being around as that means you are technically doing an integration test.  I personally am not so rigid on the definition and tend to be more pragmatic.  In other words, if it makes it easier / quicker to write tests with a database back end and you can still achieve stable, reusable tests with good coverage than I'm all for it.

One of the biggest (and most valid) arguments against using an actual database in unit tests is that they are slow.  While that is true, you will see in this article that running your tests asynchronously against a multi-threaded database can help to improve performance of the unit test runs.  While it's true testing against a live database backend will probably never be as fast as testing without the database we can make it reasonably fast with asynchronous testing.

Furthermore, by including the database back-end we can actually improve test coverage by validating that our database queries are actually performed successfully.  Because when we hit the database we can test things like database constraints or that we are not trying to put a VARCHAR in a NUMBER field.  And when we don't hit the database with a "pure" unit test, we can't verify these things.  Anyways, lets get started.

##A "Pure" unit test for a Django Model

Starting with the sample login application we have been using in this series, lets write a simple model to take us through some examples. Add the following code to `/accounts/models.py`:

```python
class Account(models.Model):
    username = models.CharField(max_length=100)
    password = models.CharField(max_length=100)
```

Now let's write a simple test for it in `/accounts/tests.py`:

```python
class AccountVows(DjangoHTTPContext):

    def topic(self):
        return self.model(Account)

    def should_have_username_field_that_is_a_string_and_has_max_length_100(self, topic):
        expect(topic).to_have_field('username', models.CharField,
                                     max_length=100)
```

The setup is very similar to all the other tests we have done previously with pyVows.  The only difference here is we are using the `DjangoHTTPContext.model` function, which just returns a `django_pyvows.assertions.Model` class to ensure the object being passed inherits from `django.db.models.Model`.

Next we use the `to_have_field` function of the `django_pyvows.assertion.Model` class to verify that we have the field setup correctly.

Once we know the field is setup correctly we can use a few more assertions in the Model class to further verify the Model under test.

## Not quite a unit test, but oh so helpful

```python
def should_be_cruddable(self,topic):
    expect(topic).to_be_cruddable()
```

Here is where we start to blur the line between _pure_ unit tests and integration tests.  The function `django_pyvows.assertions.Model.to_be_cruddable` will try to store the model in the database, with some dummy data, it will then do an update and finally delete the data.  All of that in one line of code.  Awesome!

This is a classic example of a test that not only verifies our `Account` class is setup correctly but the backend database and tables are setup correctly and that we can successfully perform CRUD operations against the database.  This is great because it make sure we don't have any sort of key constraints, user permissions or table design issues that get in the way of regular CRUD operations.

But there is some setup involved.  In particular if we were to run this test right now we would get the following failure. Go ahead and try

```sh
... long ugly stack trace ending in ...
return Database.Cursor.execute(self, query, params)
   DatabaseError: no such table: accounts_account
```

See for yourself:

```sh
$ env PYTHONPATH=$$PYTHONPATH:tddapp/ pyvows tddapp/accounts/tests.py
```

As you can see from the error message this test is trying to create and execute a query against a "live" database, which isn't there.  We can make sure it's always there by just calling syncdb from our setup function.

```python
def topic(self):
    from django.core import management
    management.call_command('syncdb',interactive=False)
    management.call_command('flush',interactive=False)
```

This is the same thing as running `syncdb` from the command line: it will create all the default django tables as well as tables for the models that you have defined. The second line will clear out any data you have loaded in those tables.  For smaller systems the syncdb will run pretty quickly so as not to harm performance too much, but for larger systems it can take a while to run the syncdb command, which is probably something that we don't want for our unit tests.

BUT WAIT, pyvows is asynchronous right?  That's right, and if you remember from [part 1](http://www.realpython.com/blog/python/asynchronous-testing-with-django-and-pyvows) sibling context's run in parallel, so you can be running all your other unit tests while the database is syncing.  Imagine a test structure like this:

```python
@Vows.batch
class MainTestClass(Vows.Context):

    class ModelVows(DjangoHTTPContext):

        def topic(self):
            from django.core import management
            management.call_command('syncdb',interactive=False)
            management.call_command('flush',interactive=False)

        class AccountVows(DjangoHTTPContext):

            def topic(self):
                return self.model(Account)

            def should_have_username_field(self,topic):
                excpect(topic).to_have_field('username')

        class SomeOtherModel(DjangoHTTPContext):

            ...model tests here...

    class ViewVows(DjangoHTTPContext):

        ...additional tests here...

    class ControllerVows(DjangoHTTPContext):

        ...additional tests here...

```

In the above example the `syncdb` and `flush` commands would run before any model tests were run.  But, concurrently to the database setup commands being run, unit tests in the `ViewVows` and `ControllerVows` context would run.  So in other words, you're not really _wasting_ time initializing the database as your other unit tests are running at the same time.

>Note: Concurrency isn't exactly parallesim especially if you're running on CPython due to limitations of the GIL.  So what is described above isn't parallel execution - it's just concurrent.  Which will ultimately be faster than synchronous execution but perhaps not as fast as it could be. Check out this [Stackoverflow question](http://stackoverflow.com/questions/15556718/greenlet-vs-threads) for a better understanding of this. It's a great place to get an overview of what is actually going on behind the scenes. Also checkout the links in the accepted answer. I found them very informative.

## Full Blown Integration Tests

Now that we have basic tests for our models running pretty quickly lets finish the login example so that the login page actually checks the database backend to ensure a valid password.  In a production system you would probably want to just use `django.contrib.auth`.  But bear with me as this example makes the point very clear (I hope :) ).  Here is what the code would look like for `templates/login.html`:

```html
<html>
  <head>
    <title>Login</title>
  </head>
  <body>
  {% if valid  %}
    <h1>Welcome {{ referrer }} user. Please login.</h1>
  {% else %}
    <h1> Incorrect login, please try again.</h1>
  {% endif %}
    <form method="POST" action="#" id="login-form">
        <p>Username: <input type="text" id="username" name="username"/></p>
        <p>Password:<input type="password" id="password" name="password"/></p>
        <input type="submit" id="login" value="Login"/>
    </form>
  </body>
</html>
```

And `views.py`:

```python
from django.http import HttpResponse
from django.shortcuts import render
from accounts.models import Account

def login(request):

    valid = True

    if request.method == "POST":
        if _is_valid_login(request.POST['username'], request.POST['password']):
            return HttpResponse("Login Successful")
        else:
            valid = False

    return render(request, 'login.html', {'valid' : valid,
                                    'referrer' : 'RealPython'})

def _is_valid_login(username, password):
    user_list = Account.objects.filter(username=username, password=password)
    return len(user_list) > 0
```

Basically we have added in the functionality to execute a query on POST to see if the username / password combination inputted by the user exists in the database. If it does, we will display the Login Successful screen.  If it doesn't, we will display the login screen with the heading 'Incorrect login, please try again.'  So let's add some tests to our existing suite to ensure that this functionality works.

Since we already covered the basic functionality in our previous test from [part 2](http://www.realpython.com/blog/python/unit-testing-with-pyvows-and-django) let's create the test to ensure an invalid login attempt returns the correct page.

```python
class PostInValidLogin(DjangoHTTPContext):

  def topic(self):
    return self.post('/login/', {'username':'user','password':'pass'})

  def should_return_invalid_login(self, (topic, content)):
    invalidLogin = render_to_string("login.html",
                     {"valid":False})
    expect(content).to_equal(invalidLogin)
```

This test is truly an integration test because it starts with sending the request to the server by using the `DjangoHTTPContext.post` function and then validates that the correct html page was returned.  So we are actually testing the entirety of the MVC here.  About the only thing we aren't testing for is any javascript or browser rending issues.  Still because we have previously tested the template rendering and the view login in isolation these type of tests help to make sure all the components are integrated correctly.

Let's add one more test to ensure a valid login.  In order to do that (since this will be a live integration test) we will need to add a user to the database.  So let's put that all together now.

```python
class PostValidLogin(DjangoHTTPContext):

    def setup(self):
        validUser = Account(username='validuser',password='pass')
                validUser.save()

    def topic(self):
       return self.post('/login/',
            {'username':'validuser','password':'pass'})

    def should_return_valid_login(self, (topic,content)):
       expect(content).to_equal("Login Successful")
```

The only difference in the ValidLoginTest is we first use the `setup` function (which pyVows ensures to be the first function executed in the class) to create the user that we want in the database.  This way we can verify that our login page is correctly querying the database.  Also notice that since we are adding data to the database we would want to make sure that our syncdb function had previously been called.  In other words we would want the `PostValidLogin` context to be a child of whatever context initialized the database, which in our case is the `AccountVows` context.  So the structure would look like this.

```python
class AccountVows(DjangoHTTPContext):

   def topic(self):
       from django.core import management
       management.call_command('syncdb',interactive=False)
       management.call_command('flush',interactive=False)
       return self.model(Account)

   ... snip ...

   class PostValidLogin(DjangoHTTPContext):

       def setup(self):
           validUser = Account(username='validuser',password='pass')
           validUser.save()

       def topic(self):
          return self.post('/login/',
                           {'username':'validuser','password':'pass'})

       def should_return_valid_login(self, (topic,content)):
          expect(content).to_equal("Login Successful")
```

This structure ensures that our database is always setup and cleared out before we run our `should_return_valid_login` test.  That way we can be sure that the database is setup correctly for our test to run.

As a final note it can be said that the `PostValidLogin` context is a bit sloppy because its not cleaning up after itself.  For our example we don't really need to as we are creating a separate database for testing and we are clearing it at the start of the test run.  But for the sake of completeness we could add a teardown function to our `PostValidLogin` context and clear out the newly created row.  Doing so would make the context look like this.

```python
class PostValidLogin(DjangoHTTPContext):

    def setup(self):
    self.validUser = Account(username='validuser',password='pass')
    self.validUser.save()

    def teardown(self):
    self.validUser.delete()

    def topic(self):
       return self.post('/login/',
            {'username':self.validUser.username,
             'password':self.validUser.password})

    def should_return_valid_login(self, (topic,content)):
       expect(content).to_equal("Login Successful")
```

Finally, run the tests:

```sh
$ env PYTHONPATH=$$PYTHONPATH:tddapp/ pyvows -vvv tddapp/accounts/tests.py
```

And you should see the following results:

```sh
Creating tables ...
Installing custom SQL ...
Installing indexes ...
Installed 0 object(s) from 0 fixture(s)
Installed 0 object(s) from 0 fixture(s)

 ============
 Vows Results
 ============


    Login page vows
      Login page url
      ✓ url should be mapped to login view
      Login page view
      ✓ should return valid http response
      ✓ should return login page
      Login page template
      ✓ should use password field
      ✓ should have login form
      ✓ should not have settings link
      ✓ should have username field
        Welcome message
        ✓ should welcome user from referrer
      Account vows
      ✓ should have username field that is a string and has max length 100
      ✓ should be cruddable
        Post in valid login
        ✓ should return invalid login
        Post valid login
        ✓ should return valid login
  ✓ OK » 12 honored • 0 broken (0.308917s)
```

## Teardown

And with the teardown of our last test context its time for the teardown of this article and the Django Testing with pyVows series.  I hope this article has shown you some techniques to speed up not only the execution but also the creation of your unit and integration test.  I don't like to be to strict on the separation between unit and integration tests as there is often a use for both. We have completely skipped the topic of mocking because with django-pyvows setting up and executing the integration tests is so fast and easy that mocks often aren't needed.  At least not for this simple case.  (But who knows maybe I'll get around to another article on mocks and when they are useful).

The main point to come away with here is that there are many methods and frameworks to test Django applications each with it's own unique set of advantages and disadvantages.  By digging into django-pyVows throughout this series I hope you have at least seen some alternative approaches to testing.  Even if you don't ultimately end up using django-pyVows I hope that the approaches and techniques you have learned can help your testing efforts in the future. Again, grab the code from the [repo](https://github.com/mjhea0/TDD-Django).

**Hit me up in the comments and let me know what you think of the series.**