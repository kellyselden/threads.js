# threads.js
[![Build Status](https://travis-ci.org/andywer/threads.js.svg?branch=master)](https://travis-ci.org/andywer/threads.js)
[![Test Coverage](https://codeclimate.com/github/andywer/threads.js/badges/coverage.svg)](https://codeclimate.com/github/andywer/threads.js/coverage)
[![Code Climate](https://codeclimate.com/github/andywer/threads.js/badges/gpa.svg)](https://codeclimate.com/github/andywer/threads.js)
[![NPM Version](https://img.shields.io/npm/v/threads.svg)](https://www.npmjs.com/package/threads)

Javascript thread library. Uses web workers when run in browsers and child processes
when run by node.js. Also supports browsers which do not support web workers.

- For client and server use
- Use different APIs (web worker, node child_process) transparently
- Thread pools
- Built-in error handling
- Well tested
- ES6 and backwards-compatible


## Basic usage

Spawn threads to do the time-consuming work and let the parent thread focus on
daily business!

```javascript
const spawn = require('threads').spawn;

const thread = spawn(function(input, done) {
  // Everything we do here will be run in parallel in another execution context.
  // Remember that this function will be executed in the thread's context,
  // so you cannot reference any value of the surrounding code.
  done({ string : input.string, integer : parseInt(input.string) });
});

thread
  .send({ string : '123' })
  // The handlers come here: (none of them is mandatory)
  .on('message', function(response) {
    console.log('123 * 2 = ', response.integer * 2);
    thread.kill();
  })
  .on('error', function(error) {
    console.error('Worker errored:', error);
  })
  .on('exit', function() {
    console.log('Worker has been terminated.');
  });
```


## Installation

### NPM (Node.js, Browserify, Webpack)

```bash
npm install --save threads
```

### Bower

```bash
bower install --save threads
```

### Script tag

```html
<script src="https://cdn.rawgit.com/andywer/threads.js/__VERSION__/dist/threads.browser.min.js"></script>
```

(where `__VERSION__` is the library's version to use, like `0.5.3`)


## How To

### Thread code in separate files

You don't have to write the thread's code inline. The file is expected to be a
commonjs module (so something that uses `module.exports = ...`), for node and
browser.

```javascript
const threads = require('threads');
const config  = threads.config;
const spawn   = threads.spawn;

// Set base paths to thread scripts
config.set({
  basepath : {
    browser : 'http://myserver.local/thread-scripts',
    node    : __dirname + '/../thread-scripts'
  }
});

const thread = spawn('worker.js');

thread
  .send({ do : 'Something awesome!' })
  .on('message', function(message) {
    console.log('worker.js replied:', message);
  });
```

worker.js:
```javascript
// Use CommonJS syntax (module.exports). Works in browser, too!
// Only limitation: You won't have require() when run in the browser.
module.exports = function(input, done) {
  done('Awesome thread script may run in browser and node.js!');
};
```


### Thread Pool

You can also create a thread pool that spawns a fixed no. of workers. Pass jobs
to the thread pool which it will queue and pass to the next idle worker.
You can also pass the number threads to be spawned. Defaults to the number of
CPU cores.

```javascript
const Pool = require('threads').Pool;

const pool = new Pool();
// Alternatively: new Pool(<number of threads to spawn>)

// Run a script
const jobA = pool
  .run('/path/to/worker')
  .send({ do : 'something' });

// Run the same script, but with a different parameter
const jobB = pool
  .send({ do : 'something else' });

// Run inline code
const jobC = pool.run(
  function(input, done) {
    const hash = md5(input);
    done(hash, input);
  }, {
    // dependencies; resolved using node's require() or the web workers importScript()
    md5 : 'js-md5'
  }
).send('Hash this string!');

jobC
  .on('done', function(hash, input) {
    console.log(`Job C hashed: md5("${input}") = "${hash}"`);
  });

pool
  .on('done', function(job, message) {
    console.log('Job done:', job);
  })
  .on('error', function(job, error) {
    console.error('Job errored:', job);
  })
  .on('finished', function() {
    console.log('Everything done, shutting down the thread pool.');
    pool.killAll();
  });
```

### Streaming

You can also spawn a thread for streaming purposes. The following example shows
a very simple use case where you keep feeding numbers to the background task
and it will return the minimum and maximum of all values you ever passed.

```javascript
const threads = require('threads');
const spawn   = threads.spawn;
const thread  = spawn(function() {});

thread
  .run(function minmax(int, done) {
    if (typeof this.min === 'undefined') {
      this.min = int;
      this.max = int;
    } else {
      this.min = Math.min(this.min, int);
      this.max = Math.max(this.max, int);
    }
    done({ min : this.min, max : this.max }});
  })
  .send(2)
  .send(3)
  .send(4)
  .send(1)
  .send(5)
  .on('message', function(minmax) {
    console.log('min:', minmax.min, ', max:', minmax.max);
  })
  .on('done', function() {
    thread.kill();
  });
```

### Retraining

As it turns out, `thread.run()` is no one-way road.

```javascript
thread
  .run(function doThis(input, done) {
    done('My first job!');
  })
  .send()
  .run(function doThat(input, done) {
    done('Old job was boring. Trying something new!');
  })
  .send();
```

### Promises

Instead of using callbacks, you can also turn thread messages and pool jobs into
promises.

```javascript
spawn(myThreadFile)
  .send({ important : 'data' })
  .promise()
  .then(function success(message) {}, function error(error) {});
```

```javascript
pool.run(fancyThreadCode);

Promise.all([
  pool.send({ data : 1 }).promise(),
  pool.send({ data : 2 }).promise()
]).then(function allResolved() {
  console.log('Everything done! It\'s closing time...');
});
```

### Transferable objects

You can also use transferable objects to improve performance when passing large
buffers (in browser). Add script files you want to run using importScripts()
(if in browser) as second parameter to thread.run().
See [Transferable Objects: Lightning Fast!](http://updates.html5rocks.com/2011/12/Transferable-Objects-Lightning-Fast).

Both features will be ignored by node.js version for now.

```javascript
const threads = require('threads');
const spawn   = threads.spawn;
const thread  = spawn(function() {});

const largeArrayBuffer = new Uint8Array(1024 * 1024 * 32); // 32MB
const jobData = { label : 'huge thing', data: largeArrayBuffer.buffer };

thread
  .run(function(input, done) {
    // do something cool with input.label, input.data
    // call done.transfer() if you want to use transferables in the thread's response
    // (the node.js code simply ignores the transferables)
    done.transfer({ some : { response : input.buffer } }, [input.data.buffer]);
  }, [
    // this file will be run in the thread using importScripts() if in browser
    // the node.js code will ignore this second parameter
    '/dependencies-bundle.js'
  ])
  // pass the buffers to transfer into thread context as 2nd parameter to send()
  .send(jobData, [ largeArrayBuffer.buffer ]);
```

### Progress update

The thread can also notify the main thread about its current progress.

```javascript
const threads = require('threads');
const spawn   = threads.spawn;
const thread  = spawn(function() {});

thread
  .run(function(input, done, progress) {
    setTimeout(done, 1000);
    setTimeout(function() { progress(25); }, 250);
    setTimeout(function() { progress(50); }, 500);
    setTimeout(function() { progress(75); }, 750);
  })
  .send()
  .on('progress', function(progress) {
    console.log(`Progress: ${progress}%`);
  })
  .on('done', function() {
    console.log(`Done.`);
    thread.kill();
  });
```

Output:

```
Progress: 25%
Progress: 50%
Progress: 75%
Done.
```

### Web worker fallback

You can provide a fallback if the user's browser does not support web workers.
See [webworker-fallback](https://github.com/andywer/webworker-fallback). This will not have any effect if used by node.js code.

### Use external dependencies

Not yet completely implemented.

To do:
- gulp task to bundle dependencies using browserify and expose all of them -> dependency bundle
- dependency bundle can be imported by importScripts()
- code can just call `var myDependency = require('my-dependency');`, no matter if browser or node.js


## Configuration

```javascript
const config = require('threads').config;

// These configuration properties are all optional
config.set({
  basepath : {
    node : 'path/to/my/worker/scripts/directory',
    web  : 'path-or-url/to/my/worker/scripts/directory'
  },
  fallback : {
    slaveScriptUrl : 'path-or-url/to/dist/slave.js'    // used for IE where you cannot pass code to a worker using a data URI
  }
});
```

## Changelog

### 0.7.0

Fixes a critical issue that prevented thread pools from running all jobs.
Also brings some major performance improvements for browser (web worker) - based setups.

### 0.6.1

Added alias for threads: Event `done` as alias for `message`. Updated README example code.
Credit goes to https://github.com/andrakis

### 0.6.0

Fixes promise and async issues. `Job.clone()` has been dropped.
Credit goes to https://github.com/maysale01


## License

This library is published under the MIT license. See [LICENSE](./LICENSE) for details.


__Have fun and build something awesome!__