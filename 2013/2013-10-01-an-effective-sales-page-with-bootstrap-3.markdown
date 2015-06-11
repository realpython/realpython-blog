# An Effective Sales Page with Bootstrap 3

What originally was a 2-part series on Bootstrap has shifted to three parts with the addition of this blog post by [William Ghelfi](http://www.williamghelfi.com/about). You can grab the final styles/pages from this [repo](https://github.com/mjhea0/bootstrap3).

> In the first [post](http://www.realpython.com/blog/design/getting-started-with-bootstrap-3), we took a look at the basics of Bootstrap 3 and how to design a basic web site. Let's take it a step further and create a nice sales landing [page](http://www.realpython.com/files/bootstrap3/part2/index.html).

<br>

One of the most common myths about Bootstrap is that you can't actually use it for anything so different and highly specialized as, say, a sales page.

And that's what it is. A myth.

Let's debunk it together in the best possible way: building it.

## Highly specialized pages are highly specialized

Successful sales pages follow precise rules. I don't know them all, and I'm not going to tell you everything there is to learn about the ones I know.

Instead, I'll try to reduce them to little effective pills you can start experimenting with.

## Attention grabbing and first steps

**What's the single best way to grab your attention?**

A big bolded question followed by a decent subtitle.

Then, proceed with briefly introducing the **pain** you are going to remove from your customer's a.. from your customer's life (!) thanks to your product.

With Bootstrap, we can build it like this:

```html
<!DOCTYPE html>
<head>
  <meta charset="utf-8" />
  <title>Bootstrap Sales Page.</title>
  <meta name="author" content="" />
  <meta name="description" content="" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <link href="http://netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap.min.css" rel="stylesheet" />
</head>
<body>

  <header class="container">
    <div class="row">
      <div class="col-md-10 col-md-offset-1">
        <h1>Have you ever seen the rain?</h1>
        <p class="lead">You should always open a sales page with a catchy question.</p>
      </div>
    </div>
  </header>

  <div class="container">
    <div class="row">

    <div class="col-xs-12 col-md-4 col-md-offset-1">
      <p>
        Proceed then with the classic triplet: the pain, the dream, the solution.
      </p>
      <p>
        <strong>The pain</strong> ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.<br /><strong>The dream</strong> enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodoconsequat. Duis aute irure dolor in reprehenderit in voluptate velit essecillum dolore eu fugiat nulla pariatur.
      </p>
      <p>
        <strong>The solution</strong> selfies semiotics keffiyeh master cleanse Vice before they sold out. Vegan 90's tofu pork belly skateboard, Truffaut tote bag.
      </p>
      <p>
        <em><strong>Interested?<br />
        <a href="#packages">Go straight to the packages.</a></strong></em>
      </p>
    </div>

      <div class="col-xs-12 col-md-6">
        <figure class="text-center">
          <img src="http://placehold.it/400x300" alt="The Amazing Product" class="img-thumbnail" />
        </figure>
      </div>

  </div>

  <hr />

  <div class="row">
  <figure class="col-md-2 col-md-offset-1">
      <img src="http://placehold.it/100x100" alt="Jonathan F. Doe" class="img-responsive img-circle pull-right" />
  </figure>
  <blockquote class="col-md-7">
      <p>
          Testimonials are important. Be sure to include some in your sales page.
      </p>
      <p>
          With Product, my sales page was <strong>online in minutes</strong>. And it rocks!.
      </p>
      <small>Jonathan F. Doe, CIO at Lorem Ipsum</small>
  </blockquote>
  </div>

</div>

</body>
</html>
```

![Step 1](https://realpython.com/images/blog_images/sales-page-bootstrap3-sshot-1.png)

Not bad, but let's add some customizations:

```html
<style>

@import url(http://fonts.googleapis.com/css?family=Open+Sans+Condensed:300,700|Open+Sans:400italic,700italic,400,700|Fjalla+One);

body {
    font-size: 18px;
    font-family: "Open Sans", Arial, sans-serif;
    color: #292b33;
}

h1, h2 {
    font-family: 'Fjalla One', 'Helvetica Neue', Arial, sans-serif;
    font-size: 52px;
    font-weight: 400;
    text-transform: uppercase;
    letter-spacing: 1px;
    text-align: left;
    margin: 1em 0 0 0;
}

p {
    line-height: 1.5;
    margin: 0 0 20px 0;
}

blockquote {
    border-left: none;
    position: relative;
}

blockquote::before {
    content: '“';
    position: absolute;
    top: 0;
    left: 0;
    font-size: 48px;
    font-family: "inherit";
    font-weight: bold;
}

blockquote p {
    margin: 0 0 10px 20px;
    font-style: italic;
    font-family: "Georgia";
}

em {
    font-family: Georgia, serif;
    font-size: 1.1em;
}

.lead {
    margin-top: 0.25em;
}

</style>
```


![Step 2](https://realpython.com/images/blog_images/sales-page-bootstrap3-sshot-2.png)

Starting to get nicer!

## All in

It's now time to better introduce **your solution to the pain**.

Start providing value: a good free sample of the product. Then lay down three different variants / packages, with different extras, and price them accordingly.

**Why three?**

Let's just say that a 99$ middle package feels both very affordable and valuable between a base package at 39$ and a complete package at 249$.

Deciding the order &mdash; lowest to highest, or highest to lowest &mdash; is just a matter of knowing what you are going after.

If your goal is to sell a higher number of the lowest package, make your visitors reach that one before they reach the others. Else, act just the opposite way around.

More on the topic, in a great [post by Nathan Barry](http://thinktraffic.net/most-common-pricing-mistake).

And here's the code:

```html
    ...
    <style>
    ...

  .bg-white-dark {
    border-top: 1px solid #cccbd6;
    border-bottom: 1px solid #cccbd6;
    background-color: #e7e6f3; /* Older Browsers */
    background-color: rgba(231, 230, 243, 0.9);
}

.container {
    padding-top: 2em;
    padding-bottom: 2em;
}

ul {
    list-style-type: circle;
}
</style>
<div class="bg-white-dark" id="free-sample">
  <div class="container">
    <div class="row">

    <div class="col-md-10 col-md-offset-1">
      <h2>Get a free sample</h2>
      <p class="lead">A taste of what is included with the product.</p>
    </div>

    </div>
    <div class="row">

      <div class="col-md-10 col-md-offset-1">
        <div class="panel panel-default">
          <div class="panel-body text-center">
            <img alt="sample" src="http://placehold.it/500x250" class="img-rounded" />
            <p>Describe the sample dolor sit amet and why should I want to get it adipiscint elit.</p>
            <p>
            <a href="#" class="btn btn-lg btn-default text-uppercase">
            <span class="icon icon-download-alt"></span>&#32;
            Download the sample
            </a>
            </p>
          </div>
        </div>
      </div>

      </div>
  </div>
</div>

<div class="container" id="packages">

  <div class="row">
    <div class="col-md-12">
      <h2>
        The Complete Package
        <span class="pull-right">
            <a class="btn btn-success btn-lg" href="#">
                <span class="text-uppercase"><span class="text-white-dark">Buy now for</span> $249</span>
            </a>
        </span>
      </h2>
      <p class="lead">
        Cosby sweater cray skateboard.
      </p>
    </div>
  </div>

  <div class="row">
    <div class="media">
      <figure class="pull-left col-xs-12 col-md-4">
          <img src="http://placehold.it/300x250" alt="The Amazing Product" class="media-object img-thumbnail" />
      </figure>
      <div class="media-body col-xs-12 col-md-7">
        <h3 class="media-heading">The best package for lorem ipsumer</h3>
        <p>
            Mustache farm-to-table deep v cardigan, Banksy Godard roof party PBR&amp;B.
        </p>
        <ul>
            <li>Details</li>
            <li>Lorem ipsum</li>
            <li>Nostrud exercitation</li>
            <li>Resources ipsum</li>
            <li>Adipiscit resource</li>
            <li>Resource numquam</li>
            <li>Resources ipsum</li>
            <li>Adipiscit resource</li>
            <li>Resource numquam</li>
        </ul>
      </div>
    </div>
  </div>

</div>

<div class="bg-white-dark">
  <div class="container">

  <div class="row">
    <div class="col-md-12">
      <h2>
        The Amazing Product + Resources
        <span class="pull-right">
            <a class="btn btn-success btn-lg" href="#">
                <span class="text-uppercase"><span class="text-white-dark">Buy now for</span> $99</span>
            </a>
        </span>
      </h2>
        <p class="lead">
            Cosby sweater cray skateboard.
        </p>
    </div>
  </div>

      <div class="row">
        <div class="media">
          <figure class="pull-left col-xs-12 col-md-4">
              <img src="http://placehold.it/300x250" alt="The Amazing Product" class="media-object img-thumbnail" />
          </figure>
          <div class="media-body col-xs-12 col-md-7">
              <h3 class="media-heading">Perfect for nostrud lorem</h3>
              <p>
                  Mustache farm-to-table deep v cardigan, Banksy Godard roof party PBR&amp;B.
              </p>
              <ul>
                  <li>Details</li>
                  <li>Lorem ipsum</li>
                  <li>Nostrud exercitation</li>
                  <li>Resources ipsum</li>
              </ul>
            </div>
        </div>
      </div>

  </div>
</div>

<div class="container">

  <div class="row">
      <div class="col-md-12">
        <h2>
            The Amazing Product
            <span class="pull-right">
                <a class="btn btn-success btn-lg" href="#">
                    <span class="text-uppercase"><span class="text-white-dark">Buy now for</span> $39</span>
                </a>
            </span>
        </h2>
        <p class="lead">
            Cosby sweater cray skateboard.
        </p>
      </div>
  </div>

  <div class="row">
    <div class="media">
      <figure class="pull-left col-xs-12 col-md-4">
          <img src="http://placehold.it/300x250" alt="The Amazing Product" class="media-object img-thumbnail" />
      </figure>
      <div class="media-body col-xs-12 col-md-7">
        <h3 class="media-heading">The budget option</h3>
        <p>
            Mustache farm-to-table deep v cardigan, Banksy Godard roof party PBR&amp;B.
        </p>
        <p>
            Cliche sartorial roof party, shabby chic sustainable VHS food truck 90's four loko. Etsy hoodie     distillery, organic beard DIY cliche.
        </p>
      </div>
    </div>
  </div>

</div>
```

![Step 3](https://realpython.com/images/blog_images/sales-page-bootstrap3-sshot-3.png)

## Styling more

We are almost done, but even if we added that color variation with `.bg-white-dark` &mdash; which is, by the way, actually lilac just because with a name like `.bg-white-dark` you can switch to your preferred color without having to change the class name &mdash; the overall look & feel can be further improved, and it can be made even more different from the basic Bootstrap.

Let's add some more styles:

```html
...
<style>
...

.text-white-dark {
    color: #e7e6f3;
}

.btn-default {
    font-family: "Fjalla One", sans-serif;
  text-shadow: 0 -1px 0 rgba(0, 0, 0, 0.2);
  -webkit-box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.15), 0 1px 1px rgba(0, 0, 0, 0.075);
          box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.15), 0 1px 1px rgba(0, 0, 0, 0.075);
}

.btn-default:active,
.btn-default.active {
  -webkit-box-shadow: inset 0 3px 5px rgba(0, 0, 0, 0.125);
          box-shadow: inset 0 3px 5px rgba(0, 0, 0, 0.125);
}

.btn:active,
.btn.active {
  background-image: none;
}

.btn-default {
  text-shadow: 0 1px 0 #fff;
  background-image: -webkit-gradient(linear, left 0%, left 100%, from(#ffffff), to(#e6e6e6));
  background-image: -webkit-linear-gradient(top, #ffffff, 0%, #e6e6e6, 100%);
  background-image: -moz-linear-gradient(top, #ffffff 0%, #e6e6e6 100%);
  background-image: linear-gradient(to bottom, #ffffff 0%, #e6e6e6 100%);
  background-repeat: repeat-x;
  border-color: #e0e0e0;
  border-color: #ccc;
  filter: progid:DXImageTransform.Microsoft.gradient(startColorstr='#ffffffff', endColorstr='#ffe6e6e6', GradientType=0);
}

.btn-default:active,
.btn-default.active {
  background-color: #e6e6e6;
  border-color: #e0e0e0;
}

.btn-success {
    font-family: "Fjalla One", sans-serif;
}

.btn-success {
    background: #9292c0; /* Old browsers */
    background: -moz-linear-gradient(top,  #9292c0 0%, #8181b7 100%); /* FF3.6+ */
    background: -webkit-gradient(linear, left top, left bottom, color-stop(0%,#9292c0), color-stop(100%,#8181b7)); /* Chrome,Safari4+ */
    background: -webkit-linear-gradient(top,  #9292c0 0%,#8181b7 100%); /* Chrome10+,Safari5.1+ */
    background: -o-linear-gradient(top,  #9292c0 0%,#8181b7 100%); /* Opera 11.10+ */
    background: -ms-linear-gradient(top,  #9292c0 0%,#8181b7 100%); /* IE10+ */
    background: linear-gradient(to bottom,  #9292c0 0%,#8181b7 100%); /* W3C */
    filter: progid:DXImageTransform.Microsoft.gradient( startColorstr='#9292c0', endColorstr='#8181b7',GradientType=0 ); /* IE6-9 */
  border-color: #f1ddff;
}

.btn-success:hover,
.btn-success:focus,
.btn-success:active,
.btn-success.active {
    color: #fbfafc;
    background: #a3a3d1; /* Old browsers */
    background: -moz-linear-gradient(top,  #a3a3d1 0%, #9292c8 100%); /* FF3.6+ */
    background: -webkit-gradient(linear, left top, left bottom, color-stop(0%,#a3a3d1), color-stop(100%,#9292c8)); /* Chrome,Safari4+ */
    background: -webkit-linear-gradient(top,  #a3a3d1 0%,#9292c8 100%); /* Chrome10+,Safari5.1+ */
    background: -o-linear-gradient(top,  #a3a3d1 0%,#9292c8 100%); /* Opera 11.10+ */
    background: -ms-linear-gradient(top,  #a3a3d1 0%,#9292c8 100%); /* IE10+ */
    background: linear-gradient(to bottom,  #a3a3d1 0%,#9292c8 100%); /* W3C */
    filter: progid:DXImageTransform.Microsoft.gradient( startColorstr='#a3a3d1', endColorstr='#9292c8',GradientType=0 ); /* IE6-9 */
  border-color: #f1ddff;
}

.btn-success.disabled:hover,
.btn-success[disabled]:hover,
fieldset[disabled] .btn-success:hover,
.btn-success.disabled:focus,
.btn-success[disabled]:focus,
fieldset[disabled] .btn-success:focus,
.btn-success.disabled:active,
.btn-success[disabled]:active,
fieldset[disabled] .btn-success:active,
.btn-success.disabled.active,
.btn-success[disabled].active,
fieldset[disabled] .btn-success.active {
    color: #fbfafc;
    background: #a3a3d1; /* Old browsers */
    background: -moz-linear-gradient(top,  #a3a3d1 0%, #9292c8 100%); /* FF3.6+ */
    background: -webkit-gradient(linear, left top, left bottom, color-stop(0%,#a3a3d1), color-stop(100%,#9292c8)); /* Chrome,Safari4+ */
    background: -webkit-linear-gradient(top,  #a3a3d1 0%,#9292c8 100%); /* Chrome10+,Safari5.1+ */
    background: -o-linear-gradient(top,  #a3a3d1 0%,#9292c8 100%); /* Opera 11.10+ */
    background: -ms-linear-gradient(top,  #a3a3d1 0%,#9292c8 100%); /* IE10+ */
    background: linear-gradient(to bottom,  #a3a3d1 0%,#9292c8 100%); /* W3C */
    filter: progid:DXImageTransform.Microsoft.gradient( startColorstr='#a3a3d1', endColorstr='#9292c8',GradientType=0 ); /* IE6-9 */
  border-color: #f1ddff;
}

</style>
...
```

![Step 4](https://realpython.com/images/blog_images/sales-page-bootstrap3-sshot-4.png)

## Closing thoughts

That's it. We built and customized a minimal sales page with Bootstrap. The page looks nothing like the same old Bootstrap page, and &mdash; best of all &mdash; it actually works in driving sales performance! Check it out [here](http://www.realpython.com/files/bootstrap3/part2/index.html).

Ok, I guess it's *full disclosure time*: the page we built together is the core of an actual, real life tested, sales page.

I wrote *Bootstrap In Practice*, an ebook for starters, to **get** them **productive fast** and **go back to profit**, without getting stuck in the official docs or going too much / too soon in depth led by mere intellectual curiosity.

You can find the real, complete, battle-tested [sales page on my website](http://www.williamghelfi.com/bootstrap-in-practice/), where I also offer a [free 30 day course about Bootstrap tips](http://www.williamghelfi.com/bootstrap-in-practice#free-chapter) alongside a free sample chapter from the ebook.

<br>

Next time we'll add Flask to the mix and create a nice looking boilerplate that you can use for your own web app. Cheers!
