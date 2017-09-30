# Inner functions - what are they good for?

Let's look at three common reasons for writing inner functions.

**Remember**: In Python, a function is a "first-class" [citizen](https://realpython.com/blog/python/primer-on-python-decorators/#first-things-first), meaning they are on par with any other object (i.e., integers, strings, lists, modules, etc.). You can dynamically create or destroy them, pass them to other functions, return them as values, and so forth.

> This tutorial utilizes Python version 3.4.1.

## 1. Encapsulation

You use inner functions to [protect](http://en.wikipedia.org/wiki/Information_hiding) them from *anything* happening outside of the function, meaning that they are hidden from the global scope.

Here's a simple example that highlights that concept:

```python
def outer(num1):
    def inner_increment(num1):  # hidden from outer code
        return num1 + 1
    num2 = inner_increment(num1)
    print(num1, num2)

inner_increment(10)
# outer(10)
```

Try calling the `inner_increment()` function:

```
Traceback (most recent call last):
  File "inner.py", line 7, in <module>
    inner_increment()
NameError: name 'inner_increment' is not defined
```

Now comment out the `inner_increment` call and uncomment the outer function call, `outer(10)`, passing in `10` as the argument:

```
10 11
```

> Keep in mind that this is just an example. Although this code does achieve the desired result, it's probably better to make the `inner_increment()` function a top-level "private" function using a leading underscore: `_inner_increment()`.

The following recursive example is a slightly better use case for a nested function:

```python
def factorial(number):

    # error handling
    if not isinstance(number, int):
        raise TypeError("Sorry. 'number' must be an integer.")
    if not number >= 0:
        raise ValueError("Sorry. 'number' must be zero or positive.")

    def inner_factorial(number):
        if number <= 1:
            return 1
        return number*inner_factorial(number-1)
    return inner_factorial(number)

# call the outer function
print(factorial(4))
```

Test this out as well. One main advantage of using this design pattern is that by performing all argument checking in the outer function, you can safely skip error checking altogether in the inner function.

> For a more detailed discussion of recursion see, [Problem Solving with Algorithms and Data Structures](http://interactivepython.org/courselib/static/pythonds/Recursion/recursionsimple.html).

## 2. Keepin' it [DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself)

Perhaps you have a giant function that performs the same chunk of code in numerous places. For example, you might write a function which processes a file, and you want to accept either an open file object or a file name:

```python
def process(file_name):
    def do_stuff(file_process):
        for line in file_process:
            print(line)
    if isinstance(file_name, str):
        with open(file_name, 'r') as f:
            do_stuff(f)
    else:
        do_stuff(file_name)
```

> Again, it is common to just make `do_stuff()` a private top-level function, but if you want to hide it away as an internal function, you can.

How about a practical example?

Let's say you want to know the number of WiFi hot spots in New York City. And yes the city has the raw data to tell us: [datasource](https://data.cityofnewyork.us/Recreation/Wifi-Hotspot-Locations/ehc4-fktp). Visit the site and download the CSV.

```python
def process(file_name):

    def do_stuff(file_process):
        wifi_locations = {}

        for line in file_process:
            values = line.split(',')
            # Build the dict, and increment values
            wifi_locations[values[1]] = wifi_locations.get(values[1], 0) + 1

        max_key = 0
        for name, key in wifi_locations.items():
            all_locations = sum(wifi_locations.values())
            if key > max_key:
                max_key = key
                business = name
        print('There are {0} WiFi hot spots in NYC and {1} has the most with {2}.'.format(
            all_locations, business, max_key))

    if isinstance(file_name, str):
        with open(file_name, 'r') as f:
            do_stuff(f)
    else:
        do_stuff(file_name)


process("NAME_OF_THE.csv")
```

Run the function:

```
There are 1251 WiFi hot spots in NYC and Starbucks has the most with 212.
```

## 3. Closures and Factory Functions

Now we come to the most important reason to use inner functions. All of the inner function examples we've seen so far have been ordinary functions that merely happened to be nested inside another function. In other words, we could have defined these functions in another way (as discussed); there is no specific reason for why they *should* be nested.

But when it comes to closure, that is not the case: You must utilize nested functions.

### What's a closure?

A closure simply causes the inner function to *remember* the state of its environment when called. Beginners often think that a closure *is* the inner function, when it's really *caused* by the inner function. The closure "closes" the local variable on the stack and this stays around after the the stack creation has finished executing.

### An example

```python
def generate_power(number):
    """
    Examples of use:

    >>> raise_two = generate_power(2)
    >>> raise_three = generate_power(3)
    >>> print(raise_two(7))
    128
    >>> print(raise_three(5))
    243
    """

    # define the inner function ...
    def nth_power(power):
        return number ** power
    # ... which is returned by the factory function

    return nth_power
```

### What's happening here?

1. The 'generate_power()' function is a *factory function* - which simply means that it creates a new function each time it is called and then returns the newly created function. Thus, `raise_two` and `raise_three` are the newly created functions.
1. What does this new, inner function do? It takes a single argument, `power`, and returns `number**power`.
1. Where does the inner function get the value of `number` from? This is where the closure comes into play: `nth_power()` gets the value of `power` from the outer function, the *factory function*. Let's step through this process:

    - Call the outer function: `generate_power(2)`
    - Build the `nth_power()` function which takes a single argument `power`
    - Take a snapshot of the state of `nth_power()` which includes `power=2`
    - Pass that snapshot into the `generate_power()` function
    - Return the `nth_power()` function

    Put another way, the closure functions to "initialize" the number bar in the `nth_power()` function and then returns it. Now, whenever you call that newly returned function, it will always see its own private snapshot that includes `power=2`.

### Real World

How about a real world example?

```python
def has_permission(page):
    def inner(username):
        if username == 'Admin':
            return "'{0}' does have access to {1}.".format(username, page)
        else:
            return "'{0}' does NOT have access to {1}.".format(username, page)
    return inner


current_user = has_permission('Admin Area')
print(current_user('Admin'))

random_user = has_permission('Admin Area')
print(random_user('Not Admin'))
```

This is a simplified function to check if a certain user has the correct permissions to access a certain page. You could easily modify this to grab the user in session to check if they have the correct credentials to access a certain route. Instead of checking if the user is just equal to 'Admin', you could query the database to check the permission then return the correct view depending on whether the credentials are correct or not.

## Conclusion

The use of closures and factory functions is the most common, and powerful, use for inner functions. In most cases, when you see a decorated function, the decorator is a factory function which takes a function as argument, and returns a new function which includes the old function inside the closure. Stop. Take a deep breath. Grab a coffee. Read that again.

Put another way, a decorator is just syntactic sugar for implementing the process outlined in the `generate_power()` example.

I'll leave you with an example:

```python
def generate_power(exponent):
    def decorator(f):
        def inner(*args):
            result = f(*args)
            return exponent**result
        return inner
    return decorator


@generate_power(2)
def raise_two(n):
    return n

print(raise_two(7))


@generate_power(3)
def raise_three(n):
    return n

print(raise_two(5))
```

If your code editor allows it, view the `generate_power(exponent)` and `generate_power(number)` functions side-by-side to illustrate the concepts discussed. (Sublime Text has Column View, for example).

If you have not coded the two functions, open the code editor and start coding. For new programmers, coding is a hands on activity, like riding a bike you just have to do it - and do it solo. So back to the task at hand. After you typed the code, you can now clearly see that the code is similar in that it produces the same results but there are differences. For those who have never used decorators, noting these differences will be the first step in understanding them if you venture down that path.

<hr>

**If you'd like to know more about this syntax and decorators in general, check out our [Primer on Python Decorators](https://realpython.com/blog/python/primer-on-python-decorators/). Comment below with questions.**

<br>

<p style="font-size: 14px;">
  <em>Edits made by <a href="https://twitter.com/diek007">Derrick Kearney</a>. Thank you!</em>
</p>
