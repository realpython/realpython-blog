# Learn Python the Methodical Way

This is a guest blog post from **[Zev Averbach](http://twitter.com/zevav)** - a Python enthusiast who is working on a startup related to his current business, [Averbach Transcription](http://avtranscription.com). Say hi to him at PyCon 2015!

<hr>

One of the first pieces of advice I got about learning a programming language was from Logan Hanks: "Read the library." Then I read Derek Sivers' advice to basically [memorize the whole language](https://sivers.org/srs).

I don't really have the discipline to do either of these, frankly! However, what I've found to be very effective are drills. They're similar to the flash card approach recommended by Derek, but slightly less modular.

1. Make your way through a tutorial/chapter that teaches you some discrete, four-to-six-step skill.
1. Write down those steps as succinctly and generically as possible.
1. Put the tutorial/chapter and its solutions away.
1. Build your project from scratch, peeking only when you're stuck.
1. Erase what you built.
1. Do the project again.
1. Drink some water.
1. Erase what you built and do it again.
1. A day or two later, delete your work and do it again -- this time without peeking even once.
1. Erase your work and do it again.

> This method is particularly helpful for the Real Python course since it takes a project-driven approach, but it can be applied to any good Python book or course. Creating a project reinforces and builds upon existing skills, and prepares you for real-life coding challenges.


It may sound tedious (and obvious), but it feels great the first time you're able to do the whole task from memory, and it tends to stick there, especially if you return to an old drill every now and then.

Just remember: This is how you learned to walk, read, throw a ball, and drive a stick shift (no?): You learned "on the job" rather than memorizing separate components of a given activity, and you used crutches until you didn't need them anymore.

## Drills are a weapon against intimidation

When I hit chapter six of Real Python book two, it felt a bit overwhelming. I had skipped over the database chapter in book one, and there were a lot of things going on:

* SQL syntax
* creating and populating databases
* joining tables
* mapping built-in SQL functions to a Python dictionary, then executing them from it

Looking back on it now, it doesn't seem that dense, but it was tripping me up and not staying in my brain. So I devised a simple drill for the first three bullet points and put it in the corner of my screen:

1. Create a database, and add a table with a few columns. One should be a quantity.
1. Add a row to the table. Verify that it worked.
1. Add several rows to the table using a list of tuples. Why is this the preferred method? Verify that the rows were written.
1. Add another table with two common columns with the first table, and a "date" column, then populate it.
1. Join the two tables to print combined rows of data where the values from the two common columns coincide.

SQL syntax was my first downfall: I had to peek to get the order of `INSERT INTO table_name VALUES(...` and `CREATE TABLE pizza(topping_1 TEXT, topping_2 TEXT, quantity INT)`. Dot notation to select columns from multiple tables was pretty intuitive, but then I had to remember about `cursor.fetchall()`.

### Solution

```python
import sqlite3

# create a database
# add a table with a few columns. one should be quantity
with sqlite3.connect("new.db") as connection:
    c = connection.cursor()
    c.execute("CREATE TABLE pizza(topping_1 TEXT, topping_2 TEXT, \
    quantity INT)")
    # add a row to the table
    c.execute("INSERT INTO pizza VALUES('pepperoni', 'mushrooms', 5)")
```

### Verify

I like to use [the command line shell for SQLite](https://www.sqlite.org/cli.html) so I don't have to leave my beloved command line; to each her own:

```sh
$ sqlite3 new.db
sqlite> SELECT * FROM pizza;
pepperoni|mushrooms|5
```

Success! Then,

```python
# add several rows to the table using a list of tuples.
# this is preferred because it prevents SQL injection.
with sqlite3.connect("new.db") as connection:
    c = connection.cursor()
    more_pizza = [('sausage', 'fennel', 12),
                  ('broccoli', 'chicken', 2),
                  ('garlic', 'tofu', 0)]
    c.executemany('INSERT INTO pizza VALUES(?,?,?)', more_pizza)
```

Verify those puppies, and then

```python
# add another table with two common columns with the first table,
# and a "date" column, then populate it

    c.execute('CREATE TABLE orders(topping_1 TEXT, topping_2 TEXT, date TEXT)')
    some_orders = [('sausage', 'sundried tomatoes', '4/12/2014'),
               ('broccoli', 'chicken', '12/31/2014'),
               ('garlic', 'tofu', '04/01/1993'),
               ('sausage', 'fennel', '03/14/2015'),
               ('sausage', 'fennel', '03/15/2015'),
               ('broccoli', 'chicken', '01/02/2015')]
    c.executemany("INSERT INTO orders VALUES(?,?,?)", some_orders)

    # join the two tables to print combined rows of data
    # where the values from the two common columns coincide.
    c.execute("""SELECT pizza.topping_1, pizza.topping_2, pizza.quantity, \
    orders.date FROM pizza, orders WHERE pizza.topping_2 = \
    orders.topping_2""")
    rows = c.fetchall()
    for r in rows:
        print "topping 1: " + r[0]
        print "topping 2: " + r[1]
        print "quantity: " + str(r[2])
        print "date: " + r[3]
```

Verifying:

```sh
$ python pizzas.py
topping 1: sausage
topping 2: fennel
quantity: 12
date: 03/14/2015
topping 1: sausage
topping 2: fennel
quantity: 12
date: 03/15/2015
topping 1: broccoli
topping 2: chicken
quantity: 2
date: 01/02/2015
topping 1: broccoli
topping 2: chicken
quantity: 2
date: 12/31/2014
topping 1: garlic
topping 2: tofu
quantity: 0
date: 04/01/1993
```

Success!  Now, nuke that .py and do it again. Drink some water, go for a walk, do some squats, then `rm pizzas.py` and... you know what to do.