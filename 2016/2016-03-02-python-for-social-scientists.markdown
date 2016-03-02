# Python for Social Scientists

<div class="center-text">
  <img class="no-border" src="/images/blog_images/python-social-science.png" style="max-width: 100%;" alt="python for social scientists">
</div>

<br>

*This is a guest blog post by [Nick Eubank](http://www.nickeubank.com/)​, a Ph.D. Candidate in Political Economy at the Stanford Graduate School of Business*

<hr>

Python is an increasingly popular tool for data analysis in the social scientists. Empowered by a number of libraries that have reached maturity, R and Stata users are increasingly moving to Python in order to take advantage of the beauty, flexibility, and performance of Python without sacrificing the functionality these older programs have accumulated over the years.

But while Python has much to offer, existing Python resources are not always well-suited to the needs of social scientists. With that in mind, I've recently created a new resource -- [www.pythonforsocialscientists.org](http://www.pythonforsocialscientists.org) (PSS) -- tailored specifically to the goals and desires of the social scientist python user.

The site is **not** a new set of tutorials, however -- there are *more* than enough Python tutorials in the world. Rather, the aim of the site is to curate and annotate existing resources, and to provide users guidance on what topics to focus on and which to skip.

## Why a Site for Social Scientists?

Social scientists – and indeed, most data scientists – spend most of their time trying to wrestle individual, idiosyncratic datasets into the shape needed to run statistical analyses. This makes the way most social scientists use Python fundamentally different from how it is used by most software developers. Social scientists are primarily interested in writing relatively simple programs (scripts) that execute a series of commands (recoding variables, merging datasets, parsing text documents, etc.) to wrangle their data into a form they can analyze. And because they are usually writing their scripts for a specific, idiosyncratic application and set of data, they are generally not focused on writing code with lots of abstractions.

Social scientists, in other words, tend to be primarily interested in learning to *use existing tools* effectively, not develop new ones.

Because of this, social scientists learning Python tend to have different priorities in terms of skill development than software developers. Yet most tutorials online were written for developers or computer science students, so one of the aims of PSS is to provide social scientists with some guidance on the skills they should prioritize in their early training. In particular, [PSS suggests](http://www.pythonforsocialscientists.org/2_basic_python.html):

**Need immediately:**

- Data types: integers, floats, strings, booleans, lists, dictionaries, and sets (tuples are kinda optional)
- Defining functions
- Writing loops
- Understanding mutable versus immutable data types
- Methods for manipulating strings
- Importing third party modules
- Reading and interpreting errors

**Things you'll want to know at some point, but not necessary immediately:**

- Advanced debugging utilities (like pdb)
- File input / output (most libraries you'll use have tools to simplify this for you)

**Don't need:**

* Defining or writing classes
* Understanding Exceptions

## Pandas

Today, most empirical social science remains organized around tabular data, meaning data that is presented with a different variable in each column and a different observation in each row. As a result, many social scientists using Python are a little confused when they don't find a tabular data structure covered in their intro to Python tutorial. To address this confusion, PSS does its best to introduce users to the [pandas](http://pandas.pydata.org/) library as fast as possible, providing [links to tutorials and a few tips on gotchas to watch out for](http://www.pythonforsocialscientists.org/3_pandas.html).

The pandas library replicates much of the functionality that social scientists are used to finding in Stata or R -- data can be represented in a tabular format, column variables can be easily labeled, and columns of different types (like floats and strings) can be combined in the same dataset.

pandas is also the gateway to many other tools social scientists are likely to use, like graphing libraries ([seaborn](http://stanford.edu/~mwaskom/software/seaborn/) and [ggplot2](http://ggplot.yhathq.com/)) and the [statsmodels](http://statsmodels.sourceforge.net/) econometrics library.

## Other Libraries by Research Area

While all social scientists who wish to work with Python will need to understand the core language and most will want to be familiar with `pandas`, the Python eco-system is full of application-specific libraries that will only be of use to a subset of users. With that in mind, PSS provides an overview of libraries to help researchers working in different topic areas, along with links to materials on optimal use, and guidance on relevant considerations:

- [Network Analysis](http://www.pythonforsocialscientists.org/t_igraph.html): iGraph
- [Text Analysis](http://www.pythonforsocialscientists.org/t_text_analysis.html): NLTK, and if needed coreNLP
- [Econometrics](http://www.pythonforsocialscientists.org/t_statsmodels.html): statsmodels
- [Graphing](http://www.pythonforsocialscientists.org/t_seaborn.html): ggplot and seaborn
- [Big Data](http://www.pythonforsocialscientists.org/t_big_data.html): dask and pyspark
- [Geo-Spatial Analysis](http://www.pythonforsocialscientists.org/t_gis.html): arcpy or geopandas
- [Making code faster](http://www.pythonforsocialscientists.org/t_super_fast.html): `%prun` in iPython (for profiling) and numba (for JIT compilation)

## Want to Get Involved?

This site is young, so we are anxious for as much input as possible on content and design. If you have experience in this area you want to share *please* drop me an [email](http://www.nickeubank.com/cv-and-contact-info/) or [comment](https://github.com/nickeubank/pythonfordataanalysis) on Github.