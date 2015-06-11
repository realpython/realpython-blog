# Web Scraping and Crawling with Scrapy and MongoDB

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/scraping-with-scrapy-and-mongo/scrapy_part2.png" style="max-width: 100%;" alt="scrapy">
</div>

<br>

[Last time](https://realpython.com/blog/python/web-scraping-with-scrapy-and-mongodb) we implemented a basic web scraper that downloaded the latest questions from StackOverflow and stored the results in MongoDB. **In this article we'll extend our scraper so that it crawls through the pagination links at the bottom of each page and scrapes the questions (question title and URL) from each page.**

> Before you start any scraping job, review the site's terms of use policy and respect the robots.txt file. Also, adhere to ethical scraping practices by not flooding a site with numerous requests over a short span of time. **Treat any site you scrape as if it were your own.**

<hr>

This is a collaboration piece between the folks at Real Python and György - a Python enthusiast and software developer, currently working at a big data company and seeking a new job at the same time. You can ask him questions on twitter - [@kissgyorgy](https://twitter.com/kissgyorgy).

## Getting Started

There are two possible ways to continue from where we left off.

The first is to extend our existing Spider by extracting every next page link from the response in the `parse_item` method with an xpath expression and just `yield` a `Request` object with a callback to the same `parse_item` method. This way scrapy will automatically make a new request to the link we specify. You can find more information on this method in the [Scrapy documentation](http://doc.scrapy.org/en/latest/topics/spiders.html#spiders).

The other, much simpler option is to utilize a different type of spider - the `CrawlSpider` ([link](http://doc.scrapy.org/en/latest/topics/spiders.html#crawlspider)). It's an extended version of the basic `Spider`, designed exactly for our use case.

## The CrawlSpider

We'll be using the same Scrapy project from the last tutorial, so grab the code from the [repo](https://github.com/realpython/stack-spider/releases/tag/part1) if you need it.

### Create the Boilerplate

Within the "stack" directory, start by [generating](http://doc.scrapy.org/en/latest/topics/commands.html#std:command-genspider) the spider boilerplate from the `crawl` template:

```sh
$ scrapy genspider stack_crawler  stackoverflow.com -t crawl
Created spider 'stack_crawler' using template 'crawl' in module:
  stack.spiders.stack_crawler
```

The Scrapy project should now look like this:

```sh
├── scrapy.cfg
└── stack
    ├── __init__.py
    ├── items.py
    ├── pipelines.py
    ├── settings.py
    └── spiders
        ├── __init__.py
        ├── stack_crawler.py
        └── stack_spider.py
```

And the *stack_crawler.py* file should look like this:

```python
# -*- coding: utf-8 -*-
import scrapy
from scrapy.contrib.linkextractors import LinkExtractor
from scrapy.contrib.spiders import CrawlSpider, Rule

from stack.items import StackItem


class StackCrawlerSpider(CrawlSpider):
    name = 'stack_crawler'
    allowed_domains = ['stackoverflow.com']
    start_urls = ['http://www.stackoverflow.com/']

    rules = (
        Rule(LinkExtractor(allow=r'Items/'), callback='parse_item', follow=True),
    )

    def parse_item(self, response):
        i = StackItem()
        #i['domain_id'] = response.xpath('//input[@id="sid"]/@value').extract()
        #i['name'] = response.xpath('//div[@id="name"]').extract()
        #i['description'] = response.xpath('//div[@id="description"]').extract()
        return i
```

We just need to make a few updates to this boilerplate...

### Update the `start_urls` list

First, add the first page of questions to the `start_urls` list:

```python
start_urls = [
    'http://stackoverflow.com/questions?pagesize=50&sort=newest'
]
```

### Update the `rules` list

Next, we need to tell the spider where it can find the next page links by adding a regular expression to the `rules` attribute:

```python
rules = [
    Rule(LinkExtractor(allow=r'questions\?page=[0-9]&sort=newest'),
         callback='parse_item', follow=True)
]
```

Scrapy will now automatically request new pages based on those links and pass the response to the `parse_item` method to extract the questions and titles.

> If you're paying close attention, this regex limits the crawling to the first 9 pages since for this demo we do not want to scrape all 176,234 pages.

### Update the `parse_item` method

Now we just need to write how to parse the pages with xpath, which we already did in the last tutorial - so just copy it over:

```python
def parse_item(self, response):
    questions = response.xpath('//div[@class="summary"]/h3')

    for question in questions:
        item = StackItem()
        item['url'] = question.xpath(
            'a[@class="question-hyperlink"]/@href').extract()[0]
        item['title'] = question.xpath(
            'a[@class="question-hyperlink"]/text()').extract()[0]
        yield item
```

That's it for the spider, but do **not** start it just yet.

### Add a Download Delay

We need to be nice to StackOverflow (and any site, for that matter) by setting a download delay in *settings.py*:

```python
DOWNLOAD_DELAY = 5
```

This tells Scrapy to wait at least 5 seconds between every new request it makes. You're essentially rate limiting yourself. If you do not do this, StackOverflow will rate limit you; and if you continue to scrape the site without imposing a rate limit, your IP address could be banned. So, be nice - *Treat any site you scrape as if it were your own.*

Now there is only one thing left to think about - storing the Data.

## MongoDB

Last time we only downloaded 50 questions, but since we are grabbing a lot more data this time, we want to avoid adding duplicate questions to the database. We can do that by using an MongoDB [upsert](http://docs.mongodb.org/manual/reference/method/db.collection.update/#upsert-option), which means we update the question title if it is already in the database and insert otherwise.

Modify the `MongoDBPipeline` we defined earlier:

```python
class MongoDBPipeline(object):

    def __init__(self):
        connection = pymongo.Connection(
            settings['MONGODB_SERVER'],
            settings['MONGODB_PORT']
        )
        db = connection[settings['MONGODB_DB']]
        self.collection = db[settings['MONGODB_COLLECTION']]

    def process_item(self, item, spider):
        for data in item:
            if not data:
                raise DropItem("Missing data!")
        self.collection.update({'url': item['url']}, dict(item), upsert=True)
        log.msg("Question added to MongoDB database!",
                level=log.DEBUG, spider=spider)
        return item
```

> For simplicity, we did not optimize the query and did not deal with indexes since this is not a production environment.

## Test

Start the spider!

```sh
$ scrapy crawl stack_crawler
```

Now sit back and watch your database fill with data!

## Conclusion

You can download the entire source code from the [Github repository](https://github.com/realpython/stack-spider/releases/tag/part2). Comment below with questions. Cheers!

> Looking for more web scraping? Be sure to check out the [Real Python courses](https://realpython.com/courses). Looking to hire a professional web scraper? Check out [GoScrape](http://www.goscrape.com/).