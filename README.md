# SafePlaces Authentication Service

[![Build Status](https://travis-ci.com/aiyan/safeplaces-auth.svg?branch=master)](https://travis-ci.com/aiyan/safeplaces-auth)
[![Coverage Status](https://coveralls.io/repos/github/aiyan/safeplaces-auth/badge.svg?branch=master)](https://coveralls.io/github/aiyan/safeplaces-auth?branch=master)

The modular authentication service for the SafePlaces backend.

**Supported strategies:**
- Auth0 asymmetric JWT
- Symmetric JWT

```shell script
# Install using NPM
npm install @aiyan/safeplaces-auth

# Install using Yarn
yarn add @aiyan/safeplaces-auth
```

## Examples

### Securing API endpoints

The authentication service can be instantiated and attached to an
existing Express application. The point or order at which you attach
the service relative to other middleware affects the point at which
it is invoked.

```javascript
const auth = require('@aiyan/safeplaces-auth');

// Instantiate a public key retrieval client.
const pkClient = new auth.JWKSClient(
  // Pass the URL for the JWKS.
  `${process.env.AUTH0_BASE_URL}/.well-known/jwks.json`,
);

// Instantiate a request verification strategy.
const auth0Strategy = new auth.strategies.Auth0({
  jwksClient: pkClient,
  // The API audience is checked against the user's access token.
  apiAudience: process.env.AUTH0_API_AUDIENCE,
});

// Instantiate a strategy enforcer.
const enforcer = new auth.Enforcer({
  strategy: auth0Strategy,
  // Pass in an asynchronous database lookup function
  // that takes the user id as the only argument.
  userGetter: id => User.findOne(id),
});

// `app` is your Express application.
// A middleware is attached to the app, and it intercepts every
// request for verification. The argument to `enforcer.secure`
// affects which routes are secured.
enforcer.secure(app);
```

For greater flexibility, you access the built-in `enforcer.handleRequest(req, res)`
function to selectively handle requests under your own logic.

```
app.use((req, res, next) => {
  if (req.headers['X-Bypass-Login']) {
    return next();
  } else {
    return enforcer
      // Enforcer ends the request with a `403 Forbidden` if it is unauthorized,
      // meaning `next` will not be called unless the request is authorized.
      .handleRequest(req, res)
      .then(() => next())
      .catch(err => next(err));
  }
});
```

For the most flexibility, you can use `enforcer.processRequest(req)` only for validating a request
and decide how to handle the result yourself, whether that be ending the request
or ignoring the unauthorized error.

```
app.use((req, res, next) => {
  enforcer
    .processRequest(req)
    // `.then` is called only if `processRequest` has
    // determined the request to be authorized.
    .then(() => next())
    // Otherwise, an error describing the validation error
    // will be thrown, and you can decide what to do with it.
    .catch(err => res.status(403).send('Forbidden'));
});
```

### Handling login requests

```javascript
// Instantiate a login handler.
const loginHandler = new auth.handlers.Login({
  auth0: {
    baseUrl: process.env.AUTH0_BASE_URL,
    apiAudience: process.env.AUTH0_API_AUDIENCE,
    clientId: process.env.AUTH0_CLIENT_ID,
    clientSecret: process.env.AUTH0_CLIENT_SECRET,
    realm: process.env.AUTH0_REALM,
  },
  cookie: {
    // Enable/disable cookie attributes depending on environment.
    secure: process.env.NODE_ENV !== 'development',
    sameSite: process.env.BYPASS_SAME_SITE !== 'true',
  },
});

// Handle all requests to the login endpoint.
// Since we are passing around the `handle` function, make sure
// to bind the handle function to its object.
app.post('/auth/login', loginHandler.handle.bind(loginHandler));
```

### Handling logout requests

```javascript
// Instantiate a logout handler.
const logoutHandler = new auth.handlers.Logout({
  redirect: 'https://example.com/logout-success/',
  cookie: {
    // Enable/disable cookie attributes depending on environment.
    secure: process.env.NODE_ENV !== 'development',
    sameSite: process.env.BYPASS_SAME_SITE !== 'true',
  },
});

// Handle all requests to the logout endpoint.
// Since we are passing around the `handle` function, make sure
// to bind the handle function to its object.
app.get('/auth/logout', logoutHandler.handle.bind(logoutHandler));
```

## Strategies

### Auth0

Validate the JSON Web Token by checking the signature with
the retrieved public key, and validate the API audience.

```javascript
const auth = require('@aiyan/safeplaces-auth');

const pkClient = new auth.JWKSClient(
  `${process.env.AUTH0_BASE_URL}/.well-known/jwks.json`,
);
const auth0Strategy = new auth.strategies.Auth0({
  jwksClient: pkClient,
  apiAudience: process.env.AUTH0_API_AUDIENCE,
});
```

### Symmetric JWT

Validate the JSON Web Token by checking the signature with
a fixed private key.

```javascript
const auth = require('@aiyan/safeplaces-auth');

const symJWTStrategy = new auth.strategies.SymJWT({
  algorithm: 'HS256',
  privateKey: process.env.JWT_SECRET,
});
```

### Dynamic strategy selection

You can also pass a function into the strategy parameter
to dynamically select the strategy based on the request
or some other variables.

The function should accept the request as the only argument
and return the desired strategy or a promise resolving the
desired strategy.

```javascript
const enforcer = new auth.Enforcer({
  strategy: req => {
    console.log(req);
    // Check something in the request.
    // ...
    // Example conditional:
    if (process.env.NODE_ENV === 'development') {
      return symJWTStrategy;
    } else {
      return auth0Strategy;
    }
  },
});
```
