# Primer on Python Decorators

Decorators provide a simple syntax for calling [higher-order functions](http://en.wikipedia.org/wiki/Higher-order_function). By definition, a decorator is a function that takes another function and extends the behavior of the latter function without explicitly modifying it. Sounds confusing - but it's really not, especially after we go over a number of examples.

**In this introductory tutorial, we'll look at what decorators are and how to create and use them.**

> You can find all the examples from this article [here](https://github.com/mjhea0/python-decorators).

*Updated 11/01/2015:* Added a brief explanation on the `functools.wraps()` decorator.

## First things first

Before you can understand decorators, you must first understand ...

### How functions work

Essentially, **functions return a value based on the given arguments**.

```python
def foo(bar):
  return bar + 1

print foo(2) == 3
```

### First Class Objects

In Python, functions are **[first-class](http://python-history.blogspot.com/2009/02/first-class-everything.html) objects**. This means that functions can be passed around, and used as arguments, just like any other value (e.g, string, int, float).

```python
def foo(bar):
  return bar+1

print foo
print foo(2)
print type(foo)

def call_foo_with_arg(foo, arg):
  return foo(arg)

print call_foo_with_arg(foo, 3)
```

### Nested Functions

Because of the first-class nature of functions in Python, you can **define functions inside other functions**. Such functions are called nested functions.

```python
def parent():
  print "Printing from the parent() function."

  def first_child():
      return "Printing from the first_child() function."

  def second_child():
      return "Printing from the second_child() function."

  print first_child()
  print second_child()
```

What happens when you call the `parent()` function? Think about this for a minute. You should get ...

```sh
Printing from the parent() function.
Printing from the first_child() function.
Printing from the second_child() function
```

Try calling the `first_child()`. You should get an error:

```sh
Traceback (most recent call last):
File "decorator3.py", line 15, in <module>
first_child()
NameError: name 'first_child' is not defined
```

**What have we learned?**

Whenever you call `parent()`, the sibling functions, `first_child()` and `second_child()` are also called *AND*
because of scope, both of the sibling functions are not available (e.g., cannot be called) outside of the parent function.

### Returning Functions

Python also allows you to **return functions from other functions**. Let's alter the previous function for this example.

```python
def parent(num):

  def first_child():
      return "Printing from the first_child() function."

  def second_child():
      return "Printing from the second_child() function."

  try:
      assert num == 10
      return first_child
  except AssertionError:
      return second_child

foo = parent(10)
bar = parent(11)

print foo
print bar

print foo()
print bar()
```

The output of the first two print statements is:

```sh
<function first_child at 0x1004a8c08>
<function second_child at 0x1004a8cf8>
```

This simply means that `foo` points to the `first_child()` function, while `bar` points to the `second_child()` function.

The output of the second two functions confirms this:

```sh
Printing from the first_child() function.
Printing from the second_child() function.
```

Finally, did you notice that in example three, we executed the sibling functions within the parent functions - e.g, `second_child()`. Meanwhile in this last example, we did not add parenthesis to the sibling functions - `first_child` - when called so that way we can use them in the future. Make sense?


## Now, my friend, you are ready to take on decorators!

Let's look at two examples ...

### Example 1:

```python
def my_decorator(some_function):

  def wrapper():

      print "Something is happening before some_function() is called."

      some_function()

      print "Something is happening after some_function() is called."

  return wrapper

def just_some_function():
  print "Wheee!"


just_some_function = my_decorator(just_some_function)

just_some_function()
```

Can you guess what the output will be? Try.

```sh
Something is happening before some_function() is called.
Wheee!
Something is happening after some_function() is called.
```

To understand what's going on here, just look back at the four previous examples. We are literally just applying everything learned. **Put simply, decorators wrap a function, modifying its behavior.**

Let's take it one step further and add an if statement.

### Example 2:

```python
def my_decorator(some_function):

  def wrapper():

      num = 10

      if num == 10:
          print "Yes!"
      else:
          print "No!"

      some_function()

      print "Something is happening after some_function() is called."

  return wrapper

def just_some_function():
 print "Wheee!"

just_some_function = my_decorator(just_some_function)

just_some_function()
```

This will output in:

```sh
Yes!
Wheee!
Something is happening after some_function() is called.
```

## Time for some syntactic sugar!

Python allows you to simplify the calling of decorators using the `@` symbol (this is called "pie" syntax):

### Let's create a module for our decorator:

```python
def my_decorator(some_function):

  def wrapper():

      num = 10

      if num == 10:
          print "Yes!"
      else:
          print "No!"

      some_function()

      print "Something is happening after some_function() is called."

  return wrapper

if __name__ == "__main__":
  my_decorator()
```

Okay. Stay with me. Let's look at how to call the function with the decorator:

```python
from decorator7 import my_decorator

@my_decorator
def just_some_function():
  print "Wheee!"

just_some_function()
```

When you run this example, you should get the same output as in the previous one:

```sh
Yes!
Wheee!
Something is happening after some_function() is called.
```

So, `@my_decorator` is just an easier way of saying `just_some_function = my_decorator(just_some_function)`.

## Real World

How about a few real world examples ...

```python
import time

def timing_function(some_function):

    """
    Outputs the time a function takes
    to execute.
    """

    def wrapper():
        t1 = time.time()
        some_function()
        t2 = time.time()
        return "Time it took to run the function: " + str((t2-t1)) + "\n"
    return wrapper

@timing_function
def my_function():
    num_list = []
    for x in (range(0,10000)):
        num_list.append(x)
    print "\nSum of all the numbers: " +str((sum(num_list)))


print my_function()
```

This returns the time before you run `my_function()` as well as the time after. Then we simply subtract the two to see how long it took to run the function.

Run it. Work through the code, line by line. Make sure you understand how it works.

```python
from time import sleep


def sleep_decorator(function):

    """
    Limits how fast the function is
    called.
    """

    def wrapper(*args, **kwargs):
        sleep(2)
        return function(*args, **kwargs)
    return wrapper


@sleep_decorator
def print_number(num):
    return num

print print_number(222)

for x in range(1,6):
    print print_number(x)
```

This decorator is used for rate limiting. Test it out.

Finally one of the most used decorators in Python is the `login_required()` decorator, which ensures that a user is logged in/properly authenticated before s/he can access a specific route (`/secret`, in this case):

```python
from functools import wraps
from flask import g, request, redirect, url_for

def login_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if g.user is None:
            return redirect(url_for('login', next=request.url))
        return f(*args, **kwargs)
    return decorated_function

@app.route('/secret')
@login_required
def secret():
    pass
```

Did you notice that the function gets passed to the `functools.wraps()` [decorator](https://docs.python.org/2/library/functools.html#functools.wraps)? This simply [preserves](http://stackoverflow.com/questions/308999/what-does-functools-wraps-do) the metadata of the wrapped function.

**Cheers!**
