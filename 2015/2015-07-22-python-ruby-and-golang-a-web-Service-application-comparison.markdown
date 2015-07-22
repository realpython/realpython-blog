# Python, Ruby, and Golang: A Web Service Application Comparison

<div class="center-text">
  <img class="no-border" src="/images/blog_images/python-ruby-golang/flask-sinatra-martini.png" style="max-width: 100%;" alt="python, ruby, and golang images">
</div>

<br>

**After a [recent comparison](https://realpython.com/blog/python/python-ruby-and-golang-a-command-line-application-comparison/) of Python, Ruby, and Golang for a command-line application I decided to use the same pattern to compare building a simple web service. I have selected [Flask](http://flask.pocoo.org/) (Python), [Sinatra](http://www.sinatrarb.com/) (Ruby), and [Martini](http://martini.codegangsta.io/) (Golang) for this comparison.** Yes, there are many other options for web application libraries in each language but I felt these three lend well to comparison.

*This is a guest blog post by **[Kyle Purdon](https://twitter.com/PurdonKyle)​**, a software engineer from Boulder.*

## Library Overviews

Here is a high-level comparison of the libraries by [Stackshare](http://stackshare.io/stackups/martini-vs-flask-vs-sinatra).

### Flask (Python)

> Flask is a micro-framework for Python based on Werkzeug, Jinja2 and good intentions.

For very simple applications, such as the one shown in this demo, Flask is a great choice. The basic Flask application is only 7 lines of code (LOC) in a single Python source file. The draw of Flask over other Python web libraries (such as [Django](https://www.djangoproject.com/) or [Pyramid](http://www.pylonsproject.org/)) is that you can start small and build up to a more complex application as needed.

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run()
```

### Sinatra (Ruby)

> Sinatra is a [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) for quickly creating web applications in Ruby with minimal effort.

Just like Flask, Sinatra is great for simple applications. The basic Sinatra application is only 4 LOC in a single Ruby source file. Sinatra is used instead of libraries such as [Ruby on Rails](http://rubyonrails.org/) for the same reason as Flask - you can start small and expand the application as needed.

```ruby
require 'sinatra'

get '/hi' do
  "Hello World!"
end
```

### Martini (Golang)

> Martini is a powerful package for quickly writing modular web applications/services in Golang.

Martini comes with a few more batteries included than both Sinatra and Flask but is still very lightweight to start with - only 9 LOC for the basic application. Martini has come under some [criticism](https://stephensearles.com/three-reasons-you-should-not-use-martini/) by the Golang community but still has one of the highest rated Github projects of any Golang web framework. The author of Martini responded directly to the criticism [here](http://codegangsta.io/blog/2014/05/19/my-thoughts-on-martini/). Some other frameworks include [Revel](https://revel.github.io/), [Gin](https://gin-gonic.github.io/gin/), and even the built-in [net/http](http://golang.org/pkg/net/http/) library.

```go
package main

import "github.com/go-martini/martini"

func main() {
  m := martini.Classic()
  m.Get("/", func() string {
    return "Hello world!"
  })
  m.Run()
}
```

With the basics out of the way, let's build an app!

## Service Description

The service created provides a very basic blog application. The following routes are constructed:

* `GET /`: Return the blog (using a template to render).
* `GET /json`: Return the blog content in JSON format.
* `POST /new`: Add a new post (title, summary, content) to the blog.

The external interface to the blog service is exactly the same for each language. For simplicity MongoDB will be used as the data store for this example as it is the simplest to set up and we don't need to worry about schemas at all. In a normal "blog-like" application a relational database would likely be necessary.

### Add A Post

`POST /new`

```sh
curl --form title='Test Post 1' \
     --form summary='The First Test Post' \
     --form content='Lorem ipsum dolor sit amet, consectetur ...' \
     http://[IP]:[PORT]/new
```

### View The HTML

`GET /`

<div class="center-text">
  <img class="no-border" src="/images/blog_images/python-ruby-golang/blog.png" style="max-width: 100%; border: 1px solid black;" alt="python, ruby, and golang images">
</div>

<br>

### View The JSON

`GET /json`

```javascript
[
   {
      content:"Lorem ipsum dolor sit amet, consectetur ...",
      title:"Test Post 1",
      _id:{
         $oid:"558329927315660001550970"
      },
      summary:"The First Test Post"
   }
]
```

## Application Structure

Each application can be broken down into the following components:

### Application Setup

* Initialize an application
* Run the application

### Request

* Define routes on which a user can request data (GET)
* Define routes on which a user can submit data (POST)

### Response

* Render JSON (`GET /json`)
* Render a template (`GET /`)

### Database

* Initialize a connection
* Insert data
* Retrieve data

### Application Deployment

* Docker!

<hr>

**The rest of this article will compare each of these components for each library.** The purpose is not to suggest that one of these libraries is better than the other - it is to provide a specific comparison between the three tools:

* [Flask](http://flask.pocoo.org/) (Python)
* [Sinatra](http://www.sinatrarb.com/) (Ruby)
* [Martini](http://martini.codegangsta.io/) (Golang)

## Project Setup

All of the [projects](https://github.com/realpython/flask-sinatra-martini) are bootstrapped using [docker](https://www.docker.com/) and [docker-compose](https://docs.docker.com/compose/). Before diving into how each application is bootstrapped under the hood we can just use docker to get each up and running in exactly the same way - `docker-compose up`

Seriously, that's it! Now for each application there is a `Dockerfile` and a `docker-compose.yml` file that specify what happens when you run the above command.

### Python (flask) - *Dockerfile*

```
FROM python:3.4

ADD . /app
WORKDIR /app

RUN pip install -r requirements.txt
```

This `Dockerfile` says that we are starting from a base image with Python 3.4 installed, adding our application to the `/app` directory and using [pip](https://pypi.python.org/pypi/pip) to install our application requirements specified in `requirements.txt`.

### Ruby (sinatra)

```
FROM ruby:2.2

ADD . /app
WORKDIR /app

RUN bundle install
```

This `Dockerfile` says that we are starting from a base image with Ruby 2.2 installed, adding our application to the `/app` directory and using [bundler](http://bundler.io/) to install our application requirements specified in the `Gemfile`.

### Golang (martini)

```
FROM golang:1.3

ADD . /go/src/github.com/kpurdon/go-blog
WORKDIR /go/src/github.com/kpurdon/go-blog

RUN go get github.com/go-martini/martini && \
    go get github.com/martini-contrib/render && \
    go get gopkg.in/mgo.v2 && \
    go get github.com/martini-contrib/binding
```

This `Dockerfile` says that we are starting from a base image with Golang 1.3 installed, adding our application to the `/go/src/github.com/kpurdon/go-blog` directory and getting all of our necessary dependencies using the `go get` command.

## Initialize/Run An Application

### Python (Flask) - *app.py*

```python
# initialize application
from flask import Flask
app = Flask(__name__)

# run application
if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

```sh
$ python app.py
```

### Ruby (Sinatra) - *app.rb*

```ruby
# initialize application
require 'sinatra'
```

```sh
$ ruby app.rb
```

### Golang (Martini) - *app.go*

```go
// initialize application
package main
import "github.com/go-martini/martini"
import "github.com/martini-contrib/render"

func main() {

    app := martini.Classic()
    app.Use(render.Renderer())

    // run application
    app.Run()

}
```

```sh
$ go run app.go
```

## Define A Route (GET/POST)

### Python (Flask)

```python
# get
@app.route('/')  # the default is GET only
def blog():
    ...

#post
@app.route('/new', methods=['POST'])
def new():
    ...
```

### Ruby (Sinatra)

```ruby
# get
get '/' do
  ...
end

# post
post '/new' do
  ...
end
```

### Golang (Martini)

```go
// define data struct
type Post struct {
  Title   string `form:"title" json:"title"`
  Summary string `form:"summary" json:"summary"`
  Content string `form:"content" json:"content"`
}

// get
app.Get("/", func(r render.Render) {
  ...
}

// post
import "github.com/martini-contrib/binding"
app.Post("/new", binding.Bind(Post{}), func(r render.Render, post Post) {
  ...
}
```

## Render A JSON Response

### Python (Flask)

Flask provides a [jsonify()](http://flask.pocoo.org/docs/0.10/api/#flask.json.jsonify) method but since the service is using MongoDB the mongodb bson utility is used.

```python
from bson.json_util import dumps
return dumps(posts) # posts is a list of dicts [{}, {}]
```

### Ruby (Sinatra)

```ruby
require 'json'
content_type :json
posts.to_json # posts is an array (from mongodb)
```

### Golang (Martini)

```go
r.JSON(200, posts) // posts is an array of Post{} structs
```

## Render An HTML Response (Templating)

### Python (Flask)

```python
return render_template('blog.html', posts=posts)
```

```html
<!doctype HTML>
<html>
  <head>
    <title>Python Flask Example</title>
  </head>
  <body>{% raw %}
    {% for post in posts %}
      <h1> {{ post.title }} </h1>
      <h3> {{ post.summary }} </h3>
      <p> {{ post.content }} </p>
      <hr>
    {% endfor %}
  {% endraw %}</body>
</html>
```

### Ruby (Sinatra)

```ruby
erb :blog
```

```html
<!doctype HTML>
<html>
  <head>
    <title>Ruby Sinatra Example</title>
  </head>
  <body>
    <% @posts.each do |post| %>
      <h1><%= post['title'] %></h1>
      <h3><%= post['summary'] %></h3>
      <p><%= post['content'] %></p>
      <hr>
    <% end %>
  </body>
</html>
```

### Golang (Martini)

```go
r.HTML(200, "blog", posts)
```

```html
<!doctype HTML>
<html>
  <head>
    <title>Golang Martini Example</title>
  </head>
  <body>{% raw %}
    {{range . }}
      <h1>{{.Title}}</h1>
      <h3>{{.Summary}}</h3>
      <p>{{.Content}}</p>
      <hr>
    {{ end }}
  {% endraw %}</body>
</html>
```

## Database Connection

All of the applications are using the mongodb driver specific to the language. The environment variable `DB_PORT_27017_TCP_ADDR` is the IP of a linked docker container (the database ip).

### Python (Flask)

```python
from pymongo import MongoClient
client = MongoClient(os.environ['DB_PORT_27017_TCP_ADDR'], 27017)
db = client.blog
```

### Ruby (Sinatra)

```ruby
require 'mongo'
db_ip = [ENV['DB_PORT_27017_TCP_ADDR']]
client = Mongo::Client.new(db_ip, database: 'blog')
```

### Golang (Martini)

```go
import "gopkg.in/mgo.v2"
session, _ := mgo.Dial(os.Getenv("DB_PORT_27017_TCP_ADDR"))
db := session.DB("blog")
defer session.Close()
```

## Insert Data From a POST

### Python (Flask)

```python
from flask import request
post = {
    'title': request.form['title'],
    'summary': request.form['summary'],
    'content': request.form['content']
}
db.blog.insert_one(post)
```

### Ruby (Sinatra)

```ruby
client[:posts].insert_one(params) # params is a hash generated by sinatra
```

### Golang (Martini)

```go
db.C("posts").Insert(post) // post is an instance of the Post{} struct
```

## Retrieve Data

### Python (Flask)

```python
posts = db.blog.find()
```

### Ruby (Sinatra)

```ruby
@posts = client[:posts].find.to_a
```

### Golang (Martini)

```go
var posts []Post
db.C("posts").Find(nil).All(&posts)
```

## Application Deployment (docker!)

A great solution to deploying all of these applications is to use [docker](https://www.docker.com/) and [docker-compose](https://docs.docker.com/compose/).

### Python (Flask)

**Dockerfile**

```
FROM python:3.4

ADD . /app
WORKDIR /app

RUN pip install -r requirements.txt
```

**docker-compose.yml**

```yaml
web:
  build: .
  command: python -u app.py
  ports:
    - "5000:5000"
  volumes:
    - .:/app
  links:
    - db
db:
  image: mongo:3.0.4
  command: mongod --quiet --logpath=/dev/null
```

### Ruby (Sinatra)

**Dockerfile**

```
FROM ruby:2.2

ADD . /app
WORKDIR /app

RUN bundle install
```

**docker-compose.yml**

```yaml
web:
  build: .
  command: bundle exec ruby app.rb
  ports:
    - "4567:4567"
  volumes:
    - .:/app
  links:
    - db
db:
  image: mongo:3.0.4
  command: mongod --quiet --logpath=/dev/null
```

### Golang (Martini)

**Dockerfile**

```
FROM golang:1.3

ADD . /go/src/github.com/kpurdon/go-todo
WORKDIR /go/src/github.com/kpurdon/go-todo

RUN go get github.com/go-martini/martini && go get github.com/martini-contrib/render && go get gopkg.in/mgo.v2 && go get github.com/martini-contrib/binding
```

**docker-compose.yml**

```yaml
web:
  build: .
  command: go run app.go
  ports:
    - "3000:3000"
  volumes: # look into volumes v. "ADD"
    - .:/go/src/github.com/kpurdon/go-todo
  links:
    - db
db:
  image: mongo:3.0.4
  command: mongod --quiet --logpath=/dev/null
```

## Conclusion

To conclude lets take a look at what I believe are a few categories where the presented libraries separate themselves from each other.

### Simplicity

While Flask is very lightweight and reads clearly, the Sinatra app is the simplest of the three at 23 LOC (compared to 46 for Flask and 42 for Martini). For these reasons Sinatra is the winner in this category. It should be noted however that Sinatra's simplicity is due to more default "magic" - e.g., implicit work that happens behind the scenes. For new users this can often lead to confusion.

Here is a specific example of "magic" in Sinatra:

```ruby
params # the "request.form" logic in python is done "magically" behind the scenes in Sinatra.
```

And the equivalent Flask code:

```python
from flask import request
params = {
    'title': request.form['title'],
    'summary': request.form['summary'],
    'content': request.form['content']
}
```

For beginners to programming Flask and Sinatra are certainly simpler, but for an experienced programmer with time spent in other statically typed languages Martini does provide a fairly simplistic interface.

### Documentation

The Flask documentation was the simplest to search and most approachable. While Sinatra and Martini are both well documented, the documentation itself was not as approachable. For this reason Flask is the winner in this category.

### Community

Flask is the winner hands down in this category. The Ruby community is more often than not dogmatic about Rails being the only good choice if you need anything more than a basic service (even though [Padrino](http://www.padrinorb.com/) offers this on top of Sinatra). The Golang community is still nowhere near a consensus on one (or even a few) web frameworks, which is to be expected as the language itself is so young. Python however has embraced a number of approaches to web development including Django for out-of-the-box full-featured web applications and Flask, Bottle, CheryPy, and Tornado for a micro-framework approach.

## Final Determination

Note that the point of this article was not to promote a single tool, rather to provide an unbiased comparison of Flask, Sinatra, and Martini. With that said, I would select Flask (Python) or Sinatra (Ruby). If you are coming from a language like C or Java perhaps the statically-typed nature of Golang may appeal to you. If you are a beginner, Flask might be the best choice as it is very easy to get up and running and there is very little default "magic". My recommendation is that you be flexible in your decisions when selecting a library for your project.

<hr>

Questions? Feedback? Please comment below. Thank you!

*Also, let us know if you'd be interested in seeing some benchmarks.*
