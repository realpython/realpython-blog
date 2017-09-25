# Code Evaluation with AWS Lambda and API Gateway

**This tutorial details how [AWS Lambda](https://aws.amazon.com/lambda/) and [API Gateway](https://aws.amazon.com/api-gateway/) can be used to develop a simple code evaluation API, where an end user submits code, via an AJAX form submission, which is then executed securely by a Lambda function.**

<div class="center-text">
  <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-api-gateway-python-logos.png" style="max-width: 100%;" alt="aws lambda api gateway python logos">
</div>

<br>

Check out the live demo of what you'll be building in action [here](https://realpython.github.io/aws-lambda-code-execute/).

> **WARNING:** The code found in this tutorial is used to build a toy app to prototype a proof of concept and is not meant for production use.

This tutorial assumes that you already have an account set up with [AWS](https://aws.amazon.com/). Also, we will use the `US East (N. Virginia)` / `us-east-1` region. Feel free to use the region of your choice. For more info, review the [Regions and Availability Zones](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html) guide.

## What is AWS Lambda?

Amazon Web Services (AWS) Lambda is an on-demand compute service that lets you run code in response to events or HTTP requests.

Use cases:

<br>
<table style="font-size: 16px;border-spacing: 10px 0px;border-collapse: separate;">
  <thead>
    <tr>
      <th></th>
      <th>Event</th>
      <th>Action</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td></td>
      <td>Image added to S3</td>
      <td>Image is processed</td>
    </tr>
    <tr>
      <td></td>
      <td>HTTP Request via API Gateway</td>
      <td>HTTP Response</td>
    </tr>
    <tr>
      <td></td>
      <td>Log file added to Cloudwatch</td>
      <td>Analyze the log</td>
    </tr>
    <tr>
      <td></td>
      <td>Scheduled event</td>
      <td>Back up files</td>
    </tr>
    <tr>
      <td></td>
      <td>Scheduled event</td>
      <td>Synchronization of files</td>
    </tr>
  </tbody>
</table>
<br>

For more examples, review the [Examples of How to Use AWS Lambda](http://docs.aws.amazon.com/lambda/latest/dg/use-cases.html) guide from AWS.

You can run scripts and apps without having to provision or manage servers in a seemingly infinitely-scalable environment where you pay only for usage. This is "[serverless](https://martinfowler.com/articles/serverless.html)" computing in a nut shell. For our purposes, AWS Lambda is a perfect solution for running user-supplied code quickly, securely, and cheaply.

As of writing, Lambda [supports](https://aws.amazon.com/lambda/faqs/) code written in JavaScript (Node.js), Python, Java, and C#.

## Project Setup

Start by cloning down the base project:

```sh
$ git clone https://github.com/realpython/aws-lambda-code-execute \
  --branch v1 --single-branch
$ cd aws-lambda-code-execute
```

Then, check out the [v1](https://github.com/realpython/aws-lambda-code-execute/releases/tag/v1) tag to the master branch:

```sh
$ git checkout tags/v1 -b master
```

Open the *index.html* file in your browser of choice:

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/aws-lambda-code-execute-v1.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-code-execute-v1.png" style="max-width: 100%;" alt="code eval form">
  </a>
</div>

<br>

Then, open the project in your code editor:

```sh
├── README.md
├── assets
│   ├── main.css
│   ├── main.js
│   └── vendor
│       ├── bootstrap
│       │   ├── css
│       │   │   ├── bootstrap-grid.css
│       │   │   ├── bootstrap-grid.min.css
│       │   │   ├── bootstrap-reboot.css
│       │   │   ├── bootstrap-reboot.min.css
│       │   │   ├── bootstrap.css
│       │   │   └── bootstrap.min.css
│       │   └── js
│       │       ├── bootstrap.js
│       │       └── bootstrap.min.js
│       ├── jquery
│       │   ├── jquery.js
│       │   └── jquery.min.js
│       └── popper
│           ├── popper.js
│           └── popper.min.js
└── index.html
```

Let's quickly review the code. Essentially, we just have a simple HTML form styled with [Bootstrap](http://getbootstrap.com/). The input field is replaced with [Ace](https://ace.c9.io/), an embeddable code editor, which provides basic syntax highlighting. Finally, within *assets/main.js*, a jQuery event handler is wired up to grab the code from the Ace editor, when the form is submitted, and send the data somewhere (eventually to API Gateway) via an AJAX request.

## Lambda Setup

Within the [AWS Console](https://console.aws.amazon.com), navigate to the main [Lambda page](https://console.aws.amazon.com/lambda) and click "Create a function":

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/aws-lambda-console.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-console.png" style="max-width: 100%;" alt="aws lambda console">
  </a>
</div>

### Create function

Steps...

1. *Select blueprint*: Click "Author from scratch" to start with a blank function:

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/aws-lambda-console-select-blueprint.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-console-select-blueprint.png" style="max-width: 100%;" alt="aws lambda console - select blueprint">
      </a>
    </div>

1. *Configure Triggers*: We'll set up the API Gateway integration later, so simply click "Next" to skip this part.
1. *Configure function*: Name the function `execute_python_code`, and add a basic description - `Execute user-supplied Python code`. Select "Python 3.6" in the "Runtime" drop-down.

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/aws-lambda-console-configure-function-part1.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-console-configure-function-part1.png" style="max-width: 100%;" alt="aws lambda console - configure function">
      </a>
    </div>

1. Within the inline code editor, update the `lambda_handler` function definition with:

   <figure class="code"><figcaption><span></span></figcaption><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
   <span class="line-number">2</span>
   <span class="line-number">3</span>
   <span class="line-number">4</span>
   <span class="line-number">5</span>
   <span class="line-number">6</span>
   <span class="line-number">7</span>
   <span class="line-number">8</span>
   <span class="line-number">9</span>
   <span class="line-number">10</span>
   <span class="line-number">11</span>
   <span class="line-number">12</span>
   <span class="line-number">13</span>
   <span class="line-number">14</span>
   <span class="line-number">15</span>
   <span class="line-number">16</span>
   <span class="line-number">17</span>
   <span class="line-number">18</span>
   <span class="line-number">19</span>
   <span class="line-number">20</span>
   <span class="line-number">21</span>
   </pre></td><td class="code"><pre><code class="python"><span class="line"><span class="kn">import</span> <span class="nn">sys</span>
   </span><span class="line"><span class="kn">from</span> <span class="nn">io</span> <span class="kn">import</span> <span class="n">StringIO</span>
   </span><span class="line">
   </span><span class="line">
   </span><span class="line"><span class="k">def</span> <span class="nf">lambda_handler</span><span class="p">(</span><span class="n">event</span><span class="p">,</span> <span class="n">context</span><span class="p">):</span>
   </span><span class="line">    <span class="c"># get code from payload</span>
   </span><span class="line">    <span class="n">code</span> <span class="o">=</span> <span class="n">event</span><span class="p">[</span><span class="s">'answer'</span><span class="p">]</span>
   </span><span class="line">    <span class="n">test_code</span> <span class="o">=</span> <span class="n">code</span> <span class="o">+</span> <span class="s">'</span><span class="se">\n</span><span class="s">print(sum(1,1))'</span>
   </span><span class="line">    <span class="c"># capture stdout</span>
   </span><span class="line">    <span class="nb">buffer</span> <span class="o">=</span> <span class="n">StringIO</span><span class="p">()</span>
   </span><span class="line">    <span class="n">sys</span><span class="o">.</span><span class="n">stdout</span> <span class="o">=</span> <span class="nb">buffer</span>
   </span><span class="line">    <span class="c"># execute code</span>
   </span><span class="line">    <span class="k">try</span><span class="p">:</span>
   </span><span class="line">        <span class="k">exec</span><span class="p">(</span><span class="n">test_code</span><span class="p">)</span>
   </span><span class="line">    <span class="k">except</span><span class="p">:</span>
   </span><span class="line">        <span class="k">return</span> <span class="bp">False</span>
   </span><span class="line">    <span class="c"># return stdout</span>
   </span><span class="line">    <span class="n">sys</span><span class="o">.</span><span class="n">stdout</span> <span class="o">=</span> <span class="n">sys</span><span class="o">.</span><span class="n">__stdout__</span>
   </span><span class="line">    <span class="c"># check</span>
   </span><span class="line">    <span class="k">if</span> <span class="nb">int</span><span class="p">(</span><span class="nb">buffer</span><span class="o">.</span><span class="n">getvalue</span><span class="p">())</span> <span class="o">==</span> <span class="mi">2</span><span class="p">:</span>
   </span><span class="line">        <span class="k">return</span> <span class="bp">True</span>
   </span><span class="line">    <span class="k">return</span> <span class="bp">False</span>
   </span></code></pre></td></tr></tbody></table></div></figure>

    Here, within `lambda_handler`, which is the default entry point for Lambda, we parse the JSON request body, passing the supplied code along with some test code - `sum(1,1)` - to the [exec](https://docs.python.org/3/library/functions.html#exec) function - which executes the string as Python code. Then, we simply ensure the actual results are the same as what's expected - e.g., 2 - and return the appropriate response.

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/aws-lambda-console-configure-function-part2.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-console-configure-function-part2.png" style="max-width: 100%;" alt="aws lambda console - configure function">
      </a>
    </div>

    Under "Lambda function handler and role", leave the default handler and then select "Create a new Role from template(s)" from the drop-down. Enter a "Role name", like `api_gateway_access`, and select " Simple Microservice permissions" for the "Policy templates", which provides access to API Gateway.

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/aws-lambda-console-configure-function-part3.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-console-configure-function-part3.png" style="max-width: 100%;" alt="aws lambda console - configure function">
      </a>
    </div>

    Click "Next".

1. *Review*: Create the function after a quick review.

### Test

Next click on the "Test" button to execute the newly created Lambda:

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/aws-lambda-console-function.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-console-function.png" style="max-width: 100%;" alt="aws lambda console - function">
  </a>
</div>

<br>

Using the "Hello World" event template, replace the sample with:

```json
{
  "answer": "def sum(x,y):\n    return x+y"
}
```

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/aws-lambda-console-function-test.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-console-function-test.png" style="max-width: 100%;" alt="aws lambda console - function test">
  </a>
</div>

<br>

Click the "Save and test" button at the bottom of the modal to run the test. Once done, you should see something similar to:

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/aws-lambda-console-function-test-results.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-console-function-test-results.png" style="max-width: 100%;" alt="aws lambda console - function test results">
  </a>
</div>

<br>

With that, we can move on to configuring the API Gateway to trigger the Lambda from user-submitted POST requests...

## API Gateway Setup

[API Gateway](https://aws.amazon.com/api-gateway/) is used to define and host APIs. In our example, we'll create a single HTTP POST endpoint that triggers the Lambda function when an HTTP request is received and then responds with the results of the Lambda function, either `true` or `false`.

Steps:

1. Create the API
1. Test it manually
1. Enable CORS
1. Deploy the API
1. Test via cURL

### Create the API

1. To start, from the [API Gateway page](https://console.aws.amazon.com/apigateway), click the "Get Started" button to create a new API:

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/api-gateway-console.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-console.png" style="max-width: 100%;" alt="api gateway console">
      </a>
    </div>

1. Select "New API", and then provide a descriptive name, like `code_execute_api`:

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/api-gateway-create-new-api.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-create-new-api.png" style="max-width: 100%;" alt="api gateway - create new api">
      </a>
    </div>

    <br>

    Then, create the API.

1. Select "Create Resource" from the "Actions" drop-down.

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/api-gateway-create-resource.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-create-resource.png" style="max-width: 100%;" alt="api gateway - create resource">
      </a>
    </div>

1. Name the resource `execute`, and then click "Create Resource".

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/api-gateway-create-resource-new.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-create-resource-new.png" style="max-width: 100%;" alt="api gateway - create resource">
      </a>
    </div>

1. With the resource highlighted, select "Create Method" from the "Actions" drop-down.

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/api-gateway-create-method.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-create-method.png" style="max-width: 100%;" alt="api gateway - create method">
      </a>
    </div>

1. Choose "POST" from the method drop-down. Click the checkmark next to it.

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/api-gateway-create-method-new.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-create-method-new.png" style="max-width: 100%;" alt="api gateway - create method">
      </a>
    </div>

1. In the "Setup" step, select "Lambda Function" as the "Integration type", select the "us-east-1" region in the drop-down, and enter the name of the Lambda function that you just created.

    <div class="center-text">
      <a href="/images/blog_images/aws-lambda/api-gateway-create-method-new-setup.png">
        <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-create-method-new-setup.png" style="max-width: 100%;" alt="api gateway - create method">
      </a>
    </div>

1. Click "Save", and then click "OK" to give permission to the API Gateway to run your Lambda function.

### Test it manually

To test, click on the lightning bolt that says "Test".

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/api-gateway-method-test.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-method-test.png" style="max-width: 100%;" alt="api gateway - test method">
  </a>
</div>

<br>

Scroll down to the "Request Body" input and add the same JSON code we used with the Lambda function:

```json
{
  "answer": "def sum(x,y):\n    return x+y"
}
```

Click "Test". You should see something similar to:

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/api-gateway-method-test-results.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-method-test-results.png" style="max-width: 100%;" alt="api gateway - method test results">
  </a>
</div>

### Enable CORS

Next, we need to enable [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) so that we can POST to the API endpoint from another domain.

With the resource highlighted, select "Enable CORS" from the "Actions" drop-down:

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/api-gateway-enable-cors.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-enable-cors.png" style="max-width: 100%;" alt="api gateway - enable cors">
  </a>
</div>

<br>

Just keep the defaults for now since we're still testing the API. Click the "Enable CORS and replace existing CORS headers" button.

### Deploy the API

Finally, to deploy, select "Deploy API" from the "Actions" drop-down:

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/api-gateway-deploy-api.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-deploy-api.png" style="max-width: 100%;" alt="api gateway - deploy api">
  </a>
</div>

<br>

Create a new "Deployment stage" called 'v1':

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/api-gateway-deploy-api-stage.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-deploy-api-stage.png" style="max-width: 100%;" alt="api gateway - deploy api">
  </a>
</div>

<br>

API gateway will generate a random subdomain for the API endpoint URL, and the stage name will be added to the end of the URL. You should now be able to make POST requests to a similar URL:

```
https://c0rue3ifh4.execute-api.us-east-1.amazonaws.com/v1/execute
```

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/api-gateway-deploy-api-url.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/api-gateway-deploy-api-url.png" style="max-width: 100%;" alt="api gateway - deploy api">
  </a>
</div>

### Test via cURL:

```sh
$ curl -H "Content-Type: application/json" -X POST \
  -d '{"answer":"def sum(x,y):\n    return x+y"}' \
  https://c0rue3ifh4.execute-api.us-east-1.amazonaws.com/v1/execute
```

## Update the Form

Now, to update the form so that it sends the POST request to the API Gateway endpoint, first add the URL to the `grade` function in *assets/main.js*:

```javascript
function grade(payload) {
  $.ajax({
    method: 'POST',
    url: 'https://c0rue3ifh4.execute-api.us-east-1.amazonaws.com/v1/execute',
    dataType: 'json',
    contentType: 'application/json',
    data: JSON.stringify(payload)
  })
  .done((res) => { console.log(res); })
  .catch((err) => { console.log(err); });
}
```

Then, update the `.done` and `.catch()` functions, like so:


```javascript
function grade(payload) {
  $.ajax({
    method: 'POST',
    url: 'https://c0rue3ifh4.execute-api.us-east-1.amazonaws.com/v1/execute',
    dataType: 'json',
    contentType: 'application/json',
    data: JSON.stringify(payload)
  })
  .done((res) => {
    let message = 'Incorrect. Please try again.';
    if (res) {
      message = 'Correct!';
    }
    $('.answer').html(message);
    console.log(res);
    console.log(message);
  })
  .catch((err) => {
    $('.answer').html('Something went terribly wrong!');
    console.log(err);
  });
}
```

Now, if the request is a success, the appropriate message will be added - via the jQuery [html](http://api.jquery.com/html/) method - to an HTML element with a class of `answer`. Add this element, just below the HTML form, within *index.html*:

```html
<h5 class="answer"></h5>
```

Let's add a bit of style as well to the *assets/main.css* file:

```css
.answer {
  padding-top: 30px;
  color: #dc3545;
  font-style: italic;
}
```

Test it out!

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/aws-lambda-code-execute-success.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-code-execute-success.png" style="max-width: 100%;" alt="code eval form">
  </a>
</div>

<br>

<div class="center-text">
  <a href="/images/blog_images/aws-lambda/aws-lambda-code-execute-failure.png">
    <img class="no-border" src="/images/blog_images/aws-lambda/aws-lambda-code-execute-failure.png" style="max-width: 100%;" alt="code eval form">
  </a>
</div>

## Next Steps

1. *Production*: Think about what's required for a more robust, production-ready application - HTTPS, authentication, possibly a data store. How would you implement these within AWS? Which AWS services can/would you use?
1. *Dynamic*: Right now the Lambda function can only be used to test the `sum` function. How could you make this (more) dynamic, so that it can be used to test any code challenge (maybe even in *any* language)? Try adding a [data attribute](https://developer.mozilla.org/en-US/docs/Learn/HTML/Howto/Use_data_attributes) to the DOM, so that when a user submits an exercise the test code along with solution is sent along with the POST request - i.e., `<some-html-element data-test="\nprint(sum(1,1))" data-results"2" </some-html-element>`.
1. *Stack trace*: Instead of just responding with `true` or `false`, send back the entire stack trace and add it to the DOM when the answer is incorrect.

Thanks for reading. Add questions and/or comments below. Grab the final code from the [aws-lambda-code-execute](https://github.com/realpython/aws-lambda-code-execute) repo. Cheers!
