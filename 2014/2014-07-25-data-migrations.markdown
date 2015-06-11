# Data Migrations

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-migrations.png" style="max-width: 100%;">
</div>

**Updated Feb 12, 2015**: Changed the data migration to lookup a model from the app registry.

<br>

This is the final article in our Django migrations series:

- Part 1: [Django Migrations - A Primer](https://realpython.com/blog/python/django-migrations-a-primer/)
- Part 2: [Digging Deeper into Migrations](https://realpython.com/blog/python/digging-deeper-into-migrations/)
- **Part 3: Data Migrations (current article)**
- Video: [Django 1.7 Migrations - primer](https://realpython.com/blog/python/django-migrations-a-primer/#video)

Back again.

Migrations are mainly for keeping the data model of you database up-to-date, but a database is more than just a data model. Most notably, it's also a large collection of data. So any discussion of database migrations wouldn't be complete without also talking about data migrations.

## Data Migrations Defined

Data migrations are used in a number of scenarios. Two very popular ones are:

1. When you would like to load "system data" that your application depends upon being present to operate successfully.
2. When a change to a data model forces the need to change the existing data.

> Do note that loading dummy data for testing is not in the above list. You could use migrations to do that, but migrations are often run on production servers, so you probably don't want to be creating a bunch of dummy test data on your production server.

## Examples

Continuing from the previous Django Project, as an example of creating some "system data", let's create some historical bitcoin prices. Django migrations will help us out, by creating an empty migration file and putting it in the right place if we type:

```
$ ./manage.py makemigrations --empty historical_data
```

This should create a file called `historical_data/migrations/003_auto<date_time_stamp>.py`. Let's change the name to `003_load_historical_data.py` and then open it up. You'll have a default structure which looks like:

```python
# encoding: utf8
from django.db import models, migrations


class Migration(migrations.Migration):

    dependencies = [
        ('historical_data', '0002_auto_20140710_0810'),
    ]

    operations = [
    ]
```

You can see it's created a base structure for us, and even inserted the dependencies. That's helpful. Now to do some data migrations, use the `RunPython` migration operation:

```python
# encoding: utf8
from django.db import models, migrations
from datetime import date

def load_data(apps, schema_editor):
    PriceHistory = apps.get_model("historical_data", "PriceHistory")

    PriceHistory(date=date(2013,11,29),
         price=1234.00,
         volume=354564,
         total_btc=12054375,
         ).save()
    PriceHistory(date=date(2012,11,29),
         price=12.15,
         volume=187947,
         total_btc=10504650,
         ).save()


class Migration(migrations.Migration):

    dependencies = [
        ('historical_data', '0002_auto_20140710_0810'),
    ]

    operations = [
        migrations.RunPython(load_data)
    ]
```

We start off by defining the function `load_data` which - you guessed it - loads data.

> For a real app we might want to go out to blockchain.info and grab the complete list of historic prices, but we just put a couple in there to show how the migration works.

Once we have the function we can call it from our `RunPython` operation and then this function will be executed when we run `./manage.py migrate` from the command line.

Take note of the line:

```
PriceHistory = apps.get_model("historical_data", "PriceHistory")
```

When running migrations, it's important to get the version of our `PriceHistory` model that corresponds with the point in the migration where you are at. As you run migrations, your model (`PriceHistory`) can change, if for example you add or remove a column in a subsequent migration. This can cause your data migration to fail, unless you use the above line to get the correct version of the model. For more on this, please see the [comment here](https://realpython.com/blog/python/data-migrations/#comment-1843026722).

This is arguably more work than running `syncdb` and having it load a fixture. In fact, migrations don't respect fixtures - meaning they won't automatically load them for you like `syncdb` would.

This is mainly due to philosophy.

While you could use migrations to load data, they are mainly about migrating data and/or data models. We've shown an example of loading system data, mainly because it's a simple explanation of how you would set up a data migration, but often times, data migrations are used for more complex actions like transforming your data to match the new data model.

An example might be if we decided to start storing prices from multiple exchanges instead of just one, so we could add fields like `price_gox`, `price_btc`, etc, then we could use a migration to move all data from the `price` column to the `price_btc` column.

In general when dealing with migrations in Django 1.7, it's best to think of loading data as a separate exercise from migrating the database. If you do want to continue to use/load fixtures, you can use a command like:

```
$ ./manage.py loaddata historical_data/fixtures/initial_data.json
```

This will load data from the fixture into the database.

This doesn't happen automatically as with a data migration (which is probably a good thing), but the functionality is still there; it hasn't been lost, so feel free to continue to use fixtures if you have a need. The difference is that now you load data with fixtures when you need it. This is something to keep in mind if you are using fixtures to load test data for your unit tests.

## Conclusion

This, along with the previous two articles, covers the most common scenarios you'll encounter when using migrations. There are plenty more scenarios, and if you're curious and really want to dive into migrations, the best place to go (other than the code itself) is the [official docs](https://docs.djangoproject.com/en/1.7/topics/migrations/). It's the most up-to-date and does a pretty good job of explaining how things work. *If there is a more complex scenario that you would like to see an example of, please let us know by commenting below.*

Remember that in the general case, you are dealing with either:

1. __Schema Migrations__ - a change to the structure of the database or tables with no change to the data. This is the most common type, and Django can generally create these migrations for you automatically.

2. __Data Migrations__ - a change to the data, or loading new data. Django cannot generate these for you. They must be created manually using the `RunPython` migration.

So pick the migration that is correct for you, run `makemigrations` and then just be sure to update your migration files every time you update your model - and that's more or less it. That will allow you to keep your migrations stored with your code in git and ensure that you can update your database structure without having to lose data.

**Happy migrating!**