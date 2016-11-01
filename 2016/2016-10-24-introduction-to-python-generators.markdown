# Introduction to Python Generators

Generators are functions that can be paused and resumed on the fly, returning an object that can be iterated over. Unlike lists, they are [lazy](https://en.wikipedia.org/wiki/Lazy_evaluation) and thus produce items one at a time and only when asked. So they are much more memory efficient when dealing with large datasets. **This article details *how* to create generator functions and expressions as well as *why* you would want to use them in the first place.**

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/generators/python-generators.png" style="max-width: 100%;" alt="python generators">
</div>

## Generator Functions

To create a generator, you define a function as you normally would but use the `yield` statement instead of `return`, indicating to the interpreter that this function should be treated as an [iterator](https://en.wikipedia.org/wiki/Iterator):

```python
def countdown(num):
    print('Starting')
    while num > 0:
        yield num
        num -= 1
```

The `yield` statement pauses the function and saves the local state so that it can be resumed right where it left off.

What happens when you call this function?

```sh
>>> def countdown(num):
...     print('Starting')
...     while num > 0:
...         yield num
...         num -= 1
...
>>> val = countdown(5)
>>> val
<generator object countdown at 0x10213aee8>
```

Calling the function does not execute it. We know this because the string `Starting` did not print. Instead, the function returns a generator object which is used to control execution.

Generator objects execute when `next()` is called:

```sh
>>> next(val)
Starting
5
```

When calling `next()` the first time, execution begins at the start of the function body and continues until the next `yield` statement where the value to the right of the statement is returned, subsequent calls to `next()` continue from the `yield` statement to the end of the function, and loop around and continue from the start of the function body until another `yield` is called. If yield is not called (which in our case means we don't go into the if function because num <= 0) a `StopIteration` exception is raised:

```sh
>>> next(val)
4
>>> next(val)
3
>>> next(val)
2
>>> next(val)
1
>>> next(val)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

## Generator Expressions

Just like list comprehensions, generators can also be written in the same manner except they return a generator object rather than a list:

```sh
>>> my_list = ['a', 'b', 'c', 'd']
>>> gen_obj = (x for x in my_list)
>>> for val in gen_obj:
...     print(val)
...
a
b
c
d
```

Take note of the `parens` on either side of the second line denoting a generator expression, which, for the most part, does the same thing that a list comprehension does, but does it lazily:

```sh
>>> import sys
>>> g = (i * 2 for i in range(10000) if i % 3 == 0 or i % 5 == 0)
>>> print(sys.getsizeof(g))
72
>>> l = [i * 2 for i in range(10000) if i % 3 == 0 or i % 5 == 0]
>>> print(sys.getsizeof(l))
38216
>>>
```

Be careful not to mix up the syntax of a list comprehension with a generator expression - `[]` vs `()` - since generator expressions [can run slower](http://stackoverflow.com/a/11964478/1799408) than list comprehensions (unless you run out of memory, of course):

```sh
>>> import cProfile
>>> cProfile.run('sum((i * 2 for i in range(10000000) if i % 3 == 0 or i % 5 == 0))')
         4666672 function calls in 3.531 seconds

   Ordered by: standard name

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
  4666668    2.936    0.000    2.936    0.000 <string>:1(<genexpr>)
        1    0.001    0.001    3.529    3.529 <string>:1(<module>)
        1    0.002    0.002    3.531    3.531 {built-in method exec}
        1    0.592    0.592    3.528    3.528 {built-in method sum}
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}


>>> cProfile.run('sum([i * 2 for i in range(10000000) if i % 3 == 0 or i % 5 == 0])')
         5 function calls in 3.054 seconds

   Ordered by: standard name

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    2.725    2.725    2.725    2.725 <string>:1(<listcomp>)
        1    0.078    0.078    3.054    3.054 <string>:1(<module>)
        1    0.000    0.000    3.054    3.054 {built-in method exec}
        1    0.251    0.251    0.251    0.251 {built-in method sum}
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}
```

This is particularly easy (even for senior developers) to do in the above example since both output the exact same thing in the end.

> **NOTE:** Keep in mind that generator expressions are drastically faster when the size of your data is larger than the available memory.

## Use Cases

Generators are perfect for reading a large number of large files since they yield out data a single chunk at a time irrespective of the size of the input stream. They can also result in cleaner code by decoupling the iteration process into smaller components.

### Example 1

```python
def emit_lines(pattern=None):
    lines = []
    for dir_path, dir_names, file_names in os.walk('test/'):
        for file_name in file_names:
            if file_name.endswith('.py'):
                for line in open(os.path.join(dir_path, file_name)):
                    if pattern in line:
                        lines.append(line)
    return lines
```

This function loops through a set of files in the specified directory. It opens each file and then loops through each line to test for the pattern match.

This works fine with a small number of small files. But, what if we're dealing with extremely large files? And what if there are a lot of them? Fortunately, Python's `open()` function is efficient and doesn't load the entire file into memory. But what if our matches list far exceeds the available memory on our machine?

So, instead of running out of space (large lists) and time (nearly infinite amount of data stream) when processing large amounts of data, generators are the ideal things to use, as they yield out data one time at a time (instead of creating intermediate lists).

Let's look at the generator version of the above problem and try to understand why generators are apt for such use cases using processing pipelines.

We divided our whole process into three different components:

  - Generating set of filenames
  - Generating all lines from all files
  - Filtering out lines on the basis of pattern matching

```python
def generate_filenames():
    """
    generates a sequence of opened files
    matching a specific extension
    """
    for dir_path, dir_names, file_names in os.walk('test/'):
        for file_name in file_names:
            if file_name.endswith('.py'):
                yield open(os.path.join(dir_path, file_name))

def cat_files(files):
    """
    takes in an iterable of filenames
    """
    for fname in files:
        for line in fname:
            yield line

def grep_files(lines, pattern=None):
    """
    takes in an iterable of lines
    """
    for line in lines:
        if pattern in line:
            yield line


py_files = generate_filenames()
py_file = cat_files(py_files)
lines = grep_files(py_file, 'python')
for line in lines:
    print (line)
```

In the above snippet, we do not use any extra variables to form the list of lines, instead we create a pipeline which feeds its components via the iteration process one item at a time. `grep_files` takes in a generator object of all the lines of `*.py` files. Similarly, `cat_files` takes in a generator object of all the filenames in a directory. So this is how the whole pipeline is glued via iterations.

### Example 2

Generators work great for web scraping and crawling recursively:  

```python
import requests
import re


def get_pages(link):
    links_to_visit = []
    links_to_visit.append(link)
    while links_to_visit:
        current_link = links_to_visit.pop(0)
        page = requests.get(current_link)
        for url in re.findall('<a href="([^"]+)">', str(page.content)):
            if url[0] == '/':
                url = current_link + url[1:]
            pattern = re.compile('https?')
            if pattern.match(url):
                links_to_visit.append(url)
        yield current_link


webpage = get_pages('http://sample.com')
for result in webpage:
    print(result)
```

Here, we simply fetch a single page at a time and then perform some sort of action on the page when execution occurs. What would this look like without a generator? Either the fetching and processing would have to happen within the same function (resulting in highly coupled code that's hard to test) or we'd have to fetch all the links before processing a single page.

## Conclusion

Generators allow us to ask for values as and when we need them, making our applications more memory efficient and perfect for infinite streams of data.
They can also be used to refactor out the processing from loops resulting in cleaner, decoupled code. How have you used generators in your own projects?

Want to see more examples? Check out [Generator Tricks for Systems Programmers](http://www.dabeaz.com/generators/).
