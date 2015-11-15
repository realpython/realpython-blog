# Django Migrations - A Primer

<div class="center-text">
  <img class="no-border" src="/images/blog_images/django-migrations.png" style="max-width: 100%;">
</div>

**Updates**:

- *Nov 12, 2015*: Updated to cover Django 1.8 specific functionality as it relates to migrations.

<hr>

What's new in Django 1.7? Basically migrations. While there are some other nice features, the new migrations system is the big one.

In the past you probably used [South](http://south.aeracode.org/) to handle database changes. However, in Django 1.7, migrations are now integrated into the Django Core thanks to Andrew Godwin, who ran [this Kickstarter](https://www.kickstarter.com/projects/andrewgodwin/schema-migrations-for-django). He is also the original creator of South.

In honor of this momentous update, we are going to cover migrations, how they work and how to get the most out of them across three blog posts and one video:

- **Part 1: Django Migrations - A Primer (current article)**
- Part 2: [Digging Deeper into Migrations](https://realpython.com/blog/python/digging-deeper-into-migrations/)
- Part 3: [Data Migrations](https://realpython.com/blog/python/data-migrations)
- Video: [Django 1.7 Migrations - primer](https://realpython.com/blog/python/django-migrations-a-primer/#video)

Let's begin...

## The problems that Migrations Solve

Migrations:

1. Speed up the notoriously slow process of changing a database schema.
1. Make it easy to use git to track your database schema and its associated changes. Databases simply aren't aware of git or other version control systems. Git is awesome for code, but not so much for database schemas.
1. Provide an easy way to maintain fixture data linked to the appropriate schema.
1. Keep the code and schema in sync.

Have you ever had to make a change to an existing table (like re-naming a field) and you didn't want to mess with dropping the table and re-adding it?

Migrations solve that.

Or perhaps you needed to update the schema on a live application with millions of rows of data that simply cannot be lost.

Migrations make this much easier.

In general, migrations allow you to manage and work with your database schema in the same way you would with your Django code. You can store versions of it in git, you can update it from the command line, and you don't have to worry about creating large complex SQL queries to keep everything up to date - although you still can if you love pain... I mean SQL.

## Getting Started with Migrations

For this blog post we are going to create a simple bitcoin tracker app (actually we are just going to create the model).

> Be sure check out the [video](https://realpython.com/blog/python/django-migrations-a-primer/#video) to see examples of the migrations workflow.

### Setup the project

With Django 1.8 installed you can create the project using the following commands:

```sh
$ django-admin.py startproject bitcoin_tracker
$ cd bitcoin_tracker
$ ./manage.py startapp historical_data
```

That should give you a simple project with an app called `historical_data`. Now to create a model edit `historical_data/models.py`:

```python
class PriceHistory(models.Model):
    date = models.DateTimeField(auto_now_add=True)
    price = models.DecimalField(max_digits=5, decimal_places=2)
    volume = models.PositiveIntegerField()
    total_btc = models.PositiveIntegerField()
```

Also don't forget to add the newly created app to `settings.INSTALLED_APPS`.

This is a basic model to keep track of bitcoin prices.

### Create the migrations

With the model created, the first thing we need to do is "initialize" our migrations. We can do this with the following command:

```sh
$ $ ./manage.py makemigrations historical_data
```

> **Note:** Specifying the name of the application, `historical_data`, is optional. Leaving it off will create migrations for all apps.

Upon running the above command Django migrations will create the migrations for you and output an informational message such as:

```sh
Migrations for 'historical_data':
  0001_initial.py:
    - Create model PriceHistory
```

This creates the migrations files which instruct Django on how to create the models that you need.

**Being OCD about naming**

Did you notice in the above example that Django came up with a name for the migration - *0001_initial.py*. If you're not happy with that name and you're running >= Django 1.8, then you can use the `--name` parameter to give it whatever name you want:

```sh
$ ./manage.py makemigrations historical_data --name myfirstmigration
```

This will create the same migration as before accept with the nem name of `myfirstmigration.py`

### Apply migrations

Then to apply the models (e.g., update the database) just run:

```sh
$ ./manage.py migrate
```

Not only will this apply the migrations we just created for `PriceHistory` but since this is a new app, it will create all the Django tables for us as well (just like `syncdb` did in Django versions prior to 1.7).

### Workflow

So the basic process for using migrations looks like this:

1. Create or update a model.
2. Run `./manage.py makemigrations <app_name>`.
3. Run `./manage.py migrate` to migrate everything or `./manage.py migrate <app_name>` to migrate an individual app.
4. Repeat as necessary.

That's it! Pretty straight-forward for the basic use-case. Fortunately, this basic use case will work the majority of the time!

## Apply Migrations to an Existing Project

That's all well and good if you're starting from scratch but many people will be upgrading from a previous version of Django and migrating from South.

To do that, Django recommends to just start using the new migrations system and everything should work. The recommended upgrade path from South to Django 1.8 migrations, paraphrased from [here](https://docs.djangoproject.com/en/dev/topics/migrations/#upgrading-from-south) is basically:

1. Delete all your South migration files (yup - just blow 'em away).
2. Run `./manage.py makemigrations`. Django will just make initial migrations files based upon your current models.
3. Run `./manage.py migrate`. Django will see the tables that already exist for your migrations and just mark them as completed without running them.

Now, if you have circular dependencies and are getting errors when going through the above process, you may have to fake the migration with:

```sh
$ ./manage.py migrate --fake <appname>
```

Also, if you have a need to support both South and Django Migrations at the same time then upgrade to South 1.0 and read [this article](http://treyhunner.com/2014/03/migrating-to-django-1-dot-7/).

## Listing out migrations

It's also worth mentioning that if for whatever reason you want to see what migrations are in a certain app / project you can use the `showmigrations` command like so:

```sh
$ ./manage.py showmigrations --list
```

This will list all apps in the project and the migrations associated with each app.  Also it will but a big `X` next to the migrations that have already been applied.  Useful information if you're trying to understand what migrations exists / have been applied to your project.

### If you already have a database

A similar scenario is when you already have a database (maybe because you have restored it from production) and you want to start working with migrations against that database without blowing away the data already there. In that case you can use the `--fake-inital` option:

```sh
$ ./manage.py migrate --fake-initial <appname>
```

This will look at your migration files, and basically skip the creation of tables that are already in your database. Do note though that any migrations that don't create tables but rather modify existing tables will be run. This makes it simple to work with existing databases.

## South vs Django migrations

For those familiar with South this should feel pretty familiar and probably a little bit cleaner.  For easy reference the following table compares the old South workflow to the new Django Migrations workflow:

<table style="
    font-size: 16px;border-spacing: 10px 0px;border-collapse: separate;
">
<thead>
<tr>
<th style="
">Step    </th>
<th>  South                         </th>
<th> Django migrations</th>
</tr>
</thead>
<tbody>
<tr>
<td>initial migration </td>
<td> <ol><li>run <code>syncdb</code></li><li>then <code>./manage.py schemamigration &lt;appname&gt; --initial</code></li></ol> </td>
<td> <code>./manage.py makemigrations &lt;appname&gt;</code></td>
</tr>
<tr>
<td>apply migration </td>
<td> <code>./manage.py migrate &lt;appname&gt;</code>  </td>
<td> <code>./manage.py migrate &lt;appname&gt;</code></td>
</tr>
<tr>
<td>non-first migration </td>
<td> <code>./manage.py schemamigration &lt;appname&gt; --auto</code> </td>
<td> <code>./manage.py makemigration &lt;appname&gt;</code></td>
</tr>
</tbody>
</table>

<br>

So from the table we can see that Django Migrations, basically follows the same process as South (at least for the standard migration process) - it just simplifies things a bit.

## Conclusion

That's it. It's all meant to be very straight forward. Of course there will always be edge cases when things don't work out exactly as intended - but we'll save that for next time.

Cheers!

## Video

{% youtube 7PiyO-N6Pho %}
