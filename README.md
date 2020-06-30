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
const pkClient = new auth.JWKSClient({
  jwksUri: `${process.env.AUTH0_BASE_URL}/.well-known/jwks.json`,
});

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

### Handling login requests

```javascript
// Instantiate a login handler.
const loginHandler = new auth.handlers.Login({
  auth0: {
    baseUrl: process.env.AUTH0_BASE_URL,
    apiAudience: process.env.AUTH0_API_AUDIENCE,
    clientId: process.env.AUTH0_CLIENT_ID,
    clientSecret: process.env.AUTH0_CLIENT_SECRET,
    dbConnection: process.env.AUTH0_DB_CONNECTION,
  },
  cookie: {
    // Enable/disable cookie attributes depending on environment.
    secure: process.env.NODE_ENV !== 'development',
    sameSite: process.env.BYPASS_SAME_SITE !== 'true',
  },
});

// Handle all requests to the login endpoint.
app.post('/auth/login', loginHandler.handle);
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
app.get('/auth/logout', logoutHandler.handle);
```