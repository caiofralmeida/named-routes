# Named Routes for Node.js

A `node.js` module for naming HTTP routes. Can be used with the [express.js](http://expressjs.com/) framework or independently as standalone. Originally based on [node-reversable-router](https://github.com/web-napopa/node-reversable-router).

**Feature overview**:
 - Support for named routes
 - Can be used standalone or as replacement for express.js 4 routing (see named-routes 1.1.4 for express 3).
 - URLs can be generated by providing a `name` of the route and the required `parameters`
 - Support for optional parts in the route `path` (and URL generation still works with as many optional parts as you want)
 - Support for anonymous `*` parameters inside the path
 - Supports converting the last anonymous parameter to pairs of `param`=>`value` separated by `/`
 - Improved performance on literal matches
 - Supports callbacks for router parameters. Same logic as `express` native router.
 - Supports middleware route callbacks. Same logic as `express` native router.
 - Supports array of middleware route callbacks. Same logic as `express` native router.

## Install

```
npm install named-routes
```

## Features

### Example
#### As a replacement for express framework router
In the view files:
```js
url('admin.user.edit', {id: 2}) // /admin/user/2
```

... and in the server config:
```js
var express = require('express');
var app = express();

var Router = require('named-routes');
var router = new Router();
router.extendExpress(app);
router.registerAppHelpers(app);

app.get('/admin/user/:id', 'admin.user.edit', function(req, res, next){
    //... implementation

    // the names can also be accessed here:
    var url = app.namedRoutes.build('admin.user.edit', {id: 2}); // /admin/user/2
});

app.listen(3000);
```

#### As a standalone

```js
var Router = require('named-routes')();
var router = new Router();

router.add('get', '/admin/user/:id', function() {
	var url = router.build('admin.user.edit', {id: 2}); // /admin/user/2
}, {
    name: 'admin.user.edit'
});

//...
router.dispatch(req);
```

### Benefits of named routes
You can easily check the current route in middleware without stating the defined route path. Thus avoding duplication and keeping route paths in a central place.

This allows the path to the route to be changed as frequently while the rest of the logic across middleware or views to remain the same.

### Generating URLs
URLs are generated by passing the route name and, optionally, parameters.

If you're using express:
```js
app.get('/about', 'about', function(req, res, next){ .. }
app.get('/todo/:user/:list/:id', 'todo.user.list.id', function(req, res, next){ .. }

// in the views:
url('about') // '/about'
url('todo.user.list.id', {user: 'foo', list: 93, id: 1337}) // '/todo/foo/93/1337'
url('todo.user.list.id', {user: 'foo', list: 93}) // Throws error, missing parameters

// anywhere else:
app.namedRoutes.build('about') // '/about'
app.namedRoutes.build('todo.user.list.id', {user: 'foo', list: 93, id: 1337}) // '/todo/foo/93/1337'
app.namedRoutes.build('todo.user.list.id', {user: 'foo', list: 93}) // Throws error, missing parameters
```

As a standalone:

```js
router.add('get', '/about', function() {...}, {name:'about'})
router.add('get', '/todo/:user/:list/:id', function() {...}, {name:'todo.user.list.id'})

router.build('about') // '/about'
router.build('todo.user.list.id', {user: 'foo', list: 93, id: 1337}) // '/todo/foo/93/1337'
router.build('todo.user.list.id', {user: 'foo', list: 93}) // Throws error, missing parameters
```

### AJAX
While forgetting to pass a parameter (or setting its value equal to ```undefined```) will trigger an error, you can also pass ```null``` as a parameter value to signal that you would it to be intentionally left blank (including the associated '/' character). This can be helpful when hard-coding ajax urls into front-end javascript.

```js
app.get('/todo/:user/:list/:id', 'todo.user.list.id', function(req, res, next){ .. }

// this will build simply '/todo/foo':
url('todo.user.list.id', {user: 'foo', list: null, id: null});

// useful for writing routes into ajax requests
var getTodo = '!{url('todo.user.list.id', {user: 'foo', list: null, id: null})}';
$http.get( getTodo + '/' + listID + '/' + id, function(){...})
```

The above assumes you are working in an express view. If you are not, swap out ```url``` with  ```app.namedRoutes.build``` if you are in express but outside the view and ```app.get``` with ```router.add``` and ```url``` with ```router.build``` if you using module standalone.

### Full support for optional parts of the URL
You can define routes like this:

```js
app.get('/admin/(user/(edit/:id/)(album/:albumId/):session/)test', 'admin', function(req, res, next){
    console.log(req.params);
});
```

Brackets define the limits of the optional parts. Here you have 3 optional parts. 2 of them nested in the other.

If you don't pass all the parameters inside a optional part, the part will simply be removed from the generated URL.

So in the views:
```js
url('admin', {id: 4, albumId:2, session: 'qwjdoqiwdasdj12asdiaji198a#asd'});
// will generate: /admin/user/edit/4/album/2/qwjdoqiwdasdj12asdiaji198a/test
```
```js
url('admin', {id: 4, session: 'qwjdoqiwdasdj12asdiaji198a#asd'});
// will generate: /admin/user/edit/4/qwjdoqiwdasdj12asdiaji198a/test
```
```js
url('admin', {albumId: 2, session: 'qwjdoqiwdasdj12asdiaji198a#asd'});
// will generate: /admin/user/album/2/qwjdoqiwdasdj12asdiaji198a/test
```
```js
url('admin', {id: 4, albumId:2});
// will generate: /admin/test
// because :session parameter is missing and the optional part
// that contains it contains also the other 2 parts
```

### Improved matching speed for literal matches
Significant amount of the routes in an web applications are simply hardcoded strings. Things like `/admin` or `/user/login`.
Such routes will be matched with direct check for equallity without the need for a regular expression execution.

### Anonymous `*` parameters inside the path
```js
app.get('/admin/*/user/*/:id/', 'admin.user.edit', function(req, res, next){
    console.log(req.params)
});
```
Requesting: `/admin/any/user/thing/2` will output:
```
{
  _masked: [ 'any', 'thing'],
  id: '2'
}
```

Analogous in order to generate the same url:
```js
url('admin.user.edit', {id:2, _masked: ['any','thing']})
```

### Converting the trailing `*` anonymous parameter to multiple `name:value` parameters
```js
app.get('/admin/*/user/*/:id/albums/*', 'admin.user.edit', function(req, res, next){
    console.log(req.params)
}, {
	wildcardInPairs: true
});
```
Requesting: `/admin/any/user/thing/2/albums/sort/name/order/desc` will output:
```
{
  _masked: [ 'any', 'thing'],
  id: '2',
  sort: 'name',
  order: 'desc'
}
```

Analogous in order to generate the same url:
```js
url('admin.user.edit', {id:2, _masked: ['any','thing'], sort: 'name', 'order': 'desc'})
```


## Future development planned

### Publish
 - Organise and publish tests

### Implement
 - Query based routing and generation

### Investigate
**meta-routing** Middleware depending on media? mobile, desktop, agent

## License
(The MIT License)

Copyright (c) 2014 Andreas Lubbe <npm@lubbe.org>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the 'Software'), to deal in the Software without restriction, including without limitation the
rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
