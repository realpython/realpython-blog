---
layout: post
title: "Digging Deeper into Migrations"
date: 2014-07-23 07:14:57 -0500
toc: true
comments: true
category_side_bar: true
categories: [python, django, migrations]

keywords: "python, django, web development, python tutorial, django 1.7, migrations"
description: "This tutorial takes you deeper through the new Django migration system that is integrated in the Django, taking you through the migration files themselves."
---

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-migrations.png" style="max-width: 100%;">
</div>

This is the second article in our Django migrations series:

- Part 1: [Django Migrations - A Primer](https://realpython.com/blog/python/django-migrations-a-primer/)
- **Part 2: Digging Deeper into Migrations (current article)**
- Part 3: [Data Migrations](https://realpython.com/blog/python/data-migrations)
- Video: [Django 1.7 Migrations - primer](https://realpython.com/blog/python/django-migrations-a-primer/#video)

We're back!

Last time we went over the basics of using the new Django migrations system.

Jumping right back in to where we left off, what happens when things don't work as they should? Well, that's when you may have to go searching through your previous migrations to try to figure out what's going on. *To help with that let's dig a bit deeper to get a better understanding of how migrations work.*

## How Migrations Know What to Migrate

Try this. From the `bitcoin_tracker` app run the migration again (`./manage.py migrate`). What happens? Nothing. And that's exactly the point.

By default Django will never run a migration more than once on the same database. This is managed by a table called `django_migrations` that is created in your database the first time migrations are ran. For each migration that is ran or faked, a new row is inserted into the table.

For example, here is what the table might look like after running our initial migration:

<table style="
    font-size: 16px;border-spacing: 10px 0px;border-collapse: separate;
">
    <thead>
        <tr>
            <th>ID</th>
            <th>app</th>
            <th>name</th>
            <th>applied</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1</td>
            <td>historical_payments</td>
            <td>0001_initial</td>
            <td>2014-04-16 14:12:30.839899+08</td>
        </tr>
    </tbody>
</table>

<br>

Not very interesting because there is just one migration, but there will be new rows added for each subsequent migration.

The next time migrations are run, it will skip the migration files listed in the database table. This means that even if you change the migration file manually, it will be skipped if there is already an entry for it in the database.

This makes sense as you generally don't want to run migrations twice. But if for whatever reason you do, one way to get it to run again is to first delete the corresponding row from the database (Do note this is not an "officially recommended way", but it will work). In the case of upgrading from [South](http://south.aeracode.org/) the first time you run migrations, Django will first check the database structure, and if it is the same as the migration (i.e., the migration doesn't apply any new changes) then the migration will be "faked" meaning not really ran, but the `django_migrations` table will still be updated.

Conversely, if you want to "undo" all the migrations for a particular app, you can migrate to a special migration called zero.

For example if you type:

```sh
./manage.py migrate historical_data zero
```

It will undo/reverse all the migrations for the `historical_data` app. In addition to using zero; you can also use any arbitrary migration, and if that migration is in the past then the database will be rolled back to the state of that migration, or rolled forward if the migration hasn't yet ran. Pretty powerful stuff!

## The Migration File

What about creating the actual migration file? In other words, what exactly happens when you run `./manage.py makemigrations <appname>`?

Django migrations are actually creating a migration file that describes how to create the appropriate tables in the database. In fact, you can look at the migration file that was created. Don't worry: It's just Python.

> Don't forget to `git add` the new migrations directory so it is version-controlled.

The `historical_prices` app will now have a sub-directory called `/migrations` where all the migration files for that app will reside. Let's look at `historical_data/migrations/0001_initial.py`, as this is the file where the initial migration code is created. It should look similar to:

```python
# encoding: utf8
from django.db import models, migrations


class Migration(migrations.Migration):

    dependencies = [
    ]

    operations = [
        migrations.CreateModel(
            name='PriceHistory',
            fields=[
                ('id', models.AutoField(verbose_name='ID', serialize=False, primary_key=True, auto_created=True)),
                ('date', models.DateTimeField(auto_now_add=True)),
                ('price', models.DecimalField(decimal_places=2, max_digits=5)),
                ('volume', models.PositiveIntegerField()),
                ('total_btc', models.PositiveIntegerField()),
            ],
            options={
            },
            bases=(models.Model,),
        ),
    ]
```

For a migration to work, you must create a class called `Migration()` that inherits from `django.db.migrations.Migration`. This is the class that the migration framework will look for and execute when you ask it to run migrations (which we will do later).

The Migration class contains two main lists, `dependencies` and `operations`.

## Migration dependencies

`dependencies` is a list of migrations that must be ran prior to this migration being run.

In the case above nothing has to be run prior so there are no dependencies. But if you have foreign key relationships, for example, then you will have to ensure a model is created before you can add a foreign key to it. So let's assume we had another app called `main` that defined the table we wanted to reference in our foreign key. Then our dependencies list might look like this:

```python
dependencies = [
   ('main', '__first__'),
]
```

The dependency above says that migrations for the `main` app must be run first.

> Be sure check out the [video](https://realpython.com/blog/python/django-migrations-a-primer/#video) to see examples of utilizing foreign keys and how they affect the dependencies within the migration files.

You can also have a dependency on a specific file, like so:

```python
dependencies = [
    ('main', '0001_initial'),
]
```

This is a dependency for the file called `0001_initial` from the `main` app.

Dependencies can also be combined so you can have multiple dependencies. This functionality provides a lot of flexibility, as you can accommodate foreign keys that depend upon models from different apps. It also means that the numbering of the migrations (usually 0001, 0002, 0003, ...) doesn't strictly have to be in the order they are applied. You can add any dependency you want and thus control the order without having to re-number all the migrations.

### Migration operations

The second list in the `Migration()` class is the `operations` list. This is a list of operations to be applied as part of the migration. Generally the operations can fall under one of the following types:

* __CreateModel__: You guessed it: this creates a new model. See the migration above for an example.
* __DeleteModel__: removes a table from the database; just pass in the name of the model.
* __RenameModel__: Given the `old_name` and `new_name`, this renames the model.
* __AlterModelTable__: changes the name of the table associated with a model. Same as the `db_table` option.
* __AlterUniqueTogether__: changes unique constraints.
* __AlteIndexTogether__: changes the set of custom indexes for the model.
* __AddField__: Just like it sounds. Here is an example:

    ```python
    migrations.AddField(
        model_name='PriceHistory',
        name='market_cap',
        field=models.PositiveIntegerField(),
    ),
    ```

* __RemoveField__: We don't want that field anymore... just drop it.
* __RenameField__: Given `model_name`, `old_name` and `new_name`, this changes the field with `old_name` to `new_name`.

There are also a few "special" operations:

* __RunSQL__: This allows you to pass in raw SQL and execute it as part of your model.
* __RunPython__: passes in a callable to be executed; useful for things like data loading as part of the migration.

*You can even write your own operations.* Generally when you run `makemigrations`, Django will create the necessary migrations with the appropriate dependencies and operations that you need. However, understanding the migration files themselves and how they work give you more flexibility.

## Example

Let's make a few more changes to our model to see the effect on the migrations:

```python
class PriceHistory(models.Model):
    date = models.DateTimeField(auto_now_add=True)
    #bitcoin to the moon (we need more digits)
    price = models.DecimalField(max_digits=8, decimal_places=2)
    volume = models.PositiveIntegerField()
    total_btc = models.PositiveIntegerField()
    market_cap = models.PositiveIntegerField(null=True)
```

Being bullish on bitcoin we have decided we need a larger number for the price field, and we have also decided to keep track of the market capitalization. Notice how we made the `market_cap` field nullable. If we didn't, migrations will ask to supply a value for all the existing rows (just like South does):

```sh
You are trying to add a non-nullable field 'market_cap' to PriceHistory without a default;
we can't do that (the database needs something to populate existing rows).
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows)
 2) Quit, and let me add a default in models.py
```

Now running `./manage.py makemigrations` again will produce a new migration file `0002_auto_<date_time_stamp>`.

That file should look like this:

```python
class Migration(migrations.Migration):

    dependencies = [
        ('historical_data', '0001_initial'),
    ]

    operations = [
        migrations.AddField(
            model_name='PriceHistory',
            name='market_cap',
            field=models.PositiveIntegerField(null=True),
            preserve_default=True,
        ),
        migrations.AlterField(
            model_name='PriceHistory',
            name='price',
            field=models.DecimalField(max_digits=8, decimal_places=2),
        ),
    ]
```

Notice the 'dependencies' list which declares that we have to run our initial migration prior to running this one. Also this migration has two operations - `AddField`, which creates our newly added `market_cap`, and `AlterField`, which updates the `max_digits` of our `price` field.

It's important to understand that these operations just call the migrations framework, which handles performing the various operations against the database that is defined in your `settings.py` file.

**Out of the box migrations have support for all the standard databases that Django supports. So if you stick to the primitives listed in dependencies section you can manually create whatever migration you want, without having to worry about the underlying SQL. That's all done for you.**

## Conclusion

We've come to another end, but there's one more beginning. In the last post, we'll look at Data Migrations. Cheers!
