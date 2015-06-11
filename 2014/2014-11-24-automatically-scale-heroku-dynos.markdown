# Automatically Scale Heroku Dynos

This post details how to write a script to automate the scaling of Heroku dynos based on the time of day. We'll also look at how to add a fail-safe so that our application automatically scales if it's either down completely or experiencing a heavy load.

Let's take the following assumptions into account:


<table style="font-size: 16px;border-spacing: 20px 0px;border-collapse: separate;">
<thead>
<tr>
<th>Use</th>
<th>Time</th>
<th>Web Dynos</th>
</tr>
</thead>
<tbody>
<tr>
<td>Heavy</td>
<td>7am to 10pm</td>
<td>3</td>
</tr>
<tr>
<td>Medium</td>
<td>10pm to 3am</td>
<td>2</td>
</tr>
<tr>
<td>Low</td>
<td>3am to 7am</td>
<td>1</td>
</tr>
</tbody>
</table>

<br>

So, we need to scale out at 7am, and then scale in at 10pm, and then in again at 3am. Repeat. For simplicity's sake, the majority of our traffic comes from only a few times zones, which we've taken into account. We'll also base everything off of UTC since that's the Heroku default time zone.

> If this is for your own application, make sure you leave yourself some wiggle room before scaling in or out. You may also want to look at holidays and weekends as well. Do the math. Figure out your cost savings.

With that, let's add some tasks...

## APScheduler

For this tutorial, let's use the [Advanced Python Scheduler](http://apscheduler.readthedocs.org/en/3.0/) (APScheduler), since it's easy to use and meant to be ran along side other processes, along with the [Heroku Platform API](https://devcenter.heroku.com/categories/platform-api).

Start by installing the APScheduler:

```sh
$ pip install apscheduler==3.0.1
```

Now, create a new file called *autoscale.py* and add the following code:

```python
from apscheduler.schedulers.blocking import BlockingScheduler

sched = BlockingScheduler()


@sched.scheduled_job('interval', minutes=1)
def job():
    print 'This job is run every minute.'


sched.start()
```

Yes, this just runs a task every minute. Before moving on, let's test this out to ensure it works. Add this process to your *Procfile*. Assuming you already have a web process defined, the file should now look something like this:

```
web: gunicorn hello:app
clock: python autoscale.py
```

Commit your changes, and then push them up to Heroku.

Run the following command to scale up the clock process:

```sh
$ heroku ps:scale clock=1
```

Then open the Heroku logs to see the process in action:

```
$ heroku logs --tail
2014-11-04T14:59:22.418496+00:00 heroku[api]: Scale to clock=1, web=1 by michael@realpython.com
2014-11-04T15:00:20.357505+00:00 heroku[router]: at=info method=GET path="/" host=autoscale.herokuapp.com request_id=7537ce4a-e802-4020-9b1b-10e754263957 fwd="54.160.152.14" dyno=web.1 connect=1ms service=3ms status=200 bytes=172
2014-11-04T15:00:27.620383+00:00 app[clock.1]: This job is run every minute.
2014-11-04T15:01:27.621151+00:00 app[clock.1]: This job is run every minute.
2014-11-04T15:02:27.620780+00:00 app[clock.1]: This job is run every minute.
2014-11-04T15:03:27.621276+00:00 app[clock.1]: This job is run every minute.
```

Simple, right?

Moving on, let's add the scaling tasks to the script...

## Autoscale

Start by grabbing the API key from the Heroku [account](https://dashboard.heroku.com/account) page, and add it to a new file called *config.py*. Along with the key, also enter the name of the app you are interested in monitoring and the process.

```python
APP = "<add your app name>"
KEY = "<add your API key>"
PROCESS = "web"
```

Next, add the following function to *autoscale.py*:

```python
def scale(size):
    payload = {'quantity': size}
    json_payload = json.dumps(payload)
    url = "https://api.heroku.com/apps/" + APP + "/formation/" + PROCESS
    try:
        result = requests.patch(url, headers=HEADERS, data=json_payload)
    except:
        print "test!"
        return None
    if result.status_code == 200:
        return "Success!"
    else:
        return "Failure"
```

Update the imports and add the following configuration as well:

```python
import requests
import base64
import json

from apscheduler.schedulers.blocking import BlockingScheduler

from config import APP, KEY, PROCESS


# Generate Base64 encoded API Key
BASEKEY = base64.b64encode(":" + KEY)
# Create headers for API call
HEADERS = {
    "Accept": "application/vnd.heroku+json; version=3",
    "Authorization": BASEKEY
}
```

Here we handle the basic authorization by passing the API Key into the header, and then using the `requests` library, we call the API. For more information on this, check out the official Heroku [documentation](https://devcenter.heroku.com/articles/platform-api-quickstart#authentication). If all went well, this will scale our app appropriately.

Want to test this out? Update the `job()` function like so:

```python
@sched.scheduled_job('interval', minutes=1)
def job():
    print 'Scaling ...'
    print scale(0)
```

Commit your code, and then push to Heroku. Now if you run `heroku logs --tail`, you should see:

```sh
$ heroku logs --tail
2014-11-04T20:48:12.832034+00:00 app[clock.1]: Scaling ...
2014-11-04T20:48:12.910837+00:00 heroku[api]: Scale to clock=1, web=0 by hermanmu@gmail.com
2014-11-04T20:48:12.929993+00:00 app[clock.1]: Success!
2014-11-04T20:48:51.113079+00:00 app[clock.1]: Scaling ...
2014-11-04T20:49:10.486417+00:00 heroku[web.1]: Stopping all processes with SIGTERM
2014-11-04T20:49:11.844089+00:00 heroku[web.1]: Process exited with status 0
2014-11-04T20:49:12.816363+00:00 app[clock.1]: Scaling ...
2014-11-04T20:49:12.936135+00:00 app[clock.1]: Success!
2014-11-04T20:49:12.914887+00:00 heroku[api]: Scale to clock=1, web=0 by hermanmu@gmail.com
```

With the script working, let's update APScheduler...

## Scheduler

Since we want to, again, scale out at 7am, and then scale in at 10pm, and then in again at 3am, update the scheduled tasks like so:

```python
@sched.scheduled_job('cron', hour=7)
def scale_out_to_three():
    print 'Scaling out ...'
    print scale(3)


@sched.scheduled_job('cron', hour=22)
def scale_in_to_two():
    print 'Scaling in ...'
    print scale(2)


@sched.scheduled_job('cron', hour=3)
def scale_in_to_one():
    print 'Scaling in ...'
    print scale(1)
```

Let this run for at least 24 hours, then check your logs again to ensure it's working.

## Fail-Safe

In the event that our app goes down, let's make sure to immediately scale out, no questions asked.

First, add the following function to the script, which determines how many dynos are attached to the process:

```python
def get_current_dyno_quantity():
    url = "https://api.heroku.com/apps/" + APP + "/formation"
    try:
        result = requests.get(url, headers=HEADERS)
        for formation in json.loads(result.text):
            current_quantity = formation["quantity"]
            return current_quantity
    except:
        return None
```

Then add a new task:

```python
@sched.scheduled_job('interval', minutes=3)
def fail_safe():
    print "pinging ..."
    r = requests.get('https://APPNAME.herokuapp.com/')
    current_number_of_dynos = get_current_dyno_quantity()
    if r.status_code < 200 or r.status_code > 299:
        if current_number_of_dynos < 3:
            print 'Scaling out ...'
            print scale(3)
    if r.elapsed.microseconds / 1000 > 5000:
        if current_number_of_dynos < 3:
            print 'Scaling out ...'
            print scale(3)
```

Here we send a GET request to our app (make sure to update the URL), and if the status code falls outside the 200-range or if the response takes longer than 5000 milliseconds, then we scale out (as long as the current number of dynos does not exceed 3).

Want to test this out? Manually remove all dynos from your app, and then open the logs:

```sh
heroku ps:scale web=0
Scaling web processes... done, now running 0
$ heroku ps
=== clock: `python autoscale.py`
clock.1: up 2014/11/04 15:47:06 (~ 3m ago)

$ heroku logs --tail
2014-11-04T21:53:06.633786+00:00 app[clock.1]: pinging ...
2014-11-04T21:53:06.738860+00:00 app[clock.1]: Scaling out ...
2014-11-04T21:53:06.817780+00:00 heroku[api]: Scale to clock=1, web=3 by michael@realpython.com
2014-11-04T21:53:10.740655+00:00 heroku[web.1]: Starting process with command `gunicorn hello:app`
2014-11-04T21:53:10.634433+00:00 heroku[web.2]: Starting process with command `gunicorn hello:app`
2014-11-04T21:53:11.338596+00:00 heroku[web.3]: Starting process with command `gunicorn hello:app`
2014-11-04T21:53:11.929276+00:00 heroku[web.2]: State changed from starting to up
2014-11-04T21:53:12.731831+00:00 heroku[web.3]: State changed from starting to up
2014-11-04T21:53:12.632277+00:00 heroku[web.1]: State changed from starting to up
2014-11-04T21:56:06.611123+00:00 app[clock.1]: pinging ...
2014-11-04T21:56:06.723760+00:00 app[clock.1]: ... success!
```

Perfect!

## Next Steps

Well, we now have a script ([download](https://gist.github.com/mjhea0/e1436b693cc56ca82277)) that automates the scaling of Heroku dynos. Hopefully this will allow you to keep your application up and running while also saving some much needed cash. You should sleep a bit sounder too knowing that your application will automatically scale out if there's a huge influx of traffic.

What's next?

1. Autoscale In: Automatically scale in when the response takes less than, say, 1000 milliseconds.
1. Failure Emails/Text Messages: If *anything* breaks, send an email and/or text message.
1. Charts: Create some charts so you can better understand your traffic/peak periods/etc. via D3.

*Finally, our friends at **[Runbook](https://runbook.io/)** manage autoscaling - and much more! - from the cloud! The process is much simpler. Check them out.*

Cheers!