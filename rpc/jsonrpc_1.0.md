## json-rpc 1.0规范

JSON可能是这个地球上最简单的文本数据格式了，可读、灵活、数据量小，编解码方便、速度快，对Unicode和特殊字符支持的好。对比下XML，就知道额外的各种标签节点需要浪费多少字节数。JSON字符默认都要使用Unicode形式，所有非ACSII字符都可以用\uXXXX表示，而不需要额外的转义。相比之下，XML里需要使用转义或是CDATA（类似HTML里的PRE标签）、或是Base64才能表示特殊数据。当然缺点也很明显，比二进制数据结构的数据量大，编解码慢，没有完备的类型系统，表达能力有限。

JSON-RPC是一个使用json对象作为数据载体的远程过程调用（Remote Process Call，RPC）技术。

> "Does distributed computing have to be any harder than this? I don't think so." -- Jan-Klaas Kollhof 

JSON-RPC的设计目标就是两个字：`简单`。我们知道一个rpc框架是为了2个系统交互通信用得，这就需要定义一个中间的数据传输格式。为了跟系统本身用的平台数据结构转换，需要提供一套序列化和反序列化这个数据格式的功能。然后就是需要某种通信协议来传输实际远程调用的数据。最后还需要通信的两端有实现的代码桩（stub&skeleton），这一般是基于动态代理或AOP实现的代理，一个可供调用的接口结构，使得框架隐藏了其他所有的技术细节（数据格式、序列化、网络传输等），程序里能像本地方法调用一样调用远程的方法。总结一下：

1. 数据格式
1. 序列化功能
1. 通信协议
1. 代码桩

JSON-RPC规范里只显式规定了数据格式（即JSON），建议了通信协议（TCP或是HTTP），序列化和代码桩没有提及。当然序列化比较好办，各种语言里都有丰富的JSON序列化库（参见参考资料1）。那么代码桩就完全留给了实现JSON-RPC规范的框架自己去处理了，通信协议这一块的具体处理也大部分需要框架自己考虑。

我们知道两个系统的通信，可能是同步阻塞的（每个请求要等响应完成以后再发下一个请求），也可能是异步非阻塞的（请求1发完，继续发请求2，哪个响应回来就处理哪个响应）；也可能是单向的（one-way，发了就不管了、不要响应信息），也可能是请求-响应的（request-response），每个请求都需要有显式的回复。

JSON-RPC1.0里，把通信的双方当做对等的两个端。

- 定义了如何发请求和返回结果、出错信息，
- 如何处理单向和请求-响应的交互过程，
- 以及双向异步请求如何匹配结果和请求，
- 最后还添加了一个简单的类型提示（class hinting）扩展功能（想法很好，但是就两句话没细节，挺鸡肋的）。

我们来具体看看规范的内容。

### 请求（request）
请求端发送一个JSON对象表示自己要调用的方法，示例如下：

	{ "method": "echo", "params": ["Hello JSON-RPC"], "id": 1}

其中包含三个关键元素：

- method：表示要调用的方法
- params：表示参数的数组
- id：表示这次请求是需要响应的，服务方需要提供一个同样是id为1的响应信息。

这个id字段非常有意思，一般的同步RPC都不需要考虑这个东西，因为一般情况下，一个RPC请求只包含一个请求，并且请求会等待响应信息返回，在这之前一直会阻塞调用方的处理线程，这样整个RPC才像是调用的本地方法。

### 响应（response）

一个典型的响应信息如下：

	{ "result": "Hello JSON-RPC", "error": null, "id": 1}

关键元素也是3个：

- result：表示返回结果，如果出错result必须为null
- error：表示出错，如果请求成功，则error必须为null
- id：表示为此响应对应的请求id

### 通知（notification）

通知信息类似一个请求，但是id必须为null。通知代表单向的请求，不能回复，可以由请求方发送给服务方，也可以由服务方发送给请求方（协议原则上要求双方对等，都可以是请求方或服务方，这一点上HTTP其实是不满足的）。

### 通信（transport）

* TCP

协议推荐通信双方在TCP下使用字节流（byte stream）的方式交互。此时双方是对等的。关闭连接前必须给所有未应答的请求方发送一个错误信息。无效的请求或响应将会导致连接关闭。

* HTTP

在一些限制的条件下，也可以使用HTTP作为通信协议。此时通信双方明显不在对等，有了客户端和服务器端。

考虑到HTTP的开销问题，协议允许一次POST可以带上多个rpc请求或通知。由于HTTP下，server不能够主动访问client，所以server可以在HTTP响应中附带上自己的请求和通知。这个地方协议们没有说清楚具体操作，细节也是个蛋疼的事儿。

### 类型提示（transport）

非常简单，就是加了个属性表示构造函数`constructor`，可以用参数`[param1,...]`来初始化对象。而且如果对象初始化的时候调用了这个构造器，那么相应的属性（比如`prop1`）也会添加到对象，示例代码：

	{"__jsonclass__":["constructor", [param1,...]], "prop1": ...}

协议没有具体说清楚这个地方，具体怎么操作，怎么对应原生环境里的类型，蛋疼++。

### 一个多次通信的例子

JSON-RPC过程如下，其中`-->`代表发送数据到服务方，`<--`代表服务方的响应:

	--> {"method": "postMessage", "params": ["Hello all!"], "id": 99}
	<-- {"result": 1, "error": null, "id": 99}
	<-- {"method": "handleMessage", "params": ["user1", "we were just talking"], "id": null}
	<-- {"method": "handleMessage", "params": ["user3", "sorry, gotta go now, ttyl"], "id": null}
	--> {"method": "postMessage", "params": ["I have a question:"], "id": 101}
	<-- {"method": "userLeft", "params": ["user3"], "id": null}
	<-- {"result": 1, "error": null, "id": 101}

### 关于JSON-RPC1.0规范的进一步讨论

* 关于通知

通知机制也可以用于类似FTP协议的NOOP，或是其他通讯协议里的Keep Alive心跳机制。在具体使用的过程中，定时的发送通知来确认双方都在线，并且可以捎带异步处理的信息。

* 关于id

请求中id字段的作用，就跟JMS协议里的correlationID一样，可以起到区分多个请求的作用，这样又两个好处：

1. 多个请求可以打包成一个请求，返回也可以返回多个响应，处理方按照id区分多个数据即可；
1. 可以做异步的响应，反正我不保证顺序了，也能根据id来匹配原来是哪个请求来着。

特别是配合HTTP和通知功能，使得异步调用变得可能：客户端直接发送请求1后返回，继续发送请求2，请求3...等等；服务端在处理完成请求1后，可能把响应放到请求2或3的HTTP response报文返回，或者在接下里的某次交互里返回即可。

* 关于类型

JSON-RPC跟XML-RPC非常类型，但是借助于XML本身的schema结构，XML-RPC定义了一套基础的数据类型、以及在基础上的构造复杂数据类型的能力，这样XML-RPC中就可以用于更复杂的业务交互场景。而JSON-RPC使用JS原生的弱类型，只能表示非常简单和模糊的元数据结构，不利于复杂场景和实现代码桩的生成工具。如果有一些更有效的schema约束，则可以把JSON-RPC应用得更广泛。

JSON-RPC的历史参见：[whynamed-rpc-jsonrpc](https://github.com/kimmking/whynamed/blob/master/rpc/jsonrpc.md)


### 参考资料

- [JSON数据格式规范与各种语言下的JSON库](http://www.json.org/)
- [1.0规范全文](http://json-rpc.org/wiki/specification)
- [各种语言下的实现](http://json-rpc.org/wiki/implementations)

