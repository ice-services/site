## title: Actions

The actions are the callable/public methods of the service. The action calling represents a remote-procedure-call (RPC). It has request parameters & returns response, like a HTTP request.

If you have multiple instances of services, the broker will load balancing the request among instances. [Read more about balancing](balancing.html).

<div align="center">
![Action balancing diagram](assets/action-balancing.gif)
</div>

## Call services

To call a service, use the `broker.call` method. The broker looks for the service (and a node) which has the given action and call it. The function returns a `Promise`.

### Syntax

```js
const res = await broker.call(actionName, params, opts);
```

The `actionName` is a dot-separated string. The first part of it is the service name, while the second part of it represents the action name. So if you have a `posts` service with a `create` action, you can call it as `posts.create`.

The `params` is an object which is passed to the action as a part of the [Context](#Context). The service can access it via `ctx.params`. *It is optional.*

The `opts` is an object to set/override some request parameters, e.g.: `timeout`, `retryCount`. *It is optional.*

**Available calling options:**

| Name               | Type      | Default | Description                                                                                                                                                                                                                                                                                       |
| ------------------ | --------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `timeout`          | `Number`  | `null`  | Timeout of request in milliseconds. If the request is timed out and you don't define `fallbackResponse`, broker will throw a `RequestTimeout` error. To disable set `0`. If it's not defined, broker uses the `requestTimeout` value of broker options. [Read more](fault-tolerance.html#Timeout) |
| `retries`          | `Number`  | `null`  | Count of retry of request. If the request is timed out, broker will try to call again. To disable set `0`. If it's not defined, broker uses the `retryPolicy.retries` value of broker options. [Read more](fault-tolerance.html#Retry)                                                            |
| `fallbackResponse` | `Any`     | `null`  | Returns it, if the request has failed. [Read more](fault-tolerance.html#Fallback)                                                                                                                                                                                                                 |
| `nodeID`           | `String`  | `null`  | Target nodeID. If set, it will make a direct call to the given node.                                                                                                                                                                                                                              |
| `meta`             | `Object`  | `null`  | Metadata of request. Access it via `ctx.meta` in actions handlers. It will be transferred & merged at nested calls, as well.                                                                                                                                                                      |
| `parentCtx`        | `Context` | `null`  | Parent `Context` instance.                                                                                                                                                                                                                                                                        |
| `requestID`        | `String`  | `null`  | Request ID or correlation ID. *It appears in the metrics events.*                                                                                                                                                                                                                                 |


### Usages

**Call without params**

```js
broker.call("user.list")
    .then(res => console.log("User list: ", res));
```

**Call with params**

```js
broker.call("user.get", { id: 3 })
    .then(res => console.log("User: ", res));
```

**Call with async/await**

```js
const res = await broker.call("user.get", { id: 3 });
console.log("User: ", res);
```

**Call with options**

```js
broker.call("user.recommendation", { limit: 5 }, {
    timeout: 500,
    retries: 3,
    fallbackResponse: defaultRecommendation
}).then(res => console.log("Result: ", res));
```

**Call with error handling**

```js
broker.call("posts.update", { id: 2, title: "Modified post title" })
    .then(res => console.log("Post updated!"))
    .catch(err => console.error("Unable to update Post!", err));    
```

**Direct call: get health info from the "node-21" node**

```js
broker.call("$node.health", {}, { nodeID: "node-21" })
    .then(res => console.log("Result: ", res));    
```

### Metadata

Send meta information to services with `meta` property. Access it via `ctx.meta` in action handlers. Please note at nested calls the meta is merged.

```js
broker.createService({
    name: "test",
    actions: {
        first(ctx) {
            return ctx.call("test.second", null, { meta: {
                b: 5
            }});
        },
        second(ctx) {
            console.log(ctx.meta);
            // Prints: { a: "John", b: 5 }
        }
    }
});

broker.call("test.first", null, { meta: {
    a: "John"
}});
```

The `meta` is sent back to the caller service. Use it to send extra meta information back to the caller. E.g.: send response headers back to API gateway or set resolved logged in user to metadata.

```js
broker.createService({
    name: "test",
    actions: {
        async first(ctx) {
            await ctx.call("test.second", null, { meta: {
                a: "John"
            }});

            console.log(ctx.meta);
            // Prints: { a: "John", b: 5 }
        },
        second(ctx) {
            // Modify meta
            ctx.meta.b = 5;
        }
    }
});
```

When making internal calls to actions (`this.actions.xy()`) you should set `parentCtx` to pass `meta` data.

**Internal calls**

```js
broker.createService({
  name: "mod",
  actions: {
    hello(ctx) {
      console.log(ctx.meta);
      // Prints: { user: 'John' }
      ctx.meta.age = 123
      return this.actions.subHello(ctx.params, { parentCtx: ctx });
    },

    subHello(ctx) {
      console.log("meta from subHello:", ctx.meta);
      // Prints: { user: 'John', age: 123 }
      return "hi!";
    }
  }
});

broker.call("mod.hello", { param: 1 }, { meta: { user: "John" } });
```

### Timeout

Timeout can be set in action definition, as well. It overwrites the global broker [`requestTimeout` option](fault-tolerance.html#Timeout), but not the `timeout` in calling options.

**Example**

```js
// moleculer.config.js
module.exports = {
    nodeID: "node-1",
    requestTimeout: 3000
};
 // greeter.service.js
module.exports = {
    name: "greeter",
    actions: {
        normal: {
            handler(ctx) {
                return "Normal";
            }
        },
         slow: {
            timeout: 5000, // 5 secs
            handler(ctx) {
                return "Slow";
            }
        }
    },
```

**Calling examples**

```js
// It uses the global 3000 timeout
await broker.call("greeter.normal");
 // It uses the 5000 timeout from action definition
await broker.call("greeter.slow");
 // It uses 1000 timeout from calling option
await broker.call("greeter.slow", null, { timeout: 1000 });
```

## Streaming

Moleculer supports Node.js streams as request `params` and as response. Use it to transfer uploaded file from a gateway or encode/decode or compress/decompress streams.

### Examples

**Send a file to a service as a stream**

```js
const stream = fs.createReadStream(fileName);

broker.call("storage.save", stream, { meta: { filename: "avatar-123.jpg" }});
```

Please note, the `params` should be a stream, you cannot add any more variables to the `params`. Use the `meta` property to transfer additional data.

**Receiving a stream in a service**

```js
module.exports = {
    name: "storage",
    actions: {
        save(ctx) {
            const s = fs.createWriteStream(`/tmp/${ctx.meta.filename}`);
            ctx.params.pipe(s);
        }
    }
};
```

**Return a stream as response in a service**

```js
module.exports = {
    name: "storage",
    actions: {
        get: {
            params: {
                filename: "string"
            },
            handler(ctx) {
                return fs.createReadStream(`/tmp/${ctx.params.filename}`);
            }
        }
    }
};
```

**Process received stream on the caller side**

```js
const filename = "avatar-123.jpg";
broker.call("storage.get", { filename })
    .then(stream => {
        const s = fs.createWriteStream(`./${filename}`);
        stream.pipe(s);
        s.on("close", () => broker.logger.info("File has been received"));
    })
```

**AES encode/decode example service**

```js
const crypto = require("crypto");
const password = "moleculer";

module.exports = {
    name: "aes",
    actions: {
        encrypt(ctx) {
            const encrypt = crypto.createCipher("aes-256-ctr", password);
            return ctx.params.pipe(encrypt);
        },

        decrypt(ctx) {
            const decrypt = crypto.createDecipher("aes-256-ctr", password);
            return ctx.params.pipe(decrypt);
        }
    }
};
```

## Action visibility

The action has a `visibility` property to control the visibility & callability of service actions.

**Available values:**

- `published` or `null`: public action. It can be called locally, remotely and can be published via API Gateway
- `public`: public action, can be called locally & remotely but not published via API GW
- `protected`: can be called only locally (from local services)
- `private`: can be called only internally (via `this.actions.xy()` inside service)

**Change visibility**

```js
module.exports = {
    name: "posts",
    actions: {
        // It's published by default
        find(ctx) {},
        clean: {
            // Callable only via `this.actions.clean`
            visibility: "private",
            handler(ctx) {}
        }
    },
    methods: {
        cleanEntities() {
            // Call the action directly
            return this.actions.clean();
        }
    }
}
```

> The default values is `null` (means `published`) due to backward compatibility.

## Action hooks

Action hooks are pluggable and reusable middleware functions that can be registered `before`, `after` or on `errors` of service actions. A hook is either a `Function` or a `String`. In case of a `String` it must be equal to service's [method](services.html#Methods) name.

### Service level declaration

Hooks can be assigned to a specific action (by indicating action `name`) or all actions (`*`) in service.

{% note warn%} Please notice that hook registration order matter as it defines sequence by which hooks are executed. For more information take a look at [hook execution order](#Execution-order). {% endnote %}

**Before hooks**

```js
const DbService = require("moleculer-db");

module.exports = {
    name: "posts",
    mixins: [DbService]
    hooks: {
        before: {
            // Define a global hook for all actions
            // The hook will call the `resolveLoggedUser` method.
            "*": "resolveLoggedUser",

            // Define multiple hooks for action `remove`
            remove: [
                function isAuthenticated(ctx) {
                    if (!ctx.user)
                        throw new Error("Forbidden");
                },
                function isOwner(ctx) {
                    if (!this.checkOwner(ctx.params.id, ctx.user.id))
                        throw new Error("Only owner can remove it.");
                }
            ]
        }
    },

    methods: {
        async resolveLoggedUser(ctx) {
            if (ctx.meta.user)
                ctx.user = await ctx.call("users.get", { id: ctx.meta.user.id });
        }
    }
}
```

**After & Error hooks**

```js
const DbService = require("moleculer-db");

module.exports = {
    name: "users",
    mixins: [DbService]
    hooks: {
        after: {
            // Define a global hook for all actions to remove sensitive data
            "*": function(ctx, res) {
                // Remove password
                delete res.password;

                // Please note, must return result (either the original or a new)
                return res;
            },
            get: [
                // Add a new virtual field to the entity
                async function (ctx, res) {
                    res.friends = await ctx.call("friends.count", { query: { follower: res._id }});

                    return res;
                },
                // Populate the `referrer` field
                async function (ctx, res) {
                    if (res.referrer)
                        res.referrer = await ctx.call("users.get", { id: res._id });

                    return res;
                }
            ]
        },
        error: {
            // Global error handler
            "*": function(ctx, err) {
                this.logger.error(`Error occurred when '${ctx.action.name}' action was called`, err);

                // Throw further the error
                throw err;
            }
        }
    }
};
```

### Action level declaration

Hooks can be also registered inside action declaration.

{% note warn%} Please note that hook registration order matter as it defines sequence by which hooks are executed. For more information take a look at [hook execution order](#Execution-order). {% endnote %}

**Before & After hooks**

```js
broker.createService({
    name: "greeter",
    actions: {
        hello: {
            hooks: {
                before(ctx) {
                    broker.logger.info("Before action hook");
                },
                after(ctx, res) {
                    broker.logger.info("After action hook"));
                    return res;
                }
            },

            handler(ctx) {
                broker.logger.info("Action handler");
                return `Hello ${ctx.params.name}`;
            }
        }
    }
});
```

### Execution order

It is important to keep in mind that hooks have a specific execution order. This is especially important to remember when multiple hooks are registered at different ([service](#Service-level-declaration) and/or [action](#Action-level-declaration)) levels. Overall, the hooks have the following execution logic:

- `before` hooks: global (`*`) `->` service level `->` action level.

- `after` hooks: action level `->` service level `->` global (`*`).

**Example of a global, service & action level hook execution chain**

```js
broker.createService({
    name: "greeter",
    hooks: {
        before: {
            "*"(ctx) {
                broker.logger.info(chalk.cyan("Before all hook"));
            },
            hello(ctx) {
                broker.logger.info(chalk.magenta("  Before hook"));
            }
        },
        after: {
            "*"(ctx, res) {
                broker.logger.info(chalk.cyan("After all hook"));
                return res;
            },
            hello(ctx, res) {
                broker.logger.info(chalk.magenta("  After hook"));
                return res;
            }
        },
    },

    actions: {
        hello: {
            hooks: {
                before(ctx) {
                    broker.logger.info(chalk.yellow.bold("    Before action hook"));
                },
                after(ctx, res) {
                    broker.logger.info(chalk.yellow.bold("    After action hook"));
                    return res;
                }
            },

            handler(ctx) {
                broker.logger.info(chalk.green.bold("      Action handler"));
                return `Hello ${ctx.params.name}`;
            }
        }
    }
});
```

**Output produced by global, service & action level hooks**

```bash
INFO  - Before all hook
INFO  -   Before hook
INFO  -     Before action hook
INFO  -       Action handler
INFO  -     After action hook
INFO  -   After hook
INFO  - After all hook
```

### Reusability

The most efficient way of reusing hooks is by declaring them as service methods in a separate file and, later, import them with the [mixin](services.html#Mixins) mechanism. This way a single hook can be easily shared across multiple actions.

**Mixin**

```js
module.exports = {
    methods: {
        checkIsAuthenticated(ctx) {
            if (!ctx.meta.user)
                throw new Error("Unauthenticated");
        },
        checkUserRole(ctx) {
            if (ctx.action.role && ctx.meta.user.role != ctx.action.role)
                throw new Error("Forbidden");
        },
        checkOwner(ctx) {
            // Check the owner of entity
        }
    }
}
```

**Use mixin methods in hooks**

```js
const MyAuthMixin = require("./my.mixin");

module.exports = {
    name: "posts",
    mixins: [MyAuthMixin]
    hooks: {
        before: {
            "*": ["checkIsAuthenticated"],
            create: ["checkUserRole"],
            update: ["checkUserRole", "checkOwner"],
            remove: ["checkUserRole", "checkOwner"]
        }
    },

    actions: {
        find: {
            // No required role
            handler(ctx) {}
        },
        create: {
            role: "admin",
            handler(ctx) {}
        },
        update: {
            role: "user",
            handler(ctx) {}
        }
    }
};
```

### Local Storage

The `locals` property of `Context` object is a simple storage that can be used to store some additional data and pass it to the action handler. `locals` property and hooks are a powerful combo:

**Setting `ctx.locals` in before hook**

```js
module.exports = {
    name: "user",

    hooks: {
        before: {
            async get(ctx) {
                const entity = await this.findEntity(ctx.params.id);
                ctx.locals.entity = entity;
            }
        }
    },

    actions: {
        get: {
            params: {
                id: "number"
            },
            handler(ctx) {
                this.logger.info("Entity", ctx.locals.entity);
            }
        }
    }
}
```

## Context

When you call an action, the broker creates a `Context` instance which contains all request information and passes it to the action handler as a single argument.

**Available properties & methods of `Context`:**

| Name              | Type            | Description                                                      |
| ----------------- | --------------- | ---------------------------------------------------------------- |
| `ctx.id`          | `String`        | Context ID                                                       |
| `ctx.broker`      | `ServiceBroker` | Instance of the broker.                                          |
| `ctx.action`      | `Object`        | Instance of action definition.                                   |
| `ctx.nodeID`      | `String`        | The caller or target Node ID.                                    |
| `ctx.caller`      | `String`        | Action name of the caller. E.g.: `v3.myService.myAction`         |
| `ctx.requestID`   | `String`        | Request ID. If you make nested-calls, it will be the same ID.    |
| `ctx.parentID`    | `String`        | Parent context ID (in nested-calls).                             |
| `ctx.params`      | `Any`           | Request params. *Second argument from `broker.call`.*            |
| `ctx.meta`        | `Any`           | Request metadata. *It will be also transferred to nested-calls.* |
| `ctx.locals`      | `Any`           | Local data. *It will be also transferred to nested-calls.*       |
| `ctx.level`       | `Number`        | Request level (in nested-calls). The first level is `1`.         |
| `ctx.call()`      | `Function`      | Make nested-calls. Same arguments like in `broker.call`          |
| `ctx.emit()`      | `Function`      | Emit an event, same as `broker.emit`                             |
| `ctx.broadcast()` | `Function`      | Broadcast an event, same as `broker.broadcast`                   |


### Context tracking

If you want graceful service shutdowns, enable the Context tracking feature in broker options. If you enable it, all services will wait for all running contexts before shutdown. A timeout value can be defined with `shutdownTimeout` broker option. The default values is `5` seconds.

**Enable context tracking & change the timeout value.

```js
const broker = new ServiceBroker({
    nodeID: "node-1",
    tracking: {
        enabled: true,
        shutdownTimeout: 10 * 1000
    }
});
```

> The shutdown timeout can be overwritten by `$shutdownTimeout` property in service settings.

**Disable tracking in calling option**

```js
broker.call("posts.find", {}, { tracking: false });
```