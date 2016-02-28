gremlin-javascript
==================

A WebSocket JavaScript client for TinkerPop3 Gremlin Server. Works in Node.js and modern browsers.

## Installation

```
npm install gremlin --save
```

In the browser, you can require the module with browserify or directly insert a `<script>` tag, which will expose a global `gremlin` variable:
```html
<script type="text/javascript" src="gremlin.js"></script>
```

## Quick start

```
import { createClient } from 'gremlin';

const gremlinClient = createClient();

gremlinClient.execute('g.V', (err, results) => {
  if (err) {
    return console.error(err)
  }

  console.log(results);
});
```

## Usage

### Creating a new client

```javascript
// Assuming Node.js or Browser environment with browserify:
import Gremlin from 'gremlin';

// Will open a WebSocket to ws://localhost:8182 by default
const client = Gremlin.createClient();
```
This is a shorthand for:
```javascript
const client = Gremlin.createClient(8182, 'localhost');
```

If you want to use Gremlin Server sessions, you can set the `session` argument as true in the `options` object:
```javascript
const client = Gremlin.createClient(8182, 'localhost', { session: true });
```

The `options` object currently allows you to set the following options:
* `session`: whether to use sessions or not (default: `false`)
* `language`: the script engine to use on the server, see your gremlin-server.yaml file (default: `"gremlin-groovy"`)
* `op` (advanced usage): The name of the "operation" to execute based on the available OpProcessor (default: `"eval"`)
* `processor` (advanced usage): The name of the OpProcessor to utilize (default: `""`)
* `accept` (advanced usage): mime type of returned responses, depending on the serializer (default: `"application/json"`)

### Sending scripts to Gremlin Server for execution

The client currently supports three modes:
* streaming results
* callback mode (with internal buffer)
* streaming protocol messages (low level API, for advanced usages)

#### Stream mode: client.stream(script, bindings, message)

Return a Node.js ReadableStream set in Object mode. The stream emits a distinct `data` event per query result returned by Gremlin Server.

Internally, a 1-level flatten is performed on all raw protocol messages returned. If you do not wish this behavior and prefer handling raw protocol messages with batched results, prefer using `client.messageStream()`.

The order in which results are returned is guaranteed, allowing you to effectively use `order` steps and the like in your Gremlin traversal.

The stream emits an `end` event when the client receives the last `statusCode: 299` message returned by Gremlin Server.

```javascript
const query = client.stream('g.V()');

// If playing with classic TinkerPop graph, will emit 6 data events
query.on('data', (result) => {
  // Handle first vertex
  console.log(result);
});

query.on('end', () => {
  console.log('All results fetched');
});
```

This allows you to effectively `.pipe()` the stream to any other Node.js WritableStream/TransformStream.

#### Callback mode: client.execute(script, bindings, message, callback)

Will execute the provided callback when all results are actually returned from the server.

```javascript
client.execute('g.V()', (err, results) => {
  if (!err) {
    console.log(results) // notice how results is *always* an array
  }
});
```

The client will internally concatenate all partial results returned over different messages (depending on the total number of results and the value of `resultIterationBatchSize` set in your .yaml file).

When the client receives the final `statusCode: 299` message, the callback will be executed.

#### Message stream mode: client.messageStream(script, bindings, message)

A lower level method that returns a `ReadableStream` which emits the raw protocol messages returned by Gremlin Server as distinct `data` events.

If you wish a higher-level stream of `results` rather than protocol messages, please use `client.stream()`.

Although a public method, this is recommended for advanced usages only.

```javascript
const client = Gremlin.createClient();

const stream = client.messageStream('g.V()');

// Will emit 3 events with a resultIterationBatchSize set to 2 and classic graph defined in gremlin-server.yaml
stream.on('data', (message) => {
  console.log(message.result); // Array of 2 vertices
});
```

### Adding bound parameters to your scripts

For better performance and security concerns (script injection), you must send bound parameters (`bindings`) with your scripts.

`client.execute()`, `client.stream()` and `client.messageStream()` share the same function signature: `(script, bindings, querySettings)`.

Notes/Gotchas:
- Any bindings set to `undefined` will be automatically escaped with `null` values (first-level only) in order to generate a valid JSON string sent to Gremlin Server.
- You cannot use bindings whose names collide with Gremlin reserved keywords (statically imported variables), such as `id`, `label` and `key` (see [https://github.com/jbmusso/gremlin-javascript/issues/23](issue#23)). This is a TinkerPop3 Gremlin Server limitation. Workarounds: `vid`, `eid`, `userId`, etc.

#### (String, Object) signature

```javascript
const client = Gremlin.createClient();

client.execute('g.v(vid)', { vid: 1 }, (err, results) => {
  console.log(results[0]) // notice how results is always an array
});
```

#### (Object) signature

Expects an `Object` as first argument with a `gremlin` property holding a `String` and a `bindings` property holding an `Object` of bound parameters.

```javascript
const client = Gremlin.createClient();
const query = {
  gremlin: 'g.V(vid)',
  bindings: {
    vid: 1
  }
}

client.execute(query, (err, results) => {
  console.log(results[0])
});
```

### Overriding low level settings on a per request basis

For advanced usage, for example if you wish to set the `op` or `processor` values for a given request only, you may wish to override the client level settings in the raw message sent to Gremlin Server:

```javascript
client.execute('g.v(1)', null, { args: { language: 'nashorn' }}, (err, results) => {
  // Handle result
});
```
Basically, all you have to do is provide an Object as third parameter to any `client.stream()`, `client.execute()` or `client.streamMessage()` methods.

Because we're not sending any bound parameters (`bindings`) in this example, notice how the second argument **must** be set to `null` so the low level message object is not mistaken with bound arguments.

If you wish to also send bound parameters while overriding the low level message, you can do the following:

```javascript
client.execute('g.v(vid)', { vid: 1 }, { args: { language: 'nashorn' }}, (err, results) => {
  // Handle err and results
});
```

Or in stream mode:
```javascript
client.stream('g.v(vid)', { vid: 1 }, { args: { language: 'nashorn' }})
  .pipe(/* ... */);
```

### Using Gremlin-JavaScript syntax with Nashorn

Please see [/docs/UsingNashorn.md](Using Nashorn).

## Running the Examples

This section assumes that loaded the default TinkerPop graph with `scripts: [scripts/generate-classic.groovy]` in your `gremlin-server.yaml` config file.

To run the command line example:
```
cd examples
node node-example
```

To run the browser example:
```
cd examples
node server
```
then open [http://localhost:3000/examples/gremlin.html](http://localhost:3000/examples/gremlin.html) for a demonstration on how a list of 6 vertices is being populated as the vertices are being streamed down from Gremlin Server.

## To do list

* better error handling
* emit more client events
* reconnect WebSocket if connection is lost
* add option for secure WebSocket
* more tests
* performance optimization

## License

MIT(LICENSE)
