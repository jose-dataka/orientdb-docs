
# OrientJS driver

Official [orientdb](http://www.orientechnologies.com/orientdb/) driver for node.js. Fast, lightweight, uses the binary protocol.

[![Build Status](https://travis-ci.org/orientechnologies/orientjs.svg?branch=master)](https://travis-ci.org/orientechnologies/orientjs)


# Supported Versions

OrientJS aims to work with version 2.0.0 of OrientDB and later. While it may work with earlier versions, they are not currently supported, [pull requests are welcome!](./CONTRIBUTING.md)

> **IMPORTANT**: OrientJS does not currently support OrientDB's Tree Based [RIDBag](https://github.com/orientechnologies/orientdb/wiki/RidBag) feature because it relies on making additional network requests.
> This means that by default, the result of e.g. `JSON.stringify(record)` for a record with up to 119 edges will be very different from a record with 120+ edges.
> This can lead to very nasty surprises which may not manifest themselves during development but could appear at any time in production.
> There is an [open issue](https://github.com/orientechnologies/orientdb/issues/2315) for this in OrientDB, until that gets fixed, it is **strongly recommended** that you set `RID_BAG_EMBEDDED_TO_SBTREEBONSAI_THRESHOLD` to a very large value, e.g. 2147483647.
> Please see the [relevant section in the OrientDB manual](http://www.orientechnologies.com/docs/2.0/orientdb.wiki/RidBag.html#configuration) for more information.

# Installation

Install via npm.

```sh
npm install orientjs
```

To install OrientJS globally use the `-g` option:

```sh
npm install orientjs -g
```

# Running Tests

To run the test suite, first invoke the following command within the repo, installing the development dependencies:

```sh
npm install
```

Then run the tests:

```sh
npm test
```

# Features

- Tested with latest OrientDB (2.0.x and 2.1).
- Intuitive API, based on [bluebird](https://github.com/petkaantonov/bluebird) promises.
- Fast binary protocol parser.
- Access multiple databases via the same socket.
- Migration support.
- Simple CLI.
- Connection Pooling

# Usage

### Configuring the client.

```js
var OrientDB = require('orientjs');

var server = OrientDB({
  host: 'localhost',
  port: 2424,
  username: 'root',
  password: 'yourpassword'
});
```

### Close the connection at the end

```js
// CLOSE THE CONNECTION AT THE END
server.close();
```


### Listing the databases on the server

```js
server.list()
.then(function (dbs) {
  console.log('There are ' + dbs.length + ' databases on the server.');
});
```

### Creating a new database

```js
server.create({
  name: 'mydb',
  type: 'graph',
  storage: 'plocal'
})
.then(function (db) {
  console.log('Created a database called ' + db.name);
});
```

### Using an existing database

```js
var db = server.use('mydb');
console.log('Using database: ' + db.name);

// CLOSE THE CONNECTION AT THE END
db.close();
```

### Using an existing database with credentials

```js
var db = server.use({
  name: 'mydb',
  username: 'admin',
  password: 'admin'
});
console.log('Using database: ' + db.name);
// CLOSE THE CONNECTION AT THE END
db.close();
```

### Execute an Insert Query

```js
db.query('insert into OUser (name, password, status) values (:name, :password, :status)',
  {
    params: {
      name: 'Radu',
      password: 'mypassword',
      status: 'active'
    }
  }
).then(function (response){
  console.log(response); //an Array of records inserted
});

```


### Execute a Select Query with Params

```js
db.query('select from OUser where name=:name', {
  params: {
    name: 'Radu'
  },
  limit: 1
}).then(function (results){
  console.log(results);
});

```

### Raw Execution of a Query String with Params

```js
db.exec('select from OUser where name=:name', {
  params: {
    name: 'Radu'
  }
}).then(function (response){
  console.log(response.results);
});

```

[Query Builder](OrientJS-QueryBuilder.md)


### Loading a record by RID.

```js
db.record.get('#1:1')
.then(function (record) {
  console.log('Loaded record:', record);
});
```

### Deleting a record

```js
db.record.delete('#1:1')
.then(function () {
  console.log('Record deleted');
});
```

### Listing all the classes in the database

```js
db.class.list()
.then(function (classes) {
  console.log('There are ' + classes.length + ' classes in the db:', classes);
});
```

### Creating a new class

```js
db.class.create('MyClass')
.then(function (MyClass) {
  console.log('Created class: ' + MyClass.name);
});
```

### Creating a new class that extends another

```js
db.class.create('MyOtherClass', 'MyClass')
.then(function (MyOtherClass) {
  console.log('Created class: ' + MyOtherClass.name);
});
```

### Getting an existing class

```js
db.class.get('MyClass')
.then(function (MyClass) {
  console.log('Got class: ' + MyClass.name);
});
```

### Updating an existing class

```js
db.class.update({
  name: 'MyClass',
  superClass: 'V'
})
.then(function (MyClass) {
  console.log('Updated class: ' + MyClass.name + ' that extends ' + MyClass.superClass);
});
```

### Listing properties in a class

```js
MyClass.property.list()
.then(function (properties) {
  console.log('The class has the following properties:', properties);
});
```

### Adding a property to a class

```js
MyClass.property.create({
  name: 'name',
  type: 'String'
})
.then(function () {
  console.log('Property created.')
});
```

To add multiple properties, pass multiple objects separated by comma. Example:

```js
MyClass.property.create({
  name: 'name',
  type: 'String'
}, {
  name: 'surname',
  type: 'String'
})
.then(function () {
  console.log('Property created.')
});
```

### Deleting a property from a class

```js
MyClass.property.drop('myprop')
.then(function () {
  console.log('Property deleted.');
});
```

### Renaming a property on a class

```js
MyClass.property.rename('myprop', 'mypropchanged');
.then(function () {
  console.log('Property renamed.');
});
```

### Creating a record for a class

```js
MyClass.create({
  name: 'John McFakerton',
  email: 'fake@example.com'
})
.then(function (record) {
  console.log('Created record: ', record);
});
```

### Listing records in a class

```js
MyClass.list()
.then(function (records) {
  console.log('Found ' + records.length + ' records:', records);
});
```

### Create a new index for a class property

```js
db.index.create({
  name: 'MyClass.myProp',
  type: 'unique'
})
.then(function(index){
  console.log('Created index: ', index);
});
```

### Get entry from class property index

```js
db.index.get('MyClass.myProp')
.then(function (index) {
  index.get('foo').then(console.log.bind(console));
});
```

### Creating a new, empty vertex

```js
db.create('VERTEX', 'V').one()
.then(function (vertex) {
  console.log('created vertex', vertex);
});
```

### Creating a new vertex with some properties

```js
db.create('VERTEX', 'V')
.set({
  key: 'value',
  foo: 'bar'
})
.one()
.then(function (vertex) {
  console.log('created vertex', vertex);
});
```
### Deleting a vertex

```js
db.delete('VERTEX')
.where('@rid = #12:12')
.one()
.then(function (count) {
  console.log('deleted ' + count + ' vertices');
});
```

### Creating a simple edge between vertices

```js
db.create('EDGE', 'E')
.from('#12:12')
.to('#12:13')
.one()
.then(function (edge) {
  console.log('created edge:', edge);
});
```


### Creating an edge with properties

```js
db.create('EDGE', 'E')
.from('#12:12')
.to('#12:13')
.set({
  key: 'value',
  foo: 'bar'
})
.one()
.then(function (edge) {
  console.log('created edge:', edge);
});
```

### Deleting an edge between vertices

```js
db.delete('EDGE', 'E')
.from('#12:12')
.to('#12:13')
.scalar()
.then(function (count) {
  console.log('deleted ' + count + ' edges');
});
```

### Creating a function
You can create a function by supplying a plain javascript function. Please note that the method stringifies the `function` passed so you can't use any varaibles outside the function closure.

```js
db.createFn("nameOfFunction", function(arg1, arg2) {
  return arg1 + arg2;
})
.then(function (count) {
  // Function created!
});
```

You can also omit the name and it'll default to the `Function#name`

```js
db.createFn(function nameOfFunction(arg1, arg2) {
  return arg1 + arg2;
})
.then(function (count) {
  // Function created!
});
```

[CLI](OrientJS-CLI.md)

## Events
You can also bind to the following events

### `beginQuery`
Given the query

    db.select('name, status').from('OUser').where({"status": "active"}).limit(1).fetch({"role": 1}).one();

The following event will be triggered

    db.on("beginQuery", function(obj) {
      // => {
      //  query: 'SELECT name, status FROM OUser WHERE status = :paramstatus0 LIMIT 1',
      //  mode: 'a',
      //  fetchPlan: 'role:1',
      //  limit: -1,
      //  params: { params: { paramstatus0: 'active' } }
      // }
    });


### `endQuery`
After a query has been run, you'll get the the following event emitted

    db.on("endQuery", function(obj) {
      // => {
      //   "err": errObj,
      //   "result": resultObj,
      //   "perf": {
      //     "query": timeInMs
      //   }
      // }
    });


# History

In 2012, [Gabriel Petrovay](https://github.com/gabipetrovay) created the original [node-orientdb](https://github.com/gabipetrovay/node-orientdb) library, with a straightforward callback based API.

In early 2014, [Giraldo Rosales](https://github.com/nitrog7) made a [whole host of improvements](https://github.com/nitrog7/node-orientdb), including support for orientdb 1.7 and switched to a promise based API.

Later in 2014, codemix refactored the library to make it easier to extend and maintain, and introduced an API similar to [nano](https://github.com/dscape/nano). The result is so different from the original codebase that it warranted its own name and npm package. This also gave us the opportunity to switch to semantic versioning.

In June 2015, Orient Technologies company officially adopted the Oriento driver and renamed it as OrientJS.

# Notes for contributors

Please see [CONTRIBUTING](./CONTRIBUTING.md).

# Changes

See [CHANGELOG](./CHANGELOG.md)



# License

Apache 2.0 License, see [LICENSE](./LICENSE.md)