---
layout: post
title: "Django Rest Framework - an introduction"
date: 2014-09-22 08:02:45 -0500
toc: true
comments: true
category_side_bar: true
categories: [python, django, api]

keywords: "python, REST, Django, django rest framework, django rest, API, REST API"
description: "In this tutorial, we'll develop a RESTful API with the Django Rest Framework."
---

![drf-image](https://raw.githubusercontent.com/realpython/django-form-fun/master/images/drf.png)

Let's look at how to create a RESTFul API for our Django Talk Project using [Django Rest Framework](http://www.django-rest-framework.org/) (DRF), which is an application used for rapidly building RESTful APIs based on Django models. **Put another way, we'll be converting a non-RESTful application into a RESTful one with DRF**. *We'll be using DRF version 2.4.2 for this application.*

**This tutorial covers these topics**:

1. DRF Setup
1. RESTful Structure
1. Model Serializer
1. DRF Web Browseable API

> If you missed parts [one](https://realpython.com/blog/python/django-and-ajax-form-submissions/) and [two](https://realpython.com/blog/python/django-and-ajax-form-submissions-more-practice/) be sure to check them out. Need the code? Download it from the [repo](https://github.com/realpython/django-form-fun).

**For a more in-depth tutorial on Django Rest Framework, check out the third [Real Python](https://realpython.com) course.**

## DRF setup

Install:

```sh
$ pip install djangorestframework
$ pip freeze > requirements.txt
```

Update *settings.py*:

```python
INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'talk',
    'rest_framework'
)
```

Boom!

## RESTful Structure

In a RESTful API, endpoints (URLs) define the structure of the API and how end users access data from our application using the HTTP methods: GET, POST, PUT, DELETE. Endpoints should be logically organized around *collections* and *elements*, both of which are resources. In our case, we have one single resource, `posts`, so we will use the following URLS - `/posts/` and `/posts/<id>` for collections and elements, respectively.

<table class="table table-bordered">
  <thead>
    <tr>
      <th></th>
      <th>GET</th>
      <th>POST</th>
      <th>PUT</th>
      <th>DELETE</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>/posts/</code></td>
      <td>Show all posts</td>
      <td>Add new post</td>
      <td>Update all posts</td>
      <td>Delete all posts</td>
    </tr>
    <tr>
      <td><code>/posts/&lt;id&gt;</code></td>
      <td>Show &lt;id&gt;</td>
      <td>N/A</td>
      <td>Update &lt;id&gt;</td>
      <td>Delete &lt;id&gt;</td>
    </tr>
  </tbody>
</table>

## DRF Quick Start

Let's get our new API up and running!

### Model Serializer

DRF's Serializers convert model instances to Python dictionaires, which can then be rendered in various API appropriate formats - like JSON or XML. Similar to the Django `ModelForm` class, DRF comes with a concise format for its Serializers, the `ModelSerializer` class. It's simple to use: Just tell it which fields you want to use from the model:

```python
from rest_framework import serializers
from talk.models import Post


class PostSerializer(serializers.ModelSerializer):

    class Meta:
        model = Post
        fields = ('id', 'author', 'text', 'created', 'updated')
```

Save this as *serializers.py* within the "talk" directory.

### Update Views

We need to refactor our current views to fit the RESTful paradigm. Comment out the current views and add in:

```python
from django.shortcuts import render
from django.http import HttpResponse
from rest_framework.decorators import api_view
from rest_framework.response import Response
from talk.models import Post
from talk.serializers import PostSerializer
from talk.forms import PostForm


def home(request):
    tmpl_vars = {'form': PostForm()}
    return render(request, 'talk/index.html', tmpl_vars)


@api_view(['GET'])
def post_collection(request):
    if request.method == 'GET':
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)


@api_view(['GET'])
def post_element(request, pk):
    try:
        post = Post.objects.get(pk=pk)
    except Post.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = PostSerializer(post)
        return Response(serializer.data)
```

**What's happening here**:

1. First, the `@api_view` decorator checks that the appropriate HTTP request is passed into the view function. Right now, we're only supporting GET requests.
1. Then, the view either grabs all the data, if it's for the collection, or just a single post, if it's for an element.
1. Finally, the data is serialized to JSON and returned.

> Be sure to read more about the `@api_view` from the official [documentation](http://www.django-rest-framework.org/api-guide/views#@api_view).

### Update URLs

Let's wire up some new URLs:

```python
# Talk urls
from django.conf.urls import patterns, url


urlpatterns = patterns(
    'talk.views',
    url(r'^$', 'home'),

    # api
    url(r'^api/v1/posts/$', 'post_collection'),
    url(r'^api/v1/posts/(?P<pk>[0-9]+)$', 'post_element')
)
```

### Test

We're now ready for our first test!

1. Fire up the server, then navigate to: [http://127.0.0.1:8000/api/v1/posts/?format=json](http://127.0.0.1:8000/api/v1/posts/?format=json).
1. Now let's check out the [Browsable API](http://www.django-rest-framework.org/topics/browsable-api). Navigate to [http://127.0.0.1:8000/api/v1/posts/](http://127.0.0.1:8000/api/v1/posts/)

    So, With no extra work on our end we automatically get this nice, human-readable output of our API. Nice! This is a *huge* win for DRF.

1. How about an element? Try: [http://127.0.0.1:8000/api/v1/posts/1](http://127.0.0.1:8000/api/v1/posts/1)

Before moving on you may have noticed that the author field is an `id` rather than the actual `username`. We'll address this shortly. For now, let's wire up our new API so that it works with our current application's Templates.

## Refactor for REST

### GET

On the initial page load, we want to display all posts. To do that, add the following AJAX request:

```javascript
load_posts()

// Load all posts on page load
function load_posts() {
    $.ajax({
        url : "api/v1/posts/", // the endpoint
        type : "GET", // http method
        // handle a successful response
        success : function(json) {
            for (var i = 0; i < json.length; i++) {
                console.log(json[i])
                $("#talk").prepend("<li id='post-"+json[i].id+"'><strong>"+json[i].text+"</strong> - <em> "+json[i].author+"</em> - <span> "+json[i].created+
                "</span> - <a id='delete-post-"+json[i].id+"'>delete me</a></li>");
            }
        },
        // handle a non-successful response
        error : function(xhr,errmsg,err) {
            $('#results').html("<div class='alert-box alert radius' data-alert>Oops! We have encountered an error: "+errmsg+
                " <a href='#' class='close'>&times;</a></div>"); // add the error to the dom
            console.log(xhr.status + ": " + xhr.responseText); // provide a bit more info about the error to the console
        }
    });
};
```

You've seen all this before. Notice how we're handling a success: Since the API sends back a number of objects, we need to iterate through them, appending each to the DOM. We also changed `json[i].postpk` to `json[i].id` as we are serializing the post `id`.

Test this out. Fire up the server, log in, then check out the posts.

Besides the `author` being displayed as an `id`, take note of the datetime format. This is not what we want, right? We want a *readable* datetime format. Let's update that...

### Datetime Format

We can use an awesome JavaScript library called [MomentJS](http://momentjs.com/) to easily format the date anyway we want.

First, we need to import the library to our *index.html* file:

```html
<!-- scripts -->
<script src="http://cdnjs.cloudflare.com/ajax/libs/moment.js/2.8.2/moment.min.js"></script>
<script src="static/scripts/main.js"></script>
```

Then update the for loop in *main.js*:

```javascript
for (var i = 0; i < json.length; i++) {
    dateString = convert_to_readable_date(json[i].created)
    $("#talk").prepend("<li id='post-"+json[i].id+"'><strong>"+json[i].text+
        "</strong> - <em> "+json[i].author+"</em> - <span> "+dateString+
        "</span> - <a id='delete-post-"+json[i].id+"'>delete me</a></li>");
}
```

Here we pass the date string to a new function called `convert_to_readable_date()`, which needs to be added:

```javascript
// convert ugly date to human readable date
function convert_to_readable_date(date_time_string) {
    var newDate = moment(date_time_string).format('MM/DD/YYYY, h:mm:ss a')
    return newDate
}
```

That's it. Refresh the browser. The datetime format should now look something like this - ` 08/22/2014, 6:48:29 pm`. Be sure to check out the [MomentJS](http://momentjs.com/) documentation to view more information on parsing and formatting a datetime string in JavaScript.

### POST

POST requests are handled in similar fashion. Before messing with the serializer, let's test it first by just updating the views. Maybe we'll get lucky and it will just work.

Update the `post_collection()` function in *views.py*:

```python
@api_view(['GET', 'POST'])
def post_collection(request):
    if request.method == 'GET':
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
    elif request.method == 'POST':
        data = {'text': request.DATA.get('the_post'), 'author': request.user.pk}
        serializer = PostSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

Also add the following import:

```python
from rest_framework import status
```

**What's happening here**:

1. `request.DATA` extends Django's HTTPRequest, returning the content from the request body. Read more about it [here](http://www.django-rest-framework.org/api-guide/requests).
1. If the deserialization process works, we return a response with a code of 201 (created).
1. On the other hand, if the deserialization process fails, we return a 400 response.

Update the endpoint in the `create_post()` function

From:

```javascript
url : "create_post/", // the endpoint
```

To:

```javascript
url : "api/v1/posts/", // the endpoint
```

Test it out in the browser. It should work. Don't forget to update the handling of the dates correctly as well as changing `json.postpk` to `json.id`:

```javascript
success : function(json) {
    $('#post-text').val(''); // remove the value from the input
    console.log(json); // log the returned json to the console
    dateString = convert_to_readable_date(json.created)
    $("#talk").prepend("<li id='post-"+json.id+"'><strong>"+json.text+"</strong> - <em> "+
        json.author+"</em> - <span> "+dateString+
        "</span> - <a id='delete-post-"+json.id+"'>delete me</a></li>");
    console.log("success"); // another sanity check
},
```

### Author Format

Now's a good time to pause and address the author `id` vs. `username` issue. We have a few options:

1. Be really RESTFUL and make another call to get the user info, which is not good for performance.
1. Utilize the [SlugRelatedField](http://www.django-rest-framework.org/api-guide/relations#slugrelatedfield) relation.

Let's go with the latter option. Update the serializer:

```python
from django.contrib.auth.models import User
from rest_framework import serializers
from talk.models import Post


class PostSerializer(serializers.ModelSerializer):
    author = serializers.SlugRelatedField(
        queryset=User.objects.all(), slug_field='username'
    )

    class Meta:
        model = Post
        fields = ('id', 'author', 'text', 'created', 'updated')
```

**What's happening here?**

1. The `SlugRelatedField` allows us to change the target of the `author` field from `id` to `username`.
1. Also, by default the target field - `username` - is both readable and writeable so out-of-the-box this relation will work for both GET and POST requests.

Update the `data` variable in the views as well:

```python
data = {'text': request.DATA.get('the_post'), 'author': request.user}
```

Test again. You should now see the author's `username`. Make sure both GET and POST requests are working correctly.

### Delete

Before changing or adding anything, test it out. Try the delete link. What happens? You should get a 404 error. Any idea why that would be? Or where to go to find out what the issue is? How about the `delete_post` function in our JavaScript file:

```javascript
url : "delete_post/", // the endpoint
```

That URL does not exist. Before we update it, ask yourself - "Should we target the collection or an individual element?". If you're unsure, scroll back up and look at the *RESTful Structure* table. Unless we want to delete *all* posts, then we need to hit the element endpoint:

```javascript
url : "api/v1/posts/"+post_primary_key, // the endpoint
```

Test again. Now what happens? You should see a 405 error - `405: {"detail": "Method 'DELETE' not allowed."}` - since the view is not setup to handle a DELETE request.

```python
@api_view(['GET', 'DELETE'])
def post_element(request, pk):
    try:
        post = Post.objects.get(pk=pk)
    except Post.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = PostSerializer(post)
        return Response(serializer.data)

    elif request.method == 'DELETE':
        post.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

With the DELETE HTTP verb added, we can handle the request by removing the post with the `delete()` method and returning a 204 response. Does it work? Only one way to find out. This time when you test make sure that (a) the post is actually deleted and removed from the DOM and (b) that a 204 status code is returned (you can confirm this in the *Network* tab within *Chrome Developer Tools*).

## Conclusion and Next Steps

This is all for now. Need an extra challenge? Add the ability to update posts with the PUT request.

The actual REST part is simple: You just need to update the `post_element()` function to handle PUT requests.

The client side is a bit more difficult as you need to update the actual HTML to display an input box for the user to enter the new value in, which you'll need to grab in the JavaScript file so you can send it with the PUT request.

Are you going to allow any user to update any post, regardless of whether s/he originally posted it?

If so, are you going to update the author name? Maybe add an `edited_by` field to the database? Then display an edited by note on the DOM as well. If users can only update their own posts, you need to make sure you are handling this correctly in the views and then displaying an error message if the user is trying to edit a post that s/he did not originally post. Or perhaps you could just remove the edit link (and perhaps the delete link as well) for posts that a certain user can't edit. You could turn it into a permissions issue and only let *certain* users, like moderators or admin, edit all posts while the remaining users can only update their own posts.

So many questions.

If you do decide to try this, go with the lowest hanging fruit - *simply allow any user to update any post and only update the `text` in the database.* Then test. Then add another iteration. Then test. Etc. Take notes and email us at **info@realpython.com** so we can add a supplementary blog post!

<hr>

Anyhow, next time you'll get to see us tear apart the current JavaScript code as we add in **Angular**! See you then.
