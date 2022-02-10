
# Dart 异步编程
编程中的代码执行，通常分为`同步`与`异步`两种。简单说，同步就是按照代码的编写顺序，从上到下依次执行，这也是最简单的我们最常接触的一种形式。但是同步代码的缺点也显而易见，如果其中某一行或几行代码非常耗时，那么就会阻塞，使得后面的代码不能被立刻执行。

异步的出现正是为了解决这种问题，它可以使某部分耗时代码不在当前这条执行线路上立刻执行，那究竟怎么执行呢？最常见的一种方案是使用多线程，也就相当于开辟另一条执行线，然后让耗时代码在另一条执行线上运行，这样两条执行线并列，耗时代码自然也就不能阻塞主执行线上的代码了。

多线程虽然好用，但是在大量并发时，仍然存在两个较大的缺陷，一个是开辟线程比较耗费资源，线程开多了机器吃不消，另一个则是线程的锁问题，多个线程操作共享内存时需要加锁，复杂情况下的锁竞争不仅会降低性能，还可能造成死锁。因此又出现了基于事件的异步模型。简单说就是在某个单线程中存在一个事件循环和一个事件队列，事件循环不断的从事件队列中取出事件来执行，这里的事件就好比是一段代码，每当遇到耗时的事件时，事件循环不会停下来等待结果，它会跳过耗时事件，继续执行其后的事件。当不耗时的事件都完成了，再来查看耗时事件的结果。因此，耗时事件不会阻塞整个事件循环，这让它后面的事件也会有机会得到执行。

我们很容易发现，这种基于事件的异步模型，只适合`I/O`密集型的耗时操作，因为`I/O`耗时操作，往往是把时间浪费在等待对方传送数据或者返回结果，因此这种异步模型往往用于网络服务器并发。如果是计算密集型的操作，则应当尽可能利用处理器的多核，实现并行计算。

![](https://img-blog.csdnimg.cn/20190505100632250.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9hcmN0aWNmb3guYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70)

## Dart 的事件循环

Dart 是事件驱动的体系结构，该结构基于具有单个事件循环和两个队列的单线程执行模型。 Dart虽然提供调用堆栈。 但是它使用事件在生产者和消费者之间传输上下文。 事件循环由单个线程支持，因此根本不需要同步和锁定。

Dart 的两个队列分别是
- `MicroTask queue`  微任务队列

- `Event queue` 事件队列

![](https://img-blog.csdnimg.cn/20190505100705326.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9hcmN0aWNmb3guYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70)


Dart事件循环执行如上图所示

1. 先查看`MicroTask`队列是否为空，不是则先执行`MicroTask`队列
2. 一个`MicroTask`执行完后，检查有没有下一个`MicroTask`，直到`MicroTask`队列为空，才去执行`Event`队列
3. 在`Evnet` 队列取出一个事件处理完后，再次返回第一步，去检查`MicroTask`队列是否为空


我们可以看出，将任务加入到`MicroTask`中可以被尽快执行，但也需要注意，当事件循环在处理`MicroTask`队列时，`Event`队列会被卡住，应用程序无法处理鼠标单击、I/O消息等等事件。

## 调度任务
注意，以下调用的方法，都定义在`dart:async`库中。

**将任务添加到`MicroTask`队列有两种方法**

```dart
import  'dart:async';

void  myTask(){
    print("this is my task");
}

void  main() {
    // 1. 使用 scheduleMicrotask 方法添加
    scheduleMicrotask(myTask);

    // 2. 使用Future对象添加
    new  Future.microtask(myTask);
}
```

**将任务添加到`Event`队列**

```dart
import  'dart:async';

void  myTask(){
    print("this is my task");
}

void  main() {
    new  Future(myTask);
}
```


现在学会了调度任务，赶紧编写代码验证以上的结论

```dart
import  'dart:async';

void  main() {
    print("main start");

    new  Future((){
        print("this is my task");
    });

    new  Future.microtask((){
        print("this is microtask");
    });

    print("main stop");
}
```

运行结果：

```
main start
main stop
this is microtask
this is my task
```

可以看到，代码的运行顺序并不是按照我们的编写顺序来的，将任务添加到队列并不等于立刻执行，它们是异步执行的，当前`main`方法中的代码执行完之后，才会去执行队列中的任务，且`MicroTask`队列运行在`Event`队列之前。



## 延时任务
如需要将任务延伸执行，则可使用`Future.delayed`方法

```dart
new  Future.delayed(new  Duration(seconds:1),(){
    print('task delayed');
});
```

表示在延迟时间到了之后将任务加入到`Event`队列。需要注意的是，这并不是准确的，万一前面有很耗时的任务，那么你的延迟任务不一定能准时运行。

```dart
import  'dart:async';
import  'dart:io';

void  main() {
    print("main start");

    new Future.delayed(new  Duration(seconds:1),(){
        print('task delayed');
    });

    new Future((){
        // 模拟耗时5秒
        sleep(Duration(seconds:5));
        print("5s task");
    });

    print("main stop");
}
```

运行结果：
```
main start
main stop
5s task
task delayed
```

从结果可以看出，`delayed`方法调用在前面，但是它显然并未直接将任务加入`Event`队列，而是需要等待1秒之后才会去将任务加入，但在这1秒之间，后面的`new  Future`代码直接将一个耗时任务加入到了`Event`队列，这就直接导致写在前面的`delayed`任务在1秒后只能被加入到耗时任务之后，只有当前面耗时任务完成后，它才有机会得到执行。这种机制使得延迟任务变得不太可靠，你无法确定延迟任务到底在延迟多久之后被执行。


## Future 详解
Future类是对未来结果的一个代理，它返回的并不是被调用的任务的返回值。

```dart
void  myTask(){
    print("this is my task");
}

void  main() {
    Future fut = new  Future(myTask);
}
```

如上代码，`Future`类实例`fut`并不是函数`myTask`的返回值，它只是代理了`myTask`函数，封装了该任务的执行状态。


### 创建 Future

`Future`的几种创建方法

- `Future()`
- `Future.microtask()`
- `Future.sync()`
- `Future.value()`
- `Future.delayed()`
- `Future.error()`

其中`sync`是同步方法，任务会被立即执行
```dart
import  'dart:async';

void  main() {
    print("main start");

new  Future.sync((){
    print("sync task");
});

new  Future((){
    print("async task");
});

    print("main stop");
}
```

运行结果：
```
main start
sync task
main stop
async task
```

### 注册回调
当`Future`中的任务完成后，我们往往需要一个回调，这个回调立即执行，不会被添加到事件队列。

```dart
import 'dart:async';

void main() {
  print("main start");


  Future fut =new Future.value(18);
  // 使用then注册回调
  fut.then((res){
    print(res);
  });

 // 链式调用，可以跟多个then，注册多个回调
  new Future((){
    print("async task");
  }).then((res){
    print("async task complete");
  }).then((res){
    print("async task after");
  });

  print("main stop");
}
```

运行结果：
```
main start
main stop
18
async task
async task complete
async task after
```

除了`then`方法，还可以使用`catchError`来处理异常，如下

```dart
  new Future((){
    print("async task");
  }).then((res){
    print("async task complete");
  }).catchError((e){
    print(e);
  });
```

还可以使用**静态方法**`wait` 等待多个任务全部完成后回调。

```dart
import 'dart:async';

void main() {
  print("main start");

  Future task1 = new Future((){
    print("task 1");
    return 1;
  });

  Future task2 = new Future((){
    print("task 2");
    return 2;
  });
    
  Future task3 = new Future((){
    print("task 3");
    return 3;
  });

  Future fut = Future.wait([task1, task2, task3]);
  fut.then((responses){
    print(responses);
  });

  print("main stop");
}
```

运行结果：
```
main start
main stop
task 1
task 2
task 3
[1, 2, 3]
```

如上，`wait`返回一个新的`Future`，当添加的所有`Future`完成时，在新的`Future`注册的回调将被执行。


## async 和 await
在Dart1.9中加入了`async`和`await`关键字，有了这两个关键字，我们可以更简洁的编写异步代码，而不需要调用`Future`相关的API

将 `async` 关键字作为方法声明的后缀时，具有如下意义

*  被修饰的方法会将一个 `Future` 对象作为返回值
*  该方法会**同步**执行其中的方法的代码直到**第一个 await 关键字**，然后它暂停该方法其他部分的执行；
*   一旦由 **await** 关键字引用的 **Future** 任务执行完成，**await**的下一行代码将立即执行。


```dart
// 导入io库，调用sleep函数
import 'dart:io';

// 模拟耗时操作，调用sleep函数睡眠2秒
doTask() async{
  await sleep(const Duration(seconds:2));
  return "Ok";
}

// 定义一个函数用于包装
test() async {
  var r = await doTask();
  print(r);
}

void main(){
  print("main start");
  test();
  print("main end");
}
```

运行结果：
```
main start
main end
Ok
```

需要注意，**async** 不是并行执行，它是遵循Dart **事件循环**规则来执行的，它仅仅是一个语法糖，简化`Future API`的使用。


## Isolate
前面已经说过，将非常耗时的任务添加到事件队列后，仍然会拖慢整个事件循环的处理，甚至是阻塞。可见基于事件循环的异步模型仍然是有很大缺点的，这时候我们就需要`Isolate`，这个单词的中文意思是隔离。

简单说，可以把它理解为Dart中的线程。但它又不同于线程，更恰当的说应该是微线程，或者说是协程。它与线程最大的区别就是不能共享内存，因此也不存在锁竞争问题，两个`Isolate`完全是两条独立的执行线，且每个`Isolate`都有自己的**事件循环**，它们之间只能通过发送消息通信，所以它的资源开销低于线程。

从主`Isolate`创建一个新的`Isolate`有两种方法

### spawnUri
`static Future<Isolate> spawnUri()`

`spawnUri`方法有三个必须的参数，第一个是Uri，指定一个新`Isolate`代码文件的路径，第二个是参数列表，类型是`List<String>`，第三个是动态消息。需要注意，用于运行新`Isolate`的代码文件中，必须包含一个main函数，它是新`Isolate`的入口方法，该main函数中的args参数列表，正对应`spawnUri`中的第二个参数。如不需要向新`Isolate`中传参数，该参数可传空`List`


主`Isolate`中的代码：
```dart
import 'dart:isolate'; 


void main() {
  print("main isolate start");
  create_isolate();
  print("main isolate stop");
}

// 创建一个新的 isolate
create_isolate() async{
  ReceivePort rp = new ReceivePort();
  SendPort port1 = rp.sendPort;

  Isolate newIsolate = await Isolate.spawnUri(new Uri(path: "./other_task.dart"), ["hello, isolate", "this is args"], port1);

  SendPort port2;
  rp.listen((message){
    print("main isolate message: $message");
    if (message[0] == 0){
      port2 = message[1];
    }else{
      port2?.send([1,"这条信息是 main isolate 发送的"]);
    }
  });

  // 可以在适当的时候，调用以下方法杀死创建的 isolate
  // newIsolate.kill(priority: Isolate.immediate);
}
```

创建`other_task.dart`文件，编写新`Isolate`的代码

```dart
import 'dart:isolate';
import  'dart:io';


void main(args, SendPort port1) {
  print("isolate_1 start");
  print("isolate_1 args: $args");

  ReceivePort receivePort = new ReceivePort();
  SendPort port2 = receivePort.sendPort;

  receivePort.listen((message){
    print("isolate_1 message: $message");
  });

  // 将当前 isolate 中创建的SendPort发送到主 isolate中用于通信
  port1.send([0, port2]);
  // 模拟耗时5秒
  sleep(Duration(seconds:5));
  port1.send([1, "isolate_1 任务完成"]);

  print("isolate_1 stop");
}
```

运行主`Isolate`的结果：

```dart
main isolate start
main isolate stop
isolate_1 start
isolate_1 args: [hello, isolate, this is args]
main isolate message: [0, SendPort]
isolate_1 stop
main isolate message: [1, isolate_1 任务完成]
isolate_1 message: [1, 这条信息是 main isolate 发送的]
```

![](https://img-blog.csdnimg.cn/20190505111443334.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9hcmN0aWNmb3guYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70)
整个消息通信过程如上图所示，**两个Isolate是通过两对Port对象通信，一对Port分别由用于接收消息的`ReceivePort`对象，和用于发送消息的`SendPort`对象构成。其中`SendPort`对象不用单独创建，它已经包含在`ReceivePort`对象之中。需要注意，一对Port对象只能单向发消息，这就如同一根自来水管，`ReceivePort`和`SendPort`分别位于水管的两头，水流只能从`SendPort`这头流向`ReceivePort`这头。因此，两个`Isolate`之间的消息通信肯定是需要两根这样的水管的，这就需要两对Port对象。**

理解了`Isolate`消息通信的原理，那么在Dart代码中，具体是如何操作的呢？
![](https://img-blog.csdnimg.cn/20190505121830821.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9hcmN0aWNmb3guYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70)
**`ReceivePort`对象通过调用`listen`方法，传入一个函数可用来监听并处理发送来的消息。`SendPort`对象则调用`send()`方法来发送消息。`send`方法传入的参数可以是`null`,` num`, `bool`, `double`,`String`, `List` ,`Map`或者是自定义的类。** 在上例中，我们发送的是包含两个元素的`List`对象，第一个元素是整型，表示消息类型，第二个元素则表示消息内容。

### spawn
`static Future<Isolate> spawn()`

除了使用`spawnUri`，更常用的是使用`spawn`方法来创建新的`Isolate`，我们通常希望将新创建的`Isolate`代码和`main Isolate`代码写在同一个文件，且不希望出现两个main函数，而是将指定的耗时函数运行在新的`Isolate`，这样做有利于代码的组织和代码的复用。`spawn`方法有两个必须的参数，第一个是需要运行在新`Isolate`的耗时函数，第二个是动态消息，该参数通常用于传送主`Isolate`的`SendPort`对象。

`spawn`的用法与`spawnUri`相似，且更为简洁，将上面例子稍作修改如下

```dart
import 'dart:isolate'; 
import  'dart:io';

void main() {
  print("main isolate start");
  create_isolate();
  print("main isolate end");
}

// 创建一个新的 isolate
create_isolate() async{
  ReceivePort rp = new ReceivePort();
  SendPort port1 = rp.sendPort;

  Isolate newIsolate = await Isolate.spawn(doWork, port1);

  SendPort port2;
  rp.listen((message){
    print("main isolate message: $message");
    if (message[0] == 0){
      port2 = message[1];
    }else{
      port2?.send([1,"这条信息是 main isolate 发送的"]);
    }
  });
}

// 处理耗时任务
void doWork(SendPort port1){
  print("new isolate start");
  ReceivePort rp2 = new ReceivePort();
  SendPort port2 = rp2.sendPort;

  rp2.listen((message){
    print("doWork message: $message");
  });

  // 将新isolate中创建的SendPort发送到主isolate中用于通信
  port1.send([0, port2]);
  // 模拟耗时5秒
  sleep(Duration(seconds:5));
  port1.send([1, "doWork 任务完成"]);

  print("new isolate end");
}
```

运行结果：
```
main isolate start
main isolate end
new isolate start
main isolate message: [0, SendPort]
new isolate end
main isolate message: [1, doWork 任务完成]
doWork message: [1, 这条信息是 main isolate 发送的]
```

无论是上面的`spawn`还是`spawnUri`，运行后都会包含两个Isolate，一个是主`Isolate`，一个是新`Isolate`，两个都双向绑定了消息通信的通道，即使新的`Isolate`中的任务完成了，它也不会立刻退出，因此，当使用完自己创建的`Isolate`后，最好调用`newIsolate.kill(priority: Isolate.immediate);`将`Isolate`立即杀死。

### Flutter 中创建Isolate
无论如何，在Dart中创建一个`Isolate`都显得有些繁琐，可惜的是Dart官方并未提供更高级封装。但是，如果想在Flutter中创建`Isolate`，则有更简便的API，这是由`Flutter`官方进一步封装`ReceivePort `而提供的更简洁API。[详细API文档](https://docs.flutter.io/flutter/foundation/compute.html)

使用`compute `函数来创建新的`Isolate`并执行耗时任务

```dart
import 'package:flutter/foundation.dart';
import  'dart:io';

// 创建一个新的Isolate，在其中运行任务doWork
create_new_task() async{
  var str = "New Task";
  var result = await compute(doWork, str);
  print(result);
}


String doWork(String value){
  print("new isolate doWork start");
  // 模拟耗时5秒
  sleep(Duration(seconds:5));

  print("new isolate doWork end");
  return "complete:$value";
}
```

`compute `函数有两个必须的参数，第一个是待执行的函数，这个函数必须是一个顶级函数，不能是类的实例方法，可以是类的静态方法，第二个参数为动态的消息类型，可以是被运行函数的参数。需要注意，使用`compute `应导入`'package:flutter/foundation.dart'`包。

### 使用场景
`Isolate`虽好，但也有合适的使用场景，不建议滥用`Isolate`，应尽可能多的使用Dart中的事件循环机制去处理异步任务，这样才能更好的发挥Dart语言的优势。

那么应该在什么时候使用**Future**，什么时候使用**Isolate**呢？
一个最简单的判断方法是根据某些任务的平均时间来选择：
*   方法执行在几毫秒或十几毫秒左右的，应使用`Future`
*   如果一个任务需要几百毫秒或之上的，则建议创建单独的`Isolate`

除此之外，还有一些可以参考的场景

*   JSON 解码
*   加密
*   图像处理：比如剪裁


参考资料：
[Dart 文档]( https://webdev.dartlang.org/articles/performance/event-loop)
[Isolate 文档](https://api.dartlang.org/stable/2.3.0/dart-isolate/Isolate-class.html)



# 视频课程

或关注博主视频课程，[**Flutter全栈式开发**](http://m.study.163.com/provider/480000001855430/index.htm?share=2&shareId=480000001855430)
![qr_adv](https://img-blog.csdnimg.cn/img_convert/eb3c16913c155e08e1443a0029003aa1.png)

------

**关注公众号：编程之路从0到1**
![编程之路从0到1](https://img-blog.csdnimg.cn/20190301102949549.jpg)
