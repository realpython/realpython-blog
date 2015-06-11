---
layout: post
title: "Django Rest Framework - class based views"
date: 2014-10-13 06:26:43 -0700
toc: true
comments: true
category_side_bar: true
categories: [python, django, api]

keywords: "python, REST, Django, django rest framework, django rest, API, REST API, cbv, class based views"
description: "In this part we'll refactor our Django Rest Framework views to use class based views."
---

![drf-cbv-image](https://raw.githubusercontent.com/realpython/django-form-fun/master/images/drf-cbv.png)

<br>

In this post let's continue our development of the Django Talk project along with [Django Rest Framework](http://www.django-rest-framework.org/) as we implement [class based views](http://www.django-rest-framework.org/api-guide/views#class-based-views), along with some simple refactoring to keep our code Dry. **Essentially, we're migrating an existing RESTful API based on function based views to class based views.**

> This is part four of this tutorial series.

> **Need to catch up?** Check out parts [one](https://realpython.com/blog/python/django-and-ajax-form-submissions/) and [two](https://realpython.com/blog/python/django-and-ajax-form-submissions-more-practice/) where we tackle Django and AJAX as well as part [three](https://realpython.com/blog/python/django-rest-framework-quick-start/) where we introduce the Django Rest Framework.

> **Need the code?** Download it from the [repo](https://github.com/realpython/django-form-fun).

**For a more in-depth tutorial on Django Rest Framework, be sure to check out the third [Real Python](https://realpython.com) course.**

## Refactor

Before jumping into class based views, let's do a quick refactor of our current code. Within the `def post_element()` view, make the following updates:

```python
post = get_object_or_404(Post, id=pk)

# try:
#     post = Post.objects.get(pk=pk)
# except Post.DoesNotExist:
#     return HttpResponse(status=404)
```

So here we are using the [get_object_or_404](https://docs.djangoproject.com/en/1.6/topics/http/shortcuts/#get-object-or-404) shortcut to raise a 404 error instead of an exception.

Make sure to also update the imports:

```python
from django.shortcuts import render, get_object_or_404
```

Test it out. Try viewing an element that does not exist - i.e., [http://localhost:8000/api/v1/posts/202?format=json](http://localhost:8000/api/v1/posts/202?format=json). You should see the following response in your browser:

```javascript
{
    detail: "Not found"
}
```

If you open the *Network* tab within *Chrome Developer Tools*, you should see a 404 status code. Pretty cool, right? Unfortunately, we won't be using it much longer, as it's time to say goodbye to our current function based views and add in class based views.

## Class Based Views

While functions are easy to work with, it's often beneficial to use class based views to reuse functionality, especially for large APIs with a number of endpoints.

Comment out the code for the function based views.

### Collection

Add in the following code for the class based views for the collection:

```python
class PostCollection(mixins.ListModelMixin,
                     mixins.CreateModelMixin,
                     generics.GenericAPIView):

    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

Welcome to the power of [mixins](https://docs.djangoproject.com/en/1.6/ref/class-based-views/mixins/)!

1. The `ListModelMixin` provides the `list()` function for serializing a collection to JSON and then returning it as a response to a GET request.
1. The `CreateModelMixin` meanwhile provides the `create()` function for creating a new object in response to a POST request.
1. Finally, the `GenericAPIView` mixin provides the "core" functionality needed for a RESTful API.

Please consult the official DRF [documentation](http://www.django-rest-framework.org/api-guide/generic-views#mixins) for more info on these mixins.

### Member

Now add the code for the member:

```python
class PostMember(mixins.RetrieveModelMixin,
                   mixins.DestroyModelMixin,
                   generics.GenericAPIView):

    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

Here, we simply use the `GenericAPIView` for the "core functionality" and the remaining mixins to provide the needed functionality to handle GET and DELETE requests.

### URLs

Finally, let's update *urls.py* to account for the class based views:

```python
from django.conf.urls import patterns, url
from talk import views


urlpatterns = patterns(
    'talk.views',
    url(r'^$', 'home'),

    # api
    url(r'^api/v1/posts/$', views.PostCollection.as_view()),
    url(r'^api/v1/posts/(?P<pk>[0-9]+)/$', views.PostMember.as_view())
)
```

Take note of the `as_view()` method, which provides a bit of magic to treat the class as a view function.

Test this out. Launch the development server, then:

1. Make sure all posts load
1. Add a new post
1. Remove a post

What happened? You should have seen the following error when trying to add a new post:

```javascript
400: {
    "text": ["This field is required."],
    "author": ["This field is required."]
}
```

Fortunately, this is an easy fix.

### Refactor the AJAX

We need to change the way we're sending the data with the POST request. First, update `the_post` key to `text`:

```javascript
data : { text : $('#post-text').val() }, // data sent with the post request
```

If you test it out now, the error should just indicate that we're missing the `author` field. We can grab the logged in user a number of ways, but the easiest is directly from the DOM.

> It's worth noting that we can override the default functionality in the views to grab the username from the request object. However, it's best to use DRF class based views as intended: Passing all the appropriate parameters - e.g., `text` and `author` - in the JSON request and then using the serializer to save them.

Open *index.html* within the "templates/talk" directory. At the top of the file, you'll see that we're accessing the username directly from the `request` object:

{% raw %}
```html
<h2>Hi, {{request.user.username}}</h2>
```

Let's isolate the actual username to make it easier to grab with jQuery:

```html
<h2>Hi, <span id="user">{{request.user.username}}</span></h2>
```
{% endraw %}

Now update `data` again:

```javascript
data : { text : $('#post-text').val(), author: $('#user').text()}
```

Test it out; all should be well.

## Generic Based Views

Want even more with less? Check this out. Comment out the code we just added, and then update the views like so:

```python
from talk.models import Post
from talk.forms import PostForm
from talk.serializers import PostSerializer
from rest_framework import generics
from django.shortcuts import render


def home(request):
    tmpl_vars = {'form': PostForm()}
    return render(request, 'talk/index.html', tmpl_vars)


#########################
### class based views ###
#########################

class PostCollection(generics.ListCreateAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

class PostMember(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

So, not only can we still handle all the same requests as before, but we can now handle PUT requests to update each member. More. For **a lot** less.

*Before moving on jump back to the function based views and compare the code with the class based views. Which is easier to read? Actually, which is easier to understand? Add comments if necessary to help you better understanding what's happening under the hood with the class based views. Not only do you sacrifice readability with class based views, but testing is slightly more difficult as well. However, we're now taking advantage of inheritance, and for larger projects that have a number of similar views, class based views are perfect since you don't have to write the same code over and over again. **Make sure to weigh the pros and cons before jumping to class based views.***

Be sure to test the endpoints before moving on.

1. Make sure all posts load
1. Add a new post
1. Remove a post

Curious about the PUT request? Test it out from the Browsable API - i.e., [http://localhost:8000/api/v1/posts/1/](http://localhost:8000/api/v1/posts/1/) - via the HTML form. Want to remove the PUT method handler altogether? Update the code like so:

```python
class PostMember(generics.RetrieveDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

Test it out now. Questions? Check out the [documentation](http://www.django-rest-framework.org/api-guide/generic-views#retrievedestroyapiview).

## Conclusion

This is all for now. We may jump back to the Django Rest Framework in a future post to look at pagination, permissions, and basic validation, but before that we'll add on AngularJS to the front-end to consume the data.

Cheers!
