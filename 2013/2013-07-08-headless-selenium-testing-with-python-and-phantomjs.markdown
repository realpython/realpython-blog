# Headless Selenium Testing with Python and PhantomJS

**Updated (08/25/2014):** pep8 updates, updated code

<hr>

[PhantomJS](http://phantomjs.org/) is a headless [Webkit](https://www.webkit.org/), which has a number of uses. In this example, we'll be using it, in conjunction with Selenium WebDriver, for conducting basic system tests directly from the command line. Since PhantomJS eliminates the need for a graphical browser, tests run much faster.

**Click [here](http://www.youtube.com/watch?v=X0b0xM2Ddh8) to watch the accompanying video.**

> Note: Although the video is outdated (due to changes in the scripts), it's still worth watching.

<hr>

## Setup

Install Selenium with Pip and PhantomJS with [Brew](http://brew.sh/):

```sh
$ pip install selenium
$ brew install phantomjs
```

> Having trouble installing PhantomJS with Brew? Grab the latest build [here](http://phantomjs.org/download.html).

## Examples

Now let's look at two quick examples.

### DuckDuckGo

In the first example, we're just going to search DuckDuckGo for the keyword "realpython" to find the URL of the search results.

```python
from selenium import webdriver
driver = webdriver.PhantomJS()
driver.set_window_size(1120, 550)
driver.get("https://duckduckgo.com/")
driver.find_element_by_id('search_form_input_homepage').send_keys("realpython")
driver.find_element_by_id("search_button_homepage").click()
print driver.current_url
driver.quit()
```

You can see the outputted URL in the terminal.

Here's a look at the same thing using Firefox to display the results.

```python
from selenium import webdriver
driver = webdriver.Firefox()
driver.get("https://duckduckgo.com/")
driver.find_element_by_id('search_form_input_homepage').send_keys("realpython")
driver.find_element_by_id("search_button_homepage").click()
driver.quit()
```

Did you notice how we had to create a dummy browser size on the Phantom script? This is a workaround to a bug that's currently an issue in [Github](https://github.com/ariya/phantomjs/issues/11637). Try the script without it: You'll get an `ElementNotVisibleException` exception.

Now we can write a quick test to assert that the URL brought up by the search results is correct.

```python
import unittest
from selenium import webdriver


class TestOne(unittest.TestCase):

    def setUp(self):
        self.driver = webdriver.PhantomJS()
        self.driver.set_window_size(1120, 550)

    def test_url(self):
        self.driver.get("http://duckduckgo.com/")
        self.driver.find_element_by_id(
            'search_form_input_homepage').send_keys("realpython")
        self.driver.find_element_by_id("search_button_homepage").click()
        self.assertIn(
            "https://duckduckgo.com/?q=realpython", self.driver.current_url
        )

    def tearDown(self):
        self.driver.quit()

if __name__ == '__main__':
    unittest.main()
```

The test passed.

### RealPython.com

Finally, let's look at a real world example that I run daily. Navigate to [RealPython.com](https://www.realpython.com) and I'll show you what we'll be testing. Essentially, I want to ensure that the bottom "Download Now" button has the correct product associated with it.

Here's a look at the basic unittest:

```python
import unittest
from selenium import webdriver


class TestTwo(unittest.TestCase):

    def setUp(self):
        self.driver = webdriver.PhantomJS()

    def test_url(self):
        self.driver.get("https://app.simplegoods.co/i/IQCZADOY") # url associated with button click
        button = self.driver.find_element_by_id("payment-submit").get_attribute("value")
        self.assertEquals(u'Pay - $60.00', button)

    def tearDown(self):
        self.driver.quit()

if __name__ == '__main__':
    unittest.main()
```

## Benchmarking

One main advantage of using PhantomJS over a browser is that tests are **usually** much faster. In this next example, we'll benchmark the previous test using both PhantomJS and Firefox.

```python
import unittest
from selenium import webdriver
import time


class TestThree(unittest.TestCase):

    def setUp(self):
        self.startTime = time.time()

    def test_url_fire(self):
        time.sleep(2)
        self.driver = webdriver.Firefox()
        self.driver.get("https://app.simplegoods.co/i/IQCZADOY") # url associated with button click
        button = self.driver.find_element_by_id("payment-submit").get_attribute("value")
        self.assertEquals(u'Pay - $60.00', button)

    def test_url_phantom(self):
        time.sleep(1)
        self.driver = webdriver.PhantomJS()
        self.driver.get("https://app.simplegoods.co/i/IQCZADOY") # url associated with button click
        button = self.driver.find_element_by_id("payment-submit").get_attribute("value")
        self.assertEquals(u'Pay - $60.00', button)

    def tearDown(self):
        t = time.time() - self.startTime
        print "%s: %.3f" % (self.id(), t)
        self.driver.quit()

if __name__ == '__main__':
    suite = unittest.TestLoader().loadTestsFromTestCase(TestThree)
    unittest.TextTestRunner(verbosity=0).run(suite)
```

You can see just how much faster PhantomJS is:

```sh
$ python test.py -v
__main__.TestThree.test_url_fire: 19.801
__main__.TestThree.test_url_phantom: 10.676
----------------------------------------------------------------------
Ran 2 tests in 30.683s

OK
```

<hr>
<br>

## Video

{% youtube X0b0xM2Ddh8 %}