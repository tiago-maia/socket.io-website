title: Middlewares
permalink: /docs/v4/middlewares/
release: v4
type: docs
order: 205
---

A middleware function is a function that gets executed for every incoming connection.

Middleware functions can be useful for:

- logging
- authentication / authorization
- rate limiting

Note: this function will be executed only once per connection (even if the connection consists in multiple HTTP requests).

## Registering a middleware

A middleware function has access to the [Socket instance](/docs/v4/server-socket-instance) and to the next registered middleware function.

```js
io.use((socket, next) => {
  if (isValid(socket.request)) {
    next();
  } else {
    next(new Error("invalid"));
  }
});
```

You can register several middleware functions, and they will be executed sequentially:

```js
io.use((socket, next) => {
  next();
});

io.use((socket, next) => {
  next(new Error("thou shall not pass"));
});

io.use((socket, next) => {
  // not executed, since the previous middleware has returned an error
  next();
});
```

Please make sure to call `next()` in any case. Otherwise, the connection will be left hanging until it is closed after a given timeout.

## Sending credentials

The client can send credentials with the `auth` option:

```js
// plain object
const socket = io({
  auth: {
    token: "abc"
  }
});

// or with a function
const socket = io({
  auth: (cb) => {
    cb({
      token: "abc"
    });
  }
});
```

Those credentials can be accessed in the [handshake](/docs/v4/server-socket-instance/#Socket-handshake) object on the server-side:

```js
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  // ...
});
```

## Handling middleware error

If the `next` method is called with an Error object, the connection will be refused and the client will receive an `connect_error` event.

```js
// client-side
socket.on("connect_error", (err) => {
  console.log(err.message); // prints the message associated with the error
});
```

You can attach additional details to the Error object:

```js
// server-side
io.use((socket, next) => {
  const err = new Error("not authorized");
  err.data = { content: "Please retry later" }; // additional details
  next(err);
});

// client-side
socket.on("connect_error", (err) => {
  console.log(err instanceof Error); // true
  console.log(err.message); // not authorized
  console.log(err.data); // { content: "Please retry later" }
});
```

## Compatibility with Express middleware

Most existing [Express middleware](http://expressjs.com/en/resources/middleware.html) modules should be compatible with Socket.IO, you just need a little wrapper function to make the method signatures match:

```js
const wrap = middleware => (socket, next) => middleware(socket.request, {}, next);
```

The middleware functions that end the request-response cycle and do not call `next()` will not work though.

Example with [express-session](https://www.npmjs.com/package/express-session):

```js
const session = require("express-session");

io.use(wrap(session({ secret: "cats" })));

io.on("connection", (socket) => {
  const session = socket.request.session;
});
```

Example with [Passport](http://www.passportjs.org/):

```js
const session = require("express-session");
const passport = require("passport");

io.use(wrap(session({ secret: "cats" })));
io.use(wrap(passport.initialize()));
io.use(wrap(passport.session()));

io.use((socket, next) => {
  if (socket.request.user) {
    next();
  } else {
    next(new Error("unauthorized"))
  }
});
```

A complete example with Passport can be found [here](https://github.com/socketio/socket.io/tree/master/examples/passport-example).
