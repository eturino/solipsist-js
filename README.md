# Solipsist.js

Solipsist is a library that allows you to develop JavaScript applications without worrying about the backend by providing a simple way to define object factories and mocking ajax requests.

The idea is to provide an isolated environment that mimics the behaviour of the server so you can focus completely on the frontend until it's ready to talk to the real world, avoiding the painful context-switching between client-server code.

### Status of the Library

Very early proof-of-concept version.

## Factories

With `Solipsist.Factory` you can define 'blueprints' for your data objects and then generate as many of them as you need:

```javascript

// Define some factory with six silly properties

var Factory = Solipsist.Factory;

var MyFactory = Factory({
  prop_one:    Factory.int_between(10, 20),
  prop_two:    Factory.int_between(0, 100),
  prop_three:  Factory.int_sequence(),
  prop_four:   Factory.int_sequence(1000),
  prop_five:   "Just a plain, static string",
  prop_six:    function() { return new Date; }
});

// Then, use it to generate data objects

var my_obj = MyFactory();
console.log(my_obj.prop_one);   // 13
console.log(my_obj.prop_three); // 0
console.log(my_obj.prop_four);  // 1000
console.log(my_obj.prop_five);  // "Just a plain, static string"

var other_obj = MyFactory({prop_five: "Another String"});
console.log(other_obj.prop_one);   // 17
console.log(other_obj.prop_three); // 1
console.log(other_obj.prop_four);  // 1001
console.log(other_obj.prop_five);  // "Another String"

````

You can use the object constructed by the factory to populate more complex models speficing a custom constructor with the `_constructor` special property:

```javascript

// Using backbone.js

var MyModel = Backbone.Model.extend({
  method_one: function() {
    return this.get('prop_one');
  },
  method_two: function() {
    return this.get('prop_two');
  }
});

var Factory = Solipsist.Factory;

var MyFactory = Factory({
  prop_one: Factory.int_between(10, 100),
  prop_two: Factory.int_sequence(50),
  _constructor: function(data) { return new MyModel(data); }
});

var instance = MyFactory();
console.log(instance.method_one());    // 34
console.log(instance.get('prop_two')); // 50

```

## AJAX stubs

This is the core functionality of the library and what allows you to build an isolated, client-only development environment. Use `Solipsist.Request` to define stubs that capture ajax requests from jQuery and emulate the behaviour of a server response. The syntax is very simple, following the sinatramodel of route-handler pairs.

```javascript

// Resource factory

var Factory = Solipsist.Factory;

ItemFactory = Factory({
  id: Factory.int_sequence(),
  name: Factory.string_random(10),
  description: Factory.string_random_paragraph(10, 10)
});

// AJAX stub

var Request = Solipsist.Request;

Request.get('/resource/index', function(req) {
  var response = Factory.collection_of(10, ItemFactory);
  req.success(response);
});

// Your client code

$.get('/resource/index', function(data) {
  console.log(data);
});

```

You can use URL patterns for emulate more complex requests:

```javascript

// Resource factory

var Factory = Solipsist.Factory;

ItemFactory = Factory({
  id: Factory.int_sequence(),
  name: Factory.string_random(10),
  description: Factory.string_random_paragraph(10, 10)
});

// AJAX stub

var Request = Solipsist.Request;

Request.get('/resource/:id', function(req) {
  var id = req.params['id'];
  var item = ItemFactory({id: id});
  req.success(item);
});

// Your client code

$.get('/resource/34', function(data) {
  console.log(data);
});

```

And make things more real simulating server errors and delays:

```javascript

// Resource Factory

var Factory = Solipsist.Factory;

BookFactory = Factory({
  title: Factory.string_random(10),
  pages: Factory.int_between(100, 500)
});

AuthorFactory = Factory({
  id: Factory.int_sequence(),
  name: Factory.string_random_paragraph(2, 10),
  books: function() { return Factory.collection_of(4, BookFactory); }
});

// AJAX stub

var Request = Solipsist.Request;

Request.get('/author/:id', {delay: 300}, function(req) {
  var author = AuthorFactory({id: req.params['id']});
  if (Math.random() > 0.3) {
    req.success(author);
  } else {
    req.error();
  }
});

// Your client code

$.ajax({
  url: '/author/12',
  type: 'get',
  success: function (data) {
    console.log("Success!");
    console.log(data);
  },
  error: function () {
    throw new Error('Request failed!');
  }
});

```

When you launch a POST request, all the parameters are available inside `req.params`:

```javascript

var Request = Solipsist.Request;

Request.post('/say', function(req) {
  console.log('You said: ' + req.params['message']);
  req.success();
});

$.post('/say'. {message: 'Hello, fake server!'});

```
