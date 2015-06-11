---
layout: post
title: "Python Interview Problem - Parsing CSV Files"
date: 2014-05-31 11:55:32 -0700
toc: true
comments: true
category_side_bar: true
categories: [python, fundamentals, data science]

keywords: "python, fundamentals, parse, parsing, parsing csv, csv, parsing csv python"
description: "In this post, I answer a Python interview question involving CSV parsing and test driven development."
---

So, a friend of mine recently interviewed for a back-end Python developer position, and the initial interview consisted of answering the following problem. He was given two hours.

## Problem

1. **Football**: The *football.csv* file contains the results from the English Premier League. The columns labeled ‘Goals’ and ‘Goals Allowed’ contain the total number of goals scored for and against each team in that season (so Arsenal scored 79 goals against opponents, and had 36 goals scored against them). Write a program to read the file, then print the name of the team with the smallest difference in ‘for’ and ‘against’ goals.
1. **Weather**: In *weather.csv* you’ll find daily weather data. Write a program to read the file, then output the day number (column one) with the smallest temperature spread (the maximum temperature is the second column, the minimum the third column).
1. See if you can write the same program to solve both questions.
1. Test Driven Development!

You can grab all the files [here](https://github.com/realpython/interview-questions/tree/master/parsing-data).

*Try this on your own before you look at my answer below.*

> **Note:** I have not solved this problem before, so I will be going through a number of iterations. Hopefully, this will give you insight into the process/workflow I went through. Once finished, compare your workflow with mine. What did you do differently? Finally, I will be using a true TDD approach for this. In other words, after I write a test, I will only write the bare minimum amount of code to get it to pass. That said, there will most likely be plenty of refactoring throughout. Bare with me. Patience.

## Part 1: Football

### Write your tests

In true TDD style, we start by writing our tests that will fail. Since we must first read in the CSV data, let's ensure that we can do that, by testing that the data read in is what we think it should be.

```python
import unittest
from parse_csv import read_data


class ParseCSVTest(unittest.TestCase):

    def setUp(self):
        self.data = 'football.csv'

    def test_csv_read_data_headers(self):
        self.assertEqual(
            read_data(self.data)[0],
            ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points']
            )

    def test_csv_read_data_team_name(self):
        self.assertEqual(read_data(self.data)[1][0], 'Arsenal')

    def test_csv_read_data_points(self):
        self.assertEqual(read_data(self.data)[1][7], '87')


if __name__ == '__main__':
    unittest.main()
```

Here, we are using a function imported from *parse_csv.py* called `read_data()`, which we haven't written yet, to read in the data. Then we have three unit tests to check the values of the data read in.

Run your tests:

```sh
$ python parse_csv_test.py
Traceback (most recent call last):
  File "parse_csv_test.py", line 2, in <module>
    from parse_csv import read_data
ImportError: cannot import name read_data
```

### Write the `read_data()` function

Next, let's write *just enough* code in *parse_csv.py* to get our tests to pass.

```python
import csv


def read_data(data):
    with open(data, 'r') as f:
        data = [row for row in csv.reader(f.read().splitlines())]
    return data


# ---- run code ---- #

data = "football.csv"
read_data(data)
```

Run the tests again:

```sh
$ python parse_csv_test.py -v
test_csv_read_data_headers (__main__.ParseCSVTest) ... ok
test_csv_read_data_points (__main__.ParseCSVTest) ... ok
test_csv_read_data_team_name (__main__.ParseCSVTest) ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.001s

OK
```

### Test Redux

Back to the test file. This time, we want to find the "smallest difference in ‘for’ and ‘against’ goals". So, add this import and function:

```python
from parse_csv import read_data, get_min_score_difference

...

def test_get_min_score_difference(self):
    self.assertEqual(get_min_score_difference(parsed_data), "no idea")
```

Since, we don't know off hand what the smallest difference will be, we can just add the string "no idea". You could update that when you know what the answer is after we add the `get_min_score_difference()` function to *parse_csv.py* - but how do we know if that answer is correct?

Keep in mind, that since this is a small amount of data, you can get away with calculating them all by hand and finding the smallest difference. Or you could also use test data.:

```python
def test_get_min_score_difference(self):
    parsed_data = [
        ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points'],
        ['Arsenal', '38', '26', '9', '3', '79', '36', '87'],
        ['Liverpool', '38', '24', '8', '6', '67', '30', '80']
    ]
    self.assertEqual(get_min_score_difference(parsed_data), '37')
```

Let's go with the latter, since we *know* this is correct.

Run the test. Watch it fail.

### Write the `get_min_score_difference()` function

To simplify this, let's use list comprehensions:

```python
def get_min_score_difference(parsed_data):
    parsed_data.pop(0)
    goals = [x[5] for x in parsed_data]
    goals_allowed = [x[6]for x in parsed_data]
    return min([float(x) - float(y) for x, y in zip(goals, goals_allowed)])
```

So, what's going on:

1. First, we removed the header row since it will just get in the way.
2. Next, we created two new lists - one containing the goals score, the other containing the goals allowed
3. Finally, we simply created another new list containing the values from the first two lists subtracted, then we returned the smallest value.

Test:

```sh
$ python parse_csv_test.py -v
test_csv_read_data_headers (__main__.ParseCSVTest) ... ok
test_csv_read_data_points (__main__.ParseCSVTest) ... ok
test_csv_read_data_team_name (__main__.ParseCSVTest) ... ok
test_get_min_score_difference (__main__.ParseCSVTest) ... ok

----------------------------------------------------------------------
Ran 4 tests in 0.000s

OK
```

They pass. But remember - There is another step to the TDD process: Refactoring. (*Hint, hint*).

Ask yourself: Since we eventually want the name of the team with the smallest spread, does it help to get the minimum value? No. It would be much easier to find the minimum value and then return the index value so that we can plug that in to the next function to easily get the name of the team.

First, rewrite the test:

```python
def test_get_min_score_difference(self):
    data = [
        ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points'],
        ['Arsenal', '38', '26', '9', '3', '79', '36', '87'],
        ['Liverpool', '38', '24', '8', '6', '67', '30', '80']
    ]
    self.assertEqual(get_min_score_difference(data), 1)
```

Now, let's get it to pass:

```python
def get_min_score_difference(parsed_data):
    parsed_data.pop(0)
    goals = [x[5] for x in parsed_data]
    goals_allowed = [x[6]for x in parsed_data]
    values = [float(x) - float(y) for x, y in zip(goals, goals_allowed)]
    return values.index(min(values))
```

Test again.

> If you're having trouble following the list comprehensions, try rewriting them using the regular list construct. For example:
>   ```python
    for x in parsed_data:
        goals.append(x[5])
    ```

### Final Test

Again, let's test to see which team has the smallest range between goals scored and goals allowed:

```python
def test_get_team(self):
    data = [
        ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points'],
        ['Arsenal', '38', '26', '9', '3', '79', '36', '87'],
        ['Liverpool', '38', '24', '8', '6', '67', '30', '80']
    ]
    index_value = get_min_score_difference(data)
    self.assertEqual(get_team(index_value), 'Liverpool')
```

Make sure to import in the `get_team` function. Run the tests file. This will fail. Success!

### Write the `get_team()` function

Simply use that index value from the previous function as an argument, then create a list of all the teams, and finally pass in that index value to that list of teams:

```
def get_team(index_value, parsed_data):
    teams = [x[0] for x in parsed_data]
    return teams[index_value]
```

Tests pass.

### Refactor Tests

In our tests, let's move our parsed data to the `setUp` so that we're not repeating ourselves:

```python
import unittest
from parse_csv import read_data, get_min_score_difference, get_team


class ParseCSVTest(unittest.TestCase):

    def setUp(self):
        self.data = 'football.csv'
        self.parsed_data = [
            ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points'],
            ['Arsenal', '38', '26', '9', '3', '79', '36', '87'],
            ['Liverpool', '38', '24', '8', '6', '67', '30', '80']
        ]

    def test_csv_read_data_headers(self):
        self.assertEqual(
            read_data(self.data)[0],
            ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points']
            )

    def test_csv_read_data_team_name(self):
        self.assertEqual(read_data(self.data)[1][0], 'Arsenal')

    def test_csv_read_data_points(self):
        self.assertEqual(read_data(self.data)[1][7], '87')

    def test_get_min_score_difference(self):
        self.assertEqual(get_min_score_difference(self.parsed_data), 1)

    def test_get_team(self):
        index_value = get_min_score_difference(self.parsed_data)
        self.assertEqual(get_team(index_value), 'Liverpool')


if __name__ == '__main__':
    unittest.main()
```

### Refactor Code

So, per the instructions, we need to make this code work for both CSV files. Let's refactor our code from using a procedural structure or an OOP structure, meant for code reuse:

```python
import csv


class ParseCSV(object):

    def __init__(self, data):
        self.data = data

    def read_data(self):
        with open(self.data, 'r') as f:
            parsed_data = [row for row in csv.reader(f.read().splitlines())]
        return parsed_data

    def get_min_score_difference(self, parsed_data):
        parsed_data.pop(0)
        goals = [x[5] for x in parsed_data]
        goals_allowed = [x[6]for x in parsed_data]
        values = [float(x) - float(y) for x, y in zip(goals, goals_allowed)]
        return values.index(min(values))

    def get_team(self, index_value, parsed_data):
        teams = [x[0] for x in parsed_data]
        return teams[index_value]
```

Try running the tests; they will fail:

```sh
$ python parse_csv_test.py
Traceback (most recent call last):
  File "parse_csv_test.py", line 2, in <module>
    from parse_csv import read_data, get_min_score_difference, get_team
ImportError: cannot import name read_data
```

### Refactor Tests Redux

```python
import unittest
from parse_csv import ParseCSV


class ParseCSVTest(unittest.TestCase):

    def setUp(self):
        self.data = 'football.csv'
        self.parsed_data = [
            ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points'],
            ['Arsenal', '38', '26', '9', '3', '79', '36', '87'],
            ['Liverpool', '38', '24', '8', '6', '67', '30', '80']
        ]
        self.football = ParseCSV(self.data)

    def test_csv_read_data_headers(self):
        self.assertEqual(
            self.football.read_data()[0],
            ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points']
            )

    def test_csv_read_data_team_name(self):
        self.assertEqual(self.football.read_data()[1][0], 'Arsenal')

    def test_csv_read_data_points(self):
        self.assertEqual(self.football.read_data()[1][7], '87')

    def test_get_min_score_difference(self):
        self.assertEqual(self.football.get_min_score_difference(self.parsed_data), 1)

    def test_get_team(self):
        index_value = self.football.get_min_score_difference(self.parsed_data)
        self.assertEqual(self.football.get_team(index_value, self.parsed_data), 'Liverpool')


if __name__ == '__main__':
    unittest.main()
```

There are minimal changes here. At this point, you could create a separate CSV file for testing. Instead, let's go on to the next part and look at the CSV issue later.

> Remember: Let’s not over optimize too early - just make the minimal changes necessary to get the tests to pass. We do have some naming issues and a few other issues to refactor. However, let's work on the second part of the problem, incorporating the weather data into this code, then refactor at the end.

## Part 2: Weather

### Watch our tests Fail

Before we do anything, let's use the new CSV file and watch our tests fail:

```sh
python parse_csv_test.py -v
test_csv_read_data_headers (__main__.ParseCSVTest) ... FAIL
test_csv_read_data_points (__main__.ParseCSVTest) ... FAIL
test_csv_read_data_team_name (__main__.ParseCSVTest) ... FAIL
test_get_min_score_difference (__main__.ParseCSVTest) ... ok
test_get_team (__main__.ParseCSVTest) ... ok
```

Notice how the only tests that pass are the tests that use `parsed_data` rather than the CSV. This is a good indication that we really should be using the actual data for testing. Let's quickly refactor (again!):

```python
import unittest
from parse_csv import ParseCSV


class FootballParseCSVTest(unittest.TestCase):

    def setUp(self):
        self.data = 'football.csv'
        self.football = ParseCSV(self.data)

    def test_csv_read_data_headers(self):
        self.assertEqual(
            self.football.read_data()[0],
            ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points']
            )

    def test_csv_read_data_team_name(self):
        self.assertEqual(self.football.read_data()[1][0], 'Arsenal')

    def test_csv_read_data_points(self):
        self.assertEqual(self.football.read_data()[1][7], '87')

    def test_get_min_score_difference(self):
        parsed_data = self.football.read_data()
        self.assertEqual(self.football.get_min_score_difference(parsed_data), 19)

    def test_get_team(self):
        parsed_data = self.football.read_data()
        index_value = self.football.get_min_score_difference(parsed_data)
        self.assertEqual(self.football.get_team(index_value, parsed_data), 'Leicester')


if __name__ == '__main__':
    unittest.main()
```

These will pass. Now change the data so that they run with *weather.csv*:

```sh
$ python parse_csv_test.py -v
test_csv_read_data_headers (__main__.FootballParseCSVTest) ... FAIL
test_csv_read_data_points (__main__.FootballParseCSVTest) ... FAIL
test_csv_read_data_team_name (__main__.FootballParseCSVTest) ... FAIL
test_get_min_score_difference (__main__.FootballParseCSVTest) ... ERROR
test_get_team (__main__.FootballParseCSVTest) ... ERROR
```

This is exactly what we want to see. Now we need to refactor both our code and tests to get our code to work with both data sets as well as simplify our tests to eliminate redundancy. (Notice a trend yet?)

### Refactor Tests and Code

#### Test

```python
def setUp(self):
    self.football_data = 'football.csv'
    self.weather_data = 'weather.csv'
    self.football = ParseCSV(self.football_data)
    self.weather = ParseCSV(self.weather_data)
```

Here, we are using both data sets and instantiating multiple instances of the same class, `ParseCSV()`. *Yes, there is some redundancy here, but let's just keep it simple for now. We'll refactor at the end.*

#### Test

```python
def test_csv_read_data_headers(self):
    self.assertEqual(
        self.football.read_data()[0],
        ['Team', 'Games', 'Wins', 'Losses', 'Draws', 'Goals', 'Goals Allowed', 'Points']
        )
    self.assertEqual(
        self.weather.read_data()[0],
        ['Day', 'MxT', 'MnT', 'AvT', 'AvDP', '1HrP TPcpn', 'PDir', 'AvSp', 'Dir', 'MxS', 'SkyC', 'MxR', 'Mn', 'R AvSLP']
        )

def test_csv_read_random_data_points(self):
    self.assertEqual(self.football.read_data()[1][0], 'Arsenal')
    self.assertEqual(self.football.read_data()[1][7], '87')
    self.assertEqual(self.weather.read_data()[1][0], '1')
    self.assertEqual(self.weather.read_data()[1][7], '9.6')
```

This is straightforward. Comment out the other two tests and run just this one. It should pass.

#### Test

```python
def test_get_min_difference(self):
    football_parsed_data = self.football.read_data()
    self.assertEqual(self.football.get_min_score_difference(football_parsed_data), 19)
    weather_parsed_data = self.weather.read_data()
    self.assertEqual(self.weather.get_min_score_difference(weather_parsed_data), "no idea")
```

Uncomment this test. It will fail:

```sh
$ python parse_csv_test.py -v
test_csv_read_data_headers (__main__.FootballParseCSVTest) ... ok
test_csv_read_random_data_points (__main__.FootballParseCSVTest) ... ok
test_get_min_difference (__main__.FootballParseCSVTest) ... FAIL
```

Why does it fail?

```sh
AssertionError: 2 != 'no idea'
```

This essentially is saying that the smallest difference between the second and third column is row two. Is that correct? We addressed this very same issue earlier:

> Since, we don't know off hand what the smallest difference will be, we can just add the string "no idea". You could update that when you know what the answer is after we add the `get_min_score_difference()` function to *parse_csv.py* - but how do we know if that answer is correct?

> Keep in mind, that since this is a small amount of data, you can get away with calculating them all by hand and finding the smallest difference. Or you could also use test data.

Let's go the other route this time: Just calculating it by hand. *Do not do this if you have a lot of data!*.

Do this now.

You should find that the row with the smallest difference is row 14 (or 13 minus the header). Update your tests, run them again, and they should still fail:

```sh
AssertionError: 2 != 13
```

Why is this?

#### Code

Go back to *parse.csv* and look at the `get_min_score_difference()` function:

```python
def get_min_score_difference(self, parsed_data):
    parsed_data.pop(0)
    goals = [x[5] for x in parsed_data]
    goals_allowed = [x[6]for x in parsed_data]
    values = [float(x) - float(y) for x, y in zip(goals, goals_allowed)]
    return values.index(min(values))
```

Did you figure it out?

We're passing in the column values - `x[5]` and x[6] - which are applicable only to the *football.csv*. Thus, we need to pass in either the column index values or the header names so that this function uses the right data regardless of the data set used. Since we're popping off the header, let's use the column index values:

```python
def get_min_difference(self, parsed_data, column1, column2):
    parsed_data.pop(0)
    column1_list = [x[column1] for x in parsed_data]
    column2_list = [x[column2]for x in parsed_data]
    values = [float(x) - float(y) for x, y in zip(column1_list, column2_list)]
    return values.index(min(values))
```

#### Test

Update the test:

```python
def test_get_min_difference(self):
    football_parsed_data = self.football.read_data()
    self.assertEqual(self.football.get_min_difference(football_parsed_data, 5, 6), 19)
    weather_parsed_data = self.weather.read_data()
    self.assertEqual(self.weather.get_min_difference(weather_parsed_data, 1, 2), 13)
```

And they pass:

```sh
$ python parse_csv_test.py -v
test_csv_read_data_headers (__main__.FootballParseCSVTest) ... ok
test_csv_read_random_data_points (__main__.FootballParseCSVTest) ... ok
test_get_min_difference (__main__.FootballParseCSVTest) ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.001s

OK
```

Uncomment the last test and update `get_min_score_difference()` to `get_min_difference()` then pass in the columns:

```python
def test_get_name(self):
    football_parsed_data = self.football.read_data()
    index_value = self.football.get_min_difference(football_parsed_data, 5, 6)
    self.assertEqual(self.football.get_team(index_value, football_parsed_data), 'Leicester')
    weather_parsed_data = self.weather.read_data()
    index_value = self.weather.get_min_difference(weather_parsed_data, 1, 2)
    self.assertEqual(self.weather.get_team(index_value, weather_parsed_data), '14')
```

Now they all pass:

```sh
 python parse_csv_test.py -v
test_csv_read_data_headers (__main__.FootballParseCSVTest) ... ok
test_csv_read_random_data_points (__main__.FootballParseCSVTest) ... ok
test_get_min_difference (__main__.FootballParseCSVTest) ... ok
test_get_name (__main__.FootballParseCSVTest) ... ok

----------------------------------------------------------------------
Ran 4 tests in 0.001s

OK
```

### Add test data

Finally, let's add our own test data for use rather than using the actual CSV files to isolate test data from actual, real data.

```python
import unittest
from parse_csv import ParseCSV


class testParseCSVTest(unittest.TestCase):

    def setUp(self):
        self.data = 'test.csv'
        self.test = ParseCSV(self.data)
        self.parsed_data = self.test.read_data()

    def test_csv_read_data_headers(self):
        self.assertEqual(
            self.parsed_data[0],
            ['Date', 'Open', 'High', 'Low', 'Close', 'Volume', 'Adj Close']
            )

    def test_csv_read_random_data_points(self):
        self.assertEqual(self.test.read_data()[1][0], '5/8/14')
        self.assertEqual(self.test.read_data()[1][6], '511')

    def test_get_min_difference(self):
        self.assertEqual(self.test.get_min_difference(self.parsed_data, 2, 3), 6)

    def test_get_name(self):
        index_value = self.test.get_min_difference(self.parsed_data, 2, 3)
        self.assertEqual(self.test.get_team(index_value, self.parsed_data), '4/30/14')


if __name__ == '__main__':
    unittest.main()
```

Run the tests. They should pass.

### One last thing ...

One thing to keep in mind is the nature of the `pop()` method. It's destructive, so it alters the original list, `parsed_data`, removing the headers permanently within the `get_min_difference()` function. What are the ramifications of this?

Let's test it out.

Add a `print` statement to the `test_get_name()` function:

```python
def get_team(self, index_value, parsed_data):
    print parsed_data[0]
    teams = [x[0] for x in parsed_data]
    return teams[index_value]
```

Run the tests again.

```
$ python parse_csv_test.py
...['5/8/14', '508.46', '517.23', '506.45', '511', '2015800', '511']
.
----------------------------------------------------------------------
Ran 4 tests in 0.001s

OK
```

Notice that row 0 contains data, not the headers. This can create problems. Sure, you could just assign a variable to the data when you pop it off - `headers = parsed_data.pop(0)` - then add the data back to the list. But, it's generally better to work with non-destructive methods if you do not actually need to alter the original structure.

Thus, let's use `slice` instead. Update the `get_min_difference()` function:

```python
def get_min_difference(self, parsed_data, column1, column2):
    column1_list = [x[column1] for x in parsed_data[1:]]
    column2_list = [x[column2]for x in parsed_data[1:]]
    values = [float(x) - float(y) for x, y in zip(column1_list, column2_list)]
    return values.index(min(values))
```

Run the test again. You should see the following failure:

```
$ python parse_csv_test.py
...F
======================================================================
FAIL: test_get_name (__main__.testParseCSVTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "parse_csv_test.py", line 27, in test_get_name
    self.assertEqual(self.test.get_team(index_value, self.parsed_data), '4/30/14')
AssertionError: '5/1/14' != '4/30/14'

----------------------------------------------------------------------
Ran 4 tests in 0.001s

FAILED (failures=1)
```

This is because we are now dealing with data that has the header back in within the `get_team` function while the header was sliced off in `get_min_difference()`. Thus, we need to update the `get_team()` function:

```python
def get_team(self, index_value, parsed_data):
    teams = [x[0] for x in parsed_data[1:]]
    return teams[index_value]
```

Run the tests one last time ...