# 可行性报告

## 小组成员
闫泽轩 李牧龙 罗浩铭 汤皓宇 徐航宇

## 引言

## 理论依据
### 源码基础上对spark性能瓶颈的rust重写
基于spark源码的改进是其中一种方案。在源代码上进行修改，特别是用rust重写其中的性能瓶颈模块，特别是内存密集型、CPU密集型的部分，同时保留源码的非关键部分，特别是spark中原生的丰富API，由此达到以小范围的修改达到较为显著的性能提升的效果。
在Spark的Scala源码与rust代码间的交互是这一方案中需要特别关注的点，主要是由于scala基于JVM，而rust基于机器语言，其间交互较为复杂。我们将使用Scala的JNI(Java Native Interface)与Rust的FFI，可能还会借助中介语言的接口，来实现两种语言间的交互。

### 基于不完善的rust版spark开源项目vega的实现
由于rust语言的诸多优点，用rust重写spark是一个非常有诱惑力的方案。此前，就已经有一个较为粗浅的基于rust的spark项目：vega（[Github仓库](https://github.com/rajasekarv/vega)）。这一项目完全使用rust从零写起，构建完成了一个较为简单的spark内核。不过，这一项目已经有两三年没有维护，项目里还有不少算法没有实现，特别是Spark后来的诸多优化更新，这些都是我们的改进空间。


## 技术依据
### JNI交互
Scala是在JVM上运行的语言，和Java比较相似。与其他语言交互时，主要有JNI(Java Native Interface), JNA(Java Native Access), OpenJDK project Panama三种方式。其中最常用的即为JNI接口。借由JNI，Scala可以与Java代码无缝衔接，而Java可以与C也通过JNI来交互。而Rust可通过二进制接口的方式与其他语言进行交互，特别是可以通过Rust的extern语法，十分方便地与C语言代码交互，按照C的方式调用JNI。这一套机制的组合之下，Scala和Rust的各类交互得到了保障。
同时，正如我们通常不会直接在Rust中通过二进制接口调用C的标准库函数，而是使用libc crate一样，直接使用JNI对C的接口会使得编程较为繁琐且不够安全，代码中的大量unsafe块使得程序稳定性大大下降，所以，我们将选择对JNI进行了安全的封装的接口：**jni[^1] crate**。
#### Rust调用Scala
**数据交互**
两种语言在进行交互时，必须使用两边共有的数据类型。
对于基础的参数类型，可以直接用`jni::sys::*`模块提供的系列类型来声明，对照表如下：
| Scala 类型 | Native 类型 | 类型描述         |
| ---------- | ----------- | ---------------- |
| boolean    | jboolean    | unsigned 8 bits  |
| byte       | jbyte       | signed 8 bits    |
| char       | jchar       | unsigned 16 bits |
| short      | jshort      | signed 16 bits   |
| int        | jint        | signed 32 bits   |
| long       | jlong       | signed 64 bits   |
| float      | jfloat      | 32 bits          |
| double     | jdouble     | 64 bits          |
| void       | void        | not applicable   |
对于复合类型，如对象等，则可以统一用`jni::objects::JObject`类型声明。该类型封装了由JVM返回的对象指针，并为该指针赋予了生命周期，以保证在Rust代码中的安全性。
**方法交互**
由于语言间对对象及其特性的实现不同，很难直接调用对方语言中的函数或方法。于是通常需要使用server-client模型，将执行函数或方法的任务交给sever语言，即：client传递所需的数据参数，并由server执行计算任务，并将最终结果返回给client。
基于这种模型的设计，jni提供了调用scala中函数、对象方法以及获取对象数据域的方法。它们定义于`jni::JNIEnv`中，如接受对象、方法名和方法的签名与参数的`jni::JNIEnv::call_method`，接受对象、成员名、类型的`jni::JNIEnv::get_field`等
此外，jni额外实现了一个`jni::objects::JString`接口，用以方便地实现字符串的传输。
#### Scala调用Rust
Rust可以通过`pub unsafe extern "C" fn{}`来创建导出函数，或通过jni封装的函数`JNIEnv::register_native_methods`动态注册native方法。
导出函数会通过函数名为Scala的对应类提供一个native的静态方法。
动态注册会在JNI_Onload这个导出函数里执行，jvm加载jni动态库时会执行这个函数，从而加载注册的函数。
在Rust中定义这些函数时，同样需要遵循上面的那些交互方法和规范。
### Cap'n Proto[^1]
Cap'n Proto是一种速度极快的数据交换格式，以及能力强大的RPC系统.
![capnp](./src/Capnp.png)
#### 优势:
1. 递增读取:可以在整个Cap'n Proto 信息完全传递之前进行处理，这是因为外部对象完全出现在内部对象以前。
2. 随机访问:你可以仅读取一条信息的一个字段而无需将整个对象翻译。
3. MMAP:通过memory-mapping读取一个巨型文件，OS甚至不会读取你未访问的部分。
4. 跨语言通讯:避免了从脚本语言调用C++代码的痛苦。使用Cap'n Proto可以让多种语言轻松地在同一个内存中的数据结构上进行操作。
5. 通过共享内存可以让在同一台机器上多线程共同访问。不需要通过内核来产生管道来传输数据。
6. Arena Allocation:Cap'n Proto对象始终以"arena"或者"region"风格进行分配，因此更快并且提升了缓存局部性。
7. 微小生成代码:Protobuf为每个消息类型产生专一的解析和序列化代码，这种代码会变得庞大无比。
### 优化RDD依赖关系/调度


## 创新点


## 概要设计报告


## 进度管理









