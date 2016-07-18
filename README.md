# node-streams
[Streams](https://nodejs.org/api/stream.html) in Node.js are very powerful and useful constructs. They are also one of the more difficult concepts in Node to wrap your head around. Over the past few weeks I’ve become more and more familiar with them, and the best way I’ve come to understand them is by creating simple examples of them myself. This post will go through trivial implementations of a readable, writable, and transform stream, and demonstrate how to interact with each and how they interact with each other.

For source data I have created an array of 5 objects that each have fields `id` (zero-indexed identity), `name` (“object [id]”), and `value` (random number 0-4). The data and the code to generate the data is located in the `data-sources` folder.

## Readable Streams

Readable streams are sources of data that are waiting to be read. An analogy I’ve read several places is to think of a readable stream as a faucet. The stream has an underlying source of data (the water) that’s waiting to be read (waiting for valve to open and release water).

```
var data = require('../data-sources/sourceData.json'),
    Readable = require('stream').Readable,
    util = require('util');
 
var ReadStream = function() {
  Readable.call(this, {objectMode: true});
  this.data = data;
  this.curIndex = 0;
};
util.inherits(ReadStream, Readable);
```

To create our own readable stream, we can use Node’s built-in `util.inherits()` to subclass a readable stream. This copies the prototype methods from one constructor into our new object. Notice that we’re calling the readable object’s constructor with the option `objectMode: true` You’ll see this in all of our streams. Object Mode means that we’re operating on objects instead of string and buffers, which is convenient for our trivial example.

```
ReadStream.prototype._read = function() {
  if (this.curIndex === this.data.length)
    return this.push(null);
 
  var data = this.data[this.curIndex++];
  console.log('read: ' + JSON.stringify(data));
  this.push(data);
};
module.exports = ReadStream;
```

The `_read()` function is the heart of our readable stream. This determines what data is put into the read queue by calling push() and passing in the data to be delivered to the stream consumer. Going back to our faucet analogy, this function tells the readable stream (the faucet) what data (the water) is to be delivered when the stream is consumed (valve is opened).

Once our stream has reached the end of the underlying data (here, when we’ve reached the end of our “data” array), we push null to the read queue. Our stream will signal to the consumer that the end of the stream has been reached. Finally, to make this a module to use elsewhere, we export our ReadStream object.

## Consuming our ReadStream

Readable streams can be consumed directly by attaching either a `data` or `readable` event to them. The difference between the two events is that attaching `data` will put the stream into non-flowing mode or flowing mode.

### Non-flowing Mode

In non-flowing mode, the stream pushes some of it’s data to the read queue and then emits its `readable` event.

```
var ReadStream = require('./lib/readStream.js');
var stream = new ReadStream();
stream.on('readable', function() {
  while (null !== (record = stream.read())) {
    console.log('received: ' + JSON.stringify(record));
  }
});
stream.on('end', function() {
  console.log('done');
});
```

When we receive the `readable` event, we know our stream has data in it’s buffer that’s available to be read, and start consuming the data by calling `read()` on the stream.

```
read             : {"id":0,"name":"object 0","value":2}
read             : {"id":1,"name":"object 1","value":0}
received         : {"id":0,"name":"object 0","value":2}
read             : {"id":2,"name":"object 2","value":4}
received         : {"id":1,"name":"object 1","value":0}
read             : {"id":3,"name":"object 3","value":0}
received         : {"id":2,"name":"object 2","value":4}
read             : {"id":4,"name":"object 4","value":2}
received         : {"id":3,"name":"object 3","value":0}
received         : {"id":4,"name":"object 4","value":2}
done
```

We can get a better sense of what’s going on from following the output. The stream pushes two objects to the read queue and then fires the `readable` event. Once the consumer starts reading objects, it frees up room in the read queue for our stream to continue pushing objects to. When the stream is done, and the read queue is empty, the `end` event is emitted.

### Flowing Mode

In flowing mode data is read from the readable stream unprompted and immediately provided to the consumer. This means that the consumer doesn’t have to ask for the data, it’s just fed the stream’s data until the stream ends.

```
var ReadStream = require('./lib/readStream.js');
var stream = new ReadStream();
stream.on('data', function(record) {
  console.log('received: ' + JSON.stringify(record));
});
stream.on('end', function() {
  console.log('done');
});
```

```
read             : {"id":0,"name":"object 0","value":2}
received         : {"id":0,"name":"object 0","value":2}
read             : {"id":1,"name":"object 1","value":0}
received         : {"id":1,"name":"object 1","value":0}
read             : {"id":2,"name":"object 2","value":4}
received         : {"id":2,"name":"object 2","value":4}
read             : {"id":3,"name":"object 3","value":0}
received         : {"id":3,"name":"object 3","value":0}
read             : {"id":4,"name":"object 4","value":2}
received         : {"id":4,"name":"object 4","value":2}
done
```

You can see that the consumer has access to the `record` immediately via the callback function’s parameter (named record here).

### Flow Control

One advantage of using flowing mode is that you can pause and resume streams. This is useful when you’re consuming the stream in some time-consuming fashion (such as writing to a database). Calling the aptly-named `pause()` and `resume()` functions on the stream accomplishes this.

```
stream.on('data', function(record) {
  console.log('received: ' + JSON.stringify(record));
  console.log('pausing stream for 2 seconds');
  stream.pause();
  setTimeout(function() {
    console.log('resuming stream');
    stream.resume();
  },2000);
});
```

This example uses a `setTimeout()` function to simulate something that may take some time (in this case 2 seconds).

```
read             : {"id":0,"name":"object 0","value":2}
received         : {"id":0,"name":"object 0","value":2}
pausing stream for 2 seconds
read             : {"id":1,"name":"object 1","value":0}
read             : {"id":2,"name":"object 2","value":4}
read             : {"id":3,"name":"object 3","value":0}
read             : {"id":4,"name":"object 4","value":2}
resuming stream
received         : {"id":1,"name":"object 1","value":0}
pausing stream for 2 seconds
resuming stream
received         : {"id":2,"name":"object 2","value":4}
pausing stream for 2 seconds
resuming stream
received         : {"id":3,"name":"object 3","value":0}
pausing stream for 2 seconds
resuming stream
received         : {"id":4,"name":"object 4","value":2}
pausing stream for 2 seconds
done
resuming stream
```

Once the the `pause()` function is called our consumer does not receive another `data` event until the `resume()` event is called. Notice that the stream is still pushing data to the read queue, even while our stream is paused.

## Writable Streams

[Writable](http://nodejs.org/api/stream.html#stream_class_stream_writable) streams are destinations of data. Using our faucet analogy again we can think of writable streams as a drain.

```
var Writable = require('stream').Writable,
    util = require('util');
 
var WriteStream = function() {
  Writable.call(this, {objectMode: true});
};
util.inherits(WriteStream, Writable);
 
WriteStream.prototype._write = function(chunk, encoding, callback) {
  console.log('write: ' + JSON.stringify(chunk));
  callback();
};
module.exports = WriteStream;
```

Creating our writable stream class is similar to how we created our readable stream before, subclassing the writable stream and setting the object mode to true. The `_write()` function is where we tell the stream to direct the data. In this example, we’re taking the incoming data `chunk` and writing it to the console. Once you’re done with the particular piece of data you call `callback()`. This tells the source of data that the write stream is done with the current piece of data and is ready for the next.

## Piping Streams

We can test our new WriteStream by piping our ReadStream to it. The built-in function `pipe()` attaches a readable stream to a writable stream, passing the data from one to the other.

```
var ReadStream = require('./lib/readStream.js'),
    WriteStream = require('./lib/writeStream.js');
 
var rs = new ReadStream();
var ws = new WriteStream();
rs.pipe(ws);
```

```
read             : {"id":0,"name":"object 0","value":2}
read             : {"id":1,"name":"object 1","value":0}
write            : {"id":0,"name":"object 0","value":2}
read             : {"id":2,"name":"object 2","value":4}
write            : {"id":1,"name":"object 1","value":0}
read             : {"id":3,"name":"object 3","value":0}
write            : {"id":2,"name":"object 2","value":4}
read             : {"id":4,"name":"object 4","value":2}
write            : {"id":3,"name":"object 3","value":0}
write            : {"id":4,"name":"object 4","value":2}
```

You can see that our source data automatically flows to our output without us having to listen on any events. `pipe()` manages the flow of data between streams with no intervention.

Say our writable stream takes a bit of time to handle the incoming data (again, such as writing to a database). We can simulate this by modifying WriteStream’s `_write()` function and adding a delay using `setTimeout()`.

```
WriteStream.prototype._write = function(chunk, encoding, callback) {
  console.log('write: ' + JSON.stringify(chunk));
  console.log('waiting 2 seconds');
  setTimeout(function() {
    console.log('finished waiting');
    callback();
  },2000);
};
```

Piping ReadStream into our new WriteStream gives the following output.

```
read             : {"id":0,"name":"object 0","value":2}
read             : {"id":1,"name":"object 1","value":0}
write            : {"id":0,"name":"object 0","value":2}
waiting 2 seconds
read             : {"id":2,"name":"object 2","value":4}
read             : {"id":3,"name":"object 3","value":0}
read             : {"id":4,"name":"object 4","value":2}
finished waiting
write            : {"id":1,"name":"object 1","value":0}
waiting 2 seconds
finished waiting
write            : {"id":2,"name":"object 2","value":4}
waiting 2 seconds
finished waiting
write            : {"id":3,"name":"object 3","value":0}
waiting 2 seconds
finished waiting
write            : {"id":4,"name":"object 4","value":2}
waiting 2 seconds
finished waiting
```

As we can see, WriteStream doesn’t get any new data until after `callback()` is called. `pipe()` handles all flow control so that the destination isn’t overwhelmed by the readable stream.

## Transform Streams

[Transform](http://nodejs.org/api/stream.html#stream_class_stream_transform_1) streams are intermediaries of readable and writable streams. In fact, they are both readable and writable themselves. Data goes into the transform stream and can be returned modified or unchanged, or not even returned at all. To illustrate these points we’ll go through some examples.

```
var Transform = require('stream').Transform,
    util = require('util');
 
var TransformStream = function() {
  Transform.call(this, {objectMode: true});
};
util.inherits(TransformStream, Transform);
 
TransformStream.prototype._transform = function(chunk, encoding, callback) {
  console.log('transform before : ' + JSON.stringify(chunk));
 
  if (typeof chunk.originalValue === 'undefined')
    chunk.originalValue = chunk.value;
  chunk.value++;
 
  console.log('transform after : ' + JSON.stringify(chunk));
  this.push(chunk);
  callback();
};
 
module.exports = TransformStream;
```

We’ll create our transform stream like we have with our two previous streams, subclassing Transform and setting `objectMode: true`. The method that determines what the stream does is `_transform()`. Data comes in as the `chunk` parameter (like our writable stream) and is outputted using `push()` (like our readable stream). We signal that we’re done with `chunk` by calling `callback()` (like our writable stream).

This example transform stream copies the `value` of our object to a new field called `originalValue` and then increments `value`. To see it working, we can insert it in our pipe chain from earlier.

```
var ReadStream = require('./lib/readStream.js'),
    WriteStream = require('./lib/writeStream.js'),
    TransformStream = require('./lib/transformStream.js');
 
var rs = new ReadStream();
var ws = new WriteStream();
var ts = new TransformStream();
 
rs.pipe(ts).pipe(ws);
```

One important thing I forgot to mention about `pipe()` is that it returns the destination stream. When we pipe our readable stream `rs` into our transform stream `ts` by doing `rs.pipe(ts)` it returns the transform stream, which is a readable and writable stream. We can then pipe it into `ws`, creating a full pipe chain.

```
read             : {"id":0,"name":"object 0","value":2}
read             : {"id":1,"name":"object 1","value":0}
transform before : {"id":0,"name":"object 0","value":2}
transform after  : {"id":0,"name":"object 0","value":3,"originalValue":2}
read             : {"id":2,"name":"object 2","value":4}
transform before : {"id":1,"name":"object 1","value":0}
transform after  : {"id":1,"name":"object 1","value":1,"originalValue":0}
read             : {"id":3,"name":"object 3","value":0}
transform before : {"id":2,"name":"object 2","value":4}
transform after  : {"id":2,"name":"object 2","value":5,"originalValue":4}
read             : {"id":4,"name":"object 4","value":2}
transform before : {"id":3,"name":"object 3","value":0}
transform after  : {"id":3,"name":"object 3","value":1,"originalValue":0}
transform before : {"id":4,"name":"object 4","value":2}
transform after  : {"id":4,"name":"object 4","value":3,"originalValue":2}
write            : {"id":0,"name":"object 0","value":3,"originalValue":2}
write            : {"id":1,"name":"object 1","value":1,"originalValue":0}
write            : {"id":2,"name":"object 2","value":5,"originalValue":4}
write            : {"id":3,"name":"object 3","value":1,"originalValue":0}
write            : {"id":4,"name":"object 4","value":3,"originalValue":2}
```

One nice thing about transform streams is that they make no guarantee that the output will match the input in size or frequency, which lends to some interesting uses. Say you wanted to implement a filter that blocks any object with a `value` of 0.

```
TransformStream.prototype._transform = function(chunk, encoding, callback) {
  if (chunk.value !== 0) this.push(chunk);
  callback();
};
```

```
read             : {"id":0,"name":"object 0","value":2}
read             : {"id":1,"name":"object 1","value":0}
read             : {"id":2,"name":"object 2","value":4}
read             : {"id":3,"name":"object 3","value":0}
read             : {"id":4,"name":"object 4","value":2}
write            : {"id":0,"name":"object 0","value":2}
write            : {"id":2,"name":"object 2","value":4}
write            : {"id":4,"name":"object 4","value":2}
```

Transform streams end up being the ones I write the most because it allows me to hook into established data sources and destinations to perform my desired logic.

## More knowledge

I hope this post helps demystify Node.js streams. This isn’t meant to be a comprehensive guide by any means, but a primer to get you started working with and writing your own streams. Other than the API, I’d recommend skimming the [stream-handbook](https://github.com/substack/stream-handbook) which contains some additional examples (mostly non-object mode stuff) and links to useful libraries. Finally, leave comments if there’s an important basic topic you think I’ve left out.

## Source Code

The source code presents a lib folder that includes the following files:
 * ##lib/readStream.js##  | Readable stream of objects. Source data is found in the `data-sources` folder (created by `create-source.js`).
 * ##lib/writeStream.js## | Writable stream that outputs incoming objects to the console.
 * ##lib/writeStreamDelay.js## | Writable stream that outputs incoming objects to the console then delays 2 seconds.
 * ##lib/transformStream.js## | Transform stream that stores the original `value` in `originalValue` and increments the `value` field.
 * ##lib/transformFilterStream.js## | Transform stream that filters out objects whose `value` is 0.

There are also some examples to show how to use these concepts:

 * ##read.js## | Demonstrates aforementioned `ReadStream` in [flowing and non-flowing modes](http://nodejs.org/api/stream.html#stream_class_stream_readable).
 * ##read-flowControl.js## | Demonstrates `pause()` and `resume()` in flowing mode streams.
 * ##write.js## | Pipes `ReadStream` to `WriteStream`.
 * ##transform.js## | Pipes `ReadStream` into `TransformStream` and then into `WriteStream`.
 * ##transform-filterjs## | Pipes `ReadStream` into `TransformFilterStream` and then into `WriteStream`.

## References
[html-streams](https://github.com/substack/stream-handbook/tree/master/example/html-streams)

