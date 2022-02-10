# ~~Dart 调用C语言~~ 
本篇博客研究Dart语言如何调用C语言代码混合编程，最后我们实现一个简单示例，在C语言中编写简单加解密函数，使用dart调用并传入字符串，返回加密结果，调用解密函数，恢复字符串内容。

**随着Dart SDK版本迭代，本文章部分内容已过时，最新版本教程已经上传B站，请查看 [Dart FFI 入门](https://www.bilibili.com/video/BV1v44y1i7ed)**

## 环境准备
### 编译器环境
如未安装过VS编译器，则推荐使用GCC编译器，下载一个64位Windows版本的GCC——`MinGW-W64`
[下载地址](https://sourceforge.net/projects/mingw-w64/files/)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190517193751536.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9hcmN0aWNmb3guYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70)
如上，它有两个版本，`sjlj`和`seh`后缀表示异常处理模式，`seh` 性能较好，但不支持 32位。 `sjlj` 稳定性好，可支持 32位，推荐下载`seh` 版本

将编译器安装到指定的目录，完成安装后，还需要配置一下环境变量，将安装目录下的`bin`目录加入到系统Path环境变量中，`bin`目录下包含`gcc.exe`、`make.exe`等工具链。

**测试环境**
配置完成后，检测一下环境是否搭建成功，打开`cmd`命令行，输入`gcc -v`能查看版本号则成功。

### Dart SDK环境
去往Dart 官网下载最新的2.3 版本SDK，注意，旧版本不支持`ffi` [下载地址](https://dart.dev/tools/sdk/archive)

下载安装后，同样需要配置环境变量，将`dart-sdk\bin`配置到系统Path环境变量中。


## 测试Dart `ffi`接口
关于C语言相关的各种知识，包括构建、动态库编译与加载，请学习我的[《C语言专栏》](https://blog.csdn.net/yingshukun/category_9291402.html)，只有掌握这些基础知识，才能应对各种报错问题。
### 简单示例
创建测试工程，打开`cmd`命令行
```
mkdir ffi-proj
cd ffi-proj
mkdir bin src
```

创建工程目录`ffi-proj`，在其下创建`bin `、`src`文件夹，在`bin`中创建`main.dart`文件，在`src`中创建`test.c`文件

编写`test.c`
我们在其中包含了windows头文件，用于`showBox`函数，调用Win32 API，创建一个对话框
```c
#include<windows.h>

int add(int a, int b){
    return a + b;
}


void showBox(){
    MessageBox(NULL,"Hello Dart","Title",MB_OK);
}
```

进入`src`目录下，使用gcc编译器，将C语言代码编译为dll动态库
```shell
gcc test.c -shared -o test.dll
```


编写`main.dart`

```dart
import  'dart:ffi'  as ffi;
import  'dart:io'  show Platform;

/// 根据C中的函数来定义方法签名（所谓方法签名，就是对一个方法或函数的描述，包括返回值类型，形参类型）
/// 这里需要定义两个方法签名，一个是C语言中的，一个是转换为Dart之后的
typedef NativeAddSign = ffi.Int32 Function(ffi.Int32,ffi.Int32);
typedef DartAddSign = int Function(int, int);

/// showBox函数方法签名
typedef NativeShowSign = ffi.Void Function();
typedef DartShowSign = void Function();

void main(List<String> args) {
  if (Platform.isWindows) {
    // 加载dll动态库
    ffi.DynamicLibrary dl = ffi.DynamicLibrary.open("../src/test.dll");

    // lookupFunction有两个作用，1、去动态库中查找指定的函数；2、将Native类型的C函数转化为Dart的Function类型
    var add = dl.lookupFunction<NativeAddSign, DartAddSign>("add");
    var showBox = dl.lookupFunction<NativeShowSign, DartShowSign>("showBox");

    // 调用add函数
    print(add(8, 9));
	// 调用showBox函数
    showBox();
  }
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190517200625101.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9hcmN0aWNmb3guYmxvZy5jc2RuLm5ldA==,size_16,color_FFFFFF,t_70)

### 深入用法
这里写一个稍微深入一点的示例，我们在C语言中写一个简单加密算法，然后使用dart调用C函数加密解密

编写`encrypt_test.c`，这里写一个最简单的异或加密算法，可以看到加密和解密实际上是一样的
```c
#include <string.h>
 
#define KEY 'abc'
 
void encrypt(char *str, char *r, int r_len){
    int len = strlen(str);
    for(int i = 0; i < len && i < r_len; i++){
        r[i] = str[i] ^ KEY;
    }

    if (r_len > len) r[len] = '\0';
    else r[r_len] = '\0';
    
}

void decrypt(char *str, char *r, int r_len){
    int len = strlen(str);
    for(int i = 0; i < len && i < r_len; i++){
        r[i] = str[i] ^ KEY;
    }

    if (r_len > len) r[len] = '\0';
    else r[r_len] = '\0';
}
```

编译为动态库
```
gcc encrypt_test.c -shared -o encrypt_test.dll
```

编写`main.dart`

```dart
import 'dart:ffi';
import 'dart:io' show Platform;
import "dart:convert";


/// encrypt函数方法签名，注意，这里encrypt和decrypt的方法签名实际上是一样的，两个函数返回值类型和参数类型完全相同
typedef NativeEncrypt = Void Function(CString,CString,Int32);
typedef DartEncrypt = void Function(CString,CString,int);


void main(List<String> args) {
  if (Platform.isWindows) {
    // 加载dll动态库
    DynamicLibrary dl = DynamicLibrary.open("../src/encrypt_test.dll");
    var encrypt = dl.lookupFunction<NativeEncrypt, DartEncrypt>("encrypt");
    var decrypt = dl.lookupFunction<NativeEncrypt, DartEncrypt>("decrypt");


    CString data = CString.allocate("helloworld");
    CString enResult = CString.malloc(100);
    encrypt(data,enResult,100);
    print(CString.fromUtf8(enResult));

    print("-------------------------");

    CString deResult = CString.malloc(100);
    decrypt(enResult,deResult,100);
    print(CString.fromUtf8(deResult));
  }
}

/// 创建一个类继承Pointer<Int8>指针，用于处理C语言字符串和Dart字符串的映射
class CString extends Pointer<Int8> {

  /// 申请内存空间，将Dart字符串转为C语言字符串
  factory CString.allocate(String dartStr) {
    List<int> units = Utf8Encoder().convert(dartStr);
    Pointer<Int8> str = allocate(count: units.length + 1);
    for (int i = 0; i < units.length; ++i) {
      str.elementAt(i).store(units[i]);
    }
    str.elementAt(units.length).store(0);

    return str.cast();
  }

 // 申请指定大小的堆内存空间
  factory CString.malloc(int size) {
    Pointer<Int8> str = allocate(count: size);
    return str.cast();
  }

  /// 将C语言中的字符串转为Dart中的字符串
  static String fromUtf8(CString str) {
    if (str == null) return null;
    int len = 0;
    while (str.elementAt(++len).load<int>() != 0);
    List<int> units = List(len);
    for (int i = 0; i < len; ++i) units[i] = str.elementAt(i).load();
    return Utf8Decoder().convert(units);
  }
}
```

运行结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190517204254208.jpg)
可以看到将`"helloworld"`字符串加密后变成一串乱码，解密字符串后，恢复内容


#### 完善代码
上述代码虽然实现了我们的目标，但是存在明显的内存泄露，我们使用CString 的`allocate`和`malloc`申请了堆内存，但是却没有手动释放，这样运行一段时间后可能会耗尽内存空间，手动管理内存往往是`C/C++`中最容易出问题的地方，这里我们只能进行一个简单的设计来回收内存
```dart
/// 创建Reference 类来跟踪CString申请的内存
class Reference {
   final List<Pointer<Void>> _allocations = [];

    T ref<T extends Pointer>(T ptr) {
     _allocations.add(ptr.cast());
     return ptr;
   }

	// 使用完后手动释放内存
    void finalize() {
	    for (final ptr in _allocations) {
	      ptr.free();
	    }
	    _allocations.clear();
  }
}
```

修改代码

```dart
import 'dart:ffi';
import 'dart:io' show Platform;
import "dart:convert";


/// encrypt函数方法签名，注意，这里encrypt和decrypt的方法签名实际上是一样的，两个函数返回值类型和参数类型完全相同
typedef NativeEncrypt = Void Function(CString,CString,Int32);
typedef DartEncrypt = void Function(CString,CString,int);


void main(List<String> args) {
  if (Platform.isWindows) {
    // 加载dll动态库
    DynamicLibrary dl = DynamicLibrary.open("../src/hello.dll");
    var encrypt = dl.lookupFunction<NativeEncrypt, DartEncrypt>("encrypt");
    var decrypt = dl.lookupFunction<NativeEncrypt, DartEncrypt>("decrypt");

	// 创建Reference 跟踪CString
    Reference ref = Reference();

    CString data = CString.allocate("helloworld",ref);
    CString enResult = CString.malloc(100,ref);
    encrypt(data,enResult,100);
    print(CString.fromUtf8(enResult));

    print("-------------------------");

    CString deResult = CString.malloc(100,ref);
    decrypt(enResult,deResult,100);
    print(CString.fromUtf8(deResult));

	// 用完后手动释放
    ref.finalize();
  }
}

class CString extends Pointer<Int8> {

  /// 开辟内存控件，将Dart字符串转为C语言字符串
  factory CString.allocate(String dartStr, [Reference ref]) {
    List<int> units = Utf8Encoder().convert(dartStr);
    Pointer<Int8> str = allocate(count: units.length + 1);
    for (int i = 0; i < units.length; ++i) {
      str.elementAt(i).store(units[i]);
    }
    str.elementAt(units.length).store(0);

    ref?.ref(str);
    return str.cast();
  }

  factory CString.malloc(int size, [Reference ref]) {
    Pointer<Int8> str = allocate(count: size);
    ref?.ref(str);
    return str.cast();
  }

  /// 将C语言中的字符串转为Dart中的字符串
  static String fromUtf8(CString str) {
    if (str == null) return null;
    int len = 0;
    while (str.elementAt(++len).load<int>() != 0);
    List<int> units = List(len);
    for (int i = 0; i < len; ++i) units[i] = str.elementAt(i).load();
    return Utf8Decoder().convert(units);
  }
}
```


## 总结
`dart:ffi`包目前正处理开发中，暂时释放的只有基础功能，且使用`dart:ffi`包后，Dart代码不能进行`aot`编译，不过Dart开发了`ffi`接口后，极大的扩展了dart语言的能力边界，就如同的Java的Jni一样，如果`ffi`接口开发得足够好用，Dart就能像Python那样成为一门真正的胶水语言。

大家如果有兴趣进一步研究，可以查看`dart:ffi`包源码，目前该包总共才5个dart文件，源码很少，适合学习。

参考资料：
[dart:ffi 源码](https://github.com/dart-lang/sdk/tree/master/sdk/lib/ffi)
[dart:ffi 官方示例](https://github.com/dart-lang/sdk/blob/master/samples/ffi/sqlite/docs/sqlite-tutorial.md)

## 视频课程
如需要获取完整的**Flutter全栈式开发课程**，请 [**点击跳转**](http://m.study.163.com/provider/480000001855430/index.htm?share=2&shareId=480000001855430)

![qr_adv](https://img-blog.csdnimg.cn/img_convert/eb3c16913c155e08e1443a0029003aa1.png)


**欢迎关注我的公众号：编程之路从0到1**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190301102949549.jpg)
