# Twitter Sentiment - Python, Docker, Elasticsearch, Kibana

![kibana-pie-chart](https://raw.githubusercontent.com/realpython/twitter-sentiment-elasticsearch/master/images/overall-pie-chart.png)

In this example, we'll connect to the Twitter Streaming API, gather tweets (based on a keyword), calculate the sentiment of each tweet, and build a real-time dashboard using the Elasticsearch DB and Kibana to visualize the results.

> Tools: [Docker](https://www.docker.com/) v1.3.0, [boot2docker](http://boot2docker.io/) v1.3.0, [Tweepy](http://www.tweepy.org/) v2.3.0, [TextBlob](http://textblob.readthedocs.org/en/dev/) v0.9.0, [Elasticsearch](http://www.elasticsearch.org/) v1.3.5, [Kibana](http://www.elasticsearch.org/overview/kibana/) v3.1.2

## Docker Environment

Follow the [official Docker documentation](https://docs.docker.com/installation/mac/) to install both Docker and boot2docker. Then with boot2docker up and running, run `docker version` to test the Docker installation. Create a directory to house your project, grab the Dockerfile from the [repository](https://github.com/realpython/twitter-sentiment-elasticsearch), and build the image:

```sh
$ docker build -rm -t=elasticsearch-kibana .
```

Once built, run the container:

```sh
$ docker run -d -p 8000:8000 -p 9200:9200 elasticsearch-kibana
```

Finally, run the next two commands in new terminal windows to map the IP address/port combo used by the boot2docker VM to your localhost:

```sh
$ boot2docker ssh -L8000:localhost:8000
$ boot2docker ssh -L9200:localhost:9200
```

Now you can access Elasticsearch at [http://localhost:9200](http://localhost:9200) and Kibana at [http://localhost:8000](http://localhost:8000).

## Twitter Streaming API

In order to access the [Twitter Streaming API](https://dev.twitter.com/streaming/overview), you need to register an application at [http://apps.twitter.com](http://apps.twitter.com). Once created, you should be redirected to your app's page, where you can get the consumer key and consumer secret and create an access token under the "Keys and Access Tokens" tab. Add these to a new file called *config.py*:

```python
consumer_key = "add_your_consumer_key"
consumer_secret = "add_your_consumer_secret"
access_token = "add_your_access_token"
access_token_secret = "add_your_access_token_secret"
```

> Since this file contains sensitive information do **not** add it to your Git repository.

According to the Twitter Streaming [documentation](https://dev.twitter.com/streaming/overview/connecting), "establishing a connection to the streaming APIs means making a very long lived HTTP request, and parsing the response incrementally. Conceptually, you can think of it as downloading an infinitely long file over HTTP."

So, you make a request, filter it by a specific keyword, user, and/or geographic area and then leave the connection open, collecting as many tweets as possible.

This sounds complicated, but [Tweepy](http://www.tweepy.org/) makes it easy.

## Tweepy Listener

Tweepy uses a "listener" to not only grab the streaming tweets, but filter them as well.

### The code

Save the following code as *sentiment.py*:

```python
import json
from tweepy.streaming import StreamListener
from tweepy import OAuthHandler
from tweepy import Stream
from textblob import TextBlob
from elasticsearch import Elasticsearch

# import twitter keys and tokens
from config import *

# create instance of elasticsearch
es = Elasticsearch()


class TweetStreamListener(StreamListener):

    # on success
    def on_data(self, data):

        # decode json
        dict_data = json.loads(data)

        # pass tweet into TextBlob
        tweet = TextBlob(dict_data["text"])

        # output sentiment polarity
        print tweet.sentiment.polarity

        # determine if sentiment is positive, negative, or neutral
        if tweet.sentiment.polarity < 0:
            sentiment = "negative"
        elif tweet.sentiment.polarity == 0:
            sentiment = "neutral"
        else:
            sentiment = "positive"

        # output sentiment
        print sentiment

        # add text and sentiment info to elasticsearch
        es.index(index="sentiment",
                 doc_type="test-type",
                 body={"author": dict_data["user"]["screen_name"],
                       "date": dict_data["created_at"],
                       "message": dict_data["text"],
                       "polarity": tweet.sentiment.polarity,
                       "subjectivity": tweet.sentiment.subjectivity,
                       "sentiment": sentiment})
        return True

    # on failure
    def on_error(self, status):
        print status

if __name__ == '__main__':

    # create instance of the tweepy tweet stream listener
    listener = TweetStreamListener()

    # set twitter keys/tokens
    auth = OAuthHandler(consumer_key, consumer_secret)
    auth.set_access_token(access_token, access_token_secret)

    # create instance of the tweepy stream
    stream = Stream(auth, listener)

    # search twitter for "congress" keyword
    stream.filter(track=['congress'])
```

**What's happening?**:

1. We connect to the Twitter Streaming API;
1. Filter the data by the keyword `"congress"`;
1. Decode the results (the tweets);
1. Calculate sentiment analysis via [TextBlob](http://textblob.readthedocs.org/en/dev/);
1. Determine if the overall sentiment is positive, negative, or neutral; and,
1. Finally the relevant sentiment and tweet data is added to the Elasticsearch DB.

Follow the inline comments for further details.

### TextBlob sentiment basics

To calculate the overall sentiment, we look at the [polarity](http://textblob.readthedocs.org/en/latest/_modules/textblob/blob.html#BaseBlob.polarity) score:

1. Positive - *from .01 to 1*
1. Neutral - *0*
1. Negative - *from -.01 to -1*

> Refer to the [official documentation](http://textblob.readthedocs.org/en/dev/) for more information on how TextBlob calculates sentiment.

## Elasticsearch Analysis

Over a two hour period, as I wrote this blog post, I pulled over 9,500 tweets with the keyword "congress". At this point go ahead and perform a search of your own, on a subject of interest to you. Once you have a sizable number of tweets, stop the script. Now you can perform some quick searches/analysis...

Using the index (`"sentiment"`) from the *sentiment.py* script, you can use the [Elasticsearch search API](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-search.html) to gather some basic insights.

For example:

- Full text search for "obama": [http://localhost:9200/sentiment/_search?q=obama](http://localhost:9200/sentiment/_search?q=obama)
- Author/Twitter username search: [http://localhost:9200/sentiment/_search?q=author:allvoices](http://localhost:9200/sentiment/_search?q=author:allvoices)
- Sentiment search: [http://localhost:9200/sentiment/_search?q=sentiment:positive](http://localhost:9200/sentiment/_search?q=sentiment:positive)
- Sentiment and "obama" search: [http://localhost:9200/sentiment/_search?q=sentiment:positive&message=obama](http://localhost:9200/sentiment/_search?q=sentiment:positive&message=obama)

There's much, *much* more you can do with Elasticsearch besides just searching and filtering results. Check out the [Analyze API](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/analysis-intro.html) as well as the [Elasticsearch - The Definitive Guide](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/index.html) for more ideas on how to analyze and model your data.

## Kibana Visualizer

[Kibana](http://www.elasticsearch.org/overview/kibana/) lets "you see and interact with your data" in realtime, as you're gathering data. Since it's written in JavaScript, you access it directly from your browser. Check out the basics from the [official introduction](http://www.elasticsearch.org/guide/en/kibana/current/_introduction.html) to quickly get started.

The pie chart at the top of this post came direct from Kibana, which shows the proportion of each sentiment - positive, neutral, and negative - to the whole from the tweets I pulled. Here's a few more graphs from Kibana...

*All tweets filtered by the word "obama"*

![kibana-obama-pie-chart](https://raw.githubusercontent.com/realpython/twitter-sentiment-elasticsearch/master/images/obama-pie-chart.png)

*Top twitter users by tweet count*

![top-authors](https://raw.githubusercontent.com/realpython/twitter-sentiment-elasticsearch/master/images/top-authors.png)

> Notice how the top author as 76 tweets. That's definitely worthy of a deeper look since that's a lot of tweets in a two hour period. Anyway, that author basically tweeted the same tweet 76 times - so you would want to filter out 75 of these since the overall results are currently skewed.

Aside for these charts, it's worth visualizing sentiment by location. Try this on your own. You'll have to alter the data you are grabbing from each tweet. You may also want to try visualizing the data with a histogram as well.

Finally -

1. Grab the code from the [repository](https://github.com/realpython/twitter-sentiment-elasticsearch).
1. Leave comments/questions below.

**Cheers!**
