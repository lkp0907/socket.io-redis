# socket.io-redis

[![Build Status](https://travis-ci.org/socketio/socket.io-redis.svg?branch=master)](https://travis-ci.org/socketio/socket.io-redis)
[![NPM version](https://badge.fury.io/js/socket.io-redis.svg)](http://badge.fury.io/js/socket.io-redis)

## How to use

```js
var io = require('socket.io')(3000);
var redis = require('socket.io-redis');
io.adapter(redis({ host: 'localhost', port: 6379 }));
```

By running socket.io with the `socket.io-redis` adapter you can run
multiple socket.io instances in different processes or servers that can
all broadcast and emit events to and from each other.

If you need to emit events to socket.io instances from a non-socket.io
process, you should use [socket.io-emitter](https://github.com/socketio/socket.io-emitter).

## API

### adapter(uri[, opts])

`uri` is a string like `localhost:6379` where your redis server
is located. For a list of options see below.

### adapter(opts)

The following options are allowed:

- `key`: the name of the key to pub/sub events on as prefix (`socket.io`)
- `host`: host to connect to redis on (`localhost`)
- `port`: port to connect to redis on (`6379`)
- `subEvent`: optional, the redis client event name to subscribe to (`messageBuffer`)
- `pubClient`: optional, the redis client to publish events on
- `subClient`: optional, the redis client to subscribe to events on
- `requestsTimeout`: optional, after this timeout the adapter will stop waiting from responses to request (`1000ms`)
- `withChannelMultiplexing`: optional, whether channel multiplexing is enabled (a new subscription will be trigggered for each room) (`true`)

If you decide to supply `pubClient` and `subClient`, make sure you use
[node_redis](https://github.com/mranney/node_redis) as a client or one
with an equivalent API.

### RedisAdapter

The redis adapter instances expose the following properties
that a regular `Adapter` does not

- `uid`
- `prefix`
- `pubClient`
- `subClient`
- `requestsTimeout`

### RedisAdapter#clients(rooms:Array, fn:Function)

Returns the list of client IDs connected to `rooms` across all nodes. See [Namespace#clients(fn:Function)](https://github.com/socketio/socket.io#namespaceclientsfnfunction)

```js
io.of('/').adapter.clients(function (err, clients) {
  console.log(clients); // an array containing all connected socket ids
});

io.of('/').adapter.clients(['room1', 'room2'], function (err, clients) {
  console.log(clients); // an array containing socket ids in 'room1' and/or 'room2'
});

// you can also use

io.in('room3').clients(function (err, clients) {
  console.log(clients); // an array containing socket ids in 'room3'
});
```

### RedisAdapter#clientRooms(id:String, fn:Function)

Returns the list of rooms the client with the given ID has joined (even on another node).

```js
io.of('/').adapter.clientRooms('<my-id>', function (err, rooms) {
  if (err) { /* unknown id */ }
  console.log(rooms); // an array containing every room a given id has joined.
});
```

### RedisAdapter#allRooms(fn:Function)

Returns the list of all rooms.

```js
io.of('/').adapter.allRooms(function (err, rooms) {
  console.log(rooms); // an array containing all rooms (accross every node)
});
```

### RedisAdapter#remoteJoin(id:String, room:String, fn:Function)

Makes the socket with the given id join the room. The callback will be called once the socket has joined the room, or with an `err` argument if the socket was not found.

```js
io.of('/').adapter.remoteJoin('<my-id>', 'room1', function (err) {
  if (err) { /* unknown id */ }
  // success
});
```

### RedisAdapter#remoteLeave(id:String, room:String, fn:Function)

Makes the socket with the given id leave the room. The callback will be called once the socket has left the room, or with an `err` argument if the socket was not found.

```js
io.of('/').adapter.remoteLeave('<my-id>', 'room1', function (err) {
  if (err) { /* unknown id */ }
  // success
});
```

### RedisAdapter#remoteDisconnect(id:String, close:Boolean, fn:Function)

Makes the socket with the given id to get disconnected. If `close` is set to true, it also closes the underlying socket. The callback will be called once the socket was disconnected, or with an `err` argument if the socket was not found.

```js
io.of('/').adapter.remoteDisconnect('<my-id>', true, function (err) {
  if (err) { /* unknown id */ }
  // success
});
```

### RedisAdapter#customRequest(data:Object, fn:Function)

Sends a request to every nodes, that will respond through the `customHook` method.

```js
// on every node
io.of('/').adapter.customHook = function (data, cb) {
  cb('hello ' + data);
}

// then
io.of('/').adapter.customRequest('john', function(err, replies){
  console.log(replies); // an array ['hello john', ...] with one element per node
});
```

### RedisAdapter#remoteEmit(id:String, _name:String,_data:Object, fn:Function)

Make Emit With specific Sockets.
You Can Send Message To Socket. ( Even if, another socket-server)

```js
io.of('/').adpater.remoteEmit('<socket-id>','emit-name','emit-data',function(err){

});
```

## Client error handling

Access the `pubClient` and `subClient` properties of the
Redis Adapter instance to subscribe to its `error` event:

```js
var redis = require('socket.io-redis');
var adapter = redis('localhost:6379');
adapter.pubClient.on('error', function(){});
adapter.subClient.on('error', function(){});
```

The errors emitted from `pubClient` and `subClient` will
also be forwarded to the adapter instance:

```js
var io = require('socket.io')(3000);
var redis = require('socket.io-redis');
io.adapter(redis({ host: 'localhost', port: 6379 }));
io.of('/').adapter.on('error', function(){});
```

## Custom client (eg: with authentication)

If you need to create a redisAdapter to a redis instance
that has a password, use pub/sub options instead of passing
a connection string.

```js
var redis = require('redis').createClient;
var adapter = require('socket.io-redis');
var pub = redis(port, host, { auth_pass: "pwd" });
var sub = redis(port, host, { auth_pass: "pwd" });
io.adapter(adapter({ pubClient: pub, subClient: sub }));
```

## Protocol

The `socket.io-redis` adapter broadcasts and receives messages on particularly named Redis channels. For global broadcasts the channel name is:
```
prefix + '#' + namespace + '#'
```

In broadcasting to a single room the channel name is:
```
prefix + '#' + namespace + '#' + room + '#'
```


- `prefix`: The base channel name. Default value is `socket.io`. Changed by setting `opts.key` in `adapter(opts)` constructor
- `namespace`: See https://github.com/socketio/socket.io#namespace.
- `room` : Used if targeting a specific room.

A number of other libraries adopt this protocol including:

- [socket.io-emitter](https://github.com/socketio/socket.io-emitter)
- [socket.io-python-emitter](https://github.com/GameXG/socket.io-python-emitter)


## License

MIT
