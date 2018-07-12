## 由表及里
HTTP服务器用于响应来自客户端的请求，当客户端请求数逐渐增大时服务端的处理机制有多种，如tomcat的多线程、nginx的事件循环等。而对于node而言，由于其也采用事件循环和异步I/O机制，因此在高I/O并发的场景下性能非常好，但是由于单个node程序仅仅利用单核cpu，因此为了更好利用系统资源就需要fork多个node进程执行HTTP服务器逻辑，所以node内建模块提供了**child_process和cluster**模块。利用child_process模块，我们可以执行shell命令，可以fork子进程执行代码，也可以直接执行二进制文件；利用cluster模块，使用node封装好的API、IPC通道和调度机可以非常简单的创建包括`一个master进程下HTTP代理服务器 + 多个worker进程多个HTTP应用服务器`的架构，并提供两种调度子进程算法。本文主要针对cluster模块讲述node是如何实现简介高效的服务集群创建和调度的。那么就从代码进入本文的主题：

**code1**
```
const cluster = require('cluster');
const http = require('http');

if (cluster.isMaster) {

  let numReqs = 0;
  setInterval(() => {
    console.log(`numReqs = ${numReqs}`);
  }, 1000);

  function messageHandler(msg) {
    if (msg.cmd && msg.cmd === 'notifyRequest') {
      numReqs += 1;
    }
  }

  const numCPUs = require('os').cpus().length;
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  for (const id in cluster.workers) {
    cluster.workers[id].on('message', messageHandler);
  }

} else {

  // Worker processes have a http server.
  http.Server((req, res) => {
    res.writeHead(200);
    res.end('hello world\n');

    process.send({ cmd: 'notifyRequest' });
  }).listen(8000);
}
```

主进程创建多个子进程，同时接受子进程传来的消息，循环输出处理请求的数量；

子进程创建http服务器，侦听8000端口并返回响应。

泛泛的大道理谁都了解，可是这套代码如何运行在主进程和子进程中呢？父进程如何向子进程传递客户端的请求？多个子进程共同侦听8000端口，会不会造成端口reuse error？每个服务器进程最大可有效支持多少并发量？主进程下的代理服务器如何调度请求？ 这些问题，如果不深入进去便永远只停留在写应用代码的层面，而且不了解cluster集群创建的多进程与使用child_process创建的进程集群的区别，也写不出符合业务的最优代码，因此，深入cluster还是有必要的。

## cluster与net
cluster模块与net模块息息相关，而net模块又和底层socket有联系，至于socket则涉及到了系统内核，这样便由表及里的了解了node对底层的一些优化配置，这是我们的思路。介绍前，笔者仔细研读了node的js层模块实现，在基于自身理解的基础上诠释上节代码的实现流程，力图做到清晰、易懂，如果有某些纰漏也欢迎读者指出，只有在互相交流中才能收获更多。

### 一套代码，多次执行
很多人对**code1**代码如何在主进程和子进程执行感到疑惑，怎样通过*cluster.isMaster*判断语句内的代码是在主进程执行，而其他代码在子进程执行呢？

其实只要你深入到了node源码层面，这个问题很容易作答。cluster模块的代码只有一句：
```
module.exports = ('NODE_UNIQUE_ID' in process.env) ?
                  require('internal/cluster/child') :
                  require('internal/cluster/master');
```
只需要判断当前进程有没有环境变量“NODE_UNIQUE_ID”就可知道当前进程是否是主进程；而变量“NODE_UNIQUE_ID”则是在主进程fork子进程时传递进去的参数，因此采用cluster.fork创建的子进程是一定包含“NODE_UNIQUE_ID”的。

**这里需要指出的是，必须通过cluster.fork创建的子进程才有NODE_UNIQUE_ID变量，如果通过child_process.fork的子进程，在不传递环境变量的情况下是没有NODE_UNIQUE_ID的。因此，当你在child_process.fork的子进程中执行`cluster.isMaster`判断时，返回 true。**

### 主进程与服务器
**code1**中，并没有在cluster.isMaster的条件语句中创建服务器，也没有提供服务器相关的路径、端口和fd，那么主进程中是否存在TCP服务器，有的话到底是什么时候怎么创建的？

相信大家在学习nodejs时阅读的各种书籍都介绍过在集群模式下，主进程的服务器会接受到请求然后发送给子进程，那么问题就来到主进程的服务器到底是如何创建呢？主进程服务器的创建离不开与子进程的交互，毕竟与创建服务器相关的信息全在子进程的代码中。

当子进程执行
```
http.Server((req, res) => {
    res.writeHead(200);
    res.end('hello world\n');

    process.send({ cmd: 'notifyRequest' });
  }).listen(8000);
```
时，http模块会调用net模块(确切的说，http.Server继承net.Server)，创建net.Server对象，同时侦听端口。创建net.Server实例，调用构造函数返回。创建的net.Server实例调用listen(8000)，等待accpet连接。那么，子进程如何传递服务器相关信息给主进程呢？答案就在listen函数中。我保证，net.Server.prototype.listen函数绝没有表面上看起来的那么简单，它涉及到了许多IPC通信和兼容性处理，可以说HTTP服务器创建的所有逻辑都在listen函数中。

>延伸下，在学习linux下的socket编程时，服务端的逻辑依次是执行`socket(),bind(),listen()和accept()`，在接收到客户端连接时执行`read(),write()`调用完成TCP层的通信。那么，对应到node的net模块好像只有**listen()**阶段，这是不是很难对应socket的四个阶段呢？其实不然，node的net模块把“bind，listen”操作全部写入了net.Server.prototype.listen中，清晰的对应底层socket和TCP三次握手，而向上层使用者只暴露简单的listen接口。

**code2**
```
Server.prototype.listen = function() {

  ...

  // 根据参数创建 handle句柄
  options = options._handle || options.handle || options;
  // (handle[, backlog][, cb]) where handle is an object with a handle
  if (options instanceof TCP) {
    this._handle = options;
    this[async_id_symbol] = this._handle.getAsyncId();
    listenInCluster(this, null, -1, -1, backlogFromArgs);
    return this;
  }

  ...

  var backlog;
  if (typeof options.port === 'number' || typeof options.port === 'string') {
    if (!isLegalPort(options.port)) {
      throw new RangeError('"port" argument must be >= 0 and < 65536');
    }
    backlog = options.backlog || backlogFromArgs;
    // start TCP server listening on host:port
    if (options.host) {
      lookupAndListen(this, options.port | 0, options.host, backlog,
                      options.exclusive);
    } else { // Undefined host, listens on unspecified address
      // Default addressType 4 will be used to search for master server
      listenInCluster(this, null, options.port | 0, 4,
                      backlog, undefined, options.exclusive);
    }
    return this;
  }

  ...

  throw new Error('Invalid listen argument: ' + util.inspect(options));
};

```
由于本文只探究cluster模式下HTTP服务器的相关内容，因此我们只关注有关TCP服务器部分，其他的Pipe（domain socket）服务不考虑。

listen函数可以侦听端口、路径和指定的fd，因此在listen函数的实现中判断各种参数的情况，我们最为关心的就是侦听端口的情况，在成功进入条件语句后发现所有的情况最后都执行了listenInCluster函数而返回，因此有必要继续探究。

**code3**
```
function listenInCluster(server, address, port, addressType,
                         backlog, fd, exclusive) {

  ...

  if (cluster.isMaster || exclusive) {
    server._listen2(address, port, addressType, backlog, fd);
    return;
  }

  // 后续代码为worker执行逻辑
  const serverQuery = {
    address: address,
    port: port,
    addressType: addressType,
    fd: fd,
    flags: 0
  };

  ... 

  cluster._getServer(server, serverQuery, listenOnMasterHandle);
}
```
listenInCluster函数传入了各种参数，如server实例、ip、port、ip类型（IPv6和IPv4）、backlog（底层服务端socket处理请求的最大队列）、fd等，它们不是必须传入，比如创建一个TCP服务器，就仅仅需要一个port即可。

简化后的listenInCluster函数很简单，cluster模块判断当前进程为主进程时，执行_listen2函数；否则，在子进程中执行cluster._getServer函数，同时像函数传递serverQuery对象，即创建服务器需要的相关信息。

因此，我们可以大胆假设，子进程在cluster._getServer函数中向主进程发送了创建服务器所需要的数据，即serverQuery。实际上也确实如此：

**code4**
```
cluster._getServer = function(obj, options, cb) {

  const message = util._extend({
    act: 'queryServer',
    index: indexes[indexesKey],
    data: null
  }, options);

  send(message, function modifyHandle(reply, handle) => {
    if (typeof obj._setServerData === 'function')
      obj._setServerData(reply.data);

    if (handle)
      shared(reply, handle, indexesKey, cb);  // Shared listen socket.
    else
      rr(reply, indexesKey, cb);              // Round-robin.
  });

};
```
子进程在该函数中向已建立的IPC通道发送内部消息message，该消息包含之前提到的serverQuery信息，同时包含**act: 'queryServer'**字段，等待服务端响应后继续执行回调函数modifyHandle。

主进程接收到子进程发送的内部消息，会根据**act: 'queryServer'**执行对应queryServer方法，完成服务器的创建，同时发送回复消息给子进程，子进程执行回调函数modifyHandle，继续接下来的操作。

至此，针对主进程在cluster模式下如何创建服务器的流程已完全走通，主要的逻辑是在子进程服务器的listen过程中实现。

### net模块与socket

上节提到了node中创建服务器无法与socket创建对应的问题，本节就该问题做进一步解释。在net.Server.prototype.listen函数中调用了listenInCluster函数，listenInCluster会在主进程或者子进程的回调函数中调用_listen2函数，对应底层服务端socket建立阶段的正是在这里。

```
function setupListenHandle(address, port, addressType, backlog, fd) {

  // worker进程中，_handle为fake对象，无需创建
  if (this._handle) {
    debug('setupListenHandle: have a handle already');
  } else {
    debug('setupListenHandle: create a handle');

    if (rval === null)
      rval = createServerHandle(address, port, addressType, fd);

    this._handle = rval;
  }

  this[async_id_symbol] = getNewAsyncId(this._handle);

  this._handle.onconnection = onconnection;

  var err = this._handle.listen(backlog || 511);

}
```

通过createServerHandle函数创建句柄（句柄可理解为用户空间的socket），同时给属性onconnection赋值，最后侦听端口，设定backlog。

那么，socket处理请求过程“socket(),bind()”步骤就是在createServerHandle完成。
```
function createServerHandle(address, port, addressType, fd) {
  var handle;

  // 针对网络连接，绑定地址
  if (address || port || isTCP) {
    if (!address) {
      err = handle.bind6('::', port);
      if (err) {
        handle.close();
        return createServerHandle('0.0.0.0', port);
      }
    } else if (addressType === 6) {
      err = handle.bind6(address, port);
    } else {
      err = handle.bind(address, port);
    }
  }

  return handle;
}
```
在createServerHandle中，我们看到了如何创建socket（createServerHandle在底层利用node自己封装的类库创建TCP handle），也看到了bind绑定ip和地址，那么node的net模块如何接收客户端请求呢？

必须深入c++模块才能了解node是如何实现在c++层面调用js层设置的onconnection回调属性，v8引擎提供了c++和js层的类型转换和接口透出，在c++的tcp_wrap中：
```
void TCPWrap::Listen(const FunctionCallbackInfo<Value>& args) {
  TCPWrap* wrap;
  ASSIGN_OR_RETURN_UNWRAP(&wrap,
                          args.Holder(),
                          args.GetReturnValue().Set(UV_EBADF));
  int backloxxg = args[0]->Int32Value();
  int err = uv_listen(reinterpret_cast<uv_stream_t*>(&wrap->handle_),
                      backlog,
                      OnConnection);
  args.GetReturnValue().Set(err);
}
```
我们关注uv_listen函数，它是libuv封装后的函数，传入了**handle_,backlog和OnConnection回调函数**，其中handle_为node调用libuv接口创建的socket封装，OnConnection函数为socket接收客户端连接时执行的操作。我们可能会猜测在js层设置的onconnction函数最终会在OnConnection中调用，于是进一步深入探查node的connection_wrap c++模块：

```
template <typename WrapType, typename UVType>
void ConnectionWrap<WrapType, UVType>::OnConnection(uv_stream_t* handle,
                                                    int status) {

  if (status == 0) {
    if (uv_accept(handle, client_handle))
      return;

    // Successful accept. Call the onconnection callback in JavaScript land.
    argv[1] = client_obj;
  }
  wrap_data->MakeCallback(env->onconnection_string(), arraysize(argv), argv);
}
```
过滤掉多余信息便于分析。当新的客户端连接到来时，libuv调用OnConnection，在该函数内执行uv_accept接收连接，最后将js层的回调函数onconnection[通过env->onconnection_string()获取js的回调]和接收到的客户端socket封装传入MakeCallback中。其中，argv数组的第一项为错误信息，第二项为已连接的clientSocket封装，最后在MakeCallback中执行js层的onconnection函数，该函数的参数正是argv数组传入的数据，“错误代码和clientSocket封装”。

**js层的onconnection回调**
```
function onconnection(err, clientHandle) {
  var handle = this;

  if (err) {
    self.emit('error', errnoException(err, 'accept'));
    return;
  }

  var socket = new Socket({
    handle: clientHandle,
    allowHalfOpen: self.allowHalfOpen,
    pauseOnCreate: self.pauseOnConnect
  });
  socket.readable = socket.writable = true;

  self.emit('connection', socket);
}
```
这样，node在C++层调用js层的onconnection函数，构建node层的socket对象，并触发connection事件，完成底层socket与node net模块的连接与请求打通。

至此，我们打通了socket连接建立过程与net模块（js层）的流程的交互，这种封装让开发者在不需要查阅底层接口和数据结构的情况下，仅使用node提供的http模块就可以快速开发一个应用服务器，将目光聚集在业务逻辑中。

>backlog是已连接但未进行accept处理的socket队列大小。在linux 2.2以前，backlog大小包括了半连接状态和全连接状态两种队列大小。linux 2.2以后，分离为两个backlog来分别限制半连接SYN_RCVD状态的未完成连接队列大小跟全连接ESTABLISHED状态的已完成连接队列大小。这里的半连接状态，即在三次握手中，服务端接收到客户端SYN报文后并发送SYN+ACK报文后的状态，此时服务端等待客户端的ACK，全连接状态即服务端和客户端完成三次握手后的状态。backlog并非越大越好，当等待accept队列过长，服务端无法及时处理排队的socket，会造成客户端或者前端服务器如nignx的连接超时错误，出现**“error: Broken Pipe”**。因此，node默认在socket层设置backlog默认值为511，这是因为nginx和redis默认设置的backlog值也为此，尽量避免上述错误。

### 多个子进程与端口复用
再回到关于cluster模块的主线中来。code1中，主进程与所有子进程通过消息构建出侦听8000端口的TCP服务器，那么子进程中有没有也创建一个服务器，同时侦听8000端口呢？其实，在子进程中压根就没有这回事，如何理解呢？子进程中确实创建了net.Server对象，可是它没有像主进程那样在libuv层构建socket句柄，子进程的net.Server对象使用的是一个人为fake出的一个假句柄来“欺骗”使用者端口已侦听，这样做的目的是为了集群的负载均衡，这又涉及到了cluster模块的均衡策略的话题上。

在本节有关cluster集群端口侦听以及请求处理的描述，都是基于cluster模式的默认策略RoundRobin之上讨论的，关于调度策略的讨论，我们放在下节进行。

在**主进程与服务器**这一章节最后，我们只了解到主进程是如何创建侦听给定端口的TCP服务器的，此时子进程还在等待主进程创建后发送的消息。当主进程发送创建服务器成功的消息后，子进程会执行modifyHandle回调函数。还记得这个函数吗？**主进程与服务器**这一章节最后已经贴出来它的源码：
```
function modifyHandle(reply, handle) => {
    if (typeof obj._setServerData === 'function')
      obj._setServerData(reply.data);

    if (handle)
      shared(reply, handle, indexesKey, cb);  // Shared listen socket.
    else
      rr(reply, indexesKey, cb);              // Round-robin.
  }
```
它会根据主进程是否返回handle句柄（即libuv对socket的封装）来选择执行函数。由于cluter默认采用RoundRobin调度策略，因此主进程返回的handle为null，执行函数rr。在该函数中，做了上文提到的hack操作，作者fake了一个假的handle对象，“欺骗”上层调用者：
```
function listen(backlog) {
    return 0;
  }

  const handle = { close, listen, ref: noop, unref: noop };

  handles[key] = handle;
  cb(0, handle);
```
看到了吗？fake出的handle.listen并没有调用libuv层的Listen方法，它直接返回了。这意味着什么？？子进程压根没有创建底层的服务端socket做侦听，所以在子进程创建的HTTP服务器侦听的端口根本不会出现端口复用的情况。 最后，调用cb函数，将fake后的handle传递给上层net.Server，设置net.Server对底层的socket的引用。此后，子进程利用fake后的handle做端口侦听（其实压根啥都没有做），执行成功后返回。

那么子进程TCP服务器没有创建底层socket，如何接受请求和发送响应呢？这就要依赖IPC通道了。既然主进程负责接受客户端请求，那么理所应当由主进程分发客户端请求给某个子进程，由子进程处理请求。实际上也确实是这样做的，主进程的服务器中会创建RoundRobinHandle决定分发请求给哪一个子进程，筛选出子进程后发送newconn消息给对应子进程：
```
  const message = { act: 'newconn', key: this.key };

  sendHelper(worker.process, message, handle, (reply) => {
    if (reply.accepted)
      handle.close();
    else
      this.distribute(0, handle);  // Worker is shutting down. Send to another.

    this.handoff(worker);
  });
```
子进程接收到newconn消息后，会调用内部的onconnection函数，先向主进程发送开始处理请求的消息，然后执行业务处理函数handle.onconnection。还记得这个handle.onconnection吗？它正是上节提到的node在c++层执行的js层回调函数，在handle.onconnection中构造了net.Socket对象标识已连接的socket，最后触发connection事件调用开发者的业务处理函数（此时的数据处理对应在网络模型的第四层传输层中，node的http模块会从socket中获取数据做应用层的封装，解析出请求头、请求体并构造响应体），这样便从内核socket->libuv->js依次执行到开发者的业务逻辑中。

到此为止，相信读者已经明白node是如何处理客户端的请求了，那么下一步继续探究node是如何分发客户端的请求给子进程的。

### 请求分发策略
上节提到cluster模块默认采用RoundRobin调度策略，那么还有其他策略可以选择吗？答案是肯定的，在windows机器中，cluster模块采用的是共享服务端socket方式，通俗点说就是由操作系统进行调度客户端的请求，而不是由node程序调度。其实在node v0.8以前，默认的集群模式就是采用操作系统调度方式进行，直到cluster模块的加入才有了改变。

那么，RoundRobin调度策略到底是怎样的呢？
```
RoundRobinHandle.prototype.distribute = function(err, handle) {
  this.handles.push(handle);
  const worker = this.free.shift();

  if (worker)
    this.handoff(worker);
};

// 发送消息和handle给对应worker进程，处理业务逻辑
RoundRobinHandle.prototype.handoff = function(worker) {
  if (worker.id in this.all === false) {
    return;  // Worker is closing (or has closed) the server.
  }

  const handle = this.handles.shift();

  if (handle === undefined) {
    this.free.push(worker);  // Add to ready queue again.
    return;
  }

  const message = { act: 'newconn', key: this.key };

  sendHelper(worker.process, message, handle, (reply) => {
    if (reply.accepted)
      handle.close();
    else
      this.distribute(0, handle);  // Worker is shutting down. Send to another.

    this.handoff(worker);
  });
};
```
核心代码就是这两个函数，浓缩的是精华。`distribute函数负责筛选出处理请求的子进程，this.free数组存储空闲的子进程，this.handles数组存放待处理的用户请求。handoff函数获取排队中的客户端请求，并通过IPC发送句柄handle和newconn消息，等待子进程返回。当子进程返回正在处理请求消息时，在此执行handoff函数，继续分配请求给该子进程，不管该子进程上次请求是否处理完成（node的异步特性和事件循环可以让单进程处理多请求）。`按照这样的策略，主进程的服务器每接受一个req请求，执行修改后的onconnection回调，执行distribute方法，在其内部调用handoff函数，进入该子进程的处理循环中。一旦主进程没有缓存的客户端请求时（this.handles为空），便会将当前子进程加入free空闲队列，等待主进程的下一步调度。这就是cluster模式的RoundRobin调度策略，每个子进程的处理逻辑都是一个闭环，直到主进程缓存的客户端请求处理完毕时，该子进程的处理闭环才被打开。

这么简单的实现带来的效果却是不小，经过全世界这么多使用者的尝试，主进程分发请求还是很平均的，如果RoundRobin的调度需求不满足你业务中的要求，你可以尝试仿照RoundRobin模块写一个另类的调度算法。

那么cluster模块在windows系统中采用的shared socket策略（后文简称SS策略）是什么呢？采用SS策略调度算法，子进程的服务器工作逻辑完全不同于上文中所讲的那样，子进程创建的TCP服务器会在底层侦听端口并处理响应，这是如何实现的呢？SS策略的核心在于IPC传输句柄的文件描述符，并且在C++层设置端口的**SO_REUSEADDR**选项，最后根据传输的文件描述符还原出handle(net.TCP)，处理请求。这正是shared socket名称由来，共享文件描述符。

**子进程继承父进程fd，处理请求**
```
import socket
import os

def main():
    serversocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    serversocket.bind(("127.0.0.1", 8888))
    serversocket.listen(0)

    # Child Process
    if os.fork() == 0:
        accept_conn("child", serversocket)

    accept_conn("parent", serversocket)

def accept_conn(message, s):
    while True:
        c, addr = s.accept()
        print 'Got connection from in %s' % message
        c.send('Thank you for your connecting to %s\n' % message)
        c.close()

if __name__ == "__main__":
    main()
```
>需要指出的是，在子进程中根据文件描述符还原出的handle，不能再进行bind(ip,port)和listen(backlog)操作，只有主进程创建的handle可以调用这些函数。子进程中只能选择accept、read和write操作。

既然SS策略传递的是master进程的服务端socket的文件描述符，子进程侦听该描述符，那么由谁来调度哪个子进程处理请求呢？这就是由操作系统内核来进行调度。可是内核调度往往出现意想不到的效果，在linux下导致请求往往集中在某几个子进程中处理。这从内核的调度策略也可以推算一二，内核的进程调度离不开**上下文切换**，上下文切换的代价很高，不仅需要保存当前进程的**代码、数据和堆栈等用户空间数据，还需要保存各种寄存器，如PC，ESP**，最后还需要恢复被调度进程的上下文状态，仍然包括**代码、数据和各种寄存器**，因此代价非常大。而linux内核在调度这些子进程时往往倾向于唤醒最近被阻塞的子进程，上下文切换的代价相对较小。而且内核的调度策略往往受到当前系统的运行任务数量和资源使用情况，对专注于业务开发的http服务器影响较大，因此会造成某些子进程的负载严重不均衡的状况。那么为什么cluster模块默认会在windows机器中采用SS策略调度子进程呢？原因是node在windows平台采用的IOCP来最大化性能，它使得传递连接的句柄到其他进程的成本很高，因此采用默认的依靠操作系统调度的SS策略。

SS调度策略非常简单，主进程直接通过IPC通道发送handle给子进程即可，此处就不针对代码进行分析了。此处，笔者利用node的child_process模块实现了一个简易的SS调度策略的服务集群，读者可以更好的理解：

**master代码**
```
var net = require('net');
var cp = require('child_process');
var w1 = cp.fork('./singletest/worker.js');
var w2 = cp.fork('./singletest/worker.js');
var w3 = cp.fork('./singletest/worker.js');
var w4 = cp.fork('./singletest/worker.js');

var server = net.createServer();

server.listen(8000,function(){
  // 传递句柄
  w1.send({type: 'handle'},server);
  w2.send({type: 'handle'},server);
  w3.send({type: 'handle'},server);
  w4.send({type: 'handle'},server);
  server.close();
});
```

**child代码**
```
var server = require('http').createServer(function(req,res){
  res.write(cluster.isMaster + '');
  res.end(process.pid+'')
})

var cluster = require('cluster');
process.on('message',(data,handle)=>{
  if(data.type !== 'handle')
    return;

  handle.on('connection',function(socket){
    server.emit('connection',socket)
  });
});
```

这种方式便是SS策略的典型实现，不推荐使用者尝试。

## 结尾
开篇提到的一些问题至此都已经解答完毕，关于cluster模块的一些具体实现本文不做详细描述，有兴趣感受node源码的同学可以在阅读本文的基础上再翻阅，这样事半功倍。本文是在node源码和计算机网络基础之上混合后的产物，起因于笔者研究PM2的cluster模式下God进程的具体实现。在尝试几天仔细研读node cluster相关模块后有感于其良好的封装性，故产生将其内部实现原理和技巧向日常开发者所展示的想法，最后有了这篇文章。

那么，阅读了这篇文章，熟悉了cluster模式的具体实现原理，对于日常开发者有什么促进作用呢？首先，能不停留在**使用**层面，深入到具体实现原理中去，这便是比大多数人强了；在理解实现机制的阶段下，如果能反哺业务开发就更有意义了。比如，根据业务设计出更匹配的负载均衡逻辑；根据服务的日常QPS设置合理的backlog值等；最后，在探究实现的过程中，我们又回顾了许多离应用层开发人员难以接触到的底层网络编程和操作系统知识，这同时也是学习深入的过程。

接下来，笔者可能会抽时间针对node的其他常用模块做一次细致的解读。其实，node较为重要的**Stream**模块笔者已经分析过了，[node中的Stream](http://www.cnblogs.com/accordion/p/5560531.html)、[深入node之Transform](http://www.cnblogs.com/accordion/p/5907908.html)，经过深入探究之后在日常开发node应用中有着很大的提升作用，读者们可以尝试下。既然提到了Stream模块，那么结合本文的net模块解析，我们就非常容易理解node http模块的实现了，因为http模块正是基于**net和Stream**模块实现的。那么下一篇文章就针对http模块做深入解析吧！

## 参考文章
[Node.js v0.12的新特性 -- Cluster模式采用Round-Robin负载均衡](http://www.cnblogs.com/jasonxuli/p/4522134.html)
[TCP SOCKET中backlog参数](http://ju.outofmemory.cn/entry/191319)
