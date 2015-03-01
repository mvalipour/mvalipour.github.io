---
layout: post
title: "Building an angular/node app that is secured by OAuth and JWT"
category: "web-security"
tags: ["angularjs", "nodejs", "express", "oauth", "jwt", "authentication"]
---
{% include JB/setup %}

So I recently started building an app using angular and node (express).
To avoid any session memory and truly scalable and segregated API, I decided to build a token-based authentication for my app.

### So what is JWT?

As defined on their [website](http://jwt.io/):

> JSON Web Token (JWT) is a compact URL-safe means of representing claims to be transferred between two parties. The claims in a JWT are encoded as a JSON object that is digitally signed using JSON Web Signature (JWS).

What I found interesting about JWT is that:

<!--more-->

- JWT tokens can be decoded (to JSON object) without having the private secret. This means on our angular app (where storing/using secret is not feasible), we can still extract the user information and use it for UI purposes. -- in alternative approaches, you would have to make a request to your server in order to retrieve user info.
- At the same time, on the server, we are able to verify the validity of the token signature (using the private secret).

This is great right?! In the rest of this article I will show the setup I use to:

- Authenticate the user using their google account -- via OAuth
- Store/Retrieve the user from a mongo-db store upon successful authentication.
- Issue a JWT token and pass to client.
- Use the token on client for display purposes.
- Get the client to send the token to the server on every request.

The technologies I'm using are:

- [angularjs](https://angularjs.org/) for front-end
- [angular-jwt](https://github.com/auth0/angular-jwt) for handling jwt on angular
- [Restangular](https://github.com/mgonto/restangular) to make API calls on angular
- node/express on API server
- [passport-google-oauth](https://github.com/jaredhanson/passport-google-oauth) for oauth authentication

**[Note]** There are JWT libraries available in [many other languages](http://jwt.io/#libraries).

### Authenticating user using OAuth

In order to use the google OAuth service, you need to have an OAuth client setup in [google developer console](https://console.developers.google.com/). (Read more [here](https://developers.google.com/console/help/new/?hl=en_US#generatingoauth2))

Once you have it all setup, create a `config.js` file to store your keys in.

```language-javascript
var config = {
  auth: {
    token: {
      secret: '--some-secret-here--',
      expiresInMinutes: 20
    },
    cookieName: '_accessToken',
    google: {
      clientId: '--client-id-here--',
      clientSecret: '--client-secret-here--',
      callback: '--callback-url-here--'
    }
  }
};

module.exports = config;
```

and import it in your `server.js`:

```language-javascript
var config = require('./config');
```

Now go ahead and setup passport to use Google OAuth:

```language-javascript
    var passport = require('passport');
    var GoogleStrategy = require('passport-google-oauth').OAuth2Strategy;
```
```language-javascript
  passport.use(new GoogleStrategy({
      clientID        : config.auth.google.clientId,
      clientSecret    : config.auth.google.clientSecret,
      callbackURL     : config.auth.google.callback
  },
  function(req, token, refreshToken, profile, done) {
    process.nextTick(function() {

      database.user.findOne({ 'google_id' : profile.id }, function(err, user) {
        if (err) return done(err);

        if (user) {
            return done(null, user);
        } else {
          var newUser = {
            google_id: profile.id,
            name: profile.displayName,
            email: profile.emails[0].value
          };

          database.user.create(newUser, function(err, added) {
              if (err) {
                console.log(err);
              }

              return done(null, newUser);
          });
        }
      });
    });
  }));
```

Above code, registers the required configuration for google OAuth and on the event of the successful callback, retrieves or create the user based on their Google identifier.

Then the retrieved user is passed to the `done()` method which is used later in our pipe-line for further processing (i.e. in our case JWT token generation).

Before getting to JWT, note that you need to register your passport on your express `app` like this:

```language-javascript
  app.use(passport.initialize());

  app.get('/auth/google', passport.authenticate('google', { scope : ['profile', 'email'] }));
  app.get('/auth/google/callback', function(req, res, next){
    // generate JWT here
  });
```

The two registered routes are for starting the OAUth authentication and handling the OAuth callback respectively.

```language-javascript
    var jwt = require('jsonwebtoken');
```

```language-javascript
  app.get('/auth/google/callback', function(req, res, next){
    passport.authenticate('google', function(err, user, info){
        if (err) return next(err);
        if (!user) {
          return res.redirect('/#/login');
        }

        var userData = { name: user.name };
        var rolesData = {};
        user.roles.forEach(function(r) {
          rolesData[r] = true;
        });

        var tokenData = {
          user: userData,
          roles: rolesData
        };

        var token = jwt.sign(tokenData,
                             config.auth.token.secret,
                             { expiresInMinutes: config.auth.token.expiresInMinutes });

        res.cookie(config.auth.cookieName, token);
        res.redirect('/#/');

      })(req, res, next);
  });
```

On this handler, if there is an error, we just simply take the user back to login page.  
If successful, we prepare an object that encapsulates the user fields. (`tokenData`).

**[Important]**: note that - as mentioned above - the details you put in this object, will be retrievable by anyone **without** the need to the secret key. (you can [try it here](http://jwt.io/)). So be careful what you put in there!

Finally this object is signed using the `config.auth.token.secret` secret and `config.auth.token.expiresInMinutes` expiration. The result token is then added to the response as a cookie. and the user is taken to the home page.

Now all you need is a login screen with a link to the OAuth action:

```language-markup
  <a href="/auth/google">Login with Google!</a>
```

### Token validation on API

Later on, in this article we will see that our angular app, will send the token as a header on every request. Here we will show how your API will validate the token:

- Decode the user info
- Check if signature is valid (using the secret)
- Check expiry date

Luckily, `jwt.verify()` does all of these for us:

```language-javascript
function middleware(req, res, next) {
  var token = req.headers['x-access-token'];

  if (token) {
    try {
      var decoded = jwt.verify(token, config.auth.token.secret);

      req.principal = {
        isAuthenticated: true,
        roles: decoded.roles || [],
        user: decoded.user
      };
      return next();

    } catch (err) { console.log('ERROR when parsing access token.', err); }
  }

  return res.status(401).json({ error: 'Invalid access token!' });
}
```

now you can add this middleware to your `api` route to make them secure:

```language-javascript
     app.use('/api', middleware, controllersApp);
```

`controllerApp` is the app on which you add all your API routes. (for simplicity I have excluded the code for this).

### Using the token on our angular app

We will be using [angular-jwt](https://github.com/auth0/angular-jwt) to decode the token:

```language-javascript
angular.module('portal', [
'ui.router',
'ngCookies',
'restangular',
'angular-jwt'])

.factory('principal', function($cookies, jwtHelper){
  var principal = { isAuthenticated: false, roles: [], user: { name: 'Guest' } };

  try{
    var token = $cookies._accessToken;
    var decoded = jwtHelper.decodeToken(token);

    if(decoded && !jwtHelper.isTokenExpired(token)){

        principal.isAuthenticated = true;
        principal.roles = decoded.roles;
        principal.user = decoded.user;
        principal.token = token;
    }
  }
  catch(e){
    console.log('ERROR while parsing principal cookie.');
  }

  principal.logout = function(){
    delete $cookies._accessToken;
  };

  return principal;
});
```

**Note** that this `principal` gets instanciated only once per application (fresh browser refresh). This is on purpose, so that retrieving it is cheap (decoding happens only once). The only thing to factor here is that on the following events that change the logged-in user state, we would have to do a browser refresh (instead of ui-state change):

- Login successful (given the user is sent out to google website, on the callback this happens anyway)
- Logout
- Server returning `unauthorized`.

for example our logout looks like this:

```language-javascript
  $scope.logout = function(){
    principal.logout();
    $window.location.href = '/';
  };
```

However this is just a choice, and you could choose to build the `principal` as a function and use it like `principal()` rather than a value.

This factory can now be used in any other part of our app to retrieve the logged-in user. e.g. on a top-menu

```language-javascript
.controller('topMenuController', function($scope, principal){
  $scope.userName = principal.user.name;
});
```

Finally will configure [Restangular](https://github.com/mgonto/restangular) to send the `x-access-token` header on every request it sends to the API:

```language-javascript
.run(function(Restangular, principal) {
    Restangular.setDefaultHeaders({'x-access-token': principal.token});
});
```

## Route auhorization on our angular app

So we want to mark our routes to be authorized for a specific user role. We set this as a `data` on our `routes` (in this case a root state for the admin area).

```language-javascript
  $stateProvider.state('admin', {
      abstract: true,
      url: '/admin',
      template: '<ui-view></ui-view>',
      data: { roles: ['admin'] }
  });
```

And of course we will handle the `stateChangeStart` event to enforce this:

```language-javascript
angular.module('portal')

.run(function($rootScope, $state, principal){
  $rootScope.$on('$stateChangeStart',
    function(event, toState, toParams, fromState, fromParams){

      var roles = toState.data ? toState.data.roles : [];
      if(roles &&
        roles.length > 0 &&
        roles.filter(function(r){ return principal.roles[r] === true; }).length === 0 ) {
          $state.go('login');

          event.preventDefault();
      }
  });
})
```
