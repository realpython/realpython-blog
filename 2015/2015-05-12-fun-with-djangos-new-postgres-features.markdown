---
layout: post
title: "Fun with Django's New Postgres Features"
date: 2015-05-12 07:39:28 -0600
toc: true
comments: true
category_side_bar: true
categories: [python, django]

keywords: "python, django, django 1.8, postgres, postgresql"
description: "Let's look at the new Postgres features introduced in Django 1.8."
---

**This blog post covers how to use the new [PostgreSQL-specific ModelFields](https://docs.djangoproject.com/en/1.8/ref/contrib/postgres/) introduced in Django 1.8 - the ArrayField, HStoreField, and Range Fields.**

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-new-postgres-features/django-and-postgres.png" style="max-width: 100%;" alt="postgres and django">
</div>

<br>

*This post is dedicated to the awesome backers of this [Kickstarter](https://www.kickstarter.com/projects/mjtamlyn/improved-postgresql-support-in-django/posts/803919) campaign put together by [Marc Tamlyn](https://twitter.com/mjtamlyn), the true playa that made it happen.*

## Playaz Club?

Since I'm a huge geek and have no chance of ever getting into a real Playaz Club (and because back in the day [4 Tay](https://www.youtube.com/watch?v=2daXghqHgjQ) was the bomb), I decided to build my own virtual online Playaz Club. What is that exactly?  A private, invite-only social network targeted at a small group of like-minded individuals.

For this post, we are going to focus on the user model and explore how Django's new [PostgreSQL features](https://docs.djangoproject.com/en/1.8/ref/contrib/postgres/fields/) support the modeling. The new features we are referring to are PostgreSQL-only, so don't bother trying this unless you have your database `ENGINE` equal to `django.db.backends.postgresql_psycopg2`. You will need version >= 2.5 of `psycopg2`. Aight playa, let's do this.

Holla if you with me! :)

## Modeling a Playa's Rep

Every playa got a rep, and they want the whole world to know about their rep. So let's create a user profile (aka a "rep") that allows for each of our playaz to express their individuality.

Here's the basic model for a playaz rep:

```python
from django.db import models
from django.contrib.auth.models import User

class Rep(models.Model):
    playa = models.OneToOneField(User)
    hood = models.CharField(max_length=100)
    area_code = models.IntegerField()
```

Nothing specific to 1.8 above. Just a standard model to extend the base Django User, cause a playa still needs a username and email address, right?  Plus we added two new fields to store the playaz hood and area code.

## Bankroll and the RangeField

For a playa, repping your hood ain't always enough. Playaz often like to flaunt their bankroll, but at the same time don't want to be letting people know exactly how large that bankroll is. We can model that with one of the new [Postgres Range Fields](https://docs.djangoproject.com/en/1.8/ref/contrib/postgres/fields/#range-fields). Of course we will use the `BigIntegerRangeField` to better model massive digits, amiright?

```python
bankroll = pgfields.BigIntegerRangeField(default=(10, 100))
```

Range fields are based on the [psycopg2 Range objects](http://initd.org/psycopg/docs/extras.html#adapt-range) and can be used for Numeric and DateRanges. With the bankroll field migrated to the database, we can interact with our range fields by passing it a range object, so creating our first playa would look something like this:

```sh
>>> from playa.models import Rep
>>> from django.contrib.auth.models import User
>>> calvin = User.objects.create_user(username="snoop", password="dogg")
>>> calvins_rep = Rep(hood="Long Beach", area_code=213)
>>> calvins_rep.bankroll = (100000000, 150000000)
>>> calvins_rep.playa = calvin
>>> calvins_rep.save()
```

Notice this line: `calvins_rep.bankroll = (100000000, 150000000)`. Here we are setting a range field by using a simple tuple. It is also possible to set the value using a `NumericRange` object like so:

```sh
from psycopg2.extras import NumericRange
br = NumericRange(lower=100000000, upper=150000000)
calvin.rep.bankroll = br
calvin.rep.save()
```

This is essentially the same as using the tuple. However, it's important to know about the `NumericRange` object as that is used to filter the model. For example, if we wanted to find all playas whose bankroll was greater than 50 million (meaning the entire bankroll range is greater than 50 million):

```python
Rep.objects.filter(bankroll__fully_gt=NumericRange(50000000, 50000000))
```

And that will return the list of those playas. Alternatively, if we wanted to find all playas whose bankroll is "somewhere around the 10 to 15 million range", we could use:

```python
Rep.objects.filter(bankroll__overlap=NumericRange(10000000, 15000000))
```

This would return all playas that have a bankroll range that includes at least some part of the 10 to 15 million range. A more absolute query would be all playas that have a bankroll fully in between a range, i.e. everybody that's making at least 10 million but not more than 15 million. That query would look like:

```python
Rep.objects.filter(bankroll__contained_by=NumericRange(10000000, 15000000))
```

More information about Range-based queries can be found [here](https://docs.djangoproject.com/en/1.8/ref/contrib/postgres/fields/#querying-range-fields).

## Skillz as ArrayField

It ain't all about the bankroll, playaz got skillz, all kinds of skillz. Let's model those with an [ArrayField](https://docs.djangoproject.com/en/1.8/ref/contrib/postgres/fields/#arrayfield).

```python
skillz = pgfields.ArrayField(
    models.CharField(max_length=100, blank=True),
    blank = True,
    null = True,
)
```

To declare the `ArrayField` we have to give it a first argument, which is the basefield. Unlike Python lists, ArrayFields must declare each element of the list as the same type. Basefield declares which type this is, and it can be any of the standard model field types. In the case above, we have just used a `CharField` as our basetype, which means `skillz` will be an array of strings.

Storing values to the `ArrayField` is exactly as you expect:

```sh
>>> from django.contrib.auth.models import User
>>> calvin = User.objects.get(username='snoop')
>>> calvin.rep.skillz = ['ballin', 'rappin', 'talk show host', 'merchandizn']
>>> calvin.rep.save()
```

### Finding playas by skillz

If we need a particular playa with a particular skill, how we gonna find them?  Use the `__contains` filter:

```python
Rep.objects.filter(skillz__contains=['rappin'])
```

For playas who have any of the skills ['rappin', 'djing', 'producing'] but no other skills, you could do a query like so:

```python
Rep.objects.filter(skillz__contained_by=['rappin', 'djing', 'producing'])
```

Or if you want to find anyone that has any of a certain list of skills:

```python
Rep.objects.filter(skillz__overlap=['rappin', 'djing', 'producing'])
```

You could even find only those people who listed a skill as their first skill (because everybody lists their best skill first):

```python
Rep.objects.filter(skillz__0='ballin')
```

## Game as HStore

Game can be thought of as a list of miscellaneous, random skills a playa might have. Since Game spans all kinds of things, let's model it as an [HStore field](https://docs.djangoproject.com/en/1.8/ref/contrib/postgres/fields/#hstorefield), which basically means we can stick any old Python dictionary in there:

```python
game = pgfields.HStoreField()
```

> Take a second to think about what we just did here. HStore is pretty huge. It basically allows for "NoSQL"-type data storage, right inside of postgreSQL. Plus, since it's inside PostgreSQL we can link (through foreign keys), tables that contain NoSQL data with tables that store regular SQL-type data. You can even store both in the same table on different columns as we are doing here. Maybe playas don't need to use that jankie, all-talk MongoDB anymore...

Coming back to implementation details, if you try to migrate the new HStore field into the database and end up with this error-

```python
django.db.utils.ProgrammingError: type "hstore" does not exist
```

-then your PostgreSQL database is pre 8.1 (time to upgrade, playa) or doesn't have the HStore extension installed. Keep in mind that in PostgreSQL the HStore extension is installed per database and not system-wide. To install it from your psql prompt, run the following SQL:

```python
CREATE EXTENSION hstore
```

Or if you want, you could do it through a SQL Migration with the following migration file (assuming you were connected to the database as the superuser):

```python
from django.db import models, migrations

class Migration(migrations.Migration):

    dependencies = []

    operations = [
        migrations.RunSQL("CREATE EXTENSION IF NOT EXISTS hstore")
    ]
```

Finally, you will also need to make sure that you have added `'django.contrib.postgres'` to `'settings.INSTALLED_APPS'` to make use of HStore fields.

With that setup we can add data to our `HStoreField` `game` by throwing a dictionary at it like so:

```sh
>>> calvin = User.objects.get(username="snoop")
>>> calvin.rep.game = {'best_album': 'Doggy Style', 'youtube-channel': \
    'https://www.youtube.com/user/westfesttv', 'twitter_follows' : '11000000'}
>>> calvin.rep.save()
```

Do keep in mind that the dict must only use strings for all keys and values.

And now for some more interesting examples...

## Propz

Let's write a "show game" function to search the playaz game and return a list of playaz that match. In geeky terms, we're searching through the HStore field for any keys passed into the function. It looks something like this:

```python
def show_game(key):
    return Rep.Objects.filter(game__has_key=key).values('game','playa__username')
```

Above we have used the `has_key` filter for the HStore field to return a queryset, then further filtered it with the values function (mainly to show that you can chain `django.contrib.postgres` stuff with regular query set stuff).

The return value would be a list of dictionaries:

```python
[
  {'playa__username': 'snoop',
  'game': {'twitter_follows': '11000000',
           'youtube-channel': 'https://www.youtube.com/user/westfesttv',
           'best_album': 'Doggy Style'
        }
  }
]
```

As they say, [Game recognizes Game](http://www.urbandictionary.com/define.php?term=Game+recognizes+game), and now we can search for game, too.

## High Rollers

If we believe what the playaz are telling us about their bankrolls, then we can use that to rank them in categories (since it's a range). Let's add a Playa ranking based upon the bankroll with the following levels:

* young buck -- bankroll less than one hundred grand

* balla -- bankroll between 100,000 and 500,000 with the skill 'ballin'

* playa -- bankroll between 500,000 and 1,000,000 with two skillz and some game

* high roller -- bankroll greater than 1,000,000

* O.G. -- has the skill 'gangsta' and game key "old-school"

The query for balla is below. This would be the strict interpretation, which would return only those whose entire bankroll range is within the specified limits:

```python
Rep.objects.filter(bankroll__contained_by=[100000, 500000], skillz__contains=['ballin'])
```

<br><hr>

Try out the [rest](https://docs.djangoproject.com/en/1.8/ref/contrib/postgres/) yourself for some practice. If you need some help, [Read the Docs](https://docs.djangoproject.com/en/1.8/ref/contrib/postgres/fields/#postgresql-specific-model-fields).
