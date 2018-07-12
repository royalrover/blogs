ThreadLocal变量的说法来自于Java，这是在多线程模型下出现并发问题的一种解决方案。
ThreadLocal变量作为线程内的局部变量，在多线程下可以保持独立，它存在于
线程的生命周期内，可以在线程运行阶段多个模块间共享数据。那么，ThreadLocal变量
又如何与node.js扯上关系呢？

### node模型
node的运行模型无需再赘言： “**事件循环 + 异步执行**”，可是node开发工程师比较感兴趣的点
大多集中在 “编码模式”上，即异步代码同步编写，由此提出了多种解决回调地狱的解决方案：
- yield
- thunk
- promise
- await

可是如果从代码执行流程的微观视角中跳出来，宏观上看待node服务器处理每个HTTP请求，就会
发现这其实是**多线程web服务器的另一种体现**，虽然设计上并不像多线程模型那么直观。在单核cpu中
每一时刻node服务器只能处理一个请求，可是node在当前请求中执行异步调用时，就会“中断”进入下一个
事件循环处理另一个请求，直到上一个请求的异步任务事件触发执行对应回调，继续执行该请求的后续逻辑。
这在某种程度上类似于CPU的时间片抢占机制，微观上的顺序执行，宏观上却是同步执行。

node在单进程单线程（js执行线程）中“模拟”了常见的多线程处理逻辑，虽然在单个node进程中无法
充分利用CPU的多核及超线程特性，可是却避免了多线程模型下的临界资源同步和线程上下文
切换的问题，同时内存资源开销相对较小，因此在I/O密集型的业务下使用node开发web服务
往往有着意想不到的好处。

可是在node开发中需要追踪每个请求的调用链路，通过获取请求头的traceId字段在每一级
的调用链路中传递该字段，包括“http请求、dubbo调用、dao操作、redis和日志打点”等操作。
这样通过追踪traceId，就可以分析请求所经过的所有中间链路，评估每个环节的时延与瓶颈，
更容易进行性能优化和错误排查。

那么，如何在业务代码中无侵入性的获取到相关的traceId呢？这就引出了本文的ThreadLocal变量。

## 传统的日志追踪模式
需手动传递traceId给日志中间件：
```
var koa = require('koa');
var app =  new koa();
var Logger = {
    info(msg,traceId){
        console.log(msg,traceId);
    }
};
let business = async function(ctx){
    let v = await new Promise((res)=>{
        setTimeout(()=>{
            Logger.info('service执行结束',ctx.request.headers['traceId'])
            res(123);
        },1000);
    });
    ctx.body = 'hello world';
    Logger.info('请求返回',ctx.request.headers['traceId'])
};

app.use(async(ctx,next)=>{
    ctx.request.headers['traceId'] = Date.now() + Math.random();
    await next();
});

app.use(async(ctx,next)=>{
    await business(ctx);
});

app.listen(8080);
```
在business业务处理函数中，在service执行结束和body返回后都进行日志打点，同时手动
传递请求头traceId给日志模块，方便相关系统追踪链路。

目前这样编码无法规范化日志接口，同时也对开发人员造成了很大的困扰。对于业务开发人员他们
理应不关心如何进行链路追踪，而目前的编码则直接侵入了业务代码中，这块功能应该由日志模块
Logger来实现，可是在与请求上下文没有任何联系的Logger模块如何获取每个请求的traceId呢？

这就需要依靠node.js中的ThreadLocal变量。文章开头提到，多线程下ThreadLocal变量是与
每个线程的生命周期对应的，那么如果在node.js的“单线程+异步调用+事件循环”的特性下实现
类似的ThreadLocal变量，不就可以在每个请求的异步回调执行时获取到对应的ThreadLocal变量，
拿到相关的上下文信息吗？

## ThreadLocal的node实现
单纯实现web服务器的中间链路请求追踪其实并不复杂，使用全局变量Map并通过每个请求的唯一标识
存储上下文信息，当执行到该请求的下一个异步调用时便通过在全局Map中获取到与该请求绑定的ThreadLocal
变量，不过这是在应用层面的一种投机行为，是与请求紧耦合的简易实现。

最彻底的方案则是在node应用层实现一种栈帧，在该栈帧内重写所有的异步函数，并添加各个
hook在异步函数的各个生命周期执行，实现异步函数执行上下文与栈帧的映射，这便是最为
彻底的ThreadLocal实现，而不是仅仅停留在与HTTP请求的映射过程中。

目前已经有**zone.js**库实现了node应用层栈帧的可控编码，同时可以在该栈帧存活阶段绑定
相关数据，我们便可以利用这种特性实现类似多线程下的ThreadLocal变量。

我们的目标是实现无侵入的编写包含链路追踪的业务代码，如下所示：

```
app.use(async(ctx,next)=>{
    let v = await new Promise((res)=>{
        setTimeout(()=>{
            Logger.info('service执行结束')
            res(123);
        },1000);
    });
    ctx.body = 'hello world';
    Logger.info('请求返回')
});
```
相比较，Logger.info中不需要手动传递traceId变量，由日志模块通过访问ThreadLocal变量获取。

通过zone.js提供的创建Zone（对应于栈帧）功能，我们不仅可以获取当前请求（类似于多线程下的单个线程）的
ThreadLocal变量，还可以获取上一个请求的相关信息。

```
require('zone.js');
var koa = require('koa');
var app =  new koa();
var Logger = {
    info(msg){
        console.log(msg,Zone.current.get('traceId'));
    }
};

var koaZoneProperties = {
    requestContext: null
};
var koaZone = Zone.current.fork({
    name: 'koa',
    properties: koaZoneProperties
});
let business = async function(ctx){
    let v = await new Promise((res)=>{
        setTimeout(()=>{
            Logger.info('service执行结束')
            res(123);
        },1000);
    });
    ctx.body = 'hello world';
    Logger.info('请求返回')
};
koaZone.run(()=>{
    app.use(async(ctx,next)=>{
        console.log(koaZone.get('requestContext'))
        ctx.request.headers['traceId'] = Date.now();
        await next();
    });
    
    app.use(async(ctx,next)=>{
        await new Promise((resolve)=>{
            let koaMidZone = koaZone.fork({
                name: 'koaMidware',
                properties: {
                    traceId: ctx.request.headers['traceId']
                }
            }).run(async()=>{
                // 保存请求上下文至parent zone
                koaZoneProperties.requestContext = ctx;
                await business(ctx);
                resolve();
            });
        });
    });
    
    app.listen(8080);
});
```
创建了两个有继承关系的zone（栈帧），koaZone的requestContext属性存储上一个请求的上下文信息；
koaMidZone的traceId属性存储traceId变量，这是一个ThreadLocal变量。
Logger.info中通过**Zone.current.get('traceId')** 获取当前“线程”的
ThreadLocal变量，无需开发人员手动传递traceId变量。

关于zone.js的其他用法，读者有兴趣可以自行研究。本文主要利用zone.js保存一个执行栈帧
内的多个异步函数的执行上下文与特定数据（即ThreadLocal变量）的映射。

## 说明
目前，这套模型已在线上业务中用来追踪各级链路，各级中间件包括dubbo client、dubbo provider、
配置中心等都依赖ThreadLocal变量实现数据透传和调用传递，因此可以放心使用。