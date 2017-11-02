# Working with Large Excel Files in Pandas

**Today we are going to learn how to work with large files in [Pandas](http://pandas.pydata.org/), focusing on reading and analyzing an Excel file and then working with a subset of the original data.**

<div class="center-text">
  <img class="no-border" src="/images/blog_images/pandas-excel/pandas-xlsx.png" style="max-width: 100%;" alt="flask-angular-auth">
</div>

<br>

> This tutorial utilizes Python (tested with 64-bit versions of v2.7.9 and v3.4.3), [Pandas](http://pandas.pydata.org/pandas-docs/version/0.16.1/) (v0.16.1), and [XlsxWriter](https://xlsxwriter.readthedocs.org/) (v0.7.3). We recommend using the [Anaconda](http://continuum.io/downloads) distribution to quickly get started, as it comes pre-installed with all the needed libraries.

*This is a collaboration piece between Shantnu Tiwari, founder of [Python For Engineers](http://pythonforengineers.com/), and the fine folks at Real Python.*

## Reading the File

The first file we'll work with is a compilation of all the car accidents in England from 1979-2004, to extract all accidents that happened in London in the year 2000.

### Excel

Start by downloading the source ZIP file from [data.gov.uk](http://data.dft.gov.uk/road-accidents-safety-data/Stats19-Data1979-2004.zip), and extract the contents. Then *try* to open *Accidents7904.csv* in Excel. *Be careful. If you don't have enough memory, this could very well crash your computer.*

What happens?

You should see a "File Not Loaded Completely" error since Excel can only [handle](http://superuser.com/questions/366468/what-is-the-maximum-allowed-rows-in-a-microsoft-excel-xls-or-xlsx) one million rows at a time.

> We tested this in [LibreOffice](https://www.libreoffice.org/) as well and received a similar error - "The data could not be loaded completely because the maximum number of rows per sheet was exceeded."

To solve this, we can open the file in Pandas. Before we start, the source code is on [Github](https://github.com/shantnu/PandasLargeFiles).

### Pandas

Within a new project directory, activate a virtualenv, and then install Pandas:

```sh
$ pip install pandas==0.16.1
```

Now let's build the script. Create a file called *pandas_accidents.py* and the add the following code:

```python
import pandas as pd


# Read the file
data = pd.read_csv("Accidents7904.csv", low_memory=False)
# Output the number of rows
print("Total rows: {0}".format(len(data)))
# See which headers are available
print(list(data))
```

Here, we imported Pandas, read in the file - which could take some time, depending on how much memory your system has - and outputted the total number of rows the file has as well as the available headers (e.g., column titles).

When ran, you should see:

```sh
Total rows: 6224198
['\xef\xbb\xbfAccident_Index', 'Location_Easting_OSGR', 'Location_Northing_OSGR',
 'Longitude', 'Latitude', 'Police_Force', 'Accident_Severity', 'Number_of_Vehicles',
 'Number_of_Casualties', 'Date', 'Day_of_Week', 'Time', 'Local_Authority_(District)',
 'Local_Authority_(Highway)', '1st_Road_Class', '1st_Road_Number', 'Road_Type',
 'Speed_limit', 'Junction_Detail', 'Junction_Control', '2nd_Road_Class',
 '2nd_Road_Number', 'Pedestrian_Crossing-Human_Control',
 'Pedestrian_Crossing-Physical_Facilities', 'Light_Conditions', 'Weather_Conditions',
 'Road_Surface_Conditions', 'Special_Conditions_at_Site', 'Carriageway_Hazards',
 'Urban_or_Rural_Area', 'Did_Police_Officer_Attend_Scene_of_Accident',
 'LSOA_of_Accident_Location']
```

So, there are over six millions rows! No wonder Excel choked. Turn your attention to the list of headers, the first one in particular:

```python
'\xef\xbb\xbfAccident_Index',
```

This should read `Accident_Index`. What's with the extra `\xef\xbb\xbf` at the beginning? Well, the `\x` actually means that the value is [hexadecimal](http://en.wikipedia.org/wiki/Hexadecimal), which is a [Byte Order Mark](http://stackoverflow.com/a/18664752/1799408), indicating that the text is Unicode.

Why does it matter to us?

*You cannot assume the files you read are clean. They might contain extra symbols like this that can throw your scripts off.*

This file is good, in that it is otherwise clean - but many files have missing data, data in internal inconsistent format, etc.. So any time you have a file to analyze, the first thing you must do is clean it. How much cleaning? Enough to allow you to do some analysis. Follow the [KISS](http://en.wikipedia.org/wiki/KISS_principle) principle.

**What sort of cleanup might you require?**

- *Fix date/time.* The same file might have dates in different formats, like the American (mm-dd-yy) or European (dd-mm-yy) formats. These need to be brought into a common format.
- *Remove any empty values.* The file might have blank columns and/or rows, and this will come up as *NaN* (Not a number) in Pandas. Pandas provides a simple way to remove these: the `dropna()` function. We saw an example of this in the [last blog post](https://realpython.com/blog/python/analyzing-obesity-in-england-with-python/).
- *Remove any garbage values that have made their way into the data.* These are values which do not make sense (like the byte order mark we saw earlier). Sometimes, it might be possible to work around them. For example, there could be a dataset where the age was entered as a floating point number (by mistake). The `int()` function then could be used to make sure all ages are in integer format.

## Analyzing

For those of you who know SQL, you can use the SELECT, WHERE, AND/OR statements with different keywords to refine your search. We can do the same in Pandas, and in a way that is [more programmer friendly](http://pandas.pydata.org/pandas-docs/version/0.16.1/comparison_with_sql.html).

To start off, let's find all the accidents that happened on a Sunday. Looking at the headers above, there is a `Day_of_Weeks` field, which we will use.

In the ZIP file you downloaded, there's a file called *Road-Accident-Safety-Data-Guide-1979-2004.xls*, which contains extra info on the codes used. If you open it up, you will see that *Sunday* has the code `1`.

```python
print("\nAccidents")
print("-----------")

# Accidents which happened on a Sunday
accidents_sunday = data[data.Day_of_Week == 1]
print("Accidents which happened on a Sunday: {0}".format(
    len(accidents_sunday)))
```

That's how simple it is.

Here, we targeted the `Day_of_Weeks` field and returned a DataFrame with the condition we checked for - `day of week == 1`.

When ran you should see:

```sh
Accidents
-----------
Accidents which happened on a Sunday: 693847
```

As you can see, there were 693,847 accidents that happened on a Sunday.

Let's make our query more complicated: Find out all accidents that happened on a Sunday and involved more than twenty cars:

```python
# Accidents which happened on a Sunday, > 20 cars
accidents_sunday_twenty_cars = data[
    (data.Day_of_Week == 1) & (data.Number_of_Vehicles > 20)]
print("Accidents which happened on a Sunday involving > 20 cars: {0}".format(
    len(accidents_sunday_twenty_cars)))
```

Run the script. Now we have 10 accidents:

```sh
Accidents
-----------
Accidents which happened on a Sunday: 693847
Accidents which happened on a Sunday involving > 20 cars: 10
```

Let's add another condition - weather.

Open the Road-Accident-Safety-Data-Guide-1979-2004.xls, and go to the *Weather* sheet. You'll see that the code `2` means, "Raining with no heavy winds".

Add that to our query:

```python
# Accidents which happened on a Sunday, > 20 cars, in the rain
accidents_sunday_twenty_cars_rain = data[
    (data.Day_of_Week == 1) & (data.Number_of_Vehicles > 20) &
    (data.Weather_Conditions == 2)]
print("Accidents which happened on a Sunday involving > 20 cars in the rain: {0}".format(
    len(accidents_sunday_twenty_cars_rain)))
```

So there were four accidents that happened on a Sunday, involving more than twenty cars, while it was raining:

```sh
Accidents
-----------
Accidents which happened on a Sunday: 693847
Accidents which happened on a Sunday involving > 20 cars: 10
Accidents which happened on a Sunday involving > 20 cars in the rain: 4
```

We could continue making this more and more complicated, as needed. For now, we'll stop since our main interest is to look at accidents in London.

If you look at *Road-Accident-Safety-Data-Guide-1979-2004.xls* again, there is a sheet called *Police Force*. The code for `1` says, "Metropolitan Police". This is what is more commonly known as *Scotland Yard*, and is the police force responsible for most (though not all) of London. For our case, this is good enough, and we can extract this info like so:

```python
# Accidents in London on a Sunday
london_data = data[data['Police_Force'] == 1 & (data.Day_of_Week == 1)]
print("\nAccidents in London from 1979-2004 on a Sunday: {0}".format(
    len(london_data)))
```

Run the script. This created a new DataFrame with the accidents handled by the "Metropolitan Police" from 1979 to 2004 on a Sunday:

```sh
Accidents
-----------
Accidents which happened on a Sunday: 693847
Accidents which happened on a Sunday involving > 20 cars: 10
Accidents which happened on a Sunday involving > 20 cars in the rain: 4

Accidents in London from 1979-2004 on a Sunday: 114624
```

What if you wanted to create a new DataFrame that only contains accidents in the year 2000?

The first thing we need to do is convert the date format to one which Python can understand using the `pd.to_datetime()` [function](http://pandas.pydata.org/pandas-docs/version/0.16.1/generated/pandas.to_datetime.html?highlight=to_datetime#pandas.to_datetime). This takes a date in any format and converts it to a format that we can understand (*yyyy-mm-dd*). Then we can create another DataFrame that only contains accidents for 2000:

```python
# Convert date to Pandas date/time
london_data_2000 = london_data[
    (pd.to_datetime(london_data['Date'], coerce=True) >
        pd.to_datetime('2000-01-01', coerce=True)) &
    (pd.to_datetime(london_data['Date'], coerce=True) <
        pd.to_datetime('2000-12-31', coerce=True))
]
print("Accidents in London in the year 2000 on a Sunday: {0}".format(
    len(london_data_2000)))
```

When ran, you should see:

```sh
Accidents which happened on a Sunday: 693847
Accidents which happened on a Sunday involving > 20 cars: 10
Accidents which happened on a Sunday involving > 20 cars in the rain: 4

Accidents in London from 1979-2004 on a Sunday: 114624
Accidents in London in the year 2000 on a Sunday: 3889
```

So, this is a bit confusing at first. Normally, to filter an array you would just use a `for` loop with a conditional:

```python
for data in array:
    if data > X and data < X:
        # do something
```

However, you really shouldn't define your own loop since many high-performance libraries, like Pandas, have helper functions in place. In this case, the above code loops over all the elements and filters out data outside the set dates, and then returns the data points that do fall within the dates.

Nice!

## Converting

Chances are that, while using Pandas, everyone else in your organization is stuck with Excel. Want to share the DataFrame with those using Excel?

First, we need to do some cleanup. Remember the byte order mark we saw earlier? That causes problems when writing this data to an Excel file - Pandas throws a *UnicodeDecodeError*. Why? Because the rest of the text is decoded as ASCII, but the hexadecimal values can't be represented in ASCII.

We could write everything as Unicode, but remember this byte order mark is an unnecessary (to us) extra we don't want or need. So we will get rid of it by renaming the column header:

```python
london_data_2000.rename(
    columns={'\xef\xbb\xbfAccident_Index': 'Accident_Index'}, inplace=True)
```

This is the way to rename a column in Pandas; a bit complicated, to be honest. `inplace = True` is needed because we want to modify the existing structure, and not create a copy, which is what Pandas does by default.

Now we can save the data to Excel:

```python
# Save to Excel
writer = pd.ExcelWriter(
    'London_Sundays_2000.xlsx', engine='xlsxwriter')
london_data_2000.to_excel(writer, 'Sheet1')
writer.save()
```

Make sure to install [XlsxWriter](https://xlsxwriter.readthedocs.org/) before running:

```sh
pip install XlsxWriter==0.7.3
```

If all went well, this should have created a file called *London_Sundays_2000.xlsx*, and then saved our data to *Sheet1*. Open this file up in Excel or LibreOffice, and confirm that the data is correct.

## Conclusion

So, what did we accomplish? Well, we took a very large file that Excel could not open and utilized Pandas to-

1. Open the file.
1. Perform SQL-like queries against the data.
1. Create a new XLSX file with a subset of the original data.

Keep in mind that even though this file is nearly 800MB, in the age of big data, it's still quite small. What if you wanted to open a 4GB file? Even if you have 8GB or more of RAM, that might still not be possible since much of your RAM is reserved for the OS and other system processes. In fact, my laptop froze a few times when first reading in the 800MB file. If I opened a 4GB file, it would have a heart attack.

So how do we proceed?

The trick is not to open the whole file in one go. That's what we'll look at in the next blog post. Until then, analyze your own data. Leave questions/comments below. Grab the code from the [repo](https://github.com/shantnu/PandasLargeFiles).
