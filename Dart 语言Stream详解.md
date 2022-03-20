
# 异步之 Stream 详解
关于Dart 语言的Stream 部分，应该回到语言本身去寻找答案，许多资料在Flutter框架中囫囵吞枣式的解释`Stream`，总有一种让人云山雾罩的感觉，事实上从Dart语言本身去了解Stream并不复杂，接下来就花点时间好好学习一下`Stream`吧！

`Stream`和 `Future`都是Dart中异步编程的核心内容，在之前的文章中已经详细叙述了关于`Future`的知识，请查看 [Dart 异步编程详解](https://blog.csdn.net/yingshukun/article/details/89840009)，本篇文章则主要基于 Dart2.5 介绍`Stream`的知识。


## 什么是Stream
`Stream`是Dart语言中的所谓异步数据序列的东西，简单理解，其实就是一个异步数据队列而已。我们知道队列的特点是先进先出的，`Stream`也正是如此

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915152423489.gif)

更形象的比喻，`Stream`就像一个传送带。可以将一侧的物品自动运送到另一侧。如上图，在另一侧，如果没有人去抓取，物品就会掉落消失。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915152851279.gif)

但如果我们在末尾设置一个监听，当物品到达末端时，就可以触发相应的响应行为。

> 在Dart语言中，`Stream`有两种类型，一种是点对点的单订阅流（Single-subscription），另一种则是广播流。

## 单订阅流
单订阅流的特点是只允许存在一个监听器，即使该监听器被取消后，也不允许再次注册监听器。

### 创建 Stream
创建一个`Stream`有9个构造方法，其中一个是构造广播流的，这里主要看一下其中5个构造单订阅流的方法

#### periodic
```java
void main(){
  test();
}

test() async{
  // 使用 periodic 创建流，第一个参数为间隔时间，第二个参数为回调函数
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), callback);
  // await for循环从流中读取
  await for(var i in stream){
    print(i);
  }
}

// 可以在回调函数中对值进行处理，这里直接返回了
int callback(int value){
  return value;
}
```

打印结果：
```
0
1
2
3
4
...
```

该方法从整数0开始，在指定的间隔时间内生成一个自然数列，以上设置为每一秒生成一次，`callback`函数用于对生成的整数进行处理，处理后再放入`Stream`中。这里并未处理，直接返回了。要注意，这个流是无限的，它没有任何一个约束条件使之停止。在后面会介绍如何给流设置条件。

#### fromFuture

```java
void main(){
  test();
}

test() async{
  print("test start");
  Future<String> fut = Future((){
      return "async task";
  });

  // 从Future创建Stream
  Stream<String> stream = Stream<String>.fromFuture(fut);
  await for(var s in stream){
    print(s);
  }
  print("test end");
}
```
打印结果：
```
test start
async task
test end
```
该方法从一个`Future`创建`Stream`，当`Future`执行完成时，就会放入`Stream`中，而后从`Stream`中将任务完成的结果取出。这种用法，很像异步任务队列。

#### fromFutures
从多个`Future`创建`Stream`，即将一系列的异步任务放入`Stream`中，每个`Future`按顺序执行，执行完成后放入`Stream`

```java
import  'dart:io';

void main() {
  test();
}

test() async{
  print("test start");
  Future<String> fut1 = Future((){
      // 模拟耗时5秒
      sleep(Duration(seconds:5));
      return "async task1";
  });
    Future<String> fut2 = Future((){
      return "async task2";
  });

  // 将多个Future放入一个列表中，将该列表传入
  Stream<String> stream = Stream<String>.fromFutures([fut1,fut2]);
  await for(var s in stream){
    print(s);
  }
  print("test end");
}
```

#### fromIterable
该方法从一个集合创建`Stream`，用法与上面例子大致相同
```java
// 从一个列表创建`Stream`
Stream<int> stream = Stream<int>.fromIterable([1,2,3]);
```

#### value
这是Dart2.5 新增的方法，用于从单个值创建`Stream`
```java
test() async{
  Stream<bool> stream = Stream<bool>.value(false);
  // await for循环从流中读取
  await for(var i in stream){
    print(i);
  }
}
```

### 监听 Stream
监听`Stream`，并从中获取数据也有三种方式，一种就是我们上文中使用的`await for`循环，这也是官方推荐的方式，看起来更简洁友好，除此之外，另两种方式分别是使用`forEach`方法或`listen`方法

```java
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), callback);
  // 使用forEach，传入一个函数进去获取并处理数据
  stream.forEach((int x){
    print(x);
  });
```

使用 `listen` 监听
`StreamSubscription<T> listen(void onData(T event), {Function onError, void onDone(), bool cancelOnError})`
```java
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), callback);
  stream.listen((x){
    print(x);
  });
```

还可以使用几个可选的参数
```java
test() async{
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), callback);
  stream = stream.take(5);
  stream.listen(
    (x)=>print(x),
  onError: (e)=>print(e),
  onDone: ()=>print("onDone"));
}
```

- `onError`：发生Error时触发
- `onDone`：完成时触发
- `unsubscribeOnError`：遇到第一个Error时是否取消监听，默认为`false`

### Stream 的一些方法
#### take 和 takeWhile
`Stream<T> take(int count)` 用于限制`Stream`中的元素数量

```java
test() async{
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), callback);
  // 当放入三个元素后，监听会停止，Stream会关闭
  stream = stream.take(3);

  await for(var i in stream){
    print(i);
  }
}
```
打印结果：
```
0
1
2
```
`Stream<T>.takeWhile(bool test(T element))` 与 `take`作用相似，只是它的参数是一个函数类型，且返回值必须是一个`bool`值
```java
  stream = stream.takeWhile((x){
    // 对当前元素进行判断，不满足条件则取消监听
    return x <= 3;
  });

```

#### skip 和 skipWhile

```java
test() async{
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), callback);
  stream = stream.take(5);
  // 表示从Stream中跳过两个元素
  stream = stream.skip(2);

  await for(var i in stream){
    print(i);
  }
}
```

打印结果：
```
2
3
4
```
请注意，该方法只是从`Stream`中获取元素时跳过，被跳过的元素依然是被执行了的，所耗费的时间依然存在，其实只是跳过了执行完的结果而已。

`Stream<T> skipWhile(bool test(T element))` 方法与`takeWhile`用法是相同的，传入一个函数对结果进行判断，表示跳过满足条件的。

#### toList
`Future<List<T>> toList()` 表示将`Stream`中所有数据存储在List中

```java
test() async{
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), callback);
  stream = stream.take(5);
  List <int> data = await stream.toList(); 
  for(var i in data){ 
      print(i);
   } 
}
```

#### 属性 length
等待并获取流中所有数据的数量
```java
test() async{
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), callback);
  stream = stream.take(5);
  var len = await stream.length;
  print(len);
}
```

### StreamController
它实际上就是`Stream`的一个帮助类，可用于整个 `Stream` 过程的控制。

```java
import 'dart:async';

void main() {
  test();
}

test() async{
  // 创建
  StreamController streamController = StreamController();
  // 放入事件
  streamController.add('element_1');
  streamController.addError("this is error");
  streamController.sink.add('element_2');
  streamController.stream.listen(
    print,
  onError: print,
  onDone: ()=>print("onDone"));
}
```

使用该类时，需要导入`'dart:async'`，其`add`方法和`sink.add`方法是相同的，都是用于放入一个元素，`addError`方法用于产生一个错误，监听方法中的`onError`可获取错误。

还可以在`StreamController`中传入一个指定的`stream`
```java
test() async{
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), (e)=>e);
  stream = stream.take(5);

  StreamController sc = StreamController();
  // 将 Stream 传入
  sc.addStream(stream);
  // 监听
  sc.stream.listen(
    print,
  onDone: ()=>print("onDone"));
}
```

现在来看一下`StreamController`的原型，它有5个可选参数
```dart
factory StreamController(
      {void onListen(),
      void onPause(),
      void onResume(),
      onCancel(),
      bool sync: false})
```

- `onListen` 注册监听时回调
- `onPause` 当流暂停时回调
- `onResume` 当流恢复时回调
- `onCancel` 当监听器被取消时回调
- `sync` 当值为`true`时表示同步控制器`SynchronousStreamController`，默认值为`false`，表示异步控制器

```java
test() async{
  // 创建
  StreamController sc = StreamController(
    onListen: ()=>print("onListen"),
    onPause: ()=>print("onPause"),
    onResume: ()=>print("onResume"),
    onCancel: ()=>print("onCancel"),
    sync:false
  );

  StreamSubscription ss = sc.stream.listen(print);

  sc.add('element_1');

  // 暂停
  ss.pause();
  // 恢复
  ss.resume();
  // 取消
  ss.cancel();

  // 关闭流
  sc.close();
}
```
打印结果：
```
onListen
onPause
onCancel
```

因为监听器被取消了，且关闭了流，导致`"element_1"`未被输出，`"onResume"`亦未输出

## 广播流
如下，在普通的单订阅流中调用两次`listen`会报错
```java
test() async{
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), (e)=>e);
  stream = stream.take(5);

  stream.listen(print);
  stream.listen(print);
}
```

```
Unhandled exception:
Bad state: Stream has already been listened to.
```

前面已经说了单订阅流的特点，而广播流则可以允许多个监听器存在，就如同广播一样，凡是监听了广播流，每个监听器都能获取到数据。要注意，如果在触发事件时将监听者正添加到广播流，则该监听器将不会接收当前正在触发的事件。如果取消监听，监听者会立即停止接收事件。

有两种方式创建广播流，一种直接从`Stream`创建，另一种使用`StreamController`创建
```java
test() async{
  // 调用 Stream 的 asBroadcastStream 方法创建
  Stream<int> stream = Stream<int>.periodic(Duration(seconds: 1), (e)=>e)
  .asBroadcastStream();
  stream = stream.take(5);

  stream.listen(print);
  stream.listen(print);
}
```

使用`StreamController`
```java
test() async{
  // 创建广播流
  StreamController sc = StreamController.broadcast();

  sc.stream.listen(print);
  sc.stream.listen(print);

  sc.add("event1");
  sc.add("event2");
}
```

## StreamTransformer 
该类可以使我们在`Stream`上执行数据转换。然后，这些转换被推回到流中，以便该流注册的所有监听器可以接收

**构造方法原型**
```dart
factory StreamTransformer.fromHandlers({
      void handleData(S data, EventSink<T> sink),
      void handleError(Object error, StackTrace stackTrace, EventSink<T> sink),
      void handleDone(EventSink<T> sink)
})
```

 - `handleData`：响应从流中发出的任何数据事件。提供的参数是来自发出事件的数据，以及`EventSink<T>`，表示正在进行此转换的当前流的实例
- `handleError`：响应从流中发出的任何错误事件
- `handleDone`：当流不再有数据要处理时调用。通常在流的`close()`方法被调用时回调

```java
void test() {
  StreamController sc = StreamController<int>();
  
  // 创建 StreamTransformer对象
  StreamTransformer stf = StreamTransformer<int, double>.fromHandlers(
    handleData: (int data, EventSink sink) {
      // 操作数据后，转换为 double 类型
      sink.add((data * 2).toDouble());
    }, 
    handleError: (error, stacktrace, sink) {
      sink.addError('wrong: $error');
    }, 
    handleDone: (sink) {
      sink.close();
    },
  );
  
  // 调用流的transform方法，传入转换对象
  Stream stream = sc.stream.transform(stf);

  stream.listen(print);

  // 添加数据，这里的类型是int
  sc.add(1);
  sc.add(2); 
  sc.add(3); 
  
  // 调用后，触发handleDone回调
  // sc.close();
}
```

打印结果：
```
2.0
4.0
6.0
```

## 总结
与流相关的操作，主要有四个类
- `Stream`
- `StreamController`
- `StreamSink`
- `StreamSubscription`

`Stream`是基础，为了更方便控制和管理`Stream`，出现了`StreamController`类。在`StreamController`类中， 提供了`StreamSink` 作为事件输入口，当我们调用`add`时，实际上是调用的`sink.add`，通过`sink`属性可以获取`StreamController`类中的`StreamSink` ，而`StreamSubscription`类则用于管理事件的注册、暂停与取消等，通过调用`stream.listen`方法返回一个`StreamSubscription`对象。



# 视频课程

或关注博主视频课程，[**Flutter全栈式开发**](http://m.study.163.com/provider/480000001855430/index.htm?share=2&shareId=480000001855430)

![qr_adv](https://img-blog.csdnimg.cn/img_convert/eb3c16913c155e08e1443a0029003aa1.png)

------

**关注公众号：编程之路从0到1**

![编程之路从0到1](https://img-blog.csdnimg.cn/20190301102949549.jpg)
