

# 标准输入输出流
- `stdin`
- `stdout`
- `stderr`


```java
// 导入io包
import 'dart:io';

void main() {
  // 向标准输出流写字符串
  stdout.write('root\$:');
  // 从标准输入流读取一行字符串
  var input = stdin.readLineSync();
  // 带换行符的写数据
  stdout.writeln("input data:$input");
  // 向标准错误流写数据
  stderr.writeln("has not error");
}
```

`stdin`除了可以使用`readLineSync`读一行字符串，还可以使用`readByteSync`读取一个字节。

# 文件操作
## 写文件
一种简便的操作方式，无需手动关闭文件，文件写入完成后会自动关闭
```java
import 'dart:io';

void main() async{
  // 创建文件
  File file = new File('test.txt');
  String content = 'The easiest way to write text to a file is to create a File';

  try {
    // 向文件写入字符串
    await file.writeAsString(content);
    print('Data written.');
  } catch (e) {
    print(e);
  }
}
```

`writeAsString`原型
```dart
  Future<File> writeAsString(String contents,
      {FileMode mode: FileMode.write,
      Encoding encoding: utf8,
      bool flush: false})
```

- `mode` 文件模式，这里默认为写模式
- `encoding` 字符编码，默认为utf-8
- `flush` 是否立刻刷新缓存，默认为false

文件模式`FileMode`的常量
|常量值| 说明  |
|:--|:--|
| read | 只读模式 |
| write | 可读可写模式，如果文件存在则会覆盖 |
| append | 追加模式，可读可写，文件存在则往末尾追加 |
| writeOnly | 只写模式  |
| writeOnlyAppend | 只写模式下的追加模式，不可读 |




除了`writeAsString`方法外，还可以使用`writeAsBytes`写入一个字节列表。需要注意的是，这两个方法都是异步执行的，返回值都是`Future`，如果有必要，也可以使用同步方法执行写入操作

- `writeAsStringSync`
- `writeAsBytesSync`

如需要更灵活的控制，可以使用如下方式操作文件，但是需要手动关闭文件

```java
import 'dart:io';

void main() async{
  // 创建文件
  File file = new File('test.txt');
  // 文件模式设置为追加
  IOSink isk = file.openWrite(mode: FileMode.append);

  // 多次写入
  isk.write('A woman is like a tea bag');
  isk.writeln('you never know how strong it is until it\'s in hot water.');
  isk.writeln('-Eleanor Roosevelt');
  await isk.close();
  print('Done!');
}
```


## 读文件
简便的方式
- `readAsBytes`
- `readAsBytesSync`
- `readAsString`
- `readAsStringSync`
- `readAsLines`
- `readAsLinesSync`

```java
void main() async{
  File file = new File('test.txt');
  try{
    String content = await file.readAsString();
    print(content);
  }catch(e){
    print(e);
  }
}
```

另一种更低级别的方式
```java
import 'dart:io';
import 'dart:convert';

void main() async{
  try {
    // LineSplitter Dart语言封装的换行符，此处将文本按行分割
    Stream lines = new File('test.txt').openRead()
    	.transform(utf8.decoder).transform(const LineSplitter());

    await for (var line in lines) {
      print(line);
    }
  } catch (_) {}
}
```

## 文件的其他操作
```java
import 'dart:io';

void main() async{
  File file = new File('test.txt');

  // 判断文件是否存在
  if(await file.exists()){
    print("文件存在");
  }else{
    print("文件不存在");
  }

  // 复制文件
  await file.copy("test-1.txt");

  // 修改文件名。当传入不同路径时，可用来移动文件
  await file.rename("test-2.txt");
  
  // 获取文件 size
  print(await file.length());
}
```
相应的，这些方法还有一个带`Sync`后缀的同步版本方法，例如`copySync`、`renameSync`等。

要获取文件更多的信息，还可以使用`File`等多个类的超类`FileSystemEntity`来操作

```java
import 'dart:io';

void main() async{
  String path = 'test.txt';

  // 判断路径是否是文件夹
  if (!await FileSystemEntity.isDirectory(path)) {
    print('$path is not a directory');
  } 

 Directory dir = Directory(r'D:\workspace\dart_space\Tutorial');
 // 目录是否存在
 if(await dir.exists()){
   // 从目录的list方法获取FileSystemEntity对象
   Stream<FileSystemEntity> fse = await dir.list();
   await for (FileSystemEntity entity in fse) {
     if(entity is File){
       print("entity is file");
     }

     // 打印文件信息
     print(await entity.stat());
     // 删除
     await entity.delete();
   }
 }else{
   // 不存在则创建。recursive为true时，创建路径上所有不存在的目录
   await dir.create(recursive: true);
 }
}
```

需注意，`delete`中包含一个可选的参数，原型`  Future<FileSystemEntity> delete({bool recursive: false})`，`recursive`默认为false，当删除目录时，目录必须为空才能删除；当`recursive`设置为true时，将删除目录下的所有子目录及文件。


------

**关注公众号：编程之路从0到1**

![编程之路从0到1](https://img-blog.csdnimg.cn/20190301102949549.jpg)

# 视频课程
或关注博主视频，[**Flutter全栈式开发**](http://m.study.163.com/provider/480000001855430/index.htm?share=2&shareId=480000001855430)
