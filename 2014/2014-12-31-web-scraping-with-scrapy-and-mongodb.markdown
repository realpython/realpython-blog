# Web Scraping with Scrapy and MongoDB

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/scraping-with-scrapy-and-mongo/scrapy.png" style="max-width: 100%;" alt="scrapy">
</div>

<br>

**In this article we're going to build a scraper for an *actual* freelance gig where the client wants a Python program to scrape data from [Stack Overflow](http://stackoverflow.com/questions?pagesize=50&sort=newest) to grab new questions (question title and URL). Scraped data should then be stored in [MongoDB](http://www.mongodb.org/).** It's worth noting that Stack Overflow has an [API](https://api.stackexchange.com/), which can be used to access the *exact* same data. However, the client wanted a scraper, so a scraper is what he got.

<br>

**Updates:**

1. 01/03/2014 - Refactored the spider. Thanks, [@kissgyorgy](https://twitter.com/kissgyorgy).
2. 02/18/2015 - Added [Part 2](https://realpython.com/blog/python/web-scraping-and-crawling-with-scrapy-and-mongodb/).

<br>

> As always, be sure to review the site's terms of use/service and respect the *robots.txt* file before starting any scraping job. Make sure to adhere to ethical scraping practices by not flooding the site with numerous requests over a short span of time. *Treat any site you scrape as if it were your own*.

## Installation

We need the [Scrapy](http://doc.scrapy.org/en/latest/index.html) library (v0.24.4) along with [PyMongo](http://api.mongodb.org/python/current/) (v2.7.2) for storing the data in MongoDB. You need to install [MongoDB](http://docs.mongodb.org/manual/installation/) as well (not covered).

### Scrapy

If you're running OSX or a flavor of Linux, install Scrapy with pip (with your virtualenv activated):

```sh
$ pip install Scrapy
```

If you are on Windows machine, you will need to manually install a number of dependencies. Please refer to the [official documentation](http://doc.scrapy.org/en/latest/intro/install.html) for detailed instructions as well as [this Youtube video](https://www.youtube.com/watch?v=eEK2kmmvIdw) that I created.

Once Scrapy is setup, verify your installation by running this command in the Python shell:

```sh
>>> import scrapy
>>>
```

If you don't get an error then you are good to go!

### PyMongo

Next, install PyMongo with pip:

```sh
$ pip install pymongo
```

Now we can start building the crawler.

## Scrapy Project

Let's start a new Scrapy project:

```sh
$ scrapy startproject stack
```

This creates a number of files and folders that includes a basic boilerplate for you to get started quickly:

```
├── scrapy.cfg
└── stack
    ├── __init__.py
    ├── items.py
    ├── pipelines.py
    ├── settings.py
    └── spiders
        └── __init__.py
```

### Specify Data

The *items.py* file is used to define storage "containers" for the data that we plan to scrape.

The `StackItem()` class inherits from `Item` ([docs](http://doc.scrapy.org/en/latest/topics/items.html)), which basically has a number of pre-defined objects that Scrapy has already built for us:

```python
import scrapy


class StackItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    pass
```

Let’s add some items that we actually want to collect. For each question the client needs the title and URL. So, update *items.py* like so:

```python
from scrapy.item import Item, Field


class StackItem(Item):
    title = Field()
    url = Field()
```

### Create the Spider

Create a file called *stack_spider.py* in the "spiders" directory. This is where the magic happens – e.g., where we'll tell Scrapy how to find the *exact* data we’re looking for. As you can imagine, this is *specific* to each individual web page.

Start by defining a class that inherits from Scrapy’s `Spider` and then adding attributes as needed:

```python
from scrapy import Spider


class StackSpider(Spider):
    name = "stack"
    allowed_domains = ["stackoverflow.com"]
    start_urls = [
        "http://stackoverflow.com/questions?pagesize=50&sort=newest",
    ]
```

The first few variables are self-explanatory ([docs](http://doc.scrapy.org/en/latest/topics/spiders.html#spider)):

- `name` defines the name of the Spider.
- `allowed_domains` contains the base-URLs for the allowed domains for the spider to crawl.
- `start_urls` is a list of URLs for the spider to start crawling from. All subsequent URLs will start from the data that the spider downloads from the URLS in `start_urls`.

### XPath Selectors

Next, Scrapy uses XPath selectors to extract data from a website. In other words, we can select certain parts of the HTML data based on a given XPath. As stated in Scrapy's [documentation](http://doc.scrapy.org/en/latest/topics/selectors.html), "XPath is a language for selecting nodes in XML documents, which can also be used with HTML."

You can easily find a specific Xpath using Chrome's Developer Tools. Simply inspect a specific HTML element, copy the XPath, and then tweak (as needed):

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/scraping-with-scrapy-and-mongo/chrome_copy_xpath.png" style="max-width: 100%;" alt="copy xpath in chrome developer tools">
</div>

<br>

Developer Tools also gives you the ability to test XPath selectors in the JavaScript Console by using `$x` - i.e., `$x("//img")`:

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/scraping-with-scrapy-and-mongo/chrome_text_xpath.png" style="max-width: 100%;" alt="test xpath in chrome developer tools">
</div>

<br>

Again, we basically tell Scrapy where to start looking for information based on a defined XPath. Let’s navigate to the [Stack Overflow](http://stackoverflow.com/questions?pagesize=50&sort=newest) site in Chrome and find the XPath selectors.

Right click on the first question and select "Inspect Element":

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/scraping-with-scrapy-and-mongo/stack_source_inspect_element.png" style="max-width: 100%;" alt="inspect element in chrome developer tools">
</div>

<br>


Now grab the XPath for the `<div class="summary">`, `//*[@id="question-summary-27624141"]/div[2]`, and then test it out in the JavaScript Console:

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/scraping-with-scrapy-and-mongo/stack_source_test_xpath.png" style="max-width: 100%;" alt="test xpath in chrome developer tools">
</div>

<br>

As you can tell, it just selects that *one* question. So we need to alter the XPath to grab *all* questions. Any ideas? It's simple: `//div[@class="summary"]/h3`. What does this mean? Essentially, this XPath states: *Grab all `<h3>` elements that are children of a `<div>` that has a class of `summary`*. Test this XPath out in the JavaScript Console.

> Notice how we are not using the actual XPath output from Chrome Developer Tools. In most cases, the output is just a helpful aside, which generally points you in the right direction for finding the working XPath.

Now let's update the *stack_spider.py* script:

```python
from scrapy import Spider
from scrapy.selector import Selector


class StackSpider(Spider):
    name = "stack"
    allowed_domains = ["stackoverflow.com"]
    start_urls = [
        "http://stackoverflow.com/questions?pagesize=50&sort=newest",
    ]

    def parse(self, response):
        questions = Selector(response).xpath('//div[@class="summary"]/h3')
```

### Extract the Data

We still need to parse and scrape the data we want, which falls within `<div class="summary"><h3>`. Again, update *stack_spider.py* like so:

```python
from scrapy import Spider
from scrapy.selector import Selector

from stack.items import StackItem


class StackSpider(Spider):
    name = "stack"
    allowed_domains = ["stackoverflow.com"]
    start_urls = [
        "http://stackoverflow.com/questions?pagesize=50&sort=newest",
    ]

    def parse(self, response):
        questions = Selector(response).xpath('//div[@class="summary"]/h3')

        for question in questions:
            item = StackItem()
            item['title'] = question.xpath(
                'a[@class="question-hyperlink"]/text()').extract()[0]
            item['url'] = question.xpath(
                'a[@class="question-hyperlink"]/@href').extract()[0]
            yield item
````

We are iterating through the `questions` and assigning the `title` and `url` values from the scraped data. Be sure to test out the XPath selectors in the JavaScript Console within Chrome Developer Tools - e.g., `$x('//div[@class="summary"]/h3/a[@class="question-hyperlink"]/text()')` and `$x('//div[@class="summary"]/h3/a[@class="question-hyperlink"]/@href')`.

## Test

Ready for the first test? Simply run the following command within the "stack" directory:

```sh
$ scrapy crawl stack
```

Along with the Scrapy stack trace, you should see 50 question titles and URLs outputted. You can render the output to a JSON file with this little command:

```sh
$ scrapy crawl stack -o items.json -t json
```

We’ve now implemented our Spider based on our data that we are seeking. Now we need to store the scraped data within MongoDB.

## Store the Data in MongoDB

Each time an item is returned, we want to validate the data and then add it to a Mongo collection.

The initial step is to create the database that we plan to use to save all of our crawled data. Open *settings.py* and specify the [pipeline](http://doc.scrapy.org/en/latest/topics/item-pipeline.html) and add the database settings:

```python
ITEM_PIPELINES = ['stack.pipelines.MongoDBPipeline', ]

MONGODB_SERVER = "localhost"
MONGODB_PORT = 27017
MONGODB_DB = "stackoverflow"
MONGODB_COLLECTION = "questions"
```

### Pipeline Management

We’ve setup our spider to crawl and parse the HTML, and we’ve set up our database settings. Now we have to connect the two together through a pipeline in *pipelines.py*.

**Connect to Database**

First, let's define a method to actually connect to the database:

```python
import pymongo

from scrapy.conf import settings


class MongoDBPipeline(object):

    def __init__(self):
        connection = pymongo.Connection(
            settings['MONGODB_SERVER'],
            settings['MONGODB_PORT']
        )
        db = connection[settings['MONGODB_DB']]
        self.collection = db[settings['MONGODB_COLLECTION']]
```

Here, we create a class, `MongoDBPipeline()`, and we have a constructor function to initialize the class by defining the Mongo settings and then connecting to the database.

**Process the Data**

Next, we need to define a method to process the parsed data:

```python
import pymongo

from scrapy.conf import settings
from scrapy.exceptions import DropItem
from scrapy import log


class MongoDBPipeline(object):

    def __init__(self):
        connection = pymongo.Connection(
            settings['MONGODB_SERVER'],
            settings['MONGODB_PORT']
        )
        db = connection[settings['MONGODB_DB']]
        self.collection = db[settings['MONGODB_COLLECTION']]

    def process_item(self, item, spider):
        valid = True
        for data in item:
            if not data:
                valid = False
                raise DropItem("Missing {0}!".format(data))
        if valid:
            self.collection.insert(dict(item))
            log.msg("Question added to MongoDB database!",
                    level=log.DEBUG, spider=spider)
        return item
```

We establish a connection to the database, unpack the data, and then save it to the database. Now we can test again!

## Test

Again, run the following command within the "stack" directory:

```sh
$ scrapy crawl stack
```

Hooray! We have successfully stored our crawled data into the database:

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/scraping-with-scrapy-and-mongo/robomongo.png" style="max-width: 100%;" alt="robomongo">
</div>

<br>

## Conclusion

This is a pretty simple example of using Scrapy to crawl and scrape a web page. The actual freelance project required the script to follow the pagination links and scrape each page using the `CrawlSpider` ([docs](http://doc.scrapy.org/en/latest/topics/spiders.html#crawlspider)), which is super easy to implement. Try implementing this on your own, and leave a comment below with the link to the Github repository for a quick code review. Need help? Start with [this script](https://github.com/realpython/stack-spider/blob/part1/stack/stack/spiders/stack_crawl.py), which is nearly complete. **Then view [Part 2](https://realpython.com/blog/python/web-scraping-and-crawling-with-scrapy-and-mongodb/) for the full solution!**

You can download the entire source code from the [Github repository](https://github.com/realpython/stack-spider/releases/tag/part1). Comment below with questions. Thanks for Reading!

Happy New Year!

:)

> Looking for more web scraping? Be sure to check out the [Real Python courses](https://realpython.com/courses). Looking to hire a professional web scraper? Check out [GoScrape](http://www.goscrape.com/).

