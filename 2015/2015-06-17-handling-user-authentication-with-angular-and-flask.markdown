# Handling User Authentication with Angular and Flask

**This post provides a solution to the question, "How do I handle user authentication with AngularJS and Flask?".**

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask-angular-auth/flask-angular-auth.png" style="max-width: 100%;" alt="flask-angular-auth">
</div>

<br>

*Updates:*

  - 04/17/2016: Updated to the latest versions of Python ([3.5.1](https://www.python.org/downloads/release/python-351/)) and AngularJS ([1.4.10](https://code.angularjs.org/1.4.10/docs/api)); added a section on persistent logins.

<hr>

Before beginning, keep in mind that this is not the *only* solution to the question at hand, and it may not even be the *right* solution for your situation. Regardless of the solution you implement, it is important to note that *since end users have full control of the browser as well as access to the front-end code, sensitive data living in your server-side API must be secure. In other words, make certain that you implement an authentication strategy on the server-side to protect sensitive API endpoints.*

That said, we need to enable the following workflow:

1. When the client accesses the main route, an index page is served, at which point Angular takes over, handling all routing on the client-side.
1. The Angular app immediately "asks" the server if a user is logged in.
1. Assuming the server indicates that a user is not logged in, the client is immediately asked to log in.
1. Once logged in, the Angular app then tracks the user's login status.

## Getting Started

First, grab the boilerplate code from the [repo](https://github.com/realpython/flask-angular-auth/releases/tag/v1.1), activate a virtual environment, and install the requirements.

Then create the initial migration:

```sh
$ python manage.py create_db
$ python manage.py db init
$ python manage.py db migrate
```

Test out the app:

```sh
$ python manage.py runserver
```

Navigate to [http://localhost:5000/](http://localhost:5000/) and you should see a simple welcome message - "Welcome!". Once done admiring the main page, kill the server, and glance over the code within the "project" folder:

```sh
├── __init__.py
├── config.py
├── models.py
└── static
    ├── app.js
    ├── index.html
    └── partials
        └── home.html
```

Nothing too spectacular. The majority of the back-end code/logic resides in the *\_\_init\_\_.py* file, while the Angular app resides in the "static" directory.

> For more on this structure, please check out the [Real Python](https://realpython.com) course.

## Building the Login API

Let's start with the back-end API...

### User Registration

Update the `register()` function in *\_\_init\_\_.py*:

```python
@app.route('/api/register', methods=['POST'])
def register():
    json_data = request.json
    user = User(
        email=json_data['email'],
        password=json_data['password']
    )
    try:
        db.session.add(user)
        db.session.commit()
        status = 'success'
    except:
        status = 'this user is already registered'
    db.session.close()
    return jsonify({'result': status})
```

Here, we set the payload sent with the POST request (from the client-side) to `json_data`, which is then used to create a `User` instance. We then attempted to add the user to the database and commit the changes. If this succeeds a user is added, and then we return a JSON response via the `jsonify` [method](http://flask.pocoo.org/docs/0.10/api/#flask.json.jsonify) method with a `status` of "success". If it fails, the session is closed and an error response, "this user is already registered", is sent.

Make sure to add the following imports as well:

```python
from flask import request, jsonify
from project.models import User
```

> The latter import must be imported after we create the `db` instance - e.g., `db = SQLAlchemy(app)` - to avoid a circular dependency.

Let's test this via curl. Fire up the server and then run the following command in a new terminal window:

```sh
$ curl -H "Accept: application/json" \
-H "Content-type: application/json" -X POST \
-d '{"email": "test@test.com", "password": "test"}' \
http://localhost:5000/api/register
```

You should see a success message:

```sh
{
  "result": "success"
}
```

Try it again, and you should see an error:

```sh
{
  "result": "this user is already registered"
}
```

Finally, open the database in the [SQLite Database Browser](http://sqlitebrowser.org/) to ensure the user did get INSERTed into the table:

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/flask-angular-auth/sqlite-database-browser-added-user.png" style="max-width: 100%;" alt="flask-angular-auth">
</div>

<br>

On to the login...

### User Login

Update the `login()` function in *\_\_init\_\_.py*:

```python
@app.route('/api/login', methods=['POST'])
def login():
    json_data = request.json
    user = User.query.filter_by(email=json_data['email']).first()
    if user and bcrypt.check_password_hash(
            user.password, json_data['password']):
        session['logged_in'] = True
        status = True
    else:
        status = False
    return jsonify({'result': status})
```

We queried the database to see if a user exists, based on the email sent in the payload, and, if so, the password is then verified. The appropriate response is returned.

Make sure to update the imports:

```python
from flask import Flask, request, jsonify, session
```

With the server running, test again with curl-

```sh
$ curl -H "Accept: application/json" \
-H "Content-type: application/json" -X POST \
-d '{"email": "test@test.com", "password": "test"}' \
http://localhost:5000/api/login
```

-and you should see:

```sh
{
  "result": true
}
```

Test again with curl, sending the wrong user credentials, and you should see:

```sh
{
  "result": false
}
```

Perfect!

### User Logout

Update the `logout()` function like so, in order to update the `session`:

```python
@app.route('/api/logout')
def logout():
    session.pop('logged_in', None)
    return jsonify({'result': 'success'})
```

This should be straightforward, and you can probably guess the response to this curl request - but test again if you'd like. Once done, let's move to the client-side!

## Developing the Angular App

> Need the code from the previous section? Grab it from the [repo](https://github.com/realpython/flask-angular-auth/releases/tag/v2.1).

Now, here's where things get a bit tricky. Again, since end users have full access to the power of the browser as well as [DevTools](https://developer.chrome.com/devtools) and the client-side code, it's vital that you not only restrict access to sensitive endpoints on the server-side - but that you also not store sensitive data on the client-side. Keep this in mind as you add auth functionality to your own application stack.

Let's jump right in by creating a [service](https://code.angularjs.org/1.4.10/docs/guide/services) to handle authentication.

### Authentication Service

Start with the basic structure of this service by adding the following code to a new file called *services.js* in the "static" directory:

```javascript
angular.module('myApp').factory('AuthService',
  ['$q', '$timeout', '$http',
  function ($q, $timeout, $http) {

    // create user variable
    var user = null;

    // return available functions for use in controllers
    return ({
      isLoggedIn: isLoggedIn,
      login: login,
      logout: logout,
      register: register
    });

}]);
```

Here, we defined the service name, `AuthService`, and injected the dependencies that we will be using - `$q`, `$timeout`, `$http` - and then returned the functions for use outside the service.

Make sure to add the script to the *index.html* file:

```html
<script src="static/services.js" type="text/javascript"></script>
```

Let's create each function...

**`isLoggedIn()`**

```javascript
function isLoggedIn() {
  if(user) {
    return true;
  } else {
    return false;
  }
}
```

This function returns `true` if `user` evaluates to `true` - e.g., a user is logged in - otherwise it returns false.

**`login()`**

```javascript
function login(email, password) {

  // create a new instance of deferred
  var deferred = $q.defer();

  // send a post request to the server
  $http.post('/api/login', {email: email, password: password})
    // handle success
    .success(function (data, status) {
      if(status === 200 && data.result){
        user = true;
        deferred.resolve();
      } else {
        user = false;
        deferred.reject();
      }
    })
    // handle error
    .error(function (data) {
      user = false;
      deferred.reject();
    });

  // return promise object
  return deferred.promise;

}
```

Here, we used the [$q](https://code.angularjs.org/1.4.10/docs/api/ng/service/$q) service to set up a [promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise), which we'll access in a future controller. We also utilized the [$http](https://code.angularjs.org/1.4.10/docs/api/ng/service/$http) service to send an AJAX request to the `/api/login` endpoint that we already set up in our back-end Flask app.

Based on the returned response, we either [resolve](https://code.angularjs.org/1.4.10/docs/api/ng/service/$q#usage) or [reject](https://code.angularjs.org/1.4.10/docs/api/ng/service/$q#usage) the object and set the value of `user` to `true` or `false`.

**`logout()`**

```javascript
function logout() {

  // create a new instance of deferred
  var deferred = $q.defer();

  // send a get request to the server
  $http.get('/api/logout')
    // handle success
    .success(function (data) {
      user = false;
      deferred.resolve();
    })
    // handle error
    .error(function (data) {
      user = false;
      deferred.reject();
    });

  // return promise object
  return deferred.promise;

}
```

Here, we followed the same formula as the `login()` function, except we sent a GET request rather than a POST and, to be cautious, instead of sending an error if the user does not exist, we just logged the user out.

**`register()`**

```javascript
function register(email, password) {

  // create a new instance of deferred
  var deferred = $q.defer();

  // send a post request to the server
  $http.post('/api/register', {email: email, password: password})
    // handle success
    .success(function (data, status) {
      if(status === 200 && data.result){
        deferred.resolve();
      } else {
        deferred.reject();
      }
    })
    // handle error
    .error(function (data) {
      deferred.reject();
    });

  // return promise object
  return deferred.promise;

}
```

Again, we followed a similar formula to the `logout()` function. Can you tell what's happening?

That's it for the service. Keep in mind that we still have not "used" this service. In order to do that, we just need to inject it into the necessary components in the Angular app. In our case, that will be the controllers, each associated with a different route...

### Client-side Routing

Add the remainder of the client-side routes to the *app.js* file:

```javascript
myApp.config(function ($routeProvider) {
  $routeProvider
    .when('/', {
      templateUrl: 'static/partials/home.html'
    })
    .when('/login', {
      templateUrl: 'static/partials/login.html',
      controller: 'loginController'
    })
    .when('/logout', {
      controller: 'logoutController'
    })
    .when('/register', {
      templateUrl: 'static/partials/register.html',
      controller: 'registerController'
    })
    .when('/one', {
      template: '<h1>This is page one!</h1>'
    })
    .when('/two', {
      template: '<h1>This is page two!</h1>'
    })
    .otherwise({
      redirectTo: '/'
    });
});
```

Here, we created five new routes. Now we can add the subsequent templates and controllers.

### Templates and Controllers

Looking back at our routes, we need to setup two partials/templates and three controllers:

```javascript
.when('/login', {
  templateUrl: 'static/partials/login.html',
  controller: 'loginController'
})
.when('/logout', {
  controller: 'logoutController'
})
.when('/register', {
  templateUrl: 'static/partials/register.html',
  controller: 'registerController'
})
```

**Login**

First, add the following HTML to a new file called *login.html*:

```html
<div class="col-md-4">
  <h1>Login</h1>
  <div ng-show="error" class="alert alert-danger">{% raw %}{{errorMessage}}{% endraw %}</div>
  <form class="form" ng-submit="login()">
    <div class="form-group">
      <label>Email</label>
      <input type="text" class="form-control" name="email" ng-model="loginForm.email" required>
    </div>
    <div class="form-group">
      <label>Password</label>
        <input type="password" class="form-control" name="password" ng-model="loginForm.password" required>
      </div>
      <div>
        <button type="submit" class="btn btn-default" ng-disabled="disabled">Login</button>
      </div>
  </form>
</div>
```

Add this file to the "partials" directory.

Take note of the form. We used the [ng-model](https://code.angularjs.org/1.4.10/docs/api/ng/directive/ngModel) directive on each of the inputs so that we can capture those values in the controller. Also, when the form is submitted, the [ng-submit](https://code.angularjs.org/1.4.10/docs/api/ng/directive/ngSubmit) directive handles the event by firing the `login()` function.

Next, within the "static" folder and add a new file called *controllers.js*. Yes, this will hold all of our Angular app's controllers. Be sure to add the script to the *index.html* file:

```html
<script src="static/controllers.js" type="text/javascript"></script>
```

Now, let's add the first controller:

```javascript
angular.module('myApp').controller('loginController',
  ['$scope', '$location', 'AuthService',
  function ($scope, $location, AuthService) {

    $scope.login = function () {

      // initial values
      $scope.error = false;
      $scope.disabled = true;

      // call login from service
      AuthService.login($scope.loginForm.email, $scope.loginForm.password)
        // handle success
        .then(function () {
          $location.path('/');
          $scope.disabled = false;
          $scope.loginForm = {};
        })
        // handle error
        .catch(function () {
          $scope.error = true;
          $scope.errorMessage = "Invalid username and/or password";
          $scope.disabled = false;
          $scope.loginForm = {};
        });

    };

}]);
```

So, when the `login()` function is fired, we set some initial values and then call `login()` from the `AuthService`, passing the user inputed email and password as arguments. The subsequent success or error is then handled and the DOM/view/template is updated appropriately.

Ready to test the first round-trip - *client to server and then back again to client*?

Fire up the server and navigate to [http://localhost:5000/#/login](http://localhost:5000/#/login) in your browser. First, try logging in with the user credentials used to register earlier - e.g, `test@test.com` and `test`. If all went well, you should be redirected to the main URL. Next, try to log in using invalid credentials. You should see the error message flash, "Invalid username and/or password".

**Logout**

Add the controller:

```javascript
angular.module('myApp').controller('logoutController',
  ['$scope', '$location', 'AuthService',
  function ($scope, $location, AuthService) {

    $scope.logout = function () {

      // call logout from service
      AuthService.logout()
        .then(function () {
          $location.path('/login');
        });

    };

}]);
```

Here, we called `AuthService.logout()` and then redirected the user to the `/login` route after the promise is resolved.

Add a button to *home.html*:

```html
<div ng-controller="logoutController">
  <a ng-click='logout()' class="btn btn-default">Logout</a>
</div>
```

And then test it out again.

**Register**

Add a new new file called *register.html* to the "partials" folder and add the following HTML:

```html
<div class="col-md-4">
  <h1>Register</h1>
  <div ng-show="error" class="alert alert-danger">{% raw %}{{errorMessage}}{% endraw %}</div>
  <form class="form" ng-submit="register()">
    <div class="form-group">
      <label>Email</label>
      <input type="text" class="form-control" name="email" ng-model="registerForm.email" required>
    </div>
    <div class="form-group">
      <label>Password</label>
        <input type="password" class="form-control" name="password" ng-model="registerForm.password" required>
      </div>
      <div>
        <button type="submit" class="btn btn-default" ng-disabled="disabled">Register</button>
      </div>
  </form>
</div>
```

Next, add the controller:

```javascript
angular.module('myApp').controller('registerController',
  ['$scope', '$location', 'AuthService',
  function ($scope, $location, AuthService) {

    $scope.register = function () {

      // initial values
      $scope.error = false;
      $scope.disabled = true;

      // call register from service
      AuthService.register($scope.registerForm.email,
                           $scope.registerForm.password)
        // handle success
        .then(function () {
          $location.path('/login');
          $scope.disabled = false;
          $scope.registerForm = {};
        })
        // handle error
        .catch(function () {
          $scope.error = true;
          $scope.errorMessage = "Something went wrong!";
          $scope.disabled = false;
          $scope.registerForm = {};
        });

    };

}]);
```

You've seen this before, so let's move right on to testing.

Fire up the server and register a new user at [http://localhost:5000/#/register](http://localhost:5000/#/register). Make sure to test logging in with that new user as well.

Well, that's it for the templates and controllers. We now need to add in functionality to check if a user is logged in on each and every change of route.

### Route Changes

Add the following code to *app.js*:

```javascript
myApp.run(function ($rootScope, $location, $route, AuthService) {
  $rootScope.$on('$routeChangeStart', function (event, next, current) {
    if (AuthService.isLoggedIn() === false) {
      $location.path('/login');
      $route.reload();
    }
  });
});
```

The [$routeChangeStart](https://code.angularjs.org/1.4.10/docs/api/ngRoute/service/$route) event happens before the actual route change occurs. So, whenever a route is accessed, before the view is served, we ensure that the user is logged in. Test this out!

### Protect Certain Routes

Right now all client-side routes require a user to be logged in. What if you want certain routes restricted and other routes open? You can add the following code to each route handler, replacing `true` with `false` for routes that you do not want to restrict:

```javascript
access: {restricted: true}
```

In our case, update the routes like so:

```javascript
myApp.config(function ($routeProvider) {
  $routeProvider
    .when('/', {
      templateUrl: 'static/partials/home.html',
      access: {restricted: true}
    })
    .when('/login', {
      templateUrl: 'static/partials/login.html',
      controller: 'loginController',
      access: {restricted: false}
    })
    .when('/logout', {
      controller: 'logoutController',
      access: {restricted: true}
    })
    .when('/register', {
      templateUrl: 'static/partials/register.html',
      controller: 'registerController',
      access: {restricted: false}
    })
    .when('/one', {
      template: '<h1>This is page one!</h1>',
      access: {restricted: true}
    })
    .when('/two', {
      template: '<h1>This is page two!</h1>',
      access: {restricted: false}
    })
    .otherwise({
      redirectTo: '/'
    });
});
```

Now just update the `$routeChangeStart` code in *main.js*:

```javascript
myApp.run(function ($rootScope, $location, $route, AuthService) {
  $rootScope.$on('$routeChangeStart', function (event, next, current) {
    if (next.access.restricted && AuthService.isLoggedIn() === false) {
      $location.path('/login');
      $route.reload();
    }
  });
});
```

Test each route out!

## Persistant Login

Finally, what happens on a page refresh? Try it.

The user is logged out, right? Why? Because the controller and services are called again, setting the `user` variable to `null`. This is a problem since the user is still logged in on the client-side.

Fortunately, the fix is simple: Within the `$routeChangeStart` we need to ALWAYS check if a user is logged in. Right now, it's checking whether `isLoggedIn()` is `false`. Let's add a new function, `getUserStatus()`, that checks the user status on the back-end:

```javascript
function getUserStatus() {
  return $http.get('/api/status')
  // handle success
  .success(function (data) {
    if(data.status){
      user = true;
    } else {
      user = false;
    }
  })
  // handle error
  .error(function (data) {
    user = false;
  });
}
```

Make sure to return the function as well:

```javascript
return ({
  isLoggedIn: isLoggedIn,
  login: login,
  logout: logout,
  register: register,
  getUserStatus: getUserStatus
});
```

Then add the route handler on the client-side:

```python
@app.route('/api/status')
def status():
    if session.get('logged_in'):
        if session['logged_in']:
            return jsonify({'status': True})
    else:
        return jsonify({'status': False})
```

Finally, update the `$routeChangeStart`:

```javascript
myApp.run(function ($rootScope, $location, $route, AuthService) {
  $rootScope.$on('$routeChangeStart',
    function (event, next, current) {
      AuthService.getUserStatus()
      .then(function(){
        if (next.access.restricted && !AuthService.isLoggedIn()){
          $location.path('/login');
          $route.reload();
        }
      });
  });
});
```

Try it out!

## Conclusion

That's it. Questions? Comment below.

One thing you should note is that the Angular app can be used with various frameworks as long as the endpoints are set up correctly in the AJAX requests. So, you can easily take the Angular portion and add it to your Django or Pyramid or NodeJS app. Try it!

Grab the final code from the [repo](https://github.com/realpython/flask-angular-auth). Cheers!