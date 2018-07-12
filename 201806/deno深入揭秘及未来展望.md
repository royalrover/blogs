## deno
node.js之父Ryan Dahl在一个月前发起了名为deno的项目，项目的初衷是**打造一个基于v8引擎的安全的TypeScript运行时**，同时实现HTML5的基础API。所谓的安全运行时，是将TS代码运行在一个沙盒里，访问受限的文件系统、网络功能，这比较类似于web里的iframe sandbox。

现阶段，deno的变化可谓翻天覆地。Ryan的项目一个月前提供了golang版本的deno简易源码，而如今不仅仅重构了项目，底层语言都切换为c++，接口也做了很大的更新，这源自于社区内热情的讨论，有太多太多的开发者、协作人员提出了太多的优化以及改进意见，这也就导致接下来未来几个月deno仍然会出现大改变，这在后文会提及。现在，我就带领大家进入最初的deno微观世界探索deno最初的设计。

## 架构
> 本文讲解deno的golang版本，当前最新的deno由于性能问题放弃了golang的实现，但这不影响我们分析deno的原理。未来在七月deno估计会释放出基于Rust的底层特权级实现，性能更优。

> 由于deno涉及之处是为了直接运行TS，因此下文会用TS来代指JS（现阶段TS没有自己的运行时，仍是基于编译为JS在运行在v8）

deno的设计初期来看比较简单，宏观上看包括三部分：deno的go运行时、v8引擎以及连接go运行时和v8的v8worker2库。
![deno简易架构](//si.geilicdn.com/hz_img_2c09000001644e9c9a1d0a02853e_1248_938_unadjust.png)
go运行时是deno的特权级，它负责deno对系统资源的申请、使用、释放；v8引擎此处不仅仅执行JS代码，同时也负责TypeScript的编译；而v8worker2负责go与v8的全双工通信，通过ArrayBuffer传输数据，传输的协议规范为protobuf。

深入到go运行时里，目前deno对TS层提供了几种能力：**Console、fetch、fs、module、timer、stack trace**，虽然有些功能没有提供用户端API，不过golang的接口已完成，扩展很容易。
![deno详细](//si.geilicdn.com/hz_img_3432000001644eae9fd50a02685e_1216_876_unadjust.png)

## go运行时
deno在特权级代码执行了3端逻辑：
1. 初始化go运行时环境
2. 初始化TS运行时环境
3. 启动go这一侧的事件循环（该事件循环不同于node的基于libuv的event loop，下文会提到）

### 初始化go运行时环境
```
    // HOME目录下创建 cache和src目录
	createDirs()
	// 利用 afero 库创建虚拟fs对象；同时订阅 v8端的 os事件，在go端实现 文件抓取、获取缓存、磁盘I/O，同时返回 proto序列化数据 给v8
	InitOS()
	// 心跳
	InitEcho()
	// 接受v8消息，进行 timeout、interval和clear
	InitTimers()
	// 订阅 fetch 事件，代理服务器。当代理请求结束时，返回两个消息：第一个为状态码；第二个为body体
	InitFetch()

	// recv为 v8->go 的回调函数，处理v8的消息
	worker = v8worker2.New(recv)

	// 初始化ts的相关环境，和go端对应
	main_js = stringAsset("main.js")
	err := worker.Load("/main.js", main_js)
	exitOnError(err)
```
依次执行以下任务：
    - 创建缓存目录，存储TS文件编译后的JS文件
    - 订阅 os 事件，处理来自v8层的操作，如fs等
    - 订阅 timer 事件，处理来自v8的定时器操作
    - 订阅 fetch 事件，处理来自v8的http request
    - 初始化v8worker2实例，实现go与v8的绑定
    - 加载js入口文件main.js，该文件定义了js的全局接口、初始化逻辑和与go运行时通信的方法，等待下一阶段的执行。
    
### 初始化js运行时环境
```
// v8端执行 denoMain函数，在main.ts中定义
deno.Eval("deno_main.js", "denoMain()")
```
上一步v8已经加载并执行了**main.js**文件，现在该执行denoMain方法了。denoMain是在main.js中定义的初始化方法，它定义了deno在js层的API以及v8worker实例，也是开发者密切相关的一层。

关于ts层的逻辑留在下文讲述。

### 启动事件循环
```
    var resChan = make(chan *BaseMsg, 10)
    var doneChan = make(chan bool)
    var wg sync.WaitGroup
    wg.Add(1)
	first := true

	// In a goroutine, we wait on for all goroutines to complete (for example
	// timers). We use this to signal to the main thread to exit.
	// wg.Add(1) basically translates to uv_ref, if this was Node.
	// wg.Done() basically translates to uv_unref
	go func() {
		wg.Wait()
		doneChan <- true
	}()

	for {
		select {
		case msg := <-resChan:
			out, err := proto.Marshal(msg)
			check(err)
			err = worker.SendBytes(out)
			stats.v8workerSend++
			stats.v8workerBytesSent += len(out)
			exitOnError(err)
			wg.Done() // Corresponds to the wg.Add(1) in Pub().
		case <-doneChan:
			// All goroutines have completed. Now we can exit main().
			checkChanEmpty()
			return
		}

		// We don't want to exit until we've received at least one message.
		// This is so the program doesn't exit after sending the "start"
		// message.
		if first {
			wg.Done()
		}
		first = false
	}
```
熟悉go语言的人会发现这是协程goroutine的典型用法：
main协程开启循环，监听来自resChan channel的消息，当接受到resChan的消息时意味着此刻go运行时需要向v8返回相关数据，如定时器执行结果、网络请求结果，执行对应的select case，通过v8worker2写入经过protobuf处理后的数据，进入下一次循环；直到go运行时此刻处理完所有的ts请求，会执行协程中的逻辑`doneChan <- true`，最终触发main协程的case `case <-doneChan`,结束事件循环退出程序。

因此，deno的golang版本的事件循环与node基于libuv的事件循环并不是一回事，因此不能一概而论。

## TS运行时与v8worker2
TS运行时对应于v8的实例isolate，在isolate上定义了handscope、context以及在handscope范围内的一系列句柄对象。TS运行时的初始化配置是在v8worker2中定义的，在v8worker2中，借助cgo模块实现go与c的通信：go可以调用c库，同时也可到处go函数给c程序使用。在本文中，这不是要讲述的重点，有兴趣的同学可以等下一篇文章的介绍。

总之，TS运行时的初始化是由go的v8worker2模块执行，它向v8暴露了global全局变量，同时提供了global变量下提供V8Worker2对象，用于v8与golang的通信。

TS运行时初始化完毕后，看是准备deno在TS层的执行环境，包括：

- 初始化定时器事件，监听go运行时返回的timer事件，该事件对象里有TS调用定时器的返回结果
- 初始化 fetch 事件，该事件对象里有TS请求net、fs的返回值
- 订阅 start 事件，等待执行deno程序

在 start事件处理函数中，deno做了两件事：
- 编译TS源文件
- 执行JS文件

deno使用typescript模块提供的LanguageServiceHost功能，采用硬编码的编译规则
```
ts.CompilerOptions = {
    allowJs: true,
    module: ts.ModuleKind.AMD,
    outDir: "$deno$",
    inlineSourceMap: true,
    lib: ["es2017"],
    inlineSources: true,
    target: ts.ScriptTarget.ES2017
  };
```
默认使用es2017规范，模块规范使用AMD规范。

### ts模块的加载
目前ts模块加载支持**fs和nfs**，也就是“相对路径加载和网络加载”，如
```
import { printHello } from "./subdir/print_hello.ts";
import { printHelloNfs } from "http://localhost:4545/testdata/subdir/print_hello.ts";

printHello();
printHelloNfs();
```
TS模块如何转换为AMD规范并且如何确定加载顺序，下面举例说明：
有两个ts文件： a.ts和say.ts
```
  a.ts:

  import say from './say';
  say('hello world');
  --------------------
  say.ts:

  export function say(msg){
    console.log(msg)
  }
```
执行命令` deno a.ts`,返回“hello world”。

经过ts运行时的编译后，a.ts的编译后的代码为：
```
define(["require", "exports", "./say.ts"], function (require, exports, say) {
    "use strict";
    Object.defineProperty(exports, "__esModule", { value: true });
    say(msg);
  });
```
其中，回调函数的require参数简单的require实现，exports为a.ts模块的导出对象，say模块则为say.ts的导出对象。

对于“./say.ts”文件，是由ts运行时通过v8worker2传递消息由go运行时获取对应源文件（此处通过fs或者net），通过ArrayBuffer传递给ts运行时，并进行编译、运行，传递给引用模块a.ts。最后，当所有依赖模块加载完毕之后，a.ts的回调函数执行，实现模块间时序的调度。

> 关于模块加载问题，社区内有提出异议，即增加绝对路径的引用方式： import "/abc/test.ts". 不过Ryan认为这种绝对路径方式会与系统的根目录进行冲突，而且不符合deno所提出的“安全的TS运行时”，这样会暴露系统的路径或文件信息。不过社区也提出了解决方案，即在deno运行时提供命令行参数 --baseDir，标识当前deno进程的根目录，防止访问系统的文件系统。

## 颇具争议的v8worker2与protobuf
其实deno的golang实现被诟病最多的也正是v8worker2与protobuf。这两个模块非常有名，但是不太适用于deno的场景。

首先说道protobuf，这是google提出的一种跨平台跨语言的结构化数据存储格式，它是有类型声明的，通过protobuf的命令行工具可以生成不同语言的代码，操作对应的数据结构。但是protobuf的性能瓶颈在于序列化与反序列化，这也正是protobuf作者在deno项目下之一Ryan的原因，他推荐使用 Cap'n Proto来进行数据传递。 Cap'n Proto比较有意思，它使用ArrayBuffer进行传递，并且不需要序列化为对应语言的相关变量，直接提供一套方法读取二进制数据（类似于访问数组使用的偏移量），更快。

对于v8worker2模块，笔者通读了这个binding实现，其实Ryan对于v8worker2已经尽可能优化了，不过并没有开启v8的snapshot特性，对于重复引入的模块会有些性能损失。但是最重要的瓶颈其实在于v8worker2依赖的cgo模块。cgo对于c库以及编译器的支持非常的不错，但是在数据类型的转换耗费性能比较多。

下图为社区针对golang版本的deno做出的go运行时的性能分析：
![性能分析](https://user-images.githubusercontent.com/20110/41383530-3ead3348-6f3f-11e8-9aa8-8a9bc9e1f5a1.png)
可以看出v8worker2的SendBytes和Load执行占比已超过70%。而这两个函数主要逻辑是使用cgo完成数据传递以及TS执行。
社区也有相关[cgo性能瓶颈](https://stackoverflow.com/questions/28272285/why-cgos-performance-is-so-slow-is-there-something-wrong-with-my-testing-code)的介绍，即go中的协程goroutien不同于OS的线程，在具体实现上取决于GOMAXPROCS设置以及调度策略。一旦通过cgo在c语言进行系统调用，那么会导致当前go routine所在的线程睡眠，直到调用返回。那么其他跑在当前线程的go routine都会被阻塞导致性能下降。因此，Ryan下个版本也会放弃使用go的v8worker2模块。

## deno的golang版本生命终结
终于到了这个话题，golang实现的deno现在已经被放弃了，这是由于性能问题导致的：
- 与c/c++绑定性能差，这是由cgo模块导致的，也直接导致deno的golang实现tps小，rt比较大
- golang的GC机制导致的性能的不确定性。目前v8采用的是标记清楚+整理的GC算法，而golang运行时也运行类似的GC算法，这样在多线程中存在两个并行的GC线程会对程序运行造成非常大的不确定性
- 社区内Rust力量壮大，Rust的服务器性能越发强大，而且没有GC机制，与c通信性能高过golang，因此也算是个推进因素

不过，虽然golang版本的deno走到了终点，我们通过Ryan的实现仍然很容易把握住deno的脉络，因此对于相关的开发者仍有借鉴和参考意义。

## deno的未来及感慨
就目前社区内部的讨论以及Ryan的决定来看，deno在七月份仍有重大改变：底层的代码会切换为Rust，会使用libdeno作为Rust和C的binding。deno社区目前还非常活跃，各种想法和思潮互相碰撞，比如关于模块管理与加载、API的设计、v8编译TS的优化等，在这个时代我们必须要跟上浪潮，学习这些弄潮人的思想及设计理念。

笔者之前非常专注于node的摄入挖掘与应用，不过自从deno出来之后带给笔者的震撼远非语言之能形容。因此学习golang、阅读v8文档通读deno，尽量走出自己的舒适区感受墙外的先进思想，碰撞中学习，求同中存异，收货颇丰。最后感慨下，是不是国内相对封闭的互联网环境导致国内前端或全栈领域的思维有些僵化，无法产生并主导这种非常有意思的idea和项目，当然也有可能是我们每天忙于业务需求中无法自拔。愿国内开发者且行且珍惜，不能被国外的同行甩开太多。