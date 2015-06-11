# Transaction Management with Django 1.6

If you ever devoted much time to Django database transaction management, you know how confusing it can get. In the past, the documentation provided quite a bit of depth, but understanding only came through building and experimenting.

There were a plethora of decorators to work with, like `commit_on_success`, `commit_manually`, `commit_unless_managed`, `rollback_unless_managed`, `enter_transaction_management`, `leave_transaction_management`, just to name a few. Fortunately, with Django 1.6 that all goes out the door. You really need to only know about a couple functions now. And we will get to those in just a second. First, we'll address these topics:

- **What is transaction management?**
- **What's wrong with transaction management prior to Django 1.6?**

Before jumping into:

- **What's right about transaction management in Django 1.6?**

And then dealing with a detailed example:

- **Stripe Example**
- **Transactions**
  - **The recommended way**
  - **Using a decorator**
  - **Transaction per HTTP Request**
- **SavePoints**
- **Nested Transactions**


## What is a transaction?

According to [SQL-92](http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt), "An SQL-transaction (sometimes simply called a "transaction") is a sequence of executions of SQL-statements that is atomic with respect to recovery". In other words, all the SQL statements are executed and committed together. Likewise, when rolled back, all the statements get rolled back together.

For example:

```python
# START
note = Note(title="my first note", text="Yay!")
note = Note(title="my second note", text="Whee!")
address1.save()
address2.save()
# COMMIT
```

So a transaction is a single unit of work in a database. And that single unit of work is demarcated by a start transaction and then a commit or an explicit rollback.

## What's wrong with transaction management prior to Django 1.6?

In order to fully answer this question, we must address how transactions are dealt with in the database, client libraries, and within Django.

### Databases

Every statement in a database has to run in a transaction, even if the transaction includes only one statement.

Most databases have an `AUTOCOMMIT` setting, which is usually set to True as a default. This `AUTOCOMMIT` wraps every statement in a transaction that is immediately committed if the statement succeeds. Of course you can manually call something like `START_TRANSACTION` which will temporarily suspend the `AUTOCOMMIT` until you call `COMMIT_TRANSACTION` or `ROLLBACK`.

*However, the take away here is that the `AUTOCOMMIT` setting applies an implicit commit after each statement*.

### Client Libraries

Then there's the Python **client libraries** like sqlite3 and mysqldb, which allow Python programs to interface with the databases themselves. Such libraries follow a set of standards for how to access and query the databases. That standard, DB API 2.0, is described in [PEP 249](http://www.python.org/dev/peps/pep-0249/). While it may make for some slightly dry reading, an important take away is that PEP 249 states that the database `AUTOCOMMIT` should be *OFF* by default.

This clearly conflicts with what's happening within the database:

- SQL statements always have to run in a transaction, which the database generally opens for you via `AUTOCOMMIT`.
- However, according to PEP 249, this should not happen.
- Client libraries must mirror what happens within the database, but since they are not allowed to turn `AUTOCOMMIT` on by default, they simply wrap your SQL statements in a transaction, just like the database.

Okay. Stay with me a little longer.

### Django

Enter Django. **Django** also has something to say about transaction management. In Django 1.5 and earlier, Django basically ran with an open transaction and auto-committed that transaction when you wrote data to the database. So every time you called something like `model.save()` or `model.update()`, Django generated the appropriate SQL statements and committed the transaction.

Also in Django 1.5 and earlier, it was recommended that you used the `TransactionMiddleware` to bind transactions to HTTP requests.  Each request was given a transaction. If the response returned with no exceptions, Django would commit the transaction but if your view function threw an error, `ROLLBACK` would be called.  This in effect, turned off `AUTOCOMMIT`. If you wanted standard, database level autocommit style transaction management you had to manage the transactions yourself - usually by using a transaction decorator on your view function such as `@transaction.commit_manually`, or `@transaction.commit_on_success`.

Take a breath. Or two.

### What does this mean?

Yeah, there is a lot going on there, and it turns out most developers just want the standard database level autocommits - meaning transactions stay behind the scenes, doing their thing, until you need to manually adjust them.

## What's right about transaction management in Django 1.6?

Now, welcome to Django 1.6. Do your best to forget everything we just talked about and simply remember that in Django 1.6, you use database `AUTOCOMMIT` and manage transactions manually when needed. Essentially, we have a much simpler model that basically does what the database was designed to do in the first place.

Enough theory. Let's code.

## Stripe Example

Here we have this example view function that handles registering a user and calling out to Stripe for credit card processing.

```python
def register(request):
    user = None
    if request.method == 'POST':
        form = UserForm(request.POST)
        if form.is_valid():

            customer = Customer.create("subscription",
              email = form.cleaned_data['email'],
              description = form.cleaned_data['name'],
              card = form.cleaned_data['stripe_token'],
              plan="gold",
            )

            cd = form.cleaned_data
            try:
                user = User.create(cd['name'], cd['email'], cd['password'],
                   cd['last_4_digits'])

                if customer:
                    user.stripe_id = customer.id
                    user.save()
                else:
                    UnpaidUsers(email=cd['email']).save()

            except IntegrityError:
                form.addError(cd['email'] + ' is already a member')
            else:
                request.session['user'] = user.pk
                return HttpResponseRedirect('/')

    else:
      form = UserForm()

    return render_to_response(
        'register.html',
        {
          'form': form,
          'months': range(1, 12),
          'publishable': settings.STRIPE_PUBLISHABLE,
          'soon': soon(),
          'user': user,
          'years': range(2011, 2036),
        },
        context_instance=RequestContext(request)
    )
```

This view first calls `Customer.create` which actually calls Stripe to handle the credit card processing. Then we create a new user. If we got a response back from Stripe we update the newly created customer with the `stripe_id`. If we don't get a customer back (Stripe is down) we will add an entry to the `UnpaidUsers` table with the newly created customers email, so we can ask them to retry their credit card details later.

The idea is that even if Stripe is down the user can still register and start using our site. We will just ask them again at a later date for the credit card info.

> I understand this may be a bit of a contrived example, and it's not the way I would implement such functionality if I had to, but the purpose is to demonstrate transactions.

Onward. Thinking about transactions, and keeping in mind that by default Django 1.6 gives us `AUTOCOMMIT` behavior for our database, let's look at the database related code a little longer.

```python
cd = form.cleaned_data
try:
    user = User.create(cd['name'], cd['email'], cd['password'], cd['last_4_digits'])

    if customer:
        user.stripe_id = customer.id
        user.save()
    else:
        UnpaidUsers(email=cd['email']).save()

except IntegrityError:
```

Can you spot any issues?  Well what happens if the `UnpaidUsers(email=cd['email']).save()` line fails?

You will have a user, registered in the system, that the system thinks has verified their credit card, but in reality they haven't verified the card.

We only want one of two outcomes:

1. The user is created (in the database) and has a `stripe_id`.
2. The user is created (in the database) and doesn't have a `stripe_id` AND an associated row in the `UnpaidUsers` table with the same email address is generated.

*Which means we want the two separate database statements to either both commit or both rollback. A perfect case for the humble transaction.*

First, let's write some tests to verify things behave the way we want them to.

```python
@mock.patch('payments.models.UnpaidUsers.save', side_effect = IntegrityError)
def test_registering_user_when_strip_is_down_all_or_nothing(self, save_mock):

    #create the request used to test the view
    self.request.session = {}
    self.request.method='POST'
    self.request.POST = {'email' : 'python@rocks.com',
                         'name' : 'pyRock',
                         'stripe_token' : '...',
                         'last_4_digits' : '4242',
                         'password' : 'bad_password',
                         'ver_password' : 'bad_password',
                        }

    #mock out stripe  and ask it to throw a connection error
    with mock.patch('stripe.Customer.create', side_effect =
                    socket.error("can't connect to stripe")) as stripe_mock:

        #run the test
        resp = register(self.request)

        #assert there is no record in the database without stripe id.
        users = User.objects.filter(email="python@rocks.com")
        self.assertEquals(len(users), 0)

        #check the associated table also didn't get updated
        unpaid = UnpaidUsers.objects.filter(email="python@rocks.com")
        self.assertEquals(len(unpaid), 0)
```

The decorator at the top of the test is a mock that will throw an 'IntegrityError' when we try to save to the `UnpaidUsers` table.

This is to answer the question, "What happens if the `UnpaidUsers(email=cd['email']).save()` line fails?" The next bit of code just creates a mocked session, with the appropriate info we need for our registration function. And then the `with mock.patch` forces the system to believe that Stripe is down ... finally we get to the test.

```python
resp = register(self.request)
```

The above line just calls our register view function passing in the mocked request. Then we just check to ensure the tables are not updated:

```python
#assert there is no record in the database without stripe_id.
users = User.objects.filter(email="python@rocks.com")
self.assertEquals(len(users), 0)

#check the associated table also didn't get updated
unpaid = UnpaidUsers.objects.filter(email="python@rocks.com")
self.assertEquals(len(unpaid), 0)
```

So it should fail if we run the test:

```sh
======================================================================
FAIL: test_registering_user_when_strip_is_down_all_or_nothing (tests.payments.testViews.RegisterPageTests)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/j1z0/.virtualenvs/django_1.6/lib/python2.7/site-packages/mock.py", line 1201, in patched
    return func(*args, **keywargs)
  File "/Users/j1z0/Code/RealPython/mvp_for_Adv_Python_Web_Book/tests/payments/testViews.py", line 266, in test_registering_user_when_strip_is_down_all_or_nothing
    self.assertEquals(len(users), 0)
AssertionError: 1 != 0

----------------------------------------------------------------------
```

Nice. Seems funny to say but that's exactly what we wanted. *Remember: we're practicing TDD here.* The error message tells us that the User is indeed being stored in the database - which is exactly what we don't want because they did not pay!

Transactions to the rescue ...

## Transactions

There are actually several ways to create transactions in Django 1.6.

Let's go through a few.

### The recommended way

According to Django 1.6 [documentation](https://docs.djangoproject.com/en/1.6/topics/db/transactions/), "Django provides a single API to control database transactions. ... Atomicity is the defining property of database transactions. atomic allows us to create a block of code within which the atomicity on the database is guaranteed. If the block of code is  successfully completed, the changes are committed to the database. If there is an exception, the changes are rolled back."

Atomic can be used as both a decorator or as a context_manager.  So if we use it as a context manager, the code in our register function would look like this:

```python
from django.db import transaction

try:
    with transaction.atomic():
        user = User.create(cd['name'], cd['email'], cd['password'], cd['last_4_digits'])

        if customer:
            user.stripe_id = customer.id
            user.save()
        else:
            UnpaidUsers(email=cd['email']).save()

except IntegrityError:
    form.addError(cd['email'] + ' is already a member')
```

Note the line `with transaction.atomic()`. All code inside that block will be executed inside a transaction. So if we re-run our tests, they all should pass! Remember a transaction is a single unit of work, so everything inside the context manager gets rolled back together when the `UnpaidUsers` call fails.

### Using a decorator

We can also try adding atomic as a decorator.

```python
@transaction.atomic():
def register(request):
    ...snip....

    try:
        user = User.create(cd['name'], cd['email'], cd['password'], cd['last_4_digits'])

        if customer:
            user.stripe_id = customer.id
            user.save()
        else:
                UnpaidUsers(email=cd['email']).save()

    except IntegrityError:
        form.addError(cd['email'] + ' is already a member')
```

If we re-run our tests, they will fail with the same error we had before.

Why is that? Why didn't the transaction roll back correctly? The reason is because `transaction.atomic` is looking for some sort of Exception and well, we caught that error (i.e. the `IntegrityError` in our try except block), so `transaction.atomic` never saw it and thus the standard `AUTOCOMMIT` functionality took over.

But of course removing the try except will cause the exception to just be thrown up the call chain and most likely blow up somewhere else. So we can't do that either.

So the trick is to put the atomic context manager inside of the try except block which is what we did in our first solution.  Looking at the correct code again:

```python
from django.db import transaction

try:
    with transaction.atomic():
        user = User.create(cd['name'], cd['email'], cd['password'], cd['last_4_digits'])

        if customer:
            user.stripe_id = customer.id
            user.save()
        else:
            UnpaidUsers(email=cd['email']).save()

except IntegrityError:
    form.addError(cd['email'] + ' is already a member')
```

When `UnpaidUsers` fires the `IntegrityError` the `transaction.atomic()` context manager will catch it and perform the rollback. By the time our code executes in the exception handler, (i.e. the `form.addError` line) the rollback will be done and we could safely make database calls if necessary. Also note any database calls before or after the `transaction.atomic()` context manager will be unaffected regardless of the final outcome of the context_manager.

### Transaction per HTTP Request

Django 1.6 (like 1.5) also allows you to operate in a "Transaction per request" mode. In this mode Django will automatically wrap your view function in a transaction. If the function throws an exception Django will roll back the transaction, otherwise it will commit the transaction.

To get it setup you have to set `ATOMIC_REQUEST` to True in the database configuration for each database that you want to have this behavior.  So in our "settings.py" we make the change like this:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(SITE_ROOT, 'test.db'),
        'ATOMIC_REQUEST': True,
    }
}
```

In practice this just behaves exactly as if you put the decorator on our view function. So it doesn't serve our purposes here.

It is however worthwhile to note that with both `ATOMIC_REQUESTS` and the `@transaction.atomic` decorator it is possible to still catch / handle those errors after they are thrown from the view.  In order to catch those errors you would have to implement some custom middleware, or you could override [urls.hadler500](https://docs.djangoproject.com/en/dev/topics/http/urls/#error-handling) or by making a [500.html template](https://docs.djangoproject.com/en/dev/topics/http/views/#the-500-server-error-view).

## SavePoints

Even though transactions are atomic they can be further broken down into savepoints. Think of savepoints as partial transactions.

So if you have a transaction that takes four SQL statements to complete, you could create a savepoint after the second statement.  Once that savepoint is created, even if the 3rd or 4th statement fail you can do a partial rollback, getting rid of the 3rd and 4th statement but keeping the first two.

So it's basically like splitting a transaction into smaller lightweight transactions allowing you to do partial rollbacks or commits.

> But do keep in mind if the main transaction where to get rolled back (perhaps because of an `IntegrityError` that was raised and not caught, then all savepoints will get rolled back as well).

Lets look at an example of how savepoints work.

```python
@transaction.atomic()
def save_points(self,save=True):

    user = User.create('jj','inception','jj','1234')
    sp1 = transaction.savepoint()

    user.name = 'starting down the rabbit hole'
    user.stripe_id = 4
    user.save()

    if save:
        transaction.savepoint_commit(sp1)
    else:
        transaction.savepoint_rollback(sp1)
```

Here the entire function is in a transaction. After creating a new user we create a savepoint and get a reference to the savepoint.  The next three statements-

```python
user.name = 'starting down the rabbit hole'
user.stripe_id = 4
user.save()
```

-are not part of the existing savepoint, so they stand the chance of being part of the next `savepoint_rollback`, or `savepoint_commit`. In the case of a `savepoint_rollback`, the line ` user = User.create('jj','inception','jj','1234')` will still be committed to the database even though the rest of the updates won't.

Put another way, these following two tests describe how the savepoints work:

```python
def test_savepoint_rollbacks(self):

    self.save_points(False)

    #verify that everything was stored
    users = User.objects.filter(email="inception")
    self.assertEquals(len(users), 1)

    #note the values here are from the original create call
    self.assertEquals(users[0].stripe_id, '')
    self.assertEquals(users[0].name, 'jj')


def test_savepoint_commit(self):
    self.save_points(True)

    #verify that everything was stored
    users = User.objects.filter(email="inception")
    self.assertEquals(len(users), 1)

    #note the values here are from the update calls
    self.assertEquals(users[0].stripe_id, '4')
    self.assertEquals(users[0].name, 'starting down the rabbit hole')
```

Also after we commit or rollback a savepoint we can continue to do work in the same transaction.  And that work will be unaffected by the outcome of the previous savepoint.

For example if we update our `save_points` function as such-

```python
@transaction.atomic()
def save_points(self,save=True):

    user = User.create('jj','inception','jj','1234')
    sp1 = transaction.savepoint()

    user.name = 'starting down the rabbit hole'
    user.save()

    user.stripe_id = 4
    user.save()

    if save:
        transaction.savepoint_commit(sp1)
    else:
        transaction.savepoint_rollback(sp1)

    user.create('limbo','illbehere@forever','mind blown',
           '1111')
```

-regardless of whether `savepoint_commit` or `savepoint_rollback` was called the 'limbo' user will still be created successfully. Unless something else causes the entire transaction to be rolled back.

## Nested Transactions

In addition to manually specifying savepoints, with `savepoint()`, `savepoint_commit`, and `savepoint_rollback`, creating a nested Transaction will automatically create a savepoint for us, and roll it back if we get an error.

Extending our example a bit further we get:

```python
@transaction.atomic()
def save_points(self,save=True):

    user = User.create('jj','inception','jj','1234')
    sp1 = transaction.savepoint()

    user.name = 'starting down the rabbit hole'
    user.save()

    user.stripe_id = 4
    user.save()

    if save:
        transaction.savepoint_commit(sp1)
    else:
        transaction.savepoint_rollback(sp1)

    try:
        with transaction.atomic():
            user.create('limbo','illbehere@forever','mind blown',
                   '1111')
            if not save: raise DatabaseError
    except DatabaseError:
        pass
```

Here we can see that after we deal with our savepoints, we are using the `transaction.atomic` context manager to encase our creation of the 'limbo' user. When that context manager is called, it is in effect creating a savepoint (because we are already in a transaction) and that savepoint will be committed or rolled back upon exiting the context manager.

Thus the following two tests describe their behavior:

```python
 def test_savepoint_rollbacks(self):

    self.save_points(False)

    #verify that everything was stored
    users = User.objects.filter(email="inception")
    self.assertEquals(len(users), 1)

    #savepoint was rolled back so we should have original values
    self.assertEquals(users[0].stripe_id, '')
    self.assertEquals(users[0].name, 'jj')

    #this save point was rolled back because of DatabaseError
    limbo = User.objects.filter(email="illbehere@forever")
    self.assertEquals(len(limbo),0)

def test_savepoint_commit(self):
    self.save_points(True)

    #verify that everything was stored
    users = User.objects.filter(email="inception")
    self.assertEquals(len(users), 1)

    #savepoint was committed
    self.assertEquals(users[0].stripe_id, '4')
    self.assertEquals(users[0].name, 'starting down the rabbit hole')

    #save point was committed by exiting the context_manager without an exception
    limbo = User.objects.filter(email="illbehere@forever")
    self.assertEquals(len(limbo),1)
```

So in reality you can use either `atomic` or `savepoint` to create savepoints inside a transaction. With `atomic` you don't have to worry explicitly about the commit / rollback, where as with `savepoint` you have full control over when that happens.


## Conclusion

If you had any previous experience with earlier versions of Django transactions, you can see how much simpler the transaction model is. Also having `AUTOCOMMIT` on by default is a great example of "sane" defaults that Django and Python both pride themselves on delivering. For many systems you won't need to deal directly with transactions, just let `AUTOCOMMIT` do its work. But if you do, hopefully this post will have given you the information you need to manage transactions in Django like a pro.

