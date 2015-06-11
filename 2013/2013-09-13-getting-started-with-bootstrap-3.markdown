# Getting Started with Bootstrap 3

Despite labeling myself a backend developer, I've been dabbling more and more with front-end design, namely because JavaScript has blurred the lines between the two. On my last project I hacked a bit with HTML and CSS to develop responsive styles - which can be a pain. Thankfully, frameworks such as Bootstrap (formerly known as Twitter Bootstrap) make such design requests fairly simple - and even simpler with the new release of Bootstrap 3.

Let's take a look.

> This is a 3-part series. In the first part, we'll only be looking at Bootstrap 3 and how to customize it using HTML and CSS. In the second part, we'll look at how to set up an effective sales landing page. Check it out [here](http://www.realpython.com/blog/design/an-effective-sales-page-with-bootstrap-3). Finally, in the last part, we'll add Python into the mix to build a landing page using Flask + Bootstrap 3.

You can grab the final styles/pages from this [repo](https://github.com/mjhea0/bootstrap3).


## Create a minimal layout

Start by downloading Bootstrap from the official [site](http://getbootstrap.com/).
Unpack the zip and grab the dist directory. This will be our root folder for this little project. Be sure to add `jquery.js` from `assets/js` to `dist/js`.

```sh
.
├── css
│   ├── bootstrap-theme.css
│   ├── bootstrap-theme.min.css
│   ├── bootstrap.css
│   └── bootstrap.min.css
├── fonts
│   ├── glyphicons-halflings-regular.eot
│   ├── glyphicons-halflings-regular.svg
│   ├── glyphicons-halflings-regular.ttf
│   └── glyphicons-halflings-regular.woff
├── index.html
└── js
    ├── bootstrap.js
    ├── bootstrap.min.js
    └── jquery.js
```

Add an index.html file with the following content:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Bootstrap Template</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="css/bootstrap.min.css" rel="stylesheet" media="screen">

    <!-- HTML5 shim and Respond.js IE8 support of HTML5 elements and media queries -->
    <!--[if lt IE 9]>
      <script src="../../assets/js/html5shiv.js"></script>
      <script src="../../assets/js/respond.min.js"></script>
    <![endif]-->
  </head>
  <body>
    <div class ="container">
      <h1>Hello, world!</h1>
    </div>
  </body>
</html>
```

Add a nav bar (which is now ALWAYS responsive), a container, Jumbotron, some [ipsum text](http://hipsteripsum.me/), and a button:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Bootstrap Template</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="css/bootstrap.min.css" rel="stylesheet" media="screen">

    <!-- HTML5 shim and Respond.js IE8 support of HTML5 elements and media queries -->
    <!--[if lt IE 9]>
      <script src="../../assets/js/html5shiv.js"></script>
      <script src="../../assets/js/respond.min.js"></script>
    <![endif]-->
  </head>
  <body>
    <div class="navbar navbar-inverse navbar-fixed-top">
    <div class="container">
      <div class="navbar-header">
        <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
          <span class="icon-bar"></span>
          <span class="icon-bar"></span>
          <span class="icon-bar"></span>
        </button>
        <a class="navbar-brand" href="#">Bootstrap <3</a>
      </div>
      <div class="collapse navbar-collapse">
        <ul class="nav navbar-nav navbar-right">
          <li class="active"><a href="#">Home</a></li>
          <li><a href="#about">About</a></li>
          <li><a href="#contact">Contact</a></li>
        </ul>
      </div><!--/.nav-collapse -->
    </div>
  </div>

  <div class="jumbotron">
    <div class="container">
      <h1>Bootstrap starter template</h1>
      <p class="lead">Bacon ipsum dolor sit amet tenderloin chuck jowl, drumstick pork loin kevin andouille tri-tip. Turkey biltong sirloin, tongue rump pork belly t-bone tail sausage venison corned beef. Jerky pig shoulder beef, tri-tip turducken kevin ribeye prosciutto spare ribs.</p>
      <a class="btn btn-primary btn-lg btn-block">Big ASS Learn more button</a>
    </div>
  </div>

  </body>
</html>
```

Add the following style to the head:

```html
<style>
  body {padding-top: 50px;}
</style>
```

Finally, to add the full responsiveness, add the following scripts just before the `</body>` tag:

```html
<script src="js/jquery.js"></script>
<script src="js/bootstrap.min.js"></script>
```

At first glance, you can see that all components are flat. And if you're not a fan of flat design there is an alternative theme. Play around with the responsiveness by resizing your browser. Looks pretty good. Responsiveness is now a default in Bootstrap 3 - so now you don't have a choice: You must develop for mobile first. <3

### Grid System

The grid system is now much easier to use. It scales up to 12 columns by default, which can be altered for four devices - large desktops, desktops, tablets, and phones. It's also fluid by default as you will soon see.

Add a row, a basic grid system, and more ipsum text after the closing jumbotron div:

```html
<div class="container">
  <div class="row">
    <div class="col-md-4">
      <p>Yr umami selfies Carles DIY, pop-up Tonx meggings stumptown freegan street art Vice ethnic. Pickled gastropub lo-fi polaroid, ennui selvage meh Tumblr organic iPhone kale chips narwhal Echo Park. Tonx literally distillery Pitchfork McSweeney's semiotics. Stumptown YOLO fanny pack bespoke, kitsch Carles gastropub vegan. Biodiesel ennui church-key McSweeney's, selvage hoodie Brooklyn 90's lomo. Quinoa photo booth cliche semiotics. Roof party Etsy ethnic, fashion axe mlkshk 8-bit paleo.</p>
    </div>
    <div class="col-md-4">
      <p>Yr umami selfies Carles DIY, pop-up Tonx meggings stumptown freegan street art Vice ethnic. Pickled gastropub lo-fi polaroid, ennui selvage meh Tumblr organic iPhone kale chips narwhal Echo Park. Tonx literally distillery Pitchfork McSweeney's semiotics. Stumptown YOLO fanny pack bespoke, kitsch Carles gastropub vegan. Biodiesel ennui church-key McSweeney's, selvage hoodie Brooklyn 90's lomo. Quinoa photo booth cliche semiotics. Roof party Etsy ethnic, fashion axe mlkshk 8-bit paleo.</p>
    </div>
    <div class="col-md-4">
      <p>Yr umami selfies Carles DIY, pop-up Tonx meggings stumptown freegan street art Vice ethnic. Pickled gastropub lo-fi polaroid, ennui selvage meh Tumblr organic iPhone kale chips narwhal Echo Park. Tonx literally distillery Pitchfork McSweeney's semiotics. Stumptown YOLO fanny pack bespoke, kitsch Carles gastropub vegan. Biodiesel ennui church-key McSweeney's, selvage hoodie Brooklyn 90's lomo. Quinoa photo booth cliche semiotics. Roof party Etsy ethnic, fashion axe mlkshk 8-bit paleo.</p>
    </div>
  </div>
</div>
```

> Since each row has 12 units, we made three columns that are 4 units each. Just remember that all columns must add up to 12. Experiment a bit: Try changing the first column to 2 units and the last column to 8 units, or add a new row altogether with another set of columns (or just wait until the next step).

Add a new row beneath the old row with the following grid (make sure it goes inside the `container` element):

```html
<div class="col-md-8 col-md-offset-2">
  <p>Yr umami selfies Carles DIY, pop-up Tonx meggings stumptown freegan street art Vice ethnic. Pickled gastropub lo-fi polaroid, ennui selvage meh Tumblr organic iPhone kale chips narwhal Echo Park. Tonx literally distillery Pitchfork McSweeney's semiotics. Stumptown YOLO fanny pack bespoke, kitsch Carles gastropub vegan. Biodiesel ennui church-key McSweeney's, selvage hoodie Brooklyn 90's lomo. Quinoa photo booth cliche semiotics. Roof party Etsy ethnic, fashion axe mlkshk 8-bit paleo.</p>
</div>
```

> Here, we used `offset`, which shifted the column to the right to center it. Essentially, we shifted over 2 units, then used an 8 unit column, leaving another 2 units after, equaling 12: 2 + 8 + 2 = 12.

Since the grid system in Bootstrap 3 is responsive, you can nest a row within another row (another new feature). The inner row - e.g., the row within a row - will have 12 columns. It may help to draw this out. Try this on your own before looking at my example.

```html
<div class="row">
  <div class="col-md-4">
    <div class="row">
      <div class="col-md-6">
        <p>Yr umami selfies Carles DIY, pop-up Tonx meggings stumptown freegan street art Vice ethnic.</p>
      </div>
      <div class="col-md-6">
        <p>Stumptown YOLO fanny pack bespoke, kitsch Carles gastropub vegan.</p>
      </div>
    </div>
    <div class="row">
      <div class="col-md-6">
        <p>Pickled gastropub lo-fi polaroid, ennui selvage meh Tumblr organic iPhone kale chips narwhal Echo Park.</p>
      </div>

      <div class="col-md-6">
        <p>Biodiesel ennui church-key McSweeney's, selvage hoodie Brooklyn 90's lomo.</p>
      </div>
    </div>
  </div>
  <div class="col-md-4">
    <p>Yr umami selfies Carles DIY, pop-up Tonx meggings stumptown freegan street art Vice ethnic. Pickled gastropub lo-fi polaroid, ennui selvage meh Tumblr organic iPhone kale chips narwhal Echo Park. Tonx literally distillery Pitchfork McSweeney's semiotics. Stumptown YOLO fanny pack bespoke, kitsch Carles gastropub vegan. Biodiesel ennui church-key McSweeney's, selvage hoodie Brooklyn 90's lomo. Quinoa photo booth cliche semiotics. Roof party Etsy ethnic, fashion axe mlkshk 8-bit paleo.</p>
  </div>
  <div class="col-md-4">
    <p>
      <ul class="list-group">
        <li class="list-group-item"><span class="badge">14</span>Quinoa photo booth</li>
        <li class="list-group-item"><span class="badge">10</span>iPhone kale chips</li>
        <li class="list-group-item"><span class="badge">6</span>fanny pack bespoke</li>
        <li class="list-group-item"><span class="badge">4</span>lo-fi polaroid</li>
        <li class="list-group-item"><span class="badge">2</span>Tonx meggings stumptown</li>
      </ul>
    </p>
  </div>
</div>
```

> So, we took the first column, with a span of 4, and nested it so that it has four boxes of equal size. Also, you probably noticed the list within the far right column. This is another new feature in Bootstrap 3, called [List Groups](http://getbootstrap.com/components/#list-group).

You can also replace the unordered list with `<div class="list-group">` and then use anchor tags to display links instead of the list items:

```html
<div class="list-group">
  <a href="#" class="list-group-item"><span class="badge">14</span>Quinoa photo booth</a>
  <a href="#" class="list-group-item"><span class="badge">10</span>iPhone kale chips</a>
  <a href="#" class="list-group-item active"><span class="badge">6</span>fanny pack bespoke</a>
  <a href="#" class="list-group-item"><span class="badge">4</span>lo-fi polaroid</a>
  <a href="#" class="list-group-item"><span class="badge">2</span>Tonx meggings stumptown</a>
</div>
```

Test that out. Have some fun with it. Try toying with the background color of the Jumbotron by adding an image, a gradient, or even a basic color: `<div class="jumbotron" style="background-color: #BAE0FF;">`.

Such little changes can make a big difference. Make sure your page looks like [mine](http://www.realpython.com/files/bootstrap3/index.html).

![image](https://www.realpython.com/images/blog_images/bootstrap3-index.png)

## An example: Narrow Marketing Bootstrap 3

Let's take this a step further and create a working example. One of my favorite examples from Bootstrap 2 is the [Narrow Marketing Template](http://getbootstrap.com/2.3.2/examples/marketing-narrow.html), which, sadly, isn't part of the [examples](http://getbootstrap.com/getting-started/#examples) included with Bootstrap 3. So let's create it ourselves from scratch.

Start with a basic page.

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Bootstrap Template</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="css/bootstrap.min.css" rel="stylesheet" media="screen">

    <!-- HTML5 shim and Respond.js IE8 support of HTML5 elements and media queries -->
    <!--[if lt IE 9]>
      <script src="../../assets/js/html5shiv.js"></script>
      <script src="../../assets/js/respond.min.js"></script>
    <![endif]-->
  </head>
  <style>
    body {padding-top: 50px;}
  </style>
  <body>

  <div class="container">

     <h1>Nothing to see .. yet.<h1>

  </div>

  <script src="js/jquery.js"></script>
  <script src="js/bootstrap.min.js"></script>
  </body>
</html>
```

Add the nav bar and update the styles:

```html
<style>
  body {
    padding-top: 20px;
    padding-bottom: 20px;
  }
  .container-narrow {
    margin: 0 auto;
    max-width: 700px;
  }

  .container-narrow > hr {
    margin: 40px 0 0 0;
  }
</style>

<body>

<div class="container-narrow">
  <ul class="nav nav-pills pull-right">
    <li class="active"><a href="#">Home</a></li>
    <li><a href="#">About</a></li>
    <li><a href="#">Contact</a></li>
  </ul>
  <h3 class="text-muted">Project name</h3>
  <hr>
```

Jumbotron. Add the styles first, and then add the Jumbotron below the `<hr>`.

```html
.jumbotron {
  margin: 5px 0;
  text-align: center;
  background-color:white;
}
.jumbotron h1 {
  font-size: 72px;
  line-height: 1;
  font-weight: bold;
}
.jumbotron .btn {
  font-size: 21px;
  padding: 14px 24px;
}

...

<div class="jumbotron">
  <h1>Super awesome marketing speak!</h1>
  <p class="lead">Cras justo odio, dapibus ac facilisis in, egestas eget quam. Fusce dapibus, tellus ac cursus commodo, tortor mauris condimentum nibh, ut fermentum massa justo sit amet risus.</p>
  <a class="btn btn-large btn-success" href="#">Sign up today</a>
</div>
```

Marketing Section. Again, add the styles first, then remove the `<container></container>` element and add in the marketing row.

```html
.marketing {
  margin: 40px 0;
}

.marketing p + h4 {
  margin-top: 28px;
}

...

<div class="row marketing">
  <div class="col-md-6">
    <h4>Subheading</h4>
    <p>Donec id elit non mi porta gravida at eget metus. Maecenas faucibus mollis interdum.</p>

    <h4>Subheading</h4>
    <p>Morbi leo risus, porta ac consectetur ac, vestibulum at eros. Cras mattis consectetur purus sit amet fermentum.</p>

    <h4>Subheading</h4>
    <p>Maecenas sed diam eget risus varius blandit sit amet non magna.</p>
  </div>

  <div class="col-md-6">
    <h4>Subheading</h4>
    <p>Donec id elit non mi porta gravida at eget metus. Maecenas faucibus mollis interdum.</p>

    <h4>Subheading</h4>
    <p>Morbi leo risus, porta ac consectetur ac, vestibulum at eros. Cras mattis consectetur purus sit amet fermentum.</p>

    <h4>Subheading</h4>
    <p>Maecenas sed diam eget risus varius blandit sit amet non magna.</p>
  </div>
</div>
```

Footer. Notice the extra `</div>`. This should close `<div class="container-narrow">`.

```html
<div class="footer">
  <p>&copy; Company 2013</p>
</div>
</div>
```

That's it. Compare this [version](http://www.realpython.com/files/bootstrap3/marketing-narrow-bootstrap3.html) to the Bootstrap 2 [version](http://getbootstrap.com/2.3.2/examples/marketing-narrow.html). Similar. But flat.

![image](https://www.realpython.com/images/blog_images/marketing-narrow-bootstrap3.png)

## Make it your own

Let's look at some quick ways that you can customize this example to make it look less "Bootstrappy".

First, add a new stylesheet called `main.css` and add the embedded styles to it. Don't forget to add the new stylesheet to the `head`, just beneath the first stylesheet. Also, add the following Google fonts as well as a link to the font awesome styles:

```html
<link href="css/main.css" rel="stylesheet" media="screen">
<link href="http://fonts.googleapis.com/css?family=Arvo" rel="stylesheet" type="text/css">
<link href="http://fonts.googleapis.com/css?family=PT+Sans" rel="stylesheet" type="text/css">
<link href="//netdna.bootstrapcdn.com/font-awesome/3.2.1/css/font-awesome.css" rel="stylesheet">
```

Let's put those new fonts to use:

```html
h1, h2, h3, h4, h5, h6 {
  font-family: 'Arvo', Georgia, Times, serif;
}

p, div {
  font-family: 'PT Sans', Helvetica, Arial, sans-serif;
}
```

> As you can probably tell, it's difficult to find two fonts that look good together. Fortunately, there are a lot of good resources - like [here](http://www.briangardner.com/google-font-combinations/) and [here](http://www.mrmcguire.com/10-useful-google-font-combinations-for-your-next-site/).

Textures can make a big difference, which is why I love [Subtle Patterns](http://subtlepatterns.com/). Update the `body` class with the following code. Make sure to make the Jumbotron transparent as well.

```html
body {
  padding-top: 10px;
  padding-bottom: 20px;
  background: url(http://subtlepatterns.com/patterns/lightpaperfibers.png) repeat 0 0;
}

...

.jumbotron {
  margin: 5px 0;
  text-align: center;
  background-color:transparent;
}
```

Update the Marketing section so that there is just one row with three columns (each spanning 4 units). Let's also put the [new](http://getbootstrap.com/components/#glyphicons) Glyphicons to use. Or you can use [Font Awesome](http://fortawesome.github.io/Font-Awesome/) since we added the stylesheet.

```html
<div class="row marketing">
  <div class="col-md-4">
    <h3><span class="glyphicon glyphicon-info-sign" style="color:#428bca;"></span> Subheading</h3>
     <p>Donec id elit non mi porta gravida at eget metus. Maecenas faucibus mollis interdum.</p>
  </div>
  <div class="col-md-4">
    <h3><span class="glyphicon glyphicon-info-sign" style="color:#428bca;"></span> Subheading</h3>
    <p>Morbi leo risus, porta ac consectetur ac, vestibulum at eros. Cras mattis consectetur purus sit amet fermentum.</p>
  </div>
  <div class="col-md-4">
    <h3><span class="glyphicon glyphicon-info-sign" style="color:#428bca;"></span> Subheading</h3>
    <p>Maecenas sed diam eget risus varius blandit sit amet non magna.</p>
  </div>
</div>
```

Above the marketing section let's add a new row with two videos, which hopefully have something to do with your products or services (or at least explains why your site exists).

```html
<div class="row">
  <div class="col-md-6">
    <center><iframe width="320" height="240" style="max-width:100%" src="http://www.youtube.com/embed/D0MoGRZRtcA?rel=0" frameborder="0" allowfullscreen></iframe>
  </div>
  <div class="col-md-6">
    <iframe width="320" height="240" style="max-width:100%" src="http://www.youtube.com/embed/EuOJRop5aBE?rel=0" frameborder="0" allowfullscreen></iframe></center>
  </div>
</div>
```

> By the way, I recorded those videos a few months ago which detail how to get started with a Django Project. Check [them](http://www.youtube.com/watch?v=D0MoGRZRtcA) out, create a basic project, and then add some styles. :)

Update the Jumbotron with some social sharing buttons and a few more icons. <3

```html
<div class="jumbotron">
  <h1>we <span class="glyphicon glyphicon-heart" style="color:#428bca;"></span> awesome marketing speak!</h1>
  <p class="lead">Cras justo odio, dapibus ac facilisis in, egestas eget quam. Fusce dapibus, tellus ac cursus commodo, tortor mauris condimentum nibh, ut fermentum massa justo sit amet risus.</p>
  <a class="btn btn-large btn-success" href="#">Sign up today <span class="glyphicon glyphicon-ok-circle"></span></a>
  <br/><br/>
  <a href="https://twitter.com/RealPython" class="twitter-follow-button" data-show-count="false" data-size="large">Follow @RealPython</a>
  <script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0],p=/^http:/.test(d.location)?'http':'https';if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src=p+'://platform.twitter.com/widgets.js';fjs.parentNode.insertBefore(js,fjs);}}(document, 'script', 'twitter-wjs');
  </script>
  &nbsp;&nbsp;&nbsp;&nbsp;
  <a href="https://twitter.com/intent/tweet?screen_name=RealPython" class="twitter-mention-button" data-size="large">Tweet to @RealPython</a>
  <script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0],p=/^http:/.test(d.location)?'http':'https';if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src=p+'://platform.twitter.com/widgets.js';fjs.parentNode.insertBefore(js,fjs);}}(document, 'script', 'twitter-wjs');
  </script>
  &nbsp;&nbsp;&nbsp;&nbsp;
  <a href="https://twitter.com/share" class="twitter-share-button" data-url="http://RealPython.com" data-text="RealPython yo -" data-size="large">Tweet</a>
  <script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0],p=/^http:/.test(d.location)?'http':'https';if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src=p+'://platform.twitter.com/widgets.js';fjs.parentNode.insertBefore(js,fjs);}}(document, 'script', 'twitter-wjs');
    </script>
</div>
```

Finally, remove the `text-mute` class from the nav bar and add one more icon:

```html
<h3><span class="glyphicon glyphicon-asterisk" style="color:#5cb85c"></span> Project name</h3>
```

And here's the final [version](http://www.realpython.com/files/bootstrap3/marketing-narrow-bootstrap3-updated.html).

![image](https://www.realpython.com/images/blog_images/marketing-narrow-bootstrap3-updated.png)

**Good? At the very least, with these basic changes - which took a good five minutes - you are now well underway to taking a generic theme and making it a bit more professional. Cheers. Ready for more? View the second [part](http://www.realpython.com/blog/design/an-effective-sales-page-with-bootstrap-3) in this three part series.**

Oh - and check out [these](http://startbootstrap.com/) starter templates to get a quick start. Just add new fonts, colors, and a few textures ...
