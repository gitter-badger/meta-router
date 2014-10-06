meta-router
=============
[![Gitter](https://badges.gitter.im/Join Chat.svg)](https://gitter.im/patrick-steele-idem/meta-router?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

This simple, declarative URL router provides Express middleware that can be used to associate metadata with a route. In addition, this module allows an incoming request to be matched to a route at the beginning of the request and allows the handling of the request to be deferred to later in the request. This is helpful in many applications, because intermediate middleware can use the metadata associated with the matched route to conditionally apply security checks, tracking, additional debugging, etc.

Internally, this module utilizes the same module used by Express to parse and match URLs—thus providing an easy transition from the builtin Express router to this router. The router also exposes an API that can be used independent of Express to match a path to route.

# Example

Let's say that you want to register authentication middleware for an application, but only a few of the routes actually require authentication. One option is to register the route-specific authentication middleware for each route using code similar to the following:

```javascript
var authMiddleware = require('my-auth-middleware');
app.get('/account', authMiddleware({ redirect: true}), require('./pages/account'));
```

While the above code will work as expected, it has a few drawbacks. The extra route-specific middleware adds clutter and the resulting code is not declarative.

To solve these problems, let's move our routes to a JSON file:

```json
[
    {
        "path": "GET /account",
        "handler": "require:./pages/account",
        "security": {
            "authenticationRequired": true,
            "redirect": true
        }
    }
]
```

By itself, the "security" metadata for the declared route will have no impact. To enforce authentication we can change the implementation of `my-auth-middleware` to be similar to the following:

```javascript
module.exports = function(req, res, next) {
    var routeConfig = req.route && req.route.config;
    if (routeConfig.security && routeConfig.security.authenticationRequired) {
        if (isUserAuthenticated(req)) {
            next();
        } else {
            // Handle un-authenticated user...
        }
    } else {
        // Route has no security policy... just continue on...
        next();
    }
}
```

Finally, to tie everything together we need to register the following middleware (order matters):

```javascript
// Match the incoming request to a route:
app.use(require('meta-router/middleware').match("routes.json"));

// Apply security (if applicable)
app.use(require('my-auth-middleware'));

// Invoke the page handler (if applicable)
app.use(require('meta-router/middleware').invokeHandler());
```


# Installation

```bash
npm install meta-router --save
```


# Basic Usage

The basic usage is shown below:

```javascript

// Match the incoming request to one of the provided routes
app.use(require('meta-router/middleware').match([
    {
        "path": "GET /users/:user",
        "middleware": [ // Any number of middleware functions to use for this route (called right before handler)
            function(req, res, next) {
                if (isNotLoggedIn(req)) {
                    res.status(401).end('Not authorized');
                } else {
                    next();
                }
            }
        ],
        "handler": function(req, res) {
            res.end('Hello user: ' + req.params.user);
        },
        "foo": "bar" // <-- Arbitrary metadata
    },
    {
        "path": "POST /users/:user/picture",
        "handler": function(req, res) {
            saveProfilePicture(req);
            res.end('User profile picture updated!');
        }
    }
]);

// Use information from the matched route
app.use(function(req, res, next) {
    var route = req.route;
    if (route) {
        console.log('Route params: ', route.params);     // e.g. { user: 'John' }
        console.log('Route path: ', route.path);     // e.g. "/users/123"
        console.log('Route path: ', route.config.path);     // e.g. "/users/:user"
        console.log('Route methods: ', route.config.methods); // e.g. ['GET']
        console.log('Route foo: ', route.config.foo);       // e.g. "bar"
    }
    next();
});

// Invoke the route handler (if available) and end the response:
app.use(require('meta-router/middleware').invokeHandler());
```

The API is described in more detail in the next section.

# API

## match() middleware

The `match()` middleware matches an incoming request to one of the possible routes. If an incoming request matched any routes passed in then the `req.route` property will be populated with information about the route.

```javascript
require('meta-router/middleware').match(routes);
```

The `routes` argument can either be an `Array` of routes or a path to a JSON routes file (explained later). The format of the routes `Array` is explained by the following example below:

```javascript
[
    {
        "path": "GET /users/:user", // HTTP method and path
        "handler": function(req, res) { // Route handler function
            ...
        }, // Route handler function
        "middleware": [  // Route-specific middleware to run right before the handler
            function foo(req, res, next) {
                // ...
                next();
            },
            function bar(req, res, next) {
                // ...
                next();
            }
        ],
        // Any additional metadata to associate with this route: (optional)
        "foo": "bar",
    },
    // Alternatively, multiple HTTP methods can be matched
    {
        "methods": ["GET", "POST"]
        "path": "/foo", // HTTP method and path
        "handler": ...,
        ...
    },
    // Or to match all HTTP methods, the method can be omitted altogether
    {
        "path": "/bar", // HTTP method and path
        "handler": ...,
        ...
    },
    // Additional routes:
    ...
]
```

If a `String` is passed to the `match()` middleware function then it is treated as a path to a JSON routes file. For example:

```javascript
app.use(require('meta-router/middleware').match("routes.json"));
```

Then in `routes.json`:

```js
[
    {
        "path": "GET /users/:user", // HTTP method and path
        "handler": "require:./path/to/user/handler/module", // Path to a module that exports a route handler function
        "middleware": [  // Route-specific middleware to run right before the handler (optional)
            "require:foo", // Path to a module that exports a route handler function
            {
                "factory": "require:bar",
                "method": "baz" // Optional name of a property name to lookup the actual factory
                "arguments": [  // Optional arguments to use when calling the factory function
                    "hello",
                    "world"
                ]
            }
        ],
        // Any additional metadata to associate with this route (optional):
        "foo": "bar"
    },
    ...
]
```

A few things to note when using a JSON routes file:

* JavaScript comments are allowed in the JSON configuration file (they are stripped out before parsing)
* [shortstop](https://github.com/krakenjs/shortstop) is used to preprocess the loaded JSON file to resolve handler functions and middleware. All of the handlers provided by [shortstop-handlers](https://github.com/krakenjs/shortstop-handlers) are registered.

## invokeHandler() middleware

The `invokeHandler()` can be used to invoke a route handler associated with the matched route.

```javascript
app.use(require('meta-router/middleware').invokeHandler());
```

If a route with a handler was matched, then the associated handler function will be invoked (passing in the `req` and `res` objects). The route handler is expected to end the response to complete the request. If no route was matched or if the route does not have an associated handler then the next middleware in the chain will be invoked.

## buildMatcher(routes)

Given an Array of routes, the `buildMatcher(routes)` method will return an object with a `match(path[, method])` method that can be used to match a path or path+method to one of the provided routes.

Usage:

```javascript
var matcher = require('meta-router').buildMatcher([
    {
        path: 'GET /users/:user',
        handler: function(req, res) {
            ...
        }
    },
    ...
]);

var match = matcher.match('/users/123', 'GET');
// match.params.user === '123'
// match.config.path === '/users/:user'
```

Since the `method` argument is optional, the following is also supported:

```javascript
var matcher = require('meta-router').buildMatcher([
    {
        path: '/users/:user',
        handler: function(req, res) {
            ...
        }
    },
    ...
]);

var match = matcher.match('/users/123');
// match.params.user === '123'
// match.config.path === '/users/:user'
```

## TODO

- Add support for `beforeHandler` and `afterHandler` functions for each route

## Maintainers

* Patrick Steele-Idem ([Github: @patrick-steele-idem](http://github.com/patrick-steele-idem)) ([Twitter: @psteeleidem](http://twitter.com/psteeleidem))

## Contribute

Pull requests, bug reports and feature requests welcome. To run tests:

```bash
npm install
npm test
```

## License

ISC
