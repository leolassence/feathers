---
layout: page
title: Authentication
description: Learn how to do user authentication and authorization
hide: true
---

[Passport](http://passportjs.org/) is the most commonly used authentication library for NodeJS and Express also really flexible. Manually setting up shared authentication between websockets and an HTTP REST API can be tricky. This is what the [feathers-passport](https://github.com/feathersjs/feathers-passport) module aims to make easier. The following examples show how to add local authentication that uses a Feathers service for storing and retrieving user information.

## Configuring Passport

The first step is to add the Passport, local strategy and feathers-passport modules to our application. Since we are using MongoDB already we will also use it as the session store through the [connect-mongo](https://github.com/kcbanner/connect-mongo) module:

> `npm install passport passport-local connect-mongo feathers-passport`

```js
// app.js
var feathers = require('feathers');
var mongodb = require('feathers-mongodb');
var bodyParser = require('body-parser');
var hooks = require('feathers-hooks');

var passport = require('passport');
var connectMongo = require('connect-mongo');
var feathersPassport = require('feathers-passport');

var app = feathers();
var todoService = mongodb({
  db: 'feathers-demo',
  collection: 'todos'
});

app.configure(feathers.rest())
  .configure(feathers.socketio())
  .configure(hooks())
  .configure(feathersPassport(function(result) {
    // MongoStore needs the session function
    var MongoStore = connectMongo(result.createSession);

    result.secret = 'feathers-rocks';
    result.store = new MongoStore({
      db: 'feathers-demo'
    });

    return result;
  }))
  .use(bodyParser.json())
  // Now we also need to parse HTML form submissions
  .use(bodyParser.urlencoded({ extended: true }))
  .use('/todos', todoService)
  .use('/', feathers.static(__dirname));
```

## User storage

Next, we create a MongoDB service for storing user information. It is always a good idea to not store plain text passwords in the database so we add a `.before` hook that salts and then hashes the password when creating a new user. This can be done in the service `.setup` which is called when the application is ready to start up. We also add an `.authenticate` method that we can use to look up a user by username and compare the hashed and salted passwords.

```js
var crypto = require('crypto');
// One-way hashes a string
var hash = function(string, salt) {
  var shasum = crypto.createHash('sha256');
  shasum.update(string + salt);
  return shasum.digest('hex');
};

var userService = mongodb({
  db: 'feathers-demo',
  collection: 'users'
}).extend({
  authenticate: function(username, password, callback) {
    // This will be used as the MongoDB query
    var query = {
      username: username
    };

    this.find({ query: query }, function(error, users) {
      if(error) {
        return callback(error);
      }

      var user = users[0];

      if(!user) {
        return callback(new Error('No user found'));
      }

      // Compare the hashed and salted passwords
      if(user.password !== hash(password, user.salt)) {
        return callback(new Error('User password does not match'));
      }

      // If we got to here, we call the callback with the user information
      return callback(null, user);
    });
  },

  setup: function() {
    // Adds the hook during service setup
    this.before({
      // Hash the password before sending it to MongoDB
      create: function(hook, next) {
        // Create a random salt string
        var salt = crypto.randomBytes(128).toString('base64');
        // Change the password to a hashed and salted password
        hook.data.password = hash(hook.data.password, salt);
        // Add the salt to the user data
        hook.data.salt = salt;
        next();
      }
    });
  }
});

app.use('/users', userService);
```

Now we need to set up Passport to use that service and tell it how to deserialize and serialize our user information. For us, the serialized form is the `_id` generated by MongoDB. To deserialize by `_id` we can simply call the user services `.get` method. Then we add the local strategy which simply calls the `.authenticate` method that we implemented in the user service.

```js
var LocalStrategy = require('passport-local').Strategy;

passport.serializeUser(function(user, done) {
  // Use the `_id` property to serialize the user
  done(null, user._id);
});

passport.deserializeUser(function(id, done) {
  // Get the user information from the service
  app.service('users').get(id, {}, done);
});

passport.use(new LocalStrategy(function(username, password, done) {
  app.service('users').authenticate(username, password, done);
}));
```

## Login

The last step is to add the authentication route that we can POST the login to:

```js
app.post('/login', passport.authenticate('local', {
    successRedirect: '/',
    failureRedirect: '/login.html',
    failureFlash: false
}));

app.listen(3000);
```

And to add a `login.html` page:

```html
<!DOCTYPE html>
<html>
<head lang="en">
  <meta charset="UTF-8">
  <title></title>
</head>
<body>
  <form action="/login" method="post">
    <div>
      <label>Username:</label>
      <input type="text" name="username"/>
    </div>
    <div>
      <label>Password:</label>
      <input type="password" name="password"/>
    </div>
    <div>
      <input type="submit" value="Log In"/>
    </div>
  </form>
</body>
</html>
```

To test the login, we might want to add a new user as well:

<blockquote><pre>curl 'http://localhost:3000/users/' -H 'Content-Type: application/json' --data-binary '{ "username": "feathers", "password": "supersecret" }'</pre></blockquote>

Now it should be possible to log in with the `feathers` username and `supersecret` password and you will get the logged in user information in every service call in `params.user`.