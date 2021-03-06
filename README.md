Authentication and Authorization
================================

This is a Sails example application to demonstrate authentication and authorization using `Passport.js`. This explains how a user can authenticate with a username and password. And also explains API authentication using bearer tokens.

---
Let us start by seeing things working! Clone this repo, install all the required modules and lift the app:

``` shell
git clone https://github.com/sails101/even-more-passport.git
cd sails-auth-example
npm install
sails lift
```

I've added two controller actions: `flash/home` and `flash/remotehome`.

[http://localhost:1337/flash/home](http://localhost:1337/flash/home) cannot be accessed without logging in. You will be redirected to login page ([http://localhost:1337/login](http://localhost:1337/login)). You can register for a new account at [http://localhost:1337/register](http://localhost:1337/register).

[http://localhost:1337/flash/remotehome](http://localhost:1337/flash/remotehome) cannot be accessed without an API token. Hope you have already obtained your API token from `flash/home`. This should be present in the request in any of the following ways:

1. As a query param: Example: [http://localhost:1337/flash/remotehome?access_token=<token>](http://localhost:1337/flash/remotehome?access_token=<token>)
2. In the body: Example: `access_token: <token>`
3. In the header: Example: `Authorization: Bearer <token>`

---

I am going to explain the step by step procedure for integrating Passport to your Sails app using a passport.js-based authentication generator called [sails-generate-auth](https://www.npmjs.com/package/sails-generate-auth)
 
#### Steps

Step 1. Install the following npm modules:

``` shell
npm install sails-generate-auth --save
npm install passport-local --save
npm install passport --save
npm install bcryptjs --save
npm install validator --save
```

`passport-local` module is for local authentication strategy. `passport`, `bcryptjs` and `validator` and dependencies for `passport` and `sails-generate-auth`

Step 2. Now, all you have to do to integrate passport is to run the following command in your application:

``` shell
sails generate auth
```

It automatically generates all the files needed by passport.

Step 3. Add the following routes to `config/routes.rb`

``` js
'get /login': 'AuthController.login',
'get /logout': 'AuthController.logout',
'get /register': 'AuthController.register',

'post /auth/local': 'AuthController.callback',
'post /auth/local/:action': 'AuthController.callback',

'get /auth/:provider': 'AuthController.provider',
'get /auth/:provider/callback': 'AuthController.callback',
'get /auth/:provider/:action': 'AuthController.callback',
```

Step 4. Next, change your `config/bootstrap.js` to load your Passport providers on startup by adding the following line:

``` js
sails.services.passport.loadStrategies();
```

You can see a detailed explanation of the steps until this in [sails-generate-auth](https://github.com/kasperisager/sails-generate-auth/) page.

Step 5. Modify `api/policies/sessionAuth.js` to set the policy for controller actions:

``` js
module.exports = function(req, res, next) {
  if(req.user) return next();
  res.redirect('/login');
};
```

Step 6. Now, make the following changes in `config/policies.js` to apply the `sessionAuth` policy to controllers:

``` js
module.exports.policies = {
  '*': ['passport', 'sessionAuth'],
  'auth': {
    '*': ['passport']
  }
}
``` 

Here you are applying `sessionAuth` policy for all the controller actions except those in `AuthController`. Because auth actions like login, logout and register need to be accessed without logging in.

Now, lift your Sails application and see things in action! 

---

Here are the steps for adding API authentication (Before following the steps, just make sure that these parts doesn't exist in the code already. I had given a pull request to the sails-generate-auth module and it got accepted recently. I believe it is yet to get reflected in the NPM module): 

Step 1: We have to create an access token when a user is created. For that, add a new attribute called `accessToken` in the model [api/models/passport.js](https://github.com/sails101/even-more-passport/blob/master/api/models/Passport.js#L64).

Step 2: Now, we need to generate the accessToken once a user is created. For that, add the following code in [api/services/protocols/local.js](https://github.com/sails101/even-more-passport/blob/master/api/services/protocols/local.js#L61) and add the accessToken while [creating the passport](https://github.com/sails101/even-more-passport/blob/master/api/services/protocols/local.js#L66):

``` js
var token = new Buffer(user.username + user.createdAt).toString('base64');

Passport.create({
 protocol : 'local',
 password : password,
 user     : user.id,
 accessToken: token  
}, function (err, passport) { .. }
```

Step 3: Install `passport-http-bearer` module and add it to [config/passport.js](https://github.com/sails101/even-more-passport/blob/master/config/passport.js#L22)

``` js
bearer: {
 strategy: require('passport-http-bearer').Strategy
}
```

Step 4: And load the strategy in [api/services/passport.js](https://github.com/sails101/even-more-passport/blob/master/api/services/passport.js#L295):

``` js
else if (key === 'bearer') {
 if (strategies.bearer) {
  Strategy = strategies[key].strategy;
  self.use(new Strategy(function(token, done) {
        
  Passport.findOne({ accessToken: token }, function(err, passport) {
   if (err) { return done(err); }
   if (!passport) { return done(null, false); }
   User.findById(passport.user, function(err, user) {
    if (err) { return done(err); }
    if (!user) { return done(null, false); }
    return done(null, user, { scope: 'all' });
   });
  });
 }));
}
```

Step 5: Create a policy file [api/policies/bearerAuth.js](https://github.com/sails101/even-more-passport/blob/master/api/policies/bearerAuth.js) for APIs:

``` js
module.exports = function(req, res, next) {
 return passport.authenticate('bearer', { session: false })(req, res, next);
};
```

Step 6: Add the policy for controller actions that are accessed via API in [config/policies.js](https://github.com/sails101/even-more-passport/blob/master/config/policies.js#L23):

``` js
'flash': {
 'remoteHome': ['passport', 'bearerAuth']
},
```

And your API authorization should start working now!

If you have any questions or encounter any problems, feel free to raise an issue in this repo. I'll help you to resolve it.
