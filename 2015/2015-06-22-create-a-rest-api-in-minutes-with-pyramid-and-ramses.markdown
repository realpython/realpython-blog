# Create a REST API in minutes with Pyramid and Ramses

*This is a guest blog post from [Chris Hart](https://twitter.com/chrstphrhrt) of [Brandicted](https://brandicted.com/) - a technologist from the great city of  Montreal.*

## Foreword

This tutorial is meant for beginners. If you get stuck along the way, try to power through and it will probably click. If there's anything you just don't get or want some help with, email [info@brandicted.com](mailto:info@brandicted.com) or leave a comment below.

## Intro

Making an API can be a lot of work. Developers need to handle details like serialization, URL mapping, validation, authentication, authorization, versioning, testing, databases, custom code for models and views, etc. Services like Firebase and Parse exist to make this way easier. Using a Backend-as-a-Service, developers can focus more on building unique user experiences.

Some drawbacks of using third party backend providers include a lack of control over the backend code, inability to self-host, no intellectual property, etc.. Having control over the code _and_ leveraging the time-saving convenience of a BaaS would be ideal, but most REST API frameworks in the wild still require a lot of boilerplate. One popular example of this would be the awesomely heavy [Django Rest Framework](http://www.django-rest-framework.org/). Another great project which requires way less boilerplate and makes building APIs super easy is [Flask-Restless](https://flask-restless.readthedocs.org/en/latest/) (highly recommended). We wanted to get rid of all boilerplate though, including the database queries that would normally need to be written for views.

Enter Ramses, a simple way to generate a powerful backend from a YAML file (actually a dialect for REST APIs called [RAML](http://raml.org/)). **In this post we'll show you how to go from zero to your own production-ready backend in a few minutes.**

> Want the code? [Ramses on Github](https://github.com/brandicted/ramses)

## Bootstrap a new product API

### Prerequisites

We assume you are working inside a fresh [virtual Python environment](https://virtualenv.pypa.io/en/latest/), and are running both [elasticsearch](https://www.elastic.co/downloads/elasticsearch) and [postgresql](http://www.postgresql.org/download/) with default configurations. We use [httpie](https://github.com/jakubroztocil/httpie) to interact with the API but you can also use curl or other http clients.

If at any time you get stuck or want to see the final working version of the code for this tutorial, [it can be found here](https://github.com/chrstphrhrt/ramses-tutorial/tree/master/pizza_factory).

### Scenario: a factory to make (hopefully) delicious pizzas

<div class="center-text">
  <img class="no-border" src="/images/blog_images/ramses/FatPizzaShopHumeHwyChullora.JPG" style="max-width: 100%;" alt="Python Pizzeria">
</div>

<br>

We want to create an API for our new pizzeria. Our backend should know about all the different toppings, cheeses, sauces, and crusts that can be used and the different combinations of them that go into making various pizza styles.

```sh
$ pip install ramses
$ pcreate -s ramses_starter pizza_factory
```

The installer will ask which database backend you want to use. Pick option "1" to use SQLAlchemy.

Change into the newly created directory and look around.

```sh
$ cd pizza_factory
```

All endpoints will be accessible at the URI /api/endpoint-name/item-id. The built-in server runs on port 6543 by default. Have a read through **local.ini** and see if it makes any sense. Then run the server to start interacting with your new backend.

```sh
$ pserve local.ini
```

Look at **api.raml** to get an idea of how endpoints are specified.

```yaml
#%RAML 0.8
---
title: pizza_factory
documentation:
    - title: pizza_factory REST API
      content: |
        Welcome to the pizza_factory API.
baseUri: http://localhost:6543/api
mediaType: application/json
protocols: [HTTP]

/items:
    displayName: Collection of items
    get:
        description: Get all item
    post:
        description: Create a new item
        body:
            application/json:
                schema: !include items.json

    /{id}:
        displayName: Collection-item
        get:
            description: Get a particular item
        delete:
            description: Delete a particular item
        patch:
            description: Update a particular item
```

As you can see, we have a resource at /api/items which is defined by the schema in **items.json**.

```sh
$ http :6543/api/items
HTTP/1.1 200 OK
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 73
Content-Type: application/json; charset=UTF-8
Date: Tue, 02 Jun 2015 16:02:09 GMT
Expires: Tue, 02 Jun 2015 16:02:09 GMT
Last-Modified: Tue, 02 Jun 2015 16:02:09 GMT
Pragma: no-cache
Server: waitress

{
    "count": 0,
    "data": [],
    "fields": "",
    "start": 0,
    "took": 1,
    "total": 0
}
```

## Data modeling

### Schemas!

Schemas describe the structure of data.

We need to create them for each of the different kinds of ingredients that we will make our pizzas with. The default schema from Ramses is a basic example in **items.json**.

Since we're going to have more than one schema in our project, let's create a new directory and move the default schema into it to keep things clean.

```sh
$ mkdir schemas
$ mv items.json schemas/
$ cd schemas/
```

**Rename items.json to pizzas.json** and open it in a text editor. Then copy its contents into new files in the same directory with the names **toppings.json**, **cheeses.json**, **sauces.json**, and **crusts.json**.

```sh
$ tree
.
├── cheeses.json
├── crusts.json
├── pizzas.json
├── sauces.json
└── toppings.json
```

In each new schema, update the value of the `"title"` field for the different kinds of things that are being described (e.g. `"title": "Pizza schema"`, `"title": "Topping schema"` etc.).

Let's edit the **pizzas.json** schema to hook up the ingredients that would go into a given style of pizza.

After the `"description"` field, add the following relations with the ingredients:

```json
...
"toppings": {
    "required": false,
    "type": "relationship",
    "args": {
        "document": "Topping",
        "ondelete": "NULLIFY",
        "backref_name": "pizza",
        "backref_ondelete": "NULLIFY"
    }
},
"cheeses": {
    "required": false,
    "type": "relationship",
    "args": {
        "document": "Cheese",
        "ondelete": "NULLIFY",
        "backref_name": "pizza",
        "backref_ondelete": "NULLIFY"
    }
},
"sauce_id": {
    "required": false,
    "type": "foreign_key",
    "args": {
        "ref_document": "Sauce",
        "ref_column": "sauce.id",
        "ref_column_type": "id_field"
    }
},
"crust_id": {
    "required": true,
    "type": "foreign_key",
    "args": {
        "ref_document": "Crust",
        "ref_column": "crust.id",
        "ref_column_type": "id_field"
    }
}
...
```

### Relations 101

We need to do the same for each of the ingredients to link them to the pizza style recipes that call for them. In **toppings.json** and **cheeses.json** we need a `"foreign_key"` field pointing to the specific pizza style that each topping would be used for (again, put this after the `"description"` field):

```json
...
"pizza_id": {
    "required": false,
    "type": "foreign_key",
    "args": {
        "ref_document": "Pizza",
        "ref_column": "pizza.id",
        "ref_column_type": "id_field"
    }
}
...
```

Then in both **sauces.json** and **crusts.json** we do the _reverse_ (by specifying `"relationship"` fields instead of `"foreign_key"` fields) because these two ingredients are being referenced by the particular instances of the pizza styles that call for them:

```json
...
"pizzas": {
    "required": false,
    "type": "relationship",
    "args": {
        "document": "Pizza",
        "ondelete": "NULLIFY",
        "backref_name": "sauce",
        "backref_ondelete": "NULLIFY"
    }
}
...
```

For **crusts.json** just make sure to set the value of `"backref_name"` to `"crust"`.

One thing to note here is that only a crust is _really_ required to make a pizza if you think long and hard about it. Maybe we'd have to call it bread at that point, but let's not get too philosophical.

Also note that _we have two different "directions"_ of pizza-to-ingredient relationships going on. Pizzas have many toppings and cheeses. These are "One (pizza) to Many (ingredients)" relationships. Pizzas only have one sauce and one crust though. Each sauce or crust may be called for by many different pizza styles. When talking about pizzas, we say there is a "Many (pizzas) to One (sauce/crust)" relationship. Whichever "direction" you want to call it by is only a matter of the entity you are talking about as a point of reference.

One-to-Many relationships have a `relationship` field on the "One" side and a `foreign_key` field on the "Many" side, e.g. pizzas (as described in **pizzas.json**) have many `"toppings"`:

```json
...
"toppings": {
    "required": false,
    "type": "relationship",
    "args": {
        "document": "Topping",
        "ondelete": "NULLIFY",
        "backref_name": "pizza",
        "backref_ondelete": "NULLIFY"
    }
...
```

...and each topping (as described in **toppings.json**) is called for by certain specific pizzas (`"pizza_id"`):

```json
...
"pizza_id": {
    "required": false,
    "type": "foreign_key",
    "args": {
        "ref_document": "Pizza",
        "ref_column": "pizza.id",
        "ref_column_type": "id_field"
    }
}
...
```

Many-to-One relationships have a `foreign_key` field on the "Many" side and a `relationship` field on the One side. That's why toppings have a `foreign_key` field pointing to specific pizzas, and pizzas have a `relationship` field pointing to all their toppings.

### Backref & ondelete arguments

**To learn about using relational database concepts in detail, refer to the [SQLAlchemy documentation](http://docs.sqlalchemy.org/en/latest/orm/basic_relationships.html).** _Very_ briefly:

A `backref` argument tells the database that when one model is referenced by another, the "referencing" model (which has a `foreign_key` field) will also provide access "backwards" to the "referenced" model.

An `ondelete` argument is telling the database that when the instance of a referenced model is deleted, to change the value of the referencing field accordingly. `NULLIFY` means that the value will be set to `null`.

## Creating endpoints

At this point, our kitchen is almost ready. In order to actually start making pizzas, we need to hook up some API endpoints to access the data models we just created.

Let's edit **api.raml** by replacing the default "items" endpoint for each of our resources like so:

```yaml
#%RAML 0.8
---
title: pizza_factory API
documentation:
    - title: pizza_factory REST API
      content: |
        Welcome to the pizza_factory API.
baseUri: http://{host}:{port}/{version}
version: v1
mediaType: application/json
protocols: [HTTP]

/toppings:
    displayName: Collection of ingredients for toppings
    get:
        description: Get all topping ingredients
    post:
        description: Create a topping ingredient
        body:
            application/json:
                schema: !include schemas/toppings.json

    /{id}:
        displayName: A particular topping ingredient
        get:
            description: Get a particular topping ingredient
        delete:
            description: Delete a particular topping ingredient
        patch:
            description: Update a particular topping ingredient

/cheeses:
    displayName: Collection of different cheeses
    get:
        description: Get all cheeses
    post:
        description: Create a new cheese
        body:
            application/json:
                schema: !include schemas/cheeses.json

    /{id}:
        displayName: A particular cheese ingredient
        get:
            description: Get a particular cheese
        delete:
            description: Delete a particular cheese
        patch:
            description: Update a particular cheese

/pizzas:
    displayName: Collection of pizza styles
    get:
        description: Get all pizza styles
    post:
        description: Create a new pizza style
        body:
            application/json:
                schema: !include schemas/pizzas.json

    /{id}:
        displayName: A particular pizza style
        get:
            description: Get a particular pizza style
        delete:
            description: Delete a particular pizza style
        patch:
            description: Update a particular pizza style

/sauces:
    displayName: Collection of different sauces
    get:
        description: Get all sauces
    post:
        description: Create a new sauce
        body:
            application/json:
                schema: !include schemas/sauces.json

    /{id}:
        displayName: A particular sauce
        get:
            description: Get a particular sauce
        delete:
            description: Delete a particular sauce
        patch:
            description: Update a particular sauce

/crusts:
    displayName: Collection of different crusts
    get:
        description: Get all crusts
    post:
        description: Create a new crust
        body:
            application/json:
                schema: !include schemas/crusts.json

    /{id}:
        displayName: A particular crust
        get:
            description: Get a particular crust
        delete:
            description: Delete a particular crust
        patch:
            description: Update a particular crust
```

**Notice the order of endpoint definitions**. `/pizzas` is placed after `/toppings` and `/cheeses` because it relates to them. `/sauces` and `/crusts` are placed after `/pizzas` because they relate to it. If you get any kind of errors about things missing or not being defined when starting the server, check the order of definition.

Now we can create our own ingredients and pizza styles!

Restart the server and get cooking.

```sh
$ pserve local.ini
```

Let's start by making a Hawaiian style pizza:

```sh
$ http POST :6543/api/toppings name=ham
HTTP/1.1 201 Created...
```

```sh
$ http POST :6543/api/toppings name=pineapple
HTTP/1.1 201 Created...
```

```sh
$ http POST :6543/api/cheeses name=mozzarella
HTTP/1.1 201 Created...
```

```sh
$ http POST :6543/api/sauces name=tomato
HTTP/1.1 201 Created...
```

```sh
$ http POST :6543/api/crusts name=plain
HTTP/1.1 201 Created...
```

```sh
$ http POST :6543/api/pizzas name=hawaiian toppings:=[1,2] cheeses:=[1] sauce=1 crust=1
```

### Voila!

<div class="center-text">
  <img class="no-border" src="/images/blog_images/ramses/Hawaiian_pizza.jpg" style="max-width: 100%;" alt="Hawaiian Pizza">
</div>

<br>

Here it is in all its greasy glory:

```sh
HTTP/1.1 201 Created
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 373
Content-Type: application/json; charset=UTF-8
Date: Fri, 05 Jun 2015 18:47:53 GMT
Expires: Fri, 05 Jun 2015 18:47:53 GMT
Last-Modified: Fri, 05 Jun 2015 18:47:53 GMT
Location: http://localhost:6543/api/pizzas/1
Pragma: no-cache
Server: waitress

{
    "data": {
        "_type": "Pizza",
        "_version": 0,
        "cheeses": [
            1
        ],
        "crust": 1,
        "crust_id": 1,
        "description": null,
        "id": 1,
        "name": "hawaiian",
        "sauce": 1,
        "sauce_id": 1,
        "self": "http://localhost:6543/api/pizzas/1",
        "toppings": [
            1,
            2
        ],
        "updated_at": null
    },
    "explanation": "",
    "id": "1",
    "message": null,
    "status_code": 201,
    "timestamp": "2015-06-05T18:47:53Z",
    "title": "Created"
}
```

## Seed data

The last step for bonus points is to import a bunch of existing ingredient records to make things more fun.

First create a `seeds/` directory inside the pizza_factory project and download the seed data:

```sh
$ mkdir seeds
$ cd seeds/
$ http -d https://raw.githubusercontent.com/chrstphrhrt/ramses-tutorial/master/pizza_factory/seeds/crusts.json
$ http -d https://raw.githubusercontent.com/chrstphrhrt/ramses-tutorial/master/pizza_factory/seeds/sauces.json
$ http -d https://raw.githubusercontent.com/chrstphrhrt/ramses-tutorial/master/pizza_factory/seeds/cheeses.json
$ http -d https://raw.githubusercontent.com/chrstphrhrt/ramses-tutorial/master/pizza_factory/seeds/toppings.json
```

Now, use the built-in post2api script to load all the ingredients into your API.

```sh
$ nefertari.post2api -f crusts.json -u http://localhost:6543/api/crusts
$ nefertari.post2api -f sauces.json -u http://localhost:6543/api/sauces
$ nefertari.post2api -f cheeses.json -u http://localhost:6543/api/cheeses
$ nefertari.post2api -f toppings.json -u http://localhost:6543/api/toppings
```

You can now list the different ingredients easily.

```sh
$ http :6543/api/toppings
```

Or search for the ingredients by name.

```sh
$ http :6543/api/toppings?name=chicken

HTTP/1.1 200 OK
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 934
Content-Type: application/json; charset=UTF-8
Date: Fri, 05 Jun 2015 19:58:48 GMT
Etag: "fd29d8eda6441cebdd632960a21c8136"
Expires: Fri, 05 Jun 2015 19:58:48 GMT
Last-Modified: Fri, 05 Jun 2015 19:58:48 GMT
Pragma: no-cache
Server: waitress

{
    "count": 4,
    "data": [
        {
            "_score": 2.3578677,
            "_type": "Topping",
            "_version": 0,
            "description": null,
            "id": 28,
            "name": "Chicken Tikka",
            "pizza": null,
            "pizza_id": null,
            "self": "http://localhost:6543/api/toppings/28",
            "updated_at": null
        },
        {
            "_score": 2.3578677,
            "_type": "Topping",
            "_version": 0,
            "description": null,
            "id": 27,
            "name": "Chicken Masala",
            "pizza": null,
            "pizza_id": null,
            "self": "http://localhost:6543/api/toppings/27",
            "updated_at": null
        },
        {
            "_score": 2.0254436,
            "_type": "Topping",
            "_version": 0,
            "description": null,
            "id": 14,
            "name": "BBQ Chicken",
            "pizza": null,
            "pizza_id": null,
            "self": "http://localhost:6543/api/toppings/14",
            "updated_at": null
        },
        {
            "_score": 2.0254436,
            "_type": "Topping",
            "_version": 0,
            "description": null,
            "id": 19,
            "name": "Cajun Chicken",
            "pizza": null,
            "pizza_id": null,
            "self": "http://localhost:6543/api/toppings/19",
            "updated_at": null
        }
    ],
    "fields": "",
    "start": 0,
    "took": 3,
    "total": 4
}
```

So, let's make one last pizza by finding the ingredients. How about a vegetarian one this time?

Maybe a bit of spinach, ricotta, sun-dried tomato sauce, and a whole wheat crust. First we find our IDs (yours may be different)..

```sh
$ http :6543/api/toppings?name=spinach
...
"id": 88,
"name": "Spinach",
...
$ http :6543/api/cheeses?name=ricotta
...
"id": 18,
"name": "Ricotta",
...
$ http :6543/api/sauces?name=sun
...
"id": 18,
"name": "Sun Dried Tomato",
...
$ http :6543/api/crusts?name=whole
...
"id": 13,
"name": "Whole Wheat",
...
```

Bake for 0 seconds, and..

```sh
$ http POST :6543/api/pizzas name="Veggie Delight" toppings:=[88] cheeses:=[18] sauce=18 crust=13

HTTP/1.1 201 Created
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 382
Content-Type: application/json; charset=UTF-8
Date: Fri, 05 Jun 2015 20:17:26 GMT
Expires: Fri, 05 Jun 2015 20:17:26 GMT
Last-Modified: Fri, 05 Jun 2015 20:17:26 GMT
Location: http://localhost:6543/api/pizzas/2
Pragma: no-cache
Server: waitress

{
    "data": {
        "_type": "Pizza",
        "_version": 0,
        "cheeses": [
            18
        ],
        "crust": 13,
        "crust_id": 13,
        "description": null,
        "id": 2,
        "name": "Veggie Delight",
        "sauce": 18,
        "sauce_id": 18,
        "self": "http://localhost:6543/api/pizzas/2",
        "toppings": [
            88
        ],
        "updated_at": null
    },
    "explanation": "",
    "id": "2",
    "message": null,
    "status_code": 201,
    "timestamp": "2015-06-05T20:17:26Z",
    "title": "Created"
}
```

Bon appétit!

Check out the full documentation for [Ramses on Readthedocs](https://ramses.readthedocs.org/en/latest/), and the somewhat more advanced [example project on Github](https://github.com/brandicted/ramses-example).
