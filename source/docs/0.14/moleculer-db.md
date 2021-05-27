title: Database Adapters
---
Moleculer framework has an official set of [DB adapters](https://github.com/moleculerjs/moleculer-db). Use them to persist your data in a database.

{% note info Database per service%}
Moleculer follows the *one database per service* pattern. To learn more about this design pattern and its implications check this [article](https://microservices.io/patterns/data/database-per-service.html). For *multiple entities/tables per service* approach check [FAQ](faq.html#DB-Adapters-moleculer-db).
{% endnote %}

## Features
* default CRUD actions
* [cached](caching.html) actions
* pagination support
* pluggable adapter ([NeDB](https://github.com/louischatriot/nedb) is the default memory adapter for testing & prototyping)
* official adapters for MongoDB, PostgreSQL, SQLite, MySQL, MSSQL.
* fields filtering
* populating
* encode/decode IDs
* entity lifecycle events for notifications

{% note info Try it in your browser! %}
[![Edit moleculer-db](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/github/moleculerjs/sandbox-moleculer-db/tree/master/?fontsize=14)
{% endnote %}

## Base Adapter [![NPM version](https://img.shields.io/npm/v/moleculer-db.svg)](https://www.npmjs.com/package/moleculer-db)

Moleculer's default adapter is based on [NeDB](https://github.com/louischatriot/nedb). Use it to quickly set up and test you prototype.

{% note warn%}
Only use this adapter for prototyping and testing. When you are ready to go into production simply swap to [Mongo](moleculer-db.html#Mongo-Adapter), [Mongoose](moleculer-db.html#Mongoose-Adapter) or [Sequelize](moleculer-db.html#Sequelize-Adapter) adapters as they all implement common [Settings](moleculer-db.html#Settings), [Actions](moleculer-db.html#Actions) and [Methods](moleculer-db.html#Methods).
{% endnote %}

### Install

```bash
$ npm install moleculer-db --save
```

### Usage

```js
"use strict";

const { ServiceBroker } = require("moleculer");
const DbService = require("moleculer-db");

const broker = new ServiceBroker();

// Create a DB service for `user` entities
broker.createService({
    name: "users",

    // Mixin DB service into (current) 'users' service
    mixins: [DbService],

    settings: {
        fields: ["_id", "username", "name"],
        entityValidator: {
			username: "string"
		}
    },

    afterConnected() {
        // Seed the DB with ˙this.create`
    }
});

broker.start()

// Create a new user
.then(() => broker.call("users.create", {
    username: "john",
    name: "John Doe",
    status: 1
}))

// Get all users
.then(() => broker.call("users.find").then(console.log));

// List users with pagination
.then(() => broker.call("users.list", { page: 2, pageSize: 10 }).then(console.log));

// Get a user
.then(() => broker.call("users.get", { id: 2 }).then(console.log));

// Update a user
.then(() => broker.call("users.update", { id: 2, name: "Jane Doe" }).then(console.log));

// Delete a user
.then(() => broker.call("users.remove", { id: 2 }).then(console.log));

```

> More examples can be found on [GitHub](https://github.com/moleculerjs/moleculer-db/tree/master/packages/moleculer-db/examples)

## Settings

All DB adapters share a common set of settings:

| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `idField` | `String` | **required** | Name of ID field. |
| `fields` | `Array.<String>` | `null` | Field filtering list. It must be an `Array`. If the value is `null` or `undefined` doesn't filter the fields of entities. |
| `populates` | `Array` | `null` | Schema for population. [Read more](#Populating). |
| `pageSize` | `Number` | **required** | Default page size in `list` action. |
| `maxPageSize` | `Number` | **required** | Maximum page size in `list` action. |
| `maxLimit` | `Number` | **required** | Maximum value of limit in `find` action. Default: `-1` (no limit) |
| `entityValidator` | `Object`, `function` | `null` | Validator schema or a function to validate the incoming entity in `create` & 'insert' actions. |

{% note warn%}
`idField` does not work with Sequelize adapter as you can freely set your own ID while creating the model.
{% endnote %}

## Actions

DB adapters also implement CRUD operations. These [actions](actions.html) are [`published`](actions.html#Action-visibility) methods and can be called by other services.

### `find` ![Cached action](https://img.shields.io/badge/cache-true-blue.svg) 

Find entities by query.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `populate` | `Array.<String>` | - | Populated fields. |
| `fields` | `Array.<String>` | - | Fields filter. |
| `limit` | `Number` | **required** | Max count of rows. |
| `offset` | `Number` | **required** | Count of skipped rows. |
| `sort` | `String` | **required** | Sorted fields. |
| `search` | `String` | **required** | Search text. |
| `searchFields` | `String` | **required** | Fields for searching. |
| `query` | `Object` | **required** | Query object. Passes to adapter. |

#### Results
**Type:** `Array.<Object>` - List of found entities.


### `count` ![Cached action](https://img.shields.io/badge/cache-true-blue.svg) 

Get count of entities by query.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `search` | `String` | **required** | Search text. |
| `searchFields` | `String` | **required** | Fields list for searching. |
| `query` | `Object` | **required** | Query object. Passes to adapter. |

#### Results
**Type:** `Number` - Count of found entities.


### `list` ![Cached action](https://img.shields.io/badge/cache-true-blue.svg) 

List entities by filters and pagination results.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `populate` | `Array.<String>` | - | Populated fields. |
| `fields` | `Array.<String>` | - | Fields filter. |
| `page` | `Number` | **required** | Page number. |
| `pageSize` | `Number` | **required** | Size of a page. |
| `sort` | `String` | **required** | Sorted fields. |
| `search` | `String` | **required** | Search text. |
| `searchFields` | `String` | **required** | Fields for searching. |
| `query` | `Object` | **required** | Query object. Passes to adapter. |

#### Results
**Type:** `Object` - List of found entities and count.

### `create`

Create a new entity.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| - | - | - | - |
*No input parameters.*

#### Results
**Type:** `Object` - Saved entity.

### `insert`

Create many new entities.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `entity` | `Object` | - | Entity to save. |
| `entities` | `Array.<Object>` | - | Entities to save. |

#### Results
**Type:** `Object`, `Array.<Object>` - Saved entity(ies).

### `get` ![Cached action](https://img.shields.io/badge/cache-true-blue.svg) 

Get entity by ID.

##### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `id` | `any`, `Array.<any>` | **required** | ID(s) of entity. |
| `populate` | `Array.<String>` | - | Field list for populate. |
| `fields` | `Array.<String>` | - | Fields filter. |
| `mapping` | `Boolean` | - | Convert the returned `Array` to `Object` where the key is the value of `id`. |

#### Results
**Type:** `Object`, `Array.<Object>` - Found entity(ies).


### `update`

Update an entity by ID.
> After update, clear the cache & call lifecycle events.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| - | - | - | - |
*No input parameters.*

#### Results
**Type:** `Object` - Updated entity.


### `remove`

Remove an entity by ID.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `id` | `any` | **required** | ID of entity. |

#### Results
**Type:** `Number` - Count of removed entities.

## Methods

DB adapters also has a set of helper [methods](services.html#Methods).

### `getById`

Get entity(ies) by ID(s).

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `id` | `String`, `Number`, `Array` | **required** | ID or IDs. |
| `decoding` | `Boolean` | **required** | Need to decode IDs. |

#### Results
**Type:** `Object`, `Array.<Object>` - Found entity(ies).


### `clearCache`

Clear cached entities

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| - | - | - | - |
*No input parameters.*

#### Results
**Type:** `Promise`


### `encodeID` 

Encode ID of entity.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `id` | `any` | **required** | - |

#### Results
**Type:** `any`


### `decodeID` 

Decode ID of entity.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `id` | `any` | **required** | - |

#### Results
**Type:** `any`

### `_find` ![Cached action](https://img.shields.io/badge/cache-true-blue.svg) 

Find entities by query.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `populate` | `Array.<String>` | - | Populated fields. |
| `fields` | `Array.<String>` | - | Fields filter. |
| `limit` | `Number` | **required** | Max count of rows. |
| `offset` | `Number` | **required** | Count of skipped rows. |
| `sort` | `String` | **required** | Sorted fields. |
| `search` | `String` | **required** | Search text. |
| `searchFields` | `String` | **required** | Fields for searching. |
| `query` | `Object` | **required** | Query object. Passes to adapter. |

#### Results
**Type:** `Array.<Object>`

List of found entities.


### `_count` ![Cached action](https://img.shields.io/badge/cache-true-blue.svg) 

Get count of entities by query.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `search` | `String` | **required** | Search text. |
| `searchFields` | `String` | **required** | Fields list for searching. |
| `query` | `Object` | **required** | Query object. Passes to adapter. |

#### Results
**Type:** `Number`

Count of found entities.


### `_list` ![Cached action](https://img.shields.io/badge/cache-true-blue.svg) 

List entities by filters and pagination results.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `populate` | `Array.<String>` | - | Populated fields. |
| `fields` | `Array.<String>` | - | Fields filter. |
| `page` | `Number` | **required** | Page number. |
| `pageSize` | `Number` | **required** | Size of a page. |
| `sort` | `String` | **required** | Sorted fields. |
| `search` | `String` | **required** | Search text. |
| `searchFields` | `String` | **required** | Fields for searching. |
| `query` | `Object` | **required** | Query object. Passes to adapter. |

#### Results
**Type:** `Object`

List of found entities and count.


### `_create` 

Create a new entity.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `params` | `Object` | - | Entity to save. |

#### Results
**Type:** `Object`

Saved entity.


### `_insert` 

Create many new entities.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `entity` | `Object` | - | Entity to save. |
| `entities` | `Array.<Object>` | - | Entities to save. |

#### Results
**Type:** `Object`, `Array.<Object>`

Saved entity(ies).


### `_get` ![Cached action](https://img.shields.io/badge/cache-true-blue.svg) 

Get entity by ID.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `id` | `any`, `Array.<any>` | **required** | ID(s) of entity. |
| `populate` | `Array.<String>` | - | Field list for populate. |
| `fields` | `Array.<String>` | - | Fields filter. |
| `mapping` | `Boolean` | - | Convert the returned `Array` to `Object` where the key is the value of `id`. |

#### Results
**Type:** `Object`, `Array.<Object>`

Found entity(ies).


### `_update` 

Update an entity by ID.
> After update, clear the cache & call lifecycle events.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `params` | `Object` | - | Entity to update. |

#### Results
**Type:** `Object`

Updated entity.


### `_remove` 

Remove an entity by ID.

#### Parameters
| Property | Type | Default | Description |
| -------- | ---- | ------- | ----------- |
| `id` | `any` | **required** | ID of entity. |

#### Results
**Type:** `Number`

Count of removed entities.

## Data Manipulation

You can easily use [Action hooks](actions.html#Action-hooks) to modify (e.g. add timestamps, hash user's passwords or remove sensitive info) before or after saving the data in DB. 

**Example of hooks adding a timestamp and removing sensitive data**
```js
"use strict";
const { ServiceBroker } = require("moleculer");
const DbService = require("moleculer-db");

const broker = new ServiceBroker();

broker.createService({
  name: "db-with-hooks",

  // Load DB actions
  mixins: [DbService],

  // Add Hooks to DB actions
  hooks: {
    before: {
      create: [
        function addTimestamp(ctx) {
          // Add timestamp
          ctx.params.createdAt = new Date();
          return ctx;
        }
      ]
    },
    after: {
      get: [
        // Arrow function as a Hook
        (ctx, res) => {
          // Remove sensitive data
          delete res.mail;
          delete res.phoneNumber;

          return res;
        }
      ]
    }
  }
});

const user = {
  name: "John Doe",
  mail: "john.doe@example.com",
  phoneNumber: 123456789
};

broker.start()
  // Insert user into DB
  // Call "create" action. Before hook will be triggered
  .then(() => broker.call("db-with-hooks.create", user))
  // Get user from DB
  // Call "get" action. After hook will be triggered
  .then(entry => broker.call("db-with-hooks.get", { id: entry._id }))
  .then(res => console.log(res))
  .catch(err => console.error(err));

```

## Populating
The service allows you to easily populate fields from other services. For example: If you have an `author` field in `post` entity, you can populate it with `users` service by ID of author. If the field is an `Array` of IDs, it will populate all entities via only one request


**Example of populate schema**
```js
broker.createService({
    name: "posts",
    mixins: [DbService],
    settings: {
        populates: {
            // Shorthand populate rule. Resolve the `voters` values with `users.get` action.
            "voters": "users.get",

            // Define the params of action call. It will receive only with username & full name of author.
            "author": {
                action: "users.get",
                params: {
                    fields: "username fullName"
                }
            },
            // In case the original field shouldn't be overwritten with the populated values.  
            // The reviewer field will be added to the result containing the values 
            // resolved by the "users.get" action based on the reviewerId field.
            "reviewer": {
                field: "reviewerId",
                action: "users.get",
                params: {
                    fields: "username fullName"
                }
            },

            // Custom populator handler function
            "rate"(ids, items, rule, ctx) {
                // items argument is a mutable array containing response items
                // the resolved value from promise do not matter - items array will always be passed as populated response
                // so you should du custom populate by mutating items, if confused check implementaion:
                // https://github.com/moleculerjs/moleculer-db/blob/master/packages/moleculer-db/src/index.js#L636
                
                return Promise.resolve(...);
            }
        }
    }
});

// List posts with populated authors
broker.call("posts.find", { populate: ["author"]}).then(console.log);
```

> The `populate` parameter is available in `find`, `list` and `get` actions.


## Lifecycle entity events
There are 3 lifecycle entity events which are called when entities are manipulated.

```js
broker.createService({
    name: "posts",
    mixins: [DbService],
    settings: {},

    afterConnected() {
        this.logger.info("Connected successfully");
    },

    entityCreated(json, ctx) {
        this.logger.info("New entity created!");
    },

    entityUpdated(json, ctx) {
        // You can also access to Context
        this.logger.info(`Entity updated by '${ctx.meta.user.name}' user!`);
    },

    entityRemoved(json, ctx) {
        this.logger.info("Entity removed", json);
    },    
});
```

> Please note! If you manipulate multiple entities (updateMany, removeMany), the `json` parameter will be a `Number` instead of entities!

## Extend with custom actions
Naturally you can extend this service with your custom actions.

```js
const DbService = require("moleculer-db");

module.exports = {
    name: "posts",
    mixins: [DbService],

    settings: {
        fields: ["_id", "title", "content", "votes"]
    },

    actions: {
        // Increment `votes` field by post ID
        vote(ctx) {
            return this.adapter.updateById(ctx.params.id, { $inc: { votes: 1 } });
        },

        // List posts of an author
        byAuthors(ctx) {
            return this.find({
                query: {
                    author: ctx.params.authorID
                },
                limit: ctx.params.limit || 10,
                sort: "-createdAt"
            });
        }
    }
}
```


## Mongo Adapter [![NPM version](https://img.shields.io/npm/v/moleculer-db-adapter-mongo.svg)](https://www.npmjs.com/package/moleculer-db-adapter-mongo)

This adapter is based on [MongoDB](http://mongodb.github.io/node-mongodb-native/).

### Install

```bash
$ npm install moleculer-db moleculer-db-adapter-mongo --save
```
{% note info Dependencies%}
To use this adapter you need to install [MongoDB](https://www.mongodb.com/) on you system.
{% endnote %}

### Usage

```js
"use strict";

const { ServiceBroker } = require("moleculer");
const DbService = require("moleculer-db");
const MongoDBAdapter = require("moleculer-db-adapter-mongo");

const broker = new ServiceBroker();

// Create a Mongoose service for `post` entities
broker.createService({
    name: "posts",
    mixins: [DbService],
    adapter: new MongoDBAdapter("mongodb://localhost/moleculer-demo"),
    collection: "posts"
});


broker.start()
// Create a new post
.then(() => broker.call("posts.create", {
    title: "My first post",
    content: "Lorem ipsum...",
    votes: 0
}))

// Get all posts
.then(() => broker.call("posts.find").then(console.log));
```

### Options

**Example with connection URI**
```js
new MongoDBAdapter("mongodb://localhost/moleculer-db")
```


**Example with connection URI & options**
```js
new MongoDBAdapter("mongodb://db-server-hostname/my-db", {
    keepAlive: 1
})

```

> More MongoDB examples can be found on [GitHub](https://github.com/moleculerjs/moleculer-db/tree/master/packages/moleculer-db-adapter-mongo/examples)

## Mongoose Adapter [![NPM version](https://img.shields.io/npm/v/moleculer-db-adapter-mongoose.svg)](https://www.npmjs.com/package/moleculer-db-adapter-mongoose)

This adapter is based on [Mongoose](https://mongoosejs.com/docs/).

### Install

```bash
$ npm install moleculer-db moleculer-db-adapter-mongoose mongoose --save
```

{% note info Dependencies%}
To use this adapter you need to install [MongoDB](https://www.mongodb.com/) on you system.
{% endnote %}

### Usage

```js
"use strict";

const { ServiceBroker } = require("moleculer");
const DbService = require("moleculer-db");
const MongooseAdapter = require("moleculer-db-adapter-mongoose");
const mongoose = require("mongoose");

const broker = new ServiceBroker();

// Create a Mongoose service for `post` entities
broker.createService({
    name: "posts",
    mixins: [DbService],
    adapter: new MongooseAdapter("mongodb://localhost/moleculer-demo"),
    model: mongoose.model("Post", mongoose.Schema({
        title: { type: String },
        content: { type: String },
        votes: { type: Number, default: 0}
    }))
});


broker.start()
// Create a new post
.then(() => broker.call("posts.create", {
    title: "My first post",
    content: "Lorem ipsum...",
    votes: 0
}))

// Get all posts
.then(() => broker.call("posts.find").then(console.log));
```

### Options

**Example with connection URI**
```js
new MongooseAdapter("mongodb://localhost/moleculer-db")
```

**Example with URI and options**
```js
new MongooseAdapter("mongodb://db-server-hostname/my-db", {
    user: process.env.MONGO_USERNAME,
    pass: process.env.MONGO_PASSWORD
    keepAlive: true
})
```

### Connect to multiple DBs

If your services are running on separate nodes and you wish to connect to multiple databases then you can use `model` in your service definition. On the other hand, if your services are running on a single node and you wish to connect to multiple databases, you should define the `schema` that will make multiple connections for you.

You can use example below for both cases with minimal changes (TypeScript).
You can find more detailed explanation at [mongoosejs documentation](https://mongoosejs.com/docs/connections.html#multiple_connections)
```typescript
// db/schema.ts
import mongoose from "mongoose";

export default new mongoose.Schema(
  {
    name: { $type: String, required: true },
    pictures: [String],
  },
  {
    minimize: false,
    strict: true,
    timestamps: true,
    typeKey: "$type",
    validateBeforeSave: true,
  },
)
  .index({ createdAt: 1 })
  .index({ name: 1 })
  .index({ updatedAt: 1 });
```

```typescript
// db/model.ts
import mongoose from "mongoose";
import schema from "./schema";

const modelName = "User";

export { modelName };
export default mongoose.model(modelName, schema);
```

```typescript
// users.service.ts
import DbService from "moleculer-db";
import MongooseDbAdapter from "moleculer-db-adapter-mongoose";
import { ServiceSchema } from "moleculer";
import hooks from "./hooks";
import methods from "./methods";
import model, { modelName } from "./db/model";
import schema from "./db/schema";

const connectionString = "mongodb://localhost:27017,localhost:27018,localhost:27019/entities?replicaSet=rs";

export default {
  adapter: new MongooseDbAdapter(connectionString, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
  }),

  hooks,
  methods,
  mixins: [DbService],
  modelName,
  name: "users",
  schema,

  settings: {
    rest: "users",
  },

  version: 2,
} as ServiceSchema;
```
Now for switch to `model` using you can do simply:
```typescript
//...
  hooks,
  methods,
  mixins: [DbService],
  model,
  name: "users",
//...
```

> More Mongoose examples can be found on [GitHub](https://github.com/moleculerjs/moleculer-db/tree/master/packages/moleculer-db-adapter-mongoose/examples)

## Sequelize Adapter [![NPM version](https://img.shields.io/npm/v/moleculer-db-adapter-sequelize.svg)](https://www.npmjs.com/package/moleculer-db-adapter-sequelize)

SQL adapter (Postgres, MySQL, SQLite & MSSQL) for Moleculer DB service with [Sequelize](https://github.com/sequelize/sequelize).

### Install

```bash
$ npm install moleculer-db-adapter-sequelize --save
```

You have to install additional packages for your database server:
```bash
# For SQLite
$ npm install sqlite3 --save

# For MySQL
$ npm install mysql2 --save

# For PostgreSQL
$ npm install pg pg-hstore --save

# For MSSQL
$ npm install tedious --save
```

### Usage

```js
"use strict";

const { ServiceBroker } = require("moleculer");
const DbService = require("moleculer-db");
const SqlAdapter = require("moleculer-db-adapter-sequelize");
const Sequelize = require("sequelize");

const broker = new ServiceBroker();

// Create a Sequelize service for `post` entities
broker.createService({
    name: "posts",
    mixins: [DbService],
    adapter: new SqlAdapter("sqlite://:memory:"),
    model: {
        name: "post",
        define: {
            title: Sequelize.STRING,
            content: Sequelize.TEXT,
            votes: Sequelize.INTEGER,
            author: Sequelize.INTEGER,
            status: Sequelize.BOOLEAN
        },
        options: {
            // Options from http://docs.sequelizejs.com/manual/tutorial/models-definition.html
        }
    },
});


broker.start()
// Create a new post
.then(() => broker.call("posts.create", {
    title: "My first post",
    content: "Lorem ipsum...",
    votes: 0
}))

// Get all posts
.then(() => broker.call("posts.find").then(console.log));
```

### Options
Every constructor arguments are passed to the `Sequelize` constructor. Read more about [Sequelize connection](http://docs.sequelizejs.com/manual/installation/getting-started.html).

**Example with connection URI**
```js
new SqlAdapter("postgres://user:pass@example.com:5432/dbname");
```

**Example with connection options**
```js
new SqlAdapter('database', 'username', 'password', {
    host: 'localhost',
    dialect: 'mysql'|'sqlite'|'postgres'|'mssql',

    pool: {
        max: 5,
        min: 0,
        idle: 10000
    },

    noSync: true // If true, the model will not be synced by Sequelize

    // SQLite only
    storage: 'path/to/database.sqlite'
});
```

> More Sequelize examples can be found on [GitHub](https://github.com/moleculerjs/moleculer-db/tree/master/packages/moleculer-db-adapter-sequelize/examples)
