# API Integration in Python - part 1

The following is a guest post by Aaron Maxwell, author of <a href="http://migrateup.com/python-newsletter/realpython0b/">the Advanced Python Newsletter</a>.

<hr><br>

<div class="center-text">
  <img class="no-border" src="/images/api-Integration-in-python/rest_logo.png" style="max-width: 100%;" alt="rest logo">
</div>

<br>

## How To Make Friends And Influence APIs

More and more, we're all writing code that works with remote APIs. Your magnificent new app gets a list of your customer's friends, or fetches the coordinates of nearby late-night burrito joints, or starts up a cloud server, or charges a credit card... You get the idea. All this happens just by making an HTTPS request.

(At least, I hope it's HTTPS. Please don't use plain HTTP. But that's a different topic.)

So how do you use this in your code?  Ideally the people behind the web service will provide an SDK or library that does all the above - you can just "pip install openburrito-sdk", or something, and just start using its functions and methods to find late-night burrito joints immediately. Then you don't have to deal with making HTTP requests at all.

That's not always available, though, especially if you are using an internally developed API... very common when your architecture is based on microservices, or trying to be based on microservices. **And that's what this article is about: Writing Python code to integrate with RESTful APIs, in a way that is as fun, easy, and quick as possible - and makes you look good doing it!** (Possibly.)

Sound exciting? Great, let's get started!

## Talking REST

First, if you are not sure what the phrase "REST API" means, <a href="#appendix-rest-in-a-nutshell">jump to the appendix</a> for a crash course.

Got it? Good, let's keep going. There are may ways such web services can be organized; and many formats in which you can pass it data, and get other data back. Right now, it's popular to make your API RESTful, or at least claim it's RESTful. And as for sending data back and forth, the JSON format is very popular.

Tragically, I have so far failed in my campaign to persuade everyone to use YAML instead of JSON. Despite this heartbreaking setback, there is a silver lining: the key principles apply to any HTTP API, using any data format.  So the examples below will be for a REST API, using JSON. But what you are about to learn will apply when I prevail, and we are all joyously using YAML. It also applies to XML or custom data formats; to un-RESTful architectures; or even next year, when we're all checking email via telepathic neural implants.

Let's use a concrete example. Imagine a todo-list API, tracking your action items on your road to success. Here are its methods and endpoints:

<dl>

<dt>GET /tasks/</dt>
<dd>Return a list of items on the todo list, in the format <code>{"id": &lt;item_id&gt;, "summary": &lt;one-line summary&gt;}</code></dd><br>


<dt>GET /tasks/&lt;item_id&gt;/</dt>
<dd>Fetch all available information for a specific todo item, in the format <code>{"id": &lt;item_id&gt;, "summary": &lt;one-line summary&gt;, "description" : &lt;free-form text field&gt;}</code></dd><br>

<dt>POST /tasks/</dt>
<dd>Create a new todo item. The POST body is a JSON object with two fields: "summary" (must be under 120 characters, no newline), and "description" (free-form text field). On success, the status code is 201, and the response body is an object with one field: the id created by the server (e.g., <code>{ "id": 3792 }</code>).</dd><br>

<dt>DELETE /tasks/&lt;item_id&gt;/</dt>
<dd>Mark the item as done. (I.e., strike it off the list, so GET /tasks/ will not show it.) The response body is empty.</dd><br>

<dt>PUT /tasks/&lt;item_id&gt;/</dt>
<dd>Modify an existing task. The PUT body is a JSON object with two fields: "summary" (must be under 120 characters, no newline), and "description" (free-form text field).</dd>

</dl>

> Unless otherwise noted, all actions return 200 on success; those referencing a task ID return 404 if the ID is not found. The response body is empty unless specified otherwise. All non-empty response bodies are JSON. All actions that take a request body are JSON (not form-encoded).

Great. Now, how do we interact with this thing? In Python, we are lucky to have an excellent HTTP library: Kenneth Reitz'
<code>requests</code>. It's one of the few projects worth treating like it is part of the standard library.

```sh
# Step one for every Python app that talks over the web.
$ pip install requests
```

This is your primary tool in writing Python code to use REST APIs - or any service exposed over HTTP, for that matter. It gets all the details right, and has a brilliantly elegant and easy to use interface. You get the point - I'm going to stop with the gushing praise now, and show you how to use it.

Let's say you want to get a list of action items, via the <code>GET /tasks/</code> endpoint:

```python
import requests

resp = requests.get('https://todolist.example.com/tasks/')
if resp.status_code != 200:
    # This means something went wrong.
    raise ApiError('GET /tasks/ {}'.format(resp.status_code))
for todo_item in resp.json():
    print('{} {}'.format(todo_item['id'], todo_item['summary']))
```

Notice that:

<ul>
    <li>The <code>requests</code> module has a function called
    <code>get</code> that does an HTTP GET.</li>
    <li>The response object has a method called
    <code>json</code>. This takes the response body from the server -
    a sequence of bytes - and transforms it to a Python list of
    dictionaries, a la <code>json.loads()</code>.</li>
</ul>


After some error checks and minimal processing, what you get out of the API call is a list of Python dictionaries, each representing a single task. You can then process this however you wish (printing them out, for example).

Now suppose I want to create a new task - add something to my todo list. In our API, this requires an HTTP POST. I start by creating a Python dictionary with the required fields, "summary" and "description", which define the task. Remember how response objects have a convenient <code>.json()</code> method? We can do something similar in the other direction:

```python
task = {"summary": "Take out trash", "description": "" }
resp = requests.post('https://todolist.example.com/tasks/', json=task)
if resp.status_code != 201:
    raise ApiError('POST /tasks/ {}'.format(resp.status_code))
print('Created task. ID: {}'.format(resp.json()["id"]))
```

Notice that:
<ul>
    <li><code>requests</code> sensibly provides a function called
    <code>post</code>, which does an HTTP POST. Dear lord, why can't
    all HTTP libraries be this sane.</li>
    <li>The <code>post</code> function takes a <code>json</code>
    argument, whose value here is a Python dictionary
    (<code>task</code>).</li>
    <li>Per the API spec and REST best practices, we know the task is
    created because of the 201 response code.</li>
</ul>

Now, since we are using JSON as our data format, we were able to take a nice shortcut here: the <code>json</code> argument to <code>post</code>. If we use that, <code>requests</code> will do the following for us:

<ul>
    <li>Convert that into a JSON representation string, a la
    <code>json.dumps()</code>, and</li>
    <li>Sets the requests' content type to <code>"application/json"</code> (by
    adding an HTTP header).</li>
</ul>

If you are using something other than JSON - some custom format; XML; or everybody's favorite, YAML - then you need
to do this manually, which is a bit more work. Here's how it looks:

```python
# The shortcut.
resp = requests.post('https://todolist.example.com/tasks/', json=task)
# The equivalent longer version.
resp = requests.post('https://todolist.example.com/tasks/',
                     data=json.dumps(task),
                     headers={'Content-Type':'application/json'},
```

We use the <code>data</code> argument now; that's how you specify the contents of the POST body. And as you can see,
<code>requests.post</code> takes an optional <code>headers</code> argument - a dictionary. This *adds* each key as a new header field to the request. <code>get()</code> and the others all accept this argument too, by the way.

## Constructing An API Library

If you are doing anything more than a few API calls, you'll want to make your own library to keep yourself sane. And of course, this also applies if *you* are the one providing the API, and want to develop that library, so people can easily use your service.

The structure of the library depends on how the API authenticates, if it does at all. For the moment, let's ignore authentication, to get the basic structure. Then we'll look at how to install the auth layer.

Glance again at the API description above. What are the specific actions and services it provides? In other words, what are some of the things it allows us to do?

<ul>
    <li>We can get a summary list of tasks that need to be done.</li>
    <li>We can get much more detailed information about a specific
    task.</li>
    <li>We can add a new task to our todo list.</li>
    <li>We can mark a task as done.</li>
    <li>We can modify an existing task - changing its description, etc.</li>
</ul>

Instead of thinking about HTTP endpoints, we can create our own internal API based on these concepts. This lets us more easily integrate the logic of using the API in our code, without being distracted by the details.

Let's start with the simplest thing that could possibly work. I'm going to create a file named <code>todo.py</code>, defining the following functions:

```python
# todo.py
def get_tasks():
    pass
def describe_task(task_id):
    pass
def add_task(summary, description=""):
    pass
def task_done(task_id):
    pass
def update_task(task_id, summary, description):
    pass
```

Notice a few design decisions I made here:

<ul>
    <li>All parameters are explicit. For <code>update_task</code>, for
    example, I have three arguments, instead of a single dictionary
    with three keys.</li>
    <li>In <code>add_task</code>, I anticipate that sometimes I will
    want to create a task with just a summary field - "get milk"
    doesn't really need elaboration, for example - so give
    <code>description</code> a sensible default.</li>
    <li>These are functions in a module. That is a useful organization
    in Python. (In Java, I'd have to create a class with methods, for
    example.)</li>
</ul>

To fill these out, I am going to define a helper:

```python
def _url(path):
    return 'https://todo.example.com' + path
```

This just constructs the full URL to make the API call, relative to the path. With this, implementing the helper is
straightforward:

```python
import requests

def get_tasks():
    return requests.get(_url('/tasks/'))

def describe_task(task_id):
    return requests.get(_url('/tasks/{:d}/'.format(task_id)))

def add_task(summary, description=""):
    return requests.post(_url('/tasks/'), json={
        'summary': summary,
        'description': description,
        })

def task_done(task_id):
    return requests.delete(_url('/tasks/{:d}/'.format(task_id)))

def update_task(task_id, summary, description):
    url = _url('/tasks/{:d}/'.format(task_id))
    return requests.put(url, json={
        'summary': summary,
        'description': description,
        })
```

I can use this like so:

```python
import todo

resp = todo.add_task("Take out trash")
if resp.status_code != 201:
    raise ApiError('Cannot create task: {}'.format(resp.status_code))
print('Created task. ID: {}'.format(resp.json()["id"]))

resp = todo.get_tasks()
if resp.status_code != 200:
    raise ApiError('Cannot fetch all tasks: {}'.format(resp.status_code))
for todo_item in resp.json():
    print('{} {}'.format(todo_item['id'], todo_item['summary']))
```

Notice that each of my library functions returns a response object, just like <code>requests.get</code> and friends. This is often a useful choice. Generally when working with APIs, you will want to inspect the status code, and also the payload (from <code>resp.json()</code> in this case). The response object provides easy access to this, and other information we might need, but did not anticipate when we first made the library.

You might be thinking this is exposing implementation details. Wouldn't it be better to construct some kind of <code>ApiResponse</code> class, that provides the needed info through a more explicit interface? While that can sometimes be the best approach, I have more often found it to be over-engineering. I recommend you start by just returning simple response objects. If you later need to install a response abstraction layer, you'll know when the time comes.

## Coming in Part 2

Your head may be swimming a little now. And it's not just because of the subliminal suggestions I've encoded throughout the background CSS, extolling the virtues of YAML. You see, we have covered a lot of important ground here... solidly rooted in modern engineering best practices.

And yet, there's more to come. Part 2 will extend our work here to deal with pagination, or getting large bodies of data that take multiple requests to fetch; authentication; and reliability - in other words, dealing with flakey APIs. To be notified when it's online, <a href="http://migrateup.com/python-newsletter/realpython0b/">subscribe to the Advanced Python Newsletter</a>.

**Be sure to also check out the [Real Python](https://realpython.com) courses to learn how to design RESTful APIs with both Flask and Django.**

## Appendix: REST in a nutshell

REST is essentially a set of useful conventions for structuring a web API. By "web API", I mean an API that you interact with over HTTP - making requests to specific URLs, and often getting relevant data back in the response.

There are whole books written about this topic, but I can give you a quick start here. In HTTP, we have different "methods", as they are called. GET and POST are the most common; these are used by web browsers to load a page and submit a form, respectively. In REST, you use these to to indicate different actions. **GET** is generally used to get information about some object or record that already exists. Crucially, the GET does not modify anything, or at least isn't supposed to. For example, imagine a kind of todo-list web service. You might do an HTTP GET to the url "/tasks/" to get a list of current tasks to be done. So it may return something like this:

```json
[
  { "id": 3643, "summary": "Wash car" },
  { "id": 3697, "summary": "Visit gym" }
]
```

This is a list of JSON objects. (A "JSON object" is a data type very similar to a Python dictionary.)

In contrast, **POST** is typically used when you want to create something. So to add a new item to the todo list, you might trigger an HTTP POST to "/tasks/". That's right, it is the same URL: that is allowed in REST. The different methods GET and POST are like different verbs, and the URL is like a noun.

When you do a POST, normally you will include a body in the request. That means you send along some sequence of bytes - some data defining the object or record you are creating. What kind of data? These days, it's very common to pass JSON objects. So the API may state that a POST to /tasks/ must include a single object with two fields, "summary" and "description", like this:

```json
{
  "summary": "Get milk",
  "description": "Need to get a half gallon of organic 2% milk."
}
```

This is a string, encoding a JSON object. The API server then parses it and creates the equivalent Python dictionary.

What happens next? Well, that depends on the API, but generally speaking you will get a response back with some useful information, along two dimensions. First is the *status code*. This is a positive number, something like 200 or 404 or 302. The meaning of each status code is well defined by the HTTP protocol standard; search for "http status codes" and the first hit will probably be the official reference. Anything in the 200s indicates success.

The other thing you get back is the *response body*. When your web browser GETs a web page, the HTML sent back is the response body. For an API, this can response body can be empty, or not - it depends on the API and the end point. For example, when we POST to /tasks/ to add something to our todo list, we may get back an automatically assigned task ID. This can again be in the form of a JSON object:

```json
{ "id": 3792 }
```

Then if we GET /tasks/ again, our list of tasks will include this new one:

```json
[
  { "id": 3643, "summary": "Wash car" },
  { "id": 3697, "summary": "Visit gym" },
  { "id": 3792, "summary": "Get milk" }
]
```

There are other methods besides GET and POST. In the HTTP standard, **PUT** is used to modify an existing resource (e.g., change a task's summary). Another method called **DELETE** will... well, delete it. You could use this when a task is done, to remove it from your list.

There is a lot more to REST than this. However, this is enough for you to get started. <a href="#talking-rest">Jump back to "Talking REST"</a>.