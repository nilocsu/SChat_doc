如何用map来处理发送消息+回调 

回顾一下： 

![TEKF9idfadakjfald](images\TEKF9idfadakjfald.png?raw=true)



先设计发送和消息结构：

```
{
	requestId: int,
	data: T
}
```



#### 使用flutter实现过程：

```dart

import 'package:web_socket_channel/io.dart';

typedef CallBack(dynamic map);
class Ws {
    Ws(){
        connect();
    }
  IOWebSocketChannel _socketChannel;
   Map<String, CallBack> callBackMap = new HashMap();
    connect() async(){
      _socketChannel = new IOWebSocketChannel.connect(uri);
      _socketChannel.stream.listen((data) async {
        LogUtil.e("revice message: " + data);

    if (data is String) {
      Map d = jsonDecode(data);
      if (d.containsKey('sendId')) {
       
        callBackMap[d['sendId']](d['data']);
        callBackMap.remove(d['sendId']);
        return;
      }
//    ···code···
      }, onError: (error, StackTrace stackTrace) async {
//    ···code···
      }, onDone: () async {
//    ···code···
      });
    }
	handlerSendEvent(data, callback) {
        var dt = new DateTime.now().millisecondsSinceEpoch;
        Map obj = new Map();
        obj['sendId'] = dt;
        obj['data'] = data;
        if (typeof (callback) == 'function') {
            this.callBackMap[dt] = callback
        }
    	_socketChann
            el.sink.add(json.encode(obj));
    }
}
Ws ws = new Ws();
```

每次发送数据 直接调用 ws.handlerSendEvent

### 神奇的Promise

如果我们想在 发送消息里面镶嵌消息，就会出现嵌套，如：

```dart
ws.handlerSendEvent(data, function(v){
	//    ···code···
	ws.handlerSendEvent(data1, function(v1){
		//    ···code···
	})
})
```

调用一两层，可还好，如果多了，代码可读性就差了



我们可以使用flutter 的Future 来封装， 要使用 `Completer`这个对象

代码如下：

```dart

import 'package:web_socket_channel/io.dart';



class Ws {
   Ws(){
       connect();
  }
  IOWebSocketChannel _socketChannel;
   Map<String, Completer> callBackMap = new HashMap();
    connect() async(){
      _socketChannel = new IOWebSocketChannel.connect(uri);
      _socketChannel.stream.listen((data) async {
        LogUtil.e("revice message: " + data);

    if (data is String) {
      Map d = jsonDecode(data);
      if (d.containsKey('sendId')) {
       if(//判断是否成功
           true
       ){
           
        callBackMap[d['sendId']]
            .complete( d['data']);
       }else{
           callBackMap[d['sendId']]
            .completeError( '待处理')
       }
        callBackMap.remove(d['sendId']);
        return;
      }
//    ···code···
      }, onError: (error, StackTrace stackTrace) async {
//    ···code···
      }, onDone: () async {
//    ···code···
      });
    }
	Future<dynamic> handlerSendEvent(data) {
    Completer completer = new Completer();
        var dt = new DateTime.now().millisecondsSinceEpoch;
        Map obj = new Map();
        obj['sendId'] = dt;
        obj['data'] = data;
        this.callBackMap[dt] = completer;
    	_socketChann
            el.sink.add(json.encode(obj));
        return completer.future;
    }
}
Ws ws = new Ws();
                                   
  
```

注意的是

```
Map<String, Completer> callBackMap = new HashMap();//Completer<T>要指定泛型 不然解析起来就麻烦了
```



发送消息如：

```dart
sendMessage()async{
    ws.handlerSendEvent(data1).then((v){}).catchError((e){});
}
```

个人觉得这样写 也会出现 嵌套现在， 但是并不是很严重， 因为 我们可以用 `await` 来处理

```dart
sendMessage()async{
    var res = await ws.handlerSendEvent(data1); // 这样可以避免嵌套过多， 就是多了很多 if
}
```







