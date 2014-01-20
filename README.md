# Overview #

** 本文尚未达到 0.1 版本状态，我还需要时间来把内容先填满 **

[![cc-by-3.0](http://i.creativecommons.org/l/by/3.0/80x15.png)](http://creativecommons.org/licenses/by/3.0/)

软件开发的生产关系，因为开源社区和 Node.JS 而有了深刻的变化。其中，`npm` 和 `stream` 居功至伟。

学习 `stream` 的基本原理不难，[原始文档](http://nodejs.org/api/stream.html) 就讲得很清楚。但是初学 `stream` 的时候，却总是把握不好该怎么用，其根本原因就是我们往往习惯了 `request/response` 的思考方式，而难以从管道和流体的方式来思考问题（也许水暖工有天生的优势:)）。

培养思考方式的最好方法，就是努力去理解“前辈”们都是怎么用这种方式解决问题的，所以在这里我列出一系列既有的 `stream` 相关的 `Node Modules`，辅之以简要的说明和相互关系的描述，以帮助我自己和大家掌握 `stream` 的思考方式。

[substack](https://github.com/substack) 写了一本 [stream-handbook](https://github.com/substack/stream-handbook)，值得配合 [原始文档](http://nodejs.org/api/stream.html) 阅读的。同时该文也试图对社区内一些有用的 `stream modules` 进行归类，不过由于作者精力有限，并未做出足够的说明。

本项目根据我自己的学习过程，对这些 `stream modules` 做一个概要性的介绍。看看大家都是怎么用 `stream` 的。

# Node.JS 内建的 Streams #

## [process.stdin](http://nodejs.org/docs/latest/api/process.html#process_process_stdin) ##

系统标准输入的 readable stream。缺省处于 `pause` 状态。当你 `pipe` 它的时候，竟会在 `next tick` 进入到 `resume` 状态。

## [process.stdout](http://nodejs.org/docs/latest/api/process.html#process_process_stdout) ##

代表系统标准输出 writable stream。

## [process.stderr](Http://nodejs.org/Docs/Latest/Api/process.html#Process_Process_Stderr) ##

代表系统标准错误输出的 writable stream。

# Stream Testing #

如果编写你自己的 `stream`，那么测试是必不可少的。下面两个 `Modules` 就是帮助你测试 `stream` 的。

## [stream-spec](https://github.com/dominictarr/stream-spec#duplex) ##

测试你编写的 stream。

``` js
var spec = require('stream-spec')
var tester = require('stream-tester')

spec(stream)
  .through({strict: true})
  .validateOnExit()

tester.createRandomStream(function () {
    return 'line ' + Math.random() + '\n'
  }, 1000)
  .pipe(stream)
  .pipe(tester.createPauseStream())
```

上例中，假设需要测试的 stream 是 `stream`，用到了 `stream-tester` 来生成了两个测试 stream，一个是 readable stream 生成了 1000 个 chunks 的数据，另外一个是 writeable stream，会随机 pause。

## [stream-test](https://github.com/dominictarr/stream-tester) ##

用来生成测试用的 stream。

``` js
createRandomStream (generator, max)
```

建立一个产生随机大小 chunk 的 stream。

``` js
createIncStream (max)
```

建立自增数字的 stream。

``` js
createPauseStream (prob, delay)
```

建立一个随机 pause 的 stream。

# Base Class #
## [duplex](https://github.com/dominictarr/duplex) ##

一个双工流的简单基类，自动处理暂停和缓存。

``` js
var duplex = require('duplex')

var d = duplex()
  .on('_data', function (data) {
    d.sendData(data)
  })
  .on('_end', function () {
    d.sendEnd()
  })
```

## [from](https://github.com/dominictarr/from) ##

最简单地建立一个 `readable` stream。

### from(function getChunk(count, next)) ###

``` js
var from = require('from')

var stream =
  from(function getChunk(count, next) {
    //do some sort of data
    this.emit('data', whatever)

    if (itsOver)
      this.emit('end')

    //ready to handle the next chunk
    next()
    //or, if it's sync:
    return true
  })
```

### from(array) ###

将 array 的内容逐一“发射”给 stream。

## [through](http://github.com/dominictarr/through) ##

这可能是我见到的被引用最多的 `stream module`。只需提供可选的 `write` 和 `end` 方法，即可实现一个 `readable` 和 `writable` stream（也称为 `duplex` stream）。`through` 也自动帮你处理 `pause` 和 `resume`。

``` js
var through = require('through')

through(function write(data) {
    this.queue(data) //data *must* not be null
  },
  function end () { //optional
    this.queue(null)
  })
}
```

## [duplexer](https://github.com/Raynos/duplexer) ##

将既有的 write stream 和 read stream 合成一个 `duplex stream` (`readable` and `writable`) 。

### duplex (writeStream, readStream) ###

``` js
var grep = cp.exec('grep Stream')
duplex(grep.stdin, grep.stdout)
```

还有一个 [duplexer2](https://github.com/deoxxa/duplexer2) 用 `stream2` 的 API 实现了 `duplexer` 的功能。

## [streamin](https://github.com/fent/node-streamin) ##


## [shoe](https://github.com/substack/shoe) ##

将 sockjs 封装为 stream 方式，可以用于 node.js 和浏览器。

``` js
var shoe = require('shoe');
var through = require('through');

var result = document.getElementById('result');

var stream = shoe('/invert');
stream.pipe(through(function (msg) {
    result.appendChild(document.createTextNode(msg));
    this.queue(String(Number(msg)^1));
})).pipe(stream);
```

## [map-stream](http://github.com/dominictarr/map-stream) ##

在 stream 上实现 map/reduce 的 map。

### map (asyncFunction[, options]) ###

``` javascript
var map = require('map-stream')

map(function (data, done) {
  //transform data
  // ...
  done(null, data)
})
```

# Network #

## [websocket-stream](https://github.com/maxogden/websocket-stream) ##

基于 [ws](https://github.com/einaros/ws) 实现的 websocket 的 stream 接口。

## [shoe]() ##

# 综合工具 #

## [event-stream](https://github.com/dominictarr/event-stream) ##

concat, merge, map/reduce, pipeline... 等 stream 工具大全集。不过建议各位直接用每个小工具本身，而不是用合集。

其中 pipeline 可以将多个piped stream 合为一个 stream。

reduce 依赖 [stream-reduce](https://github.com/parshap/node-stream-reduce)。

# State Sync. #

[scuttlebutt](https://github.com/dominictarr/scuttlebutt)

[delta-stream](https://github.com/Raynos/delta-stream)

[append-only](http://github.com/Raynos/append-only)

[crdt](https://github.com/dominictarr/crdt)

# 多路复用 #

`stream` 的多路复用值得开一个专题。所谓多路复用是指，在一个既有的 `stream`  上同时传输多个 `stream`。它们之间互不干扰。

一个很常见的场景：

你有多个 `scuttlebutt` 对象, 它们需要在 client/server 之间进行同步。为了避免你为每一个 `scuttlebutt` 对象都创建一个独立的 client/server 连接（例如：websocket），你需要让每一个 `scuttlebutt` 对象的同步 `stream` 利用既有的 `websocket-stream` 来传输。

更复杂一些的场景包括，你在既有 `websocket-stream` 即复用了 `scuttlebutt stream`，也复用了 `dnode` 的 `rpc stream`。

多路复用，让 `stream` 从 “玩具” 或者 “命令行” 工具层面上升到了真正的系统架构层面。

## [mux-demux](http://github.com/dominictarr/mux-demux) ##

## [nspoint](Https://github.com/AndreasMadsen/Nspoint) ##

将一个 `duplex stream` 通过 `namespace` 切分为多个 `duplex stream`。这是另外一种形式的多路复用：

``` js
var nspoint = require('nspoint');

// you have a some duplex `transport` socket there takes and outputs objects
// wrap it in each end with `nspoint` to split it via namespaces

// e.q. server side
var A = nspoint();
transport.pipe(A).pipe(transport);

var aFoo = A.namespace('foo');
var aBar = A.namespace('bar');

aFoo.write('foo message');
aBar.once('data', function (msg) {
  // gets gets {object: 'bar'}
  console.log(msg);
});

// e.q. client side
var B = nspoint(transport);
transport.pipe(B).pipe(transport);

var bFoo = B.namespace('foo');
var bBar = B.namespace('bar');

bFoo.once('data', function (msg) {
  // gets 'foo message'
  b2.write({object: 'bar'});
});
```

上例中，`A` 是一个 `server` 端的 `duplex stream`，通过 `A.namespace('foo')` 产生了一个新的 `aFoo stream` 和 `aBar stream`；对等地，`client` 端创建了 `B stream` ，随后也通过和 `server` 端相同的名字切分出了 `bFoo` 和 `bBar` 两个 `streams`，分别用于和 `aFoo` 和 `aBar` 通信。`server` 端 复用了 `A stream`，`client` 端复用了 'B stream'。

# RPC #

## [dnode](https://github.com/substack/dnode) ##

`dnode` 首先是一个 rpc 系统，例如:

### Server ###

``` js
var dnode = require('dnode');
var server = dnode({
    transform : function (s, cb) {
        cb(s.replace(/[aeiou]{2,}/, 'oo').toUpperCase())
    }
});
server.listen(5004);
```

### Client ###

``` js
var dnode = require('dnode');

var d = dnode.connect(5004);
d.on('remote', function (remote) {
    remote.transform('beep', function (s) {
        console.log('beep => ' + s);
        d.end();
    });
});
```

### Output ###

``` js
$ node server.js &
[1] 27574
$ node client.js
beep => BOOP
```

有趣的是 dnode 可以和既有的 stream 系统工作，例如：

### Server ###

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(s.replace(/[aeiou]{2,}/, 'oo').toUpperCase())
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

### Client ###

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (s) {
        console.log('beep => ' + s);
        d.end();
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

这样通过 `dnode` 编写的 rpc server，就可以运行在各个协议构造的 `stream` 之上了，无论是 `tcp`, `http`, `websocket-stream` 还是 `shoe`（可以运行在浏览器中）。

## [stream-router](https://github.com/Raynos/stream-router) ##

利用 [mux-demux](http://github.com/dominictarr/mux-demux) 上的 meta 实现路由。

### Server ###

``` js
var StreamRouter = require("stream-router")
    , streamRouter = StreamRouter()
    , MuxDemux = require("mux-demux")
    , net = require("net")

streamRouter.addRoute("/foo/:name", handleFoo)

net.createServer(handleTcp).listen(8642)

function handleTcp(con) {
    var mdm = MuxDemux({
        error: false
    })

    mdm.on("connection", streamRouter)

    con.pipe(mdm).pipe(con)
}

function handleFoo(stream, params) {
    stream.write(params.name)

    stream.on("data", console.log.bind(console, "server"))
}
```

### Client ###

``` js
var MuxDemux = require("mux-demux")
    , net = require("net")
    , mdm = MuxDemux({
        error: false
    })
    , con = net.connect(8642)

con.pipe(mdm).pipe(con)

var foo = mdm.createStream("/foo/bar")

foo.on("data", console.log.bind(console, "client"))

foo.write("bar")
```