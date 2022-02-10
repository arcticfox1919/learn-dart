
## Dart  网络编程
以下提供Dart 关于网络编程方面的各种代码示例，对于具体的协议方面知识，请自行学习。

### TCP 服务端

```dart
import 'dart:convert';
import 'dart:io';

void main() {
  //绑定本地localhost的8081端口(即绑定：127.0.0.1)
  ServerSocket.bind(InternetAddress.loopbackIPv4, 8081)
  .then((serverSocket) {
    serverSocket.listen((socket) {
      socket.cast<List<int>>().transform(utf8.decoder).listen(print);
    });
  });
}
```

以上为简洁示例，不是非常清晰，等价于以下代码

```dart
import 'dart:io';
import 'dart:convert';

void main() {
  startTCPServer();
}

// TCP 服务端
void startTCPServer() async{
  ServerSocket serverSocket = await ServerSocket.bind(InternetAddress.loopbackIPv4, 8081);

  //遍历所有连接到服务器的套接字
  await for(Socket socket in serverSocket) {
    // 先将字节流以utf-8进行解码
    socket.cast<List<int>>().transform(utf8.decoder).listen((data) {
      print("from ${socket.remoteAddress.address} data:" + data);
      socket.add(utf8.encode('hello client!'));
    });
  }
}
```

### TCP 客户端

对应的简洁表达如下

```dart
import 'dart:convert';
import 'dart:io';

void main() {
  // 连接127.0.0.1的8081端口
  Socket.connect('127.0.0.1', 8081).then((socket) {
    socket.write('Hello, Server!');
    socket.cast<List<int>>().transform(utf8.decoder).listen(print);
  });
}
```

更清晰写法如下

```dart
import 'dart:convert';
import 'dart:io';

void main() {
  startTCPClient();
}

// TCP 客户端
void startTCPClient() async {
  //连接服务器的8081端口
  Socket socket = await Socket.connect('127.0.0.1', 8081);
  socket.write('Hello, Server!');
  socket.cast<List<int>>().transform(utf8.decoder).listen(print);
}
```

### UDP 服务端

```dart
import 'dart:io';
import 'dart:convert';

void main() {
  startUDPServer();
}

// UDP 服务端
void startUDPServer() async {
  RawDatagramSocket rawDgramSocket = await RawDatagramSocket.bind(InternetAddress.loopbackIPv4, 8081);

  //监听套接字事件
  await for (RawSocketEvent event in rawDgramSocket) {
    // 数据包套接字不能监听数据，而是监听事件。
    // 当事件为RawSocketEvent.read的时候才能通过receive函数接收数据
    if(event == RawSocketEvent.read) {
        print(utf8.decode(rawDgramSocket.receive().data));
        rawDgramSocket.send(utf8.encode("UDP Server:already received!"), InternetAddress.loopbackIPv4, 8082);
      }
  }
}
```

### UDP 客户端

```dart
import 'dart:convert';
import 'dart:io';

void main() {
  startUDPClent();
}

// UDP 客户端
void startUDPClent() async {
   RawDatagramSocket rawDgramSocket = await RawDatagramSocket.bind('127.0.0.1', 8082);

   rawDgramSocket.send(utf8.encode("hello,world!"), InternetAddress('127.0.0.1'), 8081);

  //监听套接字事件
  await for (RawSocketEvent event in rawDgramSocket) {
    if(event == RawSocketEvent.read) {
        // 接收数据
        print(utf8.decode(rawDgramSocket.receive().data));
      }
  }
}
```

### HTTP服务器与请求

关于HTTP的详细说明，可参加本人 [相关博客](https://blog.csdn.net/yingshukun/category_9281313.html)

```dart
import 'dart:io';

void main() {
  HttpServer
      .bind(InternetAddress.loopbackIPv4, 8080)
      .then((server) {
        server.listen((HttpRequest request) {
         // 打印请求的path
          print(request.uri.path);
          if(request.uri.path.startsWith("/greet")){
            var subPathList = request.uri.path.split("/");

            if(subPathList.length >=3){
              request.response.write('Hello, ${subPathList[2]}');
              request.response.close();
            }else{
             request.response.write('Hello, ');
             request.response.close();
            }
          }else{
            request.response.write('Welcome to test server!');
            request.response.close();
          }
        });
      });
}
```

浏览器输入`http://localhost:8080/greet/zhangsan`访问

以上是使用浏览器向服务器发出请求，接下来我们使用代码模拟浏览器发请求

```dart
import 'dart:convert';
import 'dart:io';

void main() {
  HttpClient client = HttpClient();

  client.getUrl(Uri.parse("https://www.baidu.com/"))
    .then((HttpClientRequest request) {
      // 设置请求头
      request.headers.add(HttpHeaders.userAgentHeader,
      "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.102 Safari/537.36");
      return request.close();
    })
    .then((HttpClientResponse response) {
      // 处理响应
      response.transform(utf8.decoder).listen((contents) {
        print(contents);
      });
    });
}
```

通常的，我们并不会直接使用Dart 标准库提供的`http` 网络请求API，因为标准库库使用上仍然过于繁琐，第三方库则更加简洁强大。在Flutter上，主要使用`dio`库，功能十分强大，另外还可以使用官方推出的`http`库，更加简洁精炼，链接如下

- [http](https://pub.dev/packages/http)
- [dio](https://pub.dev/packages/dio)

### WebSocket

> **WebSocket**是一种在单个[TCP](https://baike.baidu.com/item/TCP)连接上进行[全双工](https://baike.baidu.com/item/%E5%85%A8%E5%8F%8C%E5%B7%A5)通信的协议。它的出现使得客户端和服务端都可以主动的推送消息，可以是文本也可以是二进制数据。而且没有同源策略的限制，不存在跨域问题。协议的标识符就是`ws`。像https一样如果加密的话就是`wxs`。

WebSocket 是独立的、创建在 TCP 上的协议。

Websocket 通过[HTTP](https://baike.baidu.com/item/HTTP)/1.1 协议的101状态码进行握手。

为了创建Websocket连接，需要通过浏览器发出请求，之后服务器进行回应，这个过程通常称为“握手”（handshaking）

#### 服务端

Web套接字服务器使用普通的HTTP服务器来接受Web套接字连接。初始握手是HTTP请求，然后将其升级为Web套接字连接。服务器使用[WebSocketTransformer](https://api.dart.dev/stable/2.7.1/dart-io/WebSocketTransformer-class.html)升级请求， 并侦听返回的Web套接字上的数据

```dart
import 'dart:io';

void main() async {
  HttpServer server = await HttpServer.bind(InternetAddress.loopbackIPv4, 8083);
  await for (HttpRequest req in server) {
    if (req.uri.path == '/ws') {
      // 将一个HttpRequest转为WebSocket连接
      WebSocket socket = await WebSocketTransformer.upgrade(req);
      socket.listen((data) {
        print("from IP ${req.connectionInfo.remoteAddress.address}:${data}");
        socket.add("WebSocket Server:already received!");
      });
    }
  }
}
```

#### 客户端

```dart
import 'dart:io';

void main() async {
  WebSocket socket = await WebSocket.connect('ws://127.0.0.1:8083/ws');
  socket.add('Hello, World!');

  await for (var data in socket) {
    print("from Server: $data");

    // 关闭连接
    socket.close();
  }
}

```


注意：本篇内容主要为Dart编程示例，在实际开发中，还有许多问题需要处理，例如TCP的粘包问题，心跳机制，并在Dart中将WebSocket结合ProtoBuf使用等，相关内容请关注后续的Flutter项目实战课程。

# 视频课程

或关注博主视频课程，[**Flutter全栈式开发**](http://m.study.163.com/provider/480000001855430/index.htm?share=2&shareId=480000001855430)

![qr_adv](https://img-blog.csdnimg.cn/img_convert/eb3c16913c155e08e1443a0029003aa1.png)

------

**关注公众号：编程之路从0到1**

![编程之路从0到1](https://img-blog.csdnimg.cn/20190301102949549.jpg)
