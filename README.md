[![Build Status](http://product-stack-ci.paralect.com/api/badges/paralect/node-mongo/status.svg)](http://product-stack-ci.paralect.com/paralect/node-mongo) [![npm version](https://badge.fury.io/js/%40paralect%2Fnode-mongo.svg)](https://badge.fury.io/js/%40paralect%2Fnode-mongo) [![Coverage Status](https://coveralls.io/repos/github/paralect/node-mongo/badge.svg?branch=master)](https://coveralls.io/github/paralect/node-mongo?branch=master)

# Handy MongoDB layer for Node.JS 8

Currently based on [monk](https://github.com/Automattic/monk).

Install as npm package: `npm i @paralect/node-mongo`

There are few reasons, why we think this layer could be helpful to many projects:

1. Every update method emits `*.updated`, `*.created`, `*.removed` events, which allow to listen for the database changes and perform business logic based on this updates. That could help keep your entities weakly coupled with each other.
2. Implements more high level api, such as paging.
3. Implements database schema validation based on [joi](https://github.com/hapijs/joi). See examples below for more details.
4. Allows you to add custom methods for services that are needed on a particular project. See examples below for more details.

## Usage example

Examples below cover all API methods currently available.

### Full API example Usage


```javascript
const connectionString = `mongodb://localhost:27017/home-db`;
const db = require('./').connect(connectionString);

// Create entity service
const usersService = db.createService('users');
// Support only query operations
const usersQueryService = db.createQueryService('users');
// Query methods:
const result = await usersService.find({ name: 'Bob' }, { page: 1, perPage: 30 });
// returns object like this:
// {
//   results: [], // array of user entities
//   pagesCount, // total number of pages
//   count, // total count of documents found by query
// }

// If page is not specified does not return count and pagesCount as it require an extra database call.
const result2 = await usersService.find({ name: 'Bob'}, { });

// returns one document or throws an error if more that one document found
const user = await usersService.findOne({ name: 'Bob' });

const usersCount = await usersService.count({ name: 'Bob'});
const isUserExists = await usersService.exists({ name: 'Bob' });
const distinctNames = await distinct('name');
const aggregate = await aggregate([{ $match: { name: 'Bob'} }]);
const mongoId = usersService.generateId();

// wait for document to appear in database, typically used in the integrational tests
await expectDocument({ name: 'Bob'}, {
  timeout: 10000,
  tick: 50,
  expectNoDocs: false,
});


// Updates

// All methods bellow:
// 1. emit updated, created and removed events
// 2. Load and save entire object

await userService.create([{ name: 'Bob' }, { name: 'Alice' }]);
await usersService.update({ _id: '1'}, (doc) => {
  doc.name = 'Alex';
});

await usersService.remove({ _id: 1 });
// if any errors happen, ensureIndex will log it as warning
usersService.ensureIndex({ _id: 1, name: 1});

// update callback is executed only if document exists
await usersService.createOrUpdate({ _id: 1 }, (doc) => {
  doc.name = 'Helen';
})

// Atomic operations. Do not emit change events.
await userService.atomic.update({ name: 'Bob' }, {
  $set: {
    name: 'Alice',
  },
}, { multi: true });

await usersService.findOneAndUpdate({ name: 'Bob'}, {
  $set: {
    name: 'Alice',
  },
});

// Subscribe to service change events:
userService.on('updated', ({ doc, prevDoc }) => {
});
userService.on('created', ({ doc, prevDoc }) => {
});
userService.on('removed', ({ doc, prevDoc }) => {
});

// Listen to the value changes between original and updated document
// Callback executed only if user lastName or firstName are different in current or updated document
const propertiesObject = { 'user.firstName': 'Bob' };
userService.onPropertiesUpdated(['user.firstName', 'user.lastName'], ({ doc, prevDoc }) => {
});

// Listen to the value changes between original and updated document
// Callback executed only if user first name changes from `Bob` to something else
userService.onPropertiesUpdated(propertiesObject, ({ doc, prevDoc }) => {
});

```

### Documents schema validation

Schema validation is based on [joi](https://github.com/hapijs/joi) and can be optionally provided to service. On every update service will validate schema before save data to the database. While schema is optional, we highly recommend use it for every service.
We believe that `joi` is one of the best libraries for validation of the schema, because it allows us to do the following things:
1) Validate the schemas with a variety of variations in data types
2) It is easy to redefine the text for validation errors
3) Write conditional validations for fields when some conditions are met for other fields
4) Do some transformations of the values (for example for string fields you can do `trim`)

```javascript
const Joi = require('Joi');

const subscriptionSchema = {
  appId: Joi.string(),
  plan: Joi.string().valid('free', 'standard'),
  subscribedOn: Joi.date().allow(null),
  cancelledOn: Joi.date().allow(null),
};

const companySchema = {
  _id: Joi.string(),
  createdOn: Joi.date(),
  updatedOn: Joi.date(),
  name: Joi.string(),
  isOnDemand: Joi.boolean().default(false),,
  status: Joi.string().valid('active', 'inactive'),
  subscriptions: Joi.array().items(
    Joi.object().keys(subscriptionSchema)
  ),
};

const joiOptions = {};

module.exports = (obj) => Joi.validate(obj, companySchema, joiOptions);

// Use schema when creating service. user.service.js file:
const schema = require('./user.schema')
const usersService = db.createService('users', schema);
```

Also, for the validation of a data schema, you can use the library [jsonschema](https://github.com/tdegrunt/jsonschema), which can also be passed as the second parameter of the function `createService`.

```javascript
// Define schema in a separate file and export method, that accepts JSON object and execute validate function of
// validator. Example of user.schema.js
//
const Validator = require('jsonschema').Validator;
const validator = new Validator();

const subscriptionSchema = {
  id: '/Subscription',
  type: 'object',
  properties: {
    appId: { type: 'String' },
    plan: { type: 'String', enum: ['free', 'standard'] },
    subscribedOn: { type: ['Date', 'null'] },
    cancelledOn: { type: ['Date', 'null'] },
  },
};

const companySchema = {
  id: '/Company',
  type: 'object',
  properties: {
    _id: { type: 'string' },
    createdOn: { type: 'Date' },
    updatedOn: { type: 'Date' },
    name: { type: 'String' },
    isOnDemand: { type: 'Boolean' },
    status: { type: 'string', enum: ['active', 'inactive'] },
    subscriptions: {
      type: 'array',
      items: { $ref: '/Subscription' },
    },
  },
  required: ['_id', 'createdOn', 'name', 'status'],
};

validator.addSchema(subscriptionSchema, '/Subscription');

module.exports = (obj) => validator.validate(obj, companySchema);

// Use schema when creating service. user.service.js file:
const schema = require('./user.schema')
const usersService = db.createService('users', schema);
```

In order to add your own methods to the services, you can use the functions `setServiceMethod` and `setQueryServiceMethod` of the object `db`. This function takes the function name as the first parameter and the custom function for the service as the second parameter. The custom function takes the service itself as the first parameter, and the remaining parameters are the parameters that are passed when this custom function is called.

```javascript
const connectionString = `mongodb://localhost:27017/home-db`;
const db = require('./').connect(connectionString);

db.setQueryServiceMethod('findById', (service, id) => {
  return service.findOne({ _id: id });
});

// Create entity service
const usersQueryService = db.createQueryService('users');

// find user by id
const user = await usersQueryService.findById('123')
```
