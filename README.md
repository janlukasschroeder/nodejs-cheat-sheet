# Node.js - Cheat Sheet

# Node.js Internals - Event Loop and how it all works

**TLDR**

What's the output of the example code look like?

```js
const fs = require('fs');

setImmediate(() => console.log(1));
Promise.resolve().then(() => console.log(2));
process.nextTick(() => console.log(3));
fs.readFile(__filename, () => {
  console.log(4);
  setTimeout(() => console.log(5));
  setImmediate(() => console.log(6));
  process.nextTick(() => console.log(7));
});
console.log(8);
```

Answer: 8, 3, 2, 1, 4, 7, 6, 5

Both, Node.js and JavaScript in your browser, have an event loop. The event loop manages executing and scheduling tasks in separate stacks. It runs in a loop.

Callbacks are executed when I/O events happen, like a message received on a socket, a file changing on disk, a `setTimeout()` callback being ready to run, etc. The operating system notifies the program that something has happened. libuv code is used to translate between the operating system and Node.js.

## Event Loop Phases

Each phase maintains a queue of callbacks that are to be executed. Callbacks are executed once the application starts processing the next phase.

![event-loop](/assets/event-loop-phases.png)

### Poll

I/O-related callbacks are executed here. The event loop starts here when main application code starts running.

### Check

`setImmediate()` callbacks are executed here.

### Close

Executes callbacks that are triggered via EventEmitter `close` events. For example, when a `net.Server` TCP server closes.

### Timers

Executes callbacks scheduled using `setTimeout()` and `setInterval()`

### Pending

Special system events run in this pase, e.g. when `net.Socket` TCP socket throws an `ECONNREFUSED` error.

---

There are also two microtask queues that can have callbacks added to them while a phase is running.

- First queue handles callbacks registered with `process.nextTick()`
- Second queue handles promises that resolve or reject.

Callbacks in the two microtask queues are executed everytime before a normal phase starts.
Callbacks in the first microtask queue run before callbacks in the second microtask queue.

When application starts running, the event loop is also started and the phases are handled one at a time. Node.js adds callbacks to the queues of different phases while the application runs. When event loops gets to a phase, it will run all the callbacks in that phase's queue. Once all callbacks of a queue are executed, the event loop moved on to the next phase.

If the application runs out of things to do but is waiting for I/O operations to complete, it'll hang out in the poll phase.

# Modules

- The `require`/`import` function is synchronous
- Node v12 and below uses `require` (CommonJS modules), Node v14 and above uses `import` (ECMAScript modules)
- Each module is only loaded and evaluated the first time it is required. Any subsequent calls of `require()` return the cached module.
- The path of the module determines if a cached version is retrieved or not, not the name of the module.
- Avoid circular dependencies

## CommonJS Modules

```js
// addTwo.js
function addTwo(num) {
  return num + 2;
}
module.exports.addTwo = addTwo;
```

```js
// app.js
const { addTwo } = require('./addTwo.js');
// Prints: 6
console.log(addTwo(4));
```

## ES Modules

ES module files can end with `.js` or `.mjs`. If file ends with `.js`, ensure to add `"type": "module"` to `package.json`

```js
// addTwo.mjs
function addTwo(num) {
  return num + 2;
}
export { addTwo };
```

```js
// app.mjs
import { addTwo } from './addTwo.mjs';
// Prints: 6
console.log(addTwo(4));
```

## IIFE - Immeditately Invoked Function Expression

IIFE is a self-invoking function to create a private scope.

## Revealing Module Pattern

```js
const iifeModule = (() => {
  const privateFunction = () => {};
  const privateArray = [];

  const exported = {
    publicFunctionA: () => {},
    publicFunctionB: () => {},
  };

  return exported;
})(); // self-invoking
```

## Substack pattern

```js
// logger.js
module.exports = (message) => {
  console.log(`info: ${message}`);
};

module.exports.verbose = (message) => {
  console.log(`verbose: ${message}`);
};
```

```js
// main.js
const logger = require('./logger.js');
logger('This is an info message');
logger.verbose('This is a verbose message');
```

## Stateful Exports

A single module instance can be shared across different modules.

```js
// logger.js
class Logger {
  constructor(name) {
    this.count = 0;
    this.name = name;
  }
  log(message) {
    this.count++;
    console.log(`[${this.name}] ${message}`);
  }
}

module.exports = new Logger('DEFAULT');
```

The same instance can be imported.

```js
// main.js
const logger = require('./logger');
logger.log('This is an info message');
```

# Asynchronous: Callbacks vs Events

Callback pattern: call a function (callback) once the execution is complete.

Observer pattern: a subject can notify (send an event) a set of observers (listeners) when a change in its state occurs.

### Callback Example

```js
// callback.js
function callbackExample(a, b, callback) {
  setTimeout(() => callback(null, a + b), 100);
}

callbackExample(1, 2, (err, result) => {
  err ? console.error(err) : console.log(result);
});
```

### Observer Example

Observers use `EventEmitter`s.

```js
// observer.js
const { EventEmitter } = require('events');

function observerExample(a, b) {
  const emitter = new EventEmitter();

  try {
    const result = a + b;
    emitter.emit('calculated', result);
  } catch (error) {
    emitter.emit('error', error);
  }
}

observerExample(1, 2)
  .on('calculated', (result) => console.log(result))
  .on('error', (error) => console.error(error));
```

When subscribing to observables with a long life span, it is extremely important that we unsubscribe our listeners once they are no longer needed. This allows us to release the memory used by the objects in a listener's scope and prevent memory leaks.

# Singleton

Use cases include: single database instance being instantiated at the beginning of the application so that every component can use the single shared instance.

```js
// db-instance.js
const Database = require('database.js');
// binding the new instance to the exported dbInstance ensures that only one instance is instantiated
// because Node.js caches the module
module.exports.dbInstance = new Database('my-db', {
  url: '',
  username: '',
  password: '',
});
```

```js
// file-1.js
const { dbInstance } = require('./db-instance');
```

```js
// file-2.js
const { dbInstance } = require('./db-instance');
```

Both modules, `file-1` and `file-2`, use the same DB instance.

# Factory

Separate the creation of an object from its implementation.

```js
// logger.js
// module is used as a factory
function Logger(name) {
  // check if 'this' exists and if 'this' is an instance of Logger
  // if not, then return new instance
  if (!(this instanceof Logger)) {
    return new Logger(name); // factory
  }
  this.name = name;
}
module.exports = Logger;
```

```js
// main.js
const Logger = require('./logger.js');
const dbLogger = Logger('DB'); // no need to call "new Logger()"
```

# Revealing Constructor

```js
//                    (1)               (2)          (3)
const object = new SomeClass(function executor(revealedMembers) {
  // manipulation code ...
});
```

(1) constructor that takes a function (2) as input
(2) the executor which is invoked at creation time and receives a subset of the object's internals as input (3)

# Builder

```js
const myBoat = new BoatBuilder()
  .withMotors(2, 'Best Motor Co. ', 'OM123')
  .withSails(1, 'fabric', 'white')
  .withCabin()
  .hullColor('blue')
  .build();
```

```js
class BoatBuilder {
  withMotors(count, brand, model) {
    this.hasMotor = true;
    this.motorCount = count;
    this.motorBrand = brand;
    this.motorModel = model;
    return this;
  }
  withSails(count, material, color) {
    this.hasSails = true;
    this.sailsCount = count;
    this.sailsMaterial = material;
    this.sailsColor = color;
    return this;
  }
  hullColor(color) {
    this.hullColor = color;
    return this;
  }
  withCabin() {
    this.hasCabin = true;
    return this;
  }
  build() {
    return new Boat({
      hasMotor: this.hasMotor,
      motorCount: this.motorCount,
      motorBrand: this.motorBrand,
      motorModel: this.motorModel,
      hasSails: this.hasSails,
      sailsCount: this.sailsCount,
      sailsMaterial: this.sailsMaterial,
      sailsColor: this.sailsColor,
      hullColor: this.hullColor,
      hasCabin: this.hasCabin,
    });
  }
}
```

# Proxy

Object that controls access to another object, called the subject.

```js
// proxy.js
function createProxy(subject) {
  return {
    //proxied method
    hello: () => subject.hello() + ' world!',

    //delegated method
    goodbye: () => subject.goodbye.apply(subject, arguments),
  };
}
module.exports = createProxy;
```

ES2015 provides `Proxy` object:

```js
const proxy = new Proxy(target, handler);
```

# Decorator

# Iterator

Iterator objects implements a `next()` method. Each time the method is called, the function returns the next element in the iteration through an object havin two properties - `done` and `value`.

- `done` is `true` when the iteration is complete. Otherwise, `done` will be `undefined` or `false`.
- `value` contains the current element of the iteration.

```js
const A_CHAR_CODE = 65;
const Z_CHAR_CODE = 90;

function createAlphabetIterator() {
  let currCode = A_CHAR_CODE;

  return {
    next() {
      const currChar = String.fromCodePoint(currCode);
      if (currCode > Z_CHAR_CODE) {
        return { done: true };
      }
      currCode++;
      return { value: currChar, done: false };
    },
  };
}

const iterator = createAlphabetIterator();

let iterationResult = iterator.next();

while (!iterationResult.done) {
  console.log(iterationResult.value);
  iterationResult = iterator.next();
}
```

# Generator

Onvoking `next()` on the generator object will start or resume the execution of the generator until the `yield` instruction is invoked or the generator returns (either implicitly or explicitly with a return instruction). `next()` method accepts an argument.

```js
function* fruitGenerator() {
  yield 'peach';
  yield 'watermelon';
  return 'summer';
}

const fruitGeneratorObj = fruitGenerator();

console.log(fruitGeneratorObj.next()); // { value: 'peach', done: false }
console.log(fruitGeneratorObj.next()); // { value: 'watermelon', done: false }
console.log(fruitGeneratorObj.next()); // { value: 'summer', done: true }

// or use a for loop
for (const fruit of fruitGenerator()) {
  console.log(fruit);
}
```

```js
// generator with argument
function* twoWayGenerator() {
  const what = yield null;
  yield 'Hello ' + what;
}
const twoWay = twoWayGenerator();
twoWay.next(); // generator reaches the first yield and pauses
console.log(twoWay.next('world')); // generator sets "what" to "world" and proceeds to the next yield, returning "Hello world"
```

# Strategy

```js
// config.js
const fs = require('fs');
const objectPath = require('object-path');

class Config {
  constructor(strategy) {
    this.data = {};
    this.strategy = strategy;
  }

  read(file) {
    this.data = this.strategy.deserialize(fs.readFileSync(file, 'utf-8')); // use strategy's implementation of deseri
  }

  save(file) {
    fs.writeFileSync(file, this.strategy.serialize(this.data));
  }

  get(path) {
    return objectPath.get(this.data, path);
  }

  set(path, value) {
    return objectPath.set(this.data, path, value);
  }
}

module.exports = Config;
```

```js
// json-strategy.js
module.exports.json = {
  deserialize: (data) => JSON.parse(data),
  serialize: (data) => JSON.stringify(data, null, '  '),
};
```

```js
// main.js
const Config = require('./config');
const jsonStrategy = require('./json-strategy');

const jsonConfig = new Config(jsonStrategy);
jsonConfig.read('./some-json-file.json');
jsonConfig.set('foo-key', 'bar-value');
jsonConfig.save('./some-json-file_modified.json');
```

# Template

Similar to a strategy.

```js
// config-template.js
import { promises as fsPromises } from 'fs';
import objectPath from 'object-path';

export class ConfigTemplate {
  async load(file) {
    this.data = this._deserialize(await fsPromises.readFile(file, 'utf-8'));
  }
  async save(file) {
    await fsPromises.writeFile(file, this._serialize(this.data));
  }
  get(path) {
    return objectPath.get(this.data, path);
  }
  set(path, value) {
    return objectPath.set(this.data, path, value);
  }
  _serialize() {
    throw new Error('_serialize() must be implemented');
  }
  _deserialize() {
    throw new Error('_deserialize() must be implemented');
  }
}
```

```js
// json-config.js
import { ConfigTemplate } from './config-template.js';

// JsonConfig extends our template
export class JsonConfig extends ConfigTemplate {
  _deserialize(data) {
    return JSON.parse(data);
  }
  _serialize(data) {
    return JSON.stringify(data, null, '  ');
  }
}
```

```js
// main.js
import { JsonConfig } from './json-config.js';

async function main() {
  const jsonConfig = new JsonConfig();
  await jsonConfig.load('./some-json-file.json');
  jsonConfig.set('foo-key', 'bar-value');
  await jsonConfig.save('./some-json-file_modified.json');
}

main();
```

# Middleware

New messages travel through each registered middleware, one after the other.

Most popular in Express: `.use()`. Using ZeroMQ as an example.

```js
// zmqMiddlewareManager.js
export class ZmqMiddlewareManager {
  constructor(socket) {
    this.socket = socket;
    this.inboundMiddleware = [];
    this.outboundMiddleware = [];
    this.handleIncomingMessages().catch((err) => console.error(err));
  }

  async handleIncomingMessages() {
    for await (const [message] of this.socket) {
      await this.executeMiddleware(this.inboundMiddleware, message).catch(
        (err) => {
          console.error('Error while processing the message', err);
        }
      );
    }
  }

  async send(message) {
    const finalMessage = await this.executeMiddleware(
      this.outboundMiddleware,
      message
    );
    return this.socket.send(finalMessage);
  }

  // allow registration of middleware
  use(middleware) {
    if (middleware.inbound) {
      this.inboundMiddleware.push(middleware.inbound);
    }
    if (middleware.outbound) {
      this.outboundMiddleware.unshift(middleware.outbound);
    }
  }

  // execute all registered middlewares
  // run inbound middlewares when msg is received, outbound middlewares when msg should be send
  async executeMiddleware(middlewares, initialMessage) {
    let message = initialMessage;
    for await (const middlewareFunc of middlewares) {
      message = await middlewareFunc.call(this, message);
    }
    return message;
  }
}
```

```js
// jsonMiddleware.js
export const jsonMiddleware = function () {
  return {
    inbound(message) {
      return JSON.parse(message.toString());
    },
    outbound(message) {
      return Buffer.from(JSON.stringify(message));
    },
  };
};
```

```js
// zlibMiddleware.js
import { inflateRaw, deflateRaw } from 'zlib';
import { promisify } from 'util';
const inflateRawAsync = promisify(inflateRaw);
const deflateRawAsync = promisify(deflateRaw);
export const zlibMiddleware = function () {
  return {
    inbound(message) {
      return inflateRawAsync(Buffer.from(message));
    },
    outbound(message) {
      return deflateRawAsync(message);
    },
  };
};
```

```js
// server.js
import zeromq from 'zeromq';
import { ZmqMiddlewareManager } from './zmqMiddlewareManager.js';
import { jsonMiddleware } from './jsonMiddleware.js';
import { zlibMiddleware } from './zlibMiddleware.js';

async function main() {
  const socket = new zeromq.Reply();
  await socket.bind('tcp://127.0.0.1:5000');
  const zmqm = new ZmqMiddlewareManager(socket);
  zmqm.use(zlibMiddleware());
  zmqm.use(jsonMiddleware());
  zmqm.use({
    async inbound(message) {
      console.log('Received', message);
      if (message.action === 'ping') {
        await this.send({ action: 'pong', echo: message.echo });
      }
      return message;
    },
  });
  console.log('Server started');
}

main();
```

# Task Pattern

```js
function createTask(target, ...args) {
  return () => {
    target(...args);
  };
}
```

# Clustering, Worker Pool & CPU Intensive Tasks

## Web Server Clustering

```js
// cpu-heavy-task.js
// listen for new messages from the parent process
process.on('message', (msg) => {
  // perform CPU heavy task
  const result = fib(parseInt(msg.n));
  // send the result of the calculation to the parent process
  process.send({ result, id: msg.id });
});

function fib(n) {
  if (n < 2) return 1;
  else return fib(n - 2) + fib(n - 1);
}
```

```js
// server.js
const http = require('http');
const { fork } = require('child_process');
let { EventEmitter } = require('events');

const eventHandler = new EventEmitter();

const child = fork(`${__dirname}/cpu-heavy-task.js`);

// when the forked process returns a response, emit an event
child.on('message', (msg) => eventHandler.emit(msg.id, msg.result));

const server = http.createServer(function (req, res) {
  // endpoint to send requests to to check if server still accepts requests once CPU heavy task runs
  // should always return a response, even with heavy load
  if (req.url === '/ping') {
    res.end(`pong`);
    return;
  }

  const id = Math.random() * 100; // generate a message ID

  child.send({ n: 50, id }); // start CPU heavy task by sending a message to the forked function

  // await response from the event handler that in turn recieves the response from the CPU heavy function
  eventHandler.once(id, (result) => {
    res.end(`${result}`);
  });
});

server.listen(8080, () => console.log('running on port 8080'));
```

## Thread Pool

TBC

## Worker in Browser

Perform blocking tasks in a seperate process using a worker.

```js
// worker.js
export default () => {
  self.addEventListener('message', (e) => {
    if (!e) return;

    // wrap in try/catch if you want to support IE11 and older browsers
    // that don't support Promises. The implementation below doesn't work
    // even when polyfills are loaded in the main project because
    // the worker runs in a different context, ie no webpack bundles here.
    try {
      const fetchData = (url, isJSON = true) => {
        return new Promise((resolve, reject) => {
          function reqListener() {
            if (this.status !== 200) {
              return reject();
            }
            const response = isJSON
              ? JSON.parse(this.responseText)
              : this.responseText;
            resolve(response);
          }
          const oReq = new XMLHttpRequest();
          oReq.addEventListener('load', reqListener);
          oReq.open('GET', url);
          oReq.send();
        });
      };

      const baseUrl = 'https://server.com/';
      const { itemId } = e.data;
      const jsonUrl = baseUrl + articleId + '.json';
      const htmlUrl = baseUrl + articleId + '.html';

      // my use case requires 2 requests in parallel.
      const tasks = [fetchData(jsonUrl), fetchData(htmlUrl, false)];

      Promise.all(tasks)
        .then((data) => {
          // send response to parent process
          postMessage({ json: data[0], html: data[1] });
        })
        .catch((error) => {
          postMessage({ isError: true, error });
        });
    } catch (error) {
      postMessage({ isError: true });
    }
  });
};
```

# Docker

TBC

# Must Have Packages

## Server

lodash (object manipulation + everything), async (pooling, max # async in parallel), axios (HTTP requests), passport (authentication strategies), helmet (Express security), express-rate-limit, moment-timezone (date/time converter), ajv (JSON validator), yup (JSON validator), validator (string validator/sanitizer), uuid v4 (generated UUIDs), jsonwebtoken (web token manipulation)

## DB Access

knex (DB schemas), sequelize (ORM for Postgres, MySQL), pq (Postgres adapter), ioredis (Redis adapter), mongoose (MongoDB adapter), graphql/apollo-server

## Dev Process

nvm (Node version manager), pm2 (server management), prettify (prettifier), eslint (code linter), dotenv (.env loading), nodemon (restart .js automatically), husky (git hooks, pre-commit, etc), jest (testing), mocha (testing), chai (testing) winston (logging), debug (debugging)

# Other Terms

- Closures
- Encapsulation
- Object augmentation (or monkey patching)
- Single Responsibility Principle = every module should have responsibility over a single functionality
- Small surface area
- Named exports
- Chaos engineering = randomly introducing bugs and errors to harden a system
- Semantic versioning
- Least Recently Used (LRU)
- Prototype pollution
- Expontential backoff
- Jitter = random variance, such as an increase of request timing of 10%.
- Tree shacking

# Bundlers

- webpack
- parcel
- rollup
- browserfy

# Server management

- `nvm` to manage local version of Node.js
- `pm2` to manage Node.js processes

# Server benchmarking

autocannon, Apache Bench (ab), wrk, Siege

# General Tools

- Sentry to capture exceptions
- Segment to funnel events
- MixPanel & Heap to track user generated events
- Zipkin and Jaeger - trace distirbuted calls
- Grafana - visualizing Graphite metrics
- Cabot - polling health of an application and triggering alerts
- Datadog, SumoLogic, Splunk, NewRelic
- ELK - ElasticSearch, Logstach, Kibana

# Databases, Storage

- PostgreSQL, MySQL, ElasticSearch, MongoDB, DynamoDB, S3, Redis, GraphQL, Neo4j, LevelDB

# Real-Time Communications

- Websocket, socket.io, RabbitMQ, ZeroMQ, Apache Kafka, Redis pub/sub

# Hosting

- AWS, Heroku, Google Cloud

# Edge Processing

- CloudFront Lambda, Cloudera Edge

# Rapid Prototyping

- netlify.com
- codesandbox.io
- runkit.com
- reqbin.com

# Authentication Providers

Okta, auth0, AWS Cognito

# References

- Distributed Systems with Node.js, O'Reilly Media
- Node.js Design Patterns: Design and implement production-grade Node.js applications using proven patterns and techniques, 3rd Edition
- Node Cookbook: Actionable solutions for the full spectrum of Node.js 8 development, 3rd Edition
