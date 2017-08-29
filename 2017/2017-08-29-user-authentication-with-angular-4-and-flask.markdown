# User Authentication with Angular 4 and Flask

**In this tutorial, we'll demonstrate how to set up token-based authentication (via JSON Web Tokens) with Angular 4 and Flask.**

<br>

<div class="center-text">
  <img class="no-border" src="/images/blog_images/angular4-auth/angular4-auth.png" style="max-width: 100%;" alt="angular4 auth">
</div>

*Main Dependencies*:

1. Angular v[4.2.4](https://github.com/angular/angular/releases/tag/4.2.4) (via Angular CLI v[1.3.2](https://github.com/angular/angular-cli/releases/tag/v1.3.2))
1. Flask v[0.12](http://flask.pocoo.org/docs/0.12/changelog/#version-0-12)
1. Python v[3.6.2](https://www.python.org/downloads/release/python-362/)

## Auth Workflow

Here's the full user auth process:

1. Client logs in and the credentials are sent to the server
2. If the credentials are correct, the server generates a token and sends it as a response to the client
3. Client receives and stores the token in Local Storage
4. Client then sends token to server on subsequent requests within the request header

## Project Setup

Start by globally installing the [Angular CLI](https://cli.angular.io/):

```sh
$ npm install -g @angular/cli@1.3.2
```

Then generate a new Angular 4 project boilerplate:

```sh
$ ng new angular4-auth
```

Fire up the app after the dependencies install:

```sh
$ cd angular4-auth
$ ng serve
```

It will probably take a minute or two to compile and build your application. Once done, navigate to [http://localhost:4200](http://localhost:4200) to ensure the app is up and running.

Open up the project in your favorite code editor, and then glance over the code:

```sh
├── e2e
│   ├── app.e2e-spec.ts
│   ├── app.po.ts
│   └── tsconfig.e2e.json
├── karma.conf.js
├── package.json
├── protractor.conf.js
├── src
│   ├── app
│   │   ├── app.component.css
│   │   ├── app.component.html
│   │   ├── app.component.spec.ts
│   │   ├── app.component.ts
│   │   └── app.module.ts
│   ├── assets
│   ├── environments
│   │   ├── environment.prod.ts
│   │   └── environment.ts
│   ├── favicon.ico
│   ├── index.html
│   ├── main.ts
│   ├── polyfills.ts
│   ├── styles.css
│   ├── test.ts
│   ├── tsconfig.app.json
│   ├── tsconfig.spec.json
│   └── typings.d.ts
├── tsconfig.json
└── tslint.json
```

In short, the client-side code lives in the "src" folder and the Angular app itself can be found in the "app" folder.

Take note of the `AppModule` within *app.module.ts*. This is used to bootstrap the Angular app. The `@NgModule` decorator takes metadata that lets Angular know how to run the app. Everything that we create in this tutorial will be added to this object.

Make sure you have a decent grasp of the app structure before moving on.

> **NOTE:** Just getting started with Angular 4? Review the [Angular Style Guide](https://angular.io/guide/styleguide), since the app generated from the CLI follows the recommended structure from that guide, as well as the [Angular4Crud Tutorial](https://github.com/mjhea0/angular4-crud/blob/master/tutorial.md).

Did you notice that the CLI initialized a new Git repo? This part is optional, but it's a good idea to create a new Github repository and update the remote:

```sh
$ git remote set-url origin <newurl>
```

Now, let's wire up a new [component](https://angular.io/guide/architecture#components)...

## Auth Component

First, use the CLI to generate a new Login component:

```sh
$ ng generate component components/login
```

This set up the component files and folders and even wired it up to *app.module.ts*. Next, let's change the *login.component.ts* file to the following:

```javascript
import { Component } from '@angular/core';

@Component({
  selector: 'login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css']
})

export class LoginComponent {
  test: string = 'just a test';
}
```

If you haven't used [TypeScript](https://www.typescriptlang.org/) before, then this code probably looks pretty foreign to you. TypeScript is a statically-typed superset of JavaScript that compiles to vanilla JavaScript, and it is the de facto programming language for building Angular 4 apps.

In Angular 4, we define a *component* by wrapping a config object with an `@Component` decorator. We can share code between packages by importing the classes we need; and, in this case, we import `Component` from the `@angular/core` package. The `LoginComponent` class is the component's controller, and we use the `export` operator to make it available for other classes to import.

Add the following HTML to the *login.component.html*:

{% raw %}
```html
<h1>Login</h1>

<p>{{test}}</p>
```
{% endraw %}

Next, configure the routes, via the [RouterModule](https://angular.io/api/router/RouterModule) in the *app.module.ts* file:

```javascript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';

import { AppComponent } from './app.component';
import { LoginComponent } from './components/login/login.component';

@NgModule({
  declarations: [
    AppComponent,
    LoginComponent,
  ],
  imports: [
    BrowserModule,
    RouterModule.forRoot([
      { path: 'login', component: LoginComponent }
    ])
  ],
  providers: [],
  bootstrap: [AppComponent]
})

export class AppModule { }
```

Finish enabling routing by replacing all HTML in the *app.component.html* file with the `<router-outlet>` tag:

```html
<router-outlet></router-outlet>
```

Run `ng serve` in your terminal, if you haven't already, and then navigate to [http://localhost:4200/login](http://localhost:4200/login). If all went well you should see the `just a test` text.

## Bootstrap Style

To quickly add some style, update the *index.html*, adding in [Bootstrap](http://getbootstrap.com/) and wrapping the `<app-root></app-root>` in a [container](http://getbootstrap.com/css/#overview-container):

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Angular4Auth</title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
  <link rel="stylesheet" href="//maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css">
</head>
<body>
  <div class="container">
    <app-root></app-root>
  </div>
</body>
</html>
```

You should see the app auto reload as soon as you save.

## Auth Service

Next, let's create a global [service](https://angular.io/guide/architecture#services) to handle a user logging in, logging out, and signing up:

```sh
$ ng generate service services/auth
```

Edit the *auth.service.ts* so that it has the following code:

```javascript
import { Injectable } from '@angular/core';

@Injectable()
export class AuthService {
  test(): string {
    return 'working';
  }
}
```

Remember how providers worked in Angular 1? They were global objects that stored a single state. When the data in a provider changed, any object that had injected that provider would receive the updates. In Angular 4, providers retain their special behavior and they are defined with the `@Injectable` decorator.

### Sanity Check

Before adding anything significant to `AuthService`, let's make sure the service itself is wired up correctly. To do that, within *login.component.ts* inject the service and call the `test()` method:

```javascript
import { Component, OnInit } from '@angular/core';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css']
})

export class LoginComponent implements OnInit {
  test: string = 'just a test';
  constructor(private auth: AuthService) {}
  ngOnInit(): void {
    console.log(this.auth.test());
  }
}
```

We've introduced some new concepts and keywords. The `constructor()` function is a special method that we use to set up a new instance of a class. The `constructor()` is where we pass any parameters that the class requires, including any providers (i.e., `AuthService`) that we want to inject. In TypeScript, we can hide variables from the outside world with the `private` keyword. Passing a `private` variable in the constructor is a shortcut to defining it within the class and then assigning the argument's value to it. Notice how the `auth` variable is accessible to the `this` object after it is passed into the constructor.

We implement the `OnInit` interface to ensure that we explicitly define a `ngOnInit()` function. Implementing `OnInit` ensures that our component will be called after the first change detection check. This function is called once when the component first initializes, making it the ideal place to configure data that relies on other Angular classes.

Unlike components, which are automatically added, services have to be manually imported and configured on the `@NgModule`. So, to get it working, you'll also have to import the `AuthService` in *app.module.ts* and add it to the `providers`:

```javascript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';

import { AppComponent } from './app.component';
import { LoginComponent } from './components/login/login.component';
import { AuthService } from './services/auth.service';

@NgModule({
  declarations: [
    AppComponent,
    LoginComponent,
  ],
  imports: [
    BrowserModule,
    RouterModule.forRoot([
      { path: 'login', component: LoginComponent }
    ])
  ],
  providers: [AuthService],
  bootstrap: [AppComponent]
})

export class AppModule { }
```

Run the server and then navigate to [http://localhost:4200/login](http://localhost:4200/login). You should see `working` logged to the JavaScript console.

### User Login

To handle logging a user in, update the `AuthService` like so:

```javascript
import { Injectable } from '@angular/core';
import { Headers, Http } from '@angular/http';
import 'rxjs/add/operator/toPromise';

@Injectable()
export class AuthService {
  private BASE_URL: string = 'http://localhost:5000/auth';
  private headers: Headers = new Headers({'Content-Type': 'application/json'});
  constructor(private http: Http) {}
  login(user): Promise<any> {
    let url: string = `${this.BASE_URL}/login`;
    return this.http.post(url, user, {headers: this.headers}).toPromise();
  }
}
```

We employ the help of some built-in Angular classes, `Headers` and `Http`, to handle our AJAX calls to the server.

Also, update the *app.module.ts* file to import the `HttpModule`.

```javascript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';
import { HttpModule } from '@angular/http';

import { AppComponent } from './app.component';
import { LoginComponent } from './components/login/login.component';
import { AuthService } from './services/auth.service';

@NgModule({
  declarations: [
    AppComponent,
    LoginComponent,
  ],
  imports: [
    BrowserModule,
    HttpModule,
    RouterModule.forRoot([
      { path: 'login', component: LoginComponent }
    ])
  ],
  providers: [AuthService],
  bootstrap: [AppComponent]
})

export class AppModule { }
```

Here, we are using the Http service to send an AJAX request to the `/user/login` endpoint. This returns a promise object.

> **NOTE:** Make sure to remove `console.log(this.auth.test());` from the `LoginComponent` component.

### User Registration

Let's go ahead and add the ability to register a user as well, which is similar to logging a user in. Update *src/app/services/auth.service.ts*, taking note of the `register` method:

```javascript
import { Injectable } from '@angular/core';
import { Headers, Http } from '@angular/http';
import 'rxjs/add/operator/toPromise';

@Injectable()
export class AuthService {
  private BASE_URL: string = 'http://localhost:5000/auth';
  private headers: Headers = new Headers({'Content-Type': 'application/json'});
  constructor(private http: Http) {}
  login(user): Promise<any> {
    let url: string = `${this.BASE_URL}/login`;
    return this.http.post(url, user, {headers: this.headers}).toPromise();
  }
  register(user): Promise<any> {
    let url: string = `${this.BASE_URL}/register`;
    return this.http.post(url, user, {headers: this.headers}).toPromise();
  }
}
```

Now, to test this we need to set up a back end...

## Server-side Setup

For the server-side, we'll use the finished project from a previous blog post, [Token-Based Authentication With Flask](https://realpython.com/blog/python/token-based-authentication-with-flask/). You can view the code from the [flask-jwt-auth](https://github.com/realpython/flask-jwt-auth) repository.

> **NOTE:** Feel free to use your own server, just make sure to update the `baseURL` in the `AuthService`.

Clone the project structure in a new terminal window:

```sh
$ git clone https://github.com/realpython/flask-jwt-auth
```

Follow the directions in the [README](https://github.com/realpython/flask-jwt-auth/blob/master/README.md) to set up the project, making sure the tests pass before moving on. Once done, run the server with `python manage.py runserver`, which will listen on port 5000.

## Sanity Check

To test, update `LoginComponent` to use the `login` and `register` methods from the service:

```javascript
import { Component, OnInit } from '@angular/core';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css']
})
export class LoginComponent implements OnInit {
  test: string = 'just a test';
  constructor(private auth: AuthService) {}
  ngOnInit(): void {
    let sampleUser: any = {
      email: 'michael@realpython.com' as string,
      password: 'michael' as string
    };
    this.auth.register(sampleUser)
    .then((user) => {
      console.log(user.json());
    })
    .catch((err) => {
      console.log(err);
    });
    this.auth.login(sampleUser).then((user) => {
      console.log(user.json());
    })
    .catch((err) => {
      console.log(err);
    });
  }
}
```

Refresh [http://localhost:4200/login](http://localhost:4200/login) in the browser and you should see a success in the JavaScript console, after the user is logged in, with the token:

```json
{
  "auth_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1M…jozfQ.bPNQb3C98yyNe0LDyl1Bfkp0Btn15QyMxZnBoE9RQMI",
  "message": "Successfully logged in.",
  "status": "success"
}
```

## Auth Login

Update *login.component.html*:

```html
<div class="row">
  <div class="col-md-4">
    <h1>Login</h1>
    <hr><br>
    <form (ngSubmit)="onLogin()" novalidate>
     <div class="form-group">
       <label for="email">Email</label>
       <input type="text" class="form-control" id="email" placeholder="enter email" [(ngModel)]="user.email" name="email" required>
     </div>
     <div class="form-group">
       <label for="password">Password</label>
       <input type="password" class="form-control" id="password" placeholder="enter password" [(ngModel)]="user.password" name="password" required>
     </div>
     <button type="submit" class="btn btn-default">Submit</button>
    </form>
  </div>
</div>
```

Take note of the form. We used the `[(ngModel)]` directive on each of the form inputs to capture those values in the controller. Also, when the form is submitted, the `ngSubmit` directive handles the event by firing the `onLogin()` method.

Now, let's update the component code, adding in `onLogin()`:

```javascript
import { Component } from '@angular/core';
import { AuthService } from '../../services/auth.service';
import { User } from '../../models/user';

@Component({
  selector: 'login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css']
})
export class LoginComponent {
  user: User = new User();
  constructor(private auth: AuthService) {}
  onLogin(): void {
    this.auth.login(this.user)
    .then((user) => {
      console.log(user.json());
    })
    .catch((err) => {
      console.log(err);
    });
  }
}
```

If you have the Angular web server running, you should see the error `Cannot find module '../../models/user'` in the browser. Before our code will work, we need to create a `User` model.

```sh
$ ng generate class models/user
```

Update *src/app/models/user.ts*:

```javascript
export class User {
  constructor(email?: string, password?: string) {}
}
```

Our `User` model has two properties, `email` and `password`. The `?` character is a special operator that indicates that initializing `User` with explicit `email` and `password` values is optional. This is equivalent to the following class in Python:

```python
class User(object):
    def __init__(self, email=None, password=None):
        self.email = email
        self.password = password
```

Don't forget to update *auth.service.ts* to use the new object.

```javascript
import { Injectable } from '@angular/core';
import { Headers, Http } from '@angular/http';
import { User } from '../models/user';
import 'rxjs/add/operator/toPromise';

@Injectable()
export class AuthService {
  private BASE_URL: string = 'http://localhost:5000/auth';
  private headers: Headers = new Headers({'Content-Type': 'application/json'});
  constructor(private http: Http) {}
  login(user: User): Promise<any> {
    let url: string = `${this.BASE_URL}/login`;
    return this.http.post(url, user, {headers: this.headers}).toPromise();
  }
  register(user: User): Promise<any> {
    let url: string = `${this.BASE_URL}/register`;
    return this.http.post(url, user, {headers: this.headers}).toPromise();
  }
}
```

One last thing. We need to import the `FormsModule` in the *app.module.ts* file.

```javascript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';
import { HttpModule } from '@angular/http';
import { FormsModule } from '@angular/forms';

import { AppComponent } from './app.component';
import { LoginComponent } from './components/login/login.component';
import { AuthService } from './services/auth.service';

@NgModule({
  declarations: [
    AppComponent,
    LoginComponent,
  ],
  imports: [
    BrowserModule,
    HttpModule,
    FormsModule,
    RouterModule.forRoot([
      { path: 'login', component: LoginComponent }
    ])
  ],
  providers: [AuthService],
  bootstrap: [AppComponent]
})

export class AppModule { }
```

So, when the form is submitted, we capture the email and password and pass them to the `login()` method on the service.

Test this out with-

- email: `michael@realpython.com`
- password: `michael`

Again, you should see a success in the javaScript console with the token.

## Auth Register

Just like for the login functionality, we need to add a component for registering a user. Start by generating a new Register component:

```sh
$ ng generate component components/register
```

Update *src/app/components/register/register.component.html*

```html
<div class="row">
  <div class="col-md-4">
    <h1>Register</h1>
    <hr><br>
    <form (ngSubmit)="onRegister()" novalidate>
     <div class="form-group">
       <label for="email">Email</label>
       <input type="text" class="form-control" id="email" placeholder="enter email" [(ngModel)]="user.email" name="email" required>
     </div>
     <div class="form-group">
       <label for="password">Password</label>
       <input type="password" class="form-control" id="password" placeholder="enter password" [(ngModel)]="user.password" name="password" required>
     </div>
     <button type="submit" class="btn btn-default">Submit</button>
    </form>
  </div>
</div>
```

Then, update *src/app/components/register/register.component.ts* as follows:


```javascript
import { Component } from '@angular/core';
import { AuthService } from '../../services/auth.service';
import { User } from '../../models/user';

@Component({
  selector: 'register',
  templateUrl: './register.component.html',
  styleUrls: ['./register.component.css']
})
export class RegisterComponent {
  user: User = new User();
  constructor(private auth: AuthService) {}
  onRegister(): void {
    this.auth.register(this.user)
    .then((user) => {
      console.log(user.json());
    })
    .catch((err) => {
      console.log(err);
    });
  }
}
```

Add a new route handler to the *app.module.ts* file:

```javascript
RouterModule.forRoot([
  { path: 'login', component: LoginComponent },
  { path: 'register', component: RegisterComponent }
])
```

Test it out by registering a new user!

## Local Storage

Next, let's add the token to Local Storage for persistence by replacing the `console.log(user.json());` with `localStorage.setItem('token', user.data.token);` in *src/app/components/login/login.component.ts*:

```javascript
onLogin(): void {
  this.auth.login(this.user)
  .then((user) => {
    localStorage.setItem('token', user.json().auth_token);
  })
  .catch((err) => {
    console.log(err);
  });
}
```

Do the same within *src/app/components/register/register.component.ts*:

```javascript
onRegister(): void {
  this.auth.register(this.user)
  .then((user) => {
    localStorage.setItem('token', user.json().auth_token);
  })
  .catch((err) => {
    console.log(err);
  });
}
```

As long as that token is present, the user can be considered logged in. And, when a user needs to make an AJAX request, that token can be used.

> **NOTE:** Besides the token, you could also add the user id and email to Local Storage. You would just need to update the server-side to send back that info when a user logs in.

Test this out. Ensure that the token is present in Local Storage after you log in.

## User Status

To test out login persistence, we can add a new view that verifies that the user is logged in and that the token is valid.

Add the following method to `AuthService`:

```javascript
ensureAuthenticated(token): Promise<any> {
  let url: string = `${this.BASE_URL}/status`;
  let headers: Headers = new Headers({
    'Content-Type': 'application/json',
    Authorization: `Bearer ${token}`
  });
  return this.http.get(url, {headers: headers}).toPromise();
}
```

Take note of `Authorization: 'Bearer ' + token`. This is called a [Bearer schema](https://security.stackexchange.com/questions/108662/why-is-bearer-required-before-the-token-in-authorization-header-in-a-http-re), which is sent along with the request. On the server, we are simply checking for the `Authorization` header, and then whether the token is valid. Can you find this code on the server-side?

Then, generate a new Status component:

```sh
$ ng generate component components/status
```

Create the HTML template, *src/app/components/status/status.component.html*:

{% raw %}
```html
<div class="row">
  <div class="col-md-4">
    <h1>User Status</h1>
    <hr><br>
    <p>Logged In? {{isLoggedIn}}</p>
  </div>
</div>
```
{% endraw %}

And change the component code in *src/app/components/status/status.component.ts*:

```javascript
import { Component, OnInit } from '@angular/core';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'status',
  templateUrl: './status.component.html',
  styleUrls: ['./status.component.css']
})
export class StatusComponent implements OnInit {
  isLoggedIn: boolean = false;
  constructor(private auth: AuthService) {}
  ngOnInit(): void {
    const token = localStorage.getItem('token');
    if (token) {
      this.auth.ensureAuthenticated(token)
      .then((user) => {
        console.log(user.json());
        if (user.json().status === 'success') {
          this.isLoggedIn = true;
        }
      })
      .catch((err) => {
        console.log(err);
      });
    }
  }
}
```

Finally, add a new route handler to the *app.module.ts* file:

```javascript
RouterModule.forRoot([
  { path: 'login', component: LoginComponent },
  { path: 'register', component: RegisterComponent },
  { path: 'status', component: StatusComponent }
])
```

Ready to test? Log in, and then navigate to [http://localhost:4200/status](http://localhost:4200/status). If there is a token in Local Storage, you should see:

```json
{
  "message": "Signature expired. Please log in again.",
  "status": "fail"
}
```

Why? Well, if you dig deeper on the server-side, you will find that the token is only valid for 5 seconds in *project/server/models.py*:

```python
def encode_auth_token(self, user_id):
    """
    Generates the Auth Token
    :return: string
    """
    try:
        payload = {
            'exp': datetime.datetime.utcnow() + datetime.timedelta(days=0, seconds=5),
            'iat': datetime.datetime.utcnow(),
            'sub': user_id
        }
        return jwt.encode(
            payload,
            app.config.get('SECRET_KEY'),
            algorithm='HS256'
        )
    except Exception as e:
        return e
```

Update this to 1 day:

```python
'exp': datetime.datetime.utcnow() + datetime.timedelta(days=1, seconds=0)
```

And then test it again. You should now see something like:

```json
{
  "data": {
    "admin": false,
    "email": "michael@realpython.com",
    "registered_on": "Sun, 13 Aug 2017 17:21:52 GMT",
    "user_id": 4
  },
  "status": "success"
}
```

Finally, let's redirect to the status page after a user successfully registers or logs in. Update *src/app/components/login/login.component.ts* like so:

```javascript
import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from '../../services/auth.service';
import { User } from '../../models/user';

@Component({
  selector: 'login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css']
})
export class LoginComponent {
  user: User = new User();
  constructor(private router: Router, private auth: AuthService) {}
  onLogin(): void {
    this.auth.login(this.user)
    .then((user) => {
      localStorage.setItem('token', user.json().auth_token);
      this.router.navigateByUrl('/status');
    })
    .catch((err) => {
      console.log(err);
    });
  }
}
```

Then update *src/app/components/register/register.component.ts*:

```javascript
import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from '../../services/auth.service';
import { User } from '../../models/user';

@Component({
  selector: 'register',
  templateUrl: './register.component.html',
  styleUrls: ['./register.component.css']
})
export class RegisterComponent {
  user: User = new User();
  constructor(private router: Router, private auth: AuthService) {}
  onRegister(): void {
    this.auth.register(this.user)
    .then((user) => {
      localStorage.setItem('token', user.json().auth_token);
      this.router.navigateByUrl('/status');
    })
    .catch((err) => {
      console.log(err);
    });
  }
}
```

Test it out!

## Route Restriction

Right now, all routes are open; so, regardless of whether a user is logged in or not, they they can access each route. Certain routes should be restricted if a user is not logged in, while other routes should be restricted if a user is logged in:

1. `/` - no restrictions
1. `/login` - restricted when logged in
1. `/register` - restricted when logged in
1. `/status` - restricted when not logged in

To achieve this, add either `EnsureAuthenticated` or `LoginRedirect` to each route, depending on whether you want to guide the user to the `status` view or the `login` view.

Start by creating two new services:

```sh
$ ng generate service services/ensure-authenticated
$ ng generate service services/login-redirect
```

Replace the code in the *ensure-authenticated.service.ts* file as follows:

```javascript
import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { AuthService } from './auth.service';

@Injectable()
export class EnsureAuthenticated implements CanActivate {
  constructor(private auth: AuthService, private router: Router) {}
  canActivate(): boolean {
    if (localStorage.getItem('token')) {
      return true;
    }
    else {
      this.router.navigateByUrl('/login');
      return false;
    }
  }
}
```

And replace the code in the *login-redirect.service.ts* like so:

```javascript
import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { AuthService } from './auth.service';

@Injectable()
export class LoginRedirect implements CanActivate {
  constructor(private auth: AuthService, private router: Router) {}
  canActivate(): boolean {
    if (localStorage.getItem('token')) {
      this.router.navigateByUrl('/status');
      return false;
    }
    else {
      return true;
    }
  }
}
```

Finally, update the *app.module.ts* file to import and configure the new services:

```javascript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';
import { HttpModule } from '@angular/http';
import { FormsModule } from '@angular/forms';

import { AppComponent } from './app.component';
import { LoginComponent } from './components/login/login.component';
import { AuthService } from './services/auth.service';
import { RegisterComponent } from './components/register/register.component';
import { StatusComponent } from './components/status/status.component';
import { EnsureAuthenticated } from './services/ensure-authenticated.service';
import { LoginRedirect } from './services/login-redirect.service';

@NgModule({
  declarations: [
    AppComponent,
    LoginComponent,
    RegisterComponent,
    StatusComponent,
  ],
  imports: [
    BrowserModule,
    HttpModule,
    FormsModule,
    RouterModule.forRoot([
      {
        path: 'login',
        component: LoginComponent,
        canActivate: [LoginRedirect]
      },
      {
        path: 'register',
        component: RegisterComponent,
        canActivate: [LoginRedirect]
      },
      {
        path: 'status',
        component: StatusComponent,
        canActivate:
        [EnsureAuthenticated]
      }
    ])
  ],
  providers: [
    AuthService,
    EnsureAuthenticated,
    LoginRedirect
  ],
  bootstrap: [AppComponent]
})

export class AppModule { }
```

Note how we are adding our services to a new route property, `canActivate`. The routing system uses the services in the `canActivate` array to determine whether to display the requested URL path. If the route has `LoginRedirect` and the user is already logged in, then they will be redirected to the `status` view. Including the `EnsureAuthenticated` service redirects the user to the `login` view if they attempt to access a URL that requires authentication.

Test one last time.

## What's Next?

In this tutorial, we went through the process of adding authentication to an Angular 4 + Flask app using JSON Web Tokens.

What’s next?

Try switching out the Flask back-end for a different web framework, like Django or Bottle, using the following endpoints:

- `/auth/register`
- `/auth/login`
- `/auth/logout`
- `/auth/user`

Add questions and/or comments below. Grab the final code from the [angular4-auth](https://github.com/realpython/angular4-auth) repo.
