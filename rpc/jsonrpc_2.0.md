## json-rpc 2.0规范解读

JSON-RPC2.0规范由JSON-RPC工作组（json-rpc@googlegroups.com）维护，发布于2010-03-26(基于2009-05-24的版本)， 最近的更新于2013-01-04。

整体来说，2.0版本的JSON-RPC规范改动的很小，大的改动大概有3点：

1. 参数可以用数组或命名参数
1. 批量请求的细节明确化了
1. 错误处理的机制标准化了

### 与1.0版本的兼容性

- 建议2.0规范的实现兼容1.0协议，但是不强制要求，如果不能兼容，建议给出友好提示。
- 请求和响应报文加了个参数表示协议的版本号：jsonrpc，它必须是“2.0”。
- method的修改：以rpc开头方法名表示rpc内部的方法和扩展，其他地方必须不能使用。
- 请求参数可以使用数组`[参数1，参数2，，，]`，也可以使用命名参数`{key:value}`。
- 请求参数为空时params可省略。
- id一般不应该为null，是数值的话不应该是小数。
- 请求里没有id时，被当做通知。（1.0时这里是id为null。）
- 请求参数必须精确匹配，包括大小写。
- 应答必须包含result或error，但是两个成员都必须不能同时包含。

### 批量请求

终于说清楚了这个批量请求怎么操作，就是一次请求里用数组包装多个请求对象。示例如下，打包5个请求：

	--> [
	        {"jsonrpc": "2.0", "method": "sum", "params": [1,2,4], "id": "1"},
	        {"jsonrpc": "2.0", "method": "notify_hello", "params": [7]},
	        {"jsonrpc": "2.0", "method": "subtract", "params": [42,23], "id": "2"},
	        {"foo": "boo"},
	        {"jsonrpc": "2.0", "method": "foo.get", "params": {"name": "myself"}, "id": "5"},
	        {"jsonrpc": "2.0", "method": "get_data", "id": "9"} 
	    ]
	<-- [
	        {"jsonrpc": "2.0", "result": 7, "id": "1"},
	        {"jsonrpc": "2.0", "result": 19, "id": "2"},
	        {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null},
	        {"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": "5"},
	        {"jsonrpc": "2.0", "result": ["hello", 5], "id": "9"}
	    ]
规范定义了所有的请求应该并发执行，并且返回不保证顺序，客户端自己使用id去匹配对应的请求和响应。而且对于请求的处理中只要有一个出错，则返回一个统一的错误信息（就是不区分哪一条失败，全部都算失败了）。
这个设计看起来是针对事务考虑的，但是在一般的使用场景里应该会比较麻烦。

### 错误对象

改进的error机制是，error变成了一个明确定义的对象。包括三个属性：

- code：数值，见下一节错误代码。
- message：字符串格式的错误信息。
- data：可选的，服务器端定义的一个数值或是对象，来附加额外的信息。

比原来的粗放型错误机制好多了。

### 错误代码

从XML-RPC借来了服务器端的错误代码：

| code | message | meaning |
| ------------- | ------------- | ------------- |
| -32700 | Parse error	Invalid | JSON was received by the server. An error occurred on the server while parsing the JSON text. |
| -32600 | Invalid Request	 | The JSON sent is not a valid Request object. |
| -32601 | Method not found | The method does not exist / is not available. |
| -32602 | Invalid params | Invalid method parameter(s). |
| -32603 | Internal error | Internal JSON-RPC error. |
| -32000 to -32099 | 	Server error | Reserved for implementation-defined server-errors. |


### 参考资料

- [2.0规范全文](http://www.jsonrpc.org/specification)
- [2.0对1.0的改进](http://www.simple-is-better.org/rpc/#differences-between-1-0-and-2-0)

