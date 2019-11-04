---
title: grpc与graphql最佳实践
tags:
  - grpc
  - graphql
categories:
  - java
img: /gallery/thumbnails/graphql_grpc.png
date: 2019-10-24 16:28:29
---

{% img /gallery/thumbnails/graphql_grpc.png graphql_grpc %}
本文分为两部分，一部分翻译自[effective-grpc](https://john-millikin.com/effective-grpc)，另一部分来自于集成grpc graphql实践。文章内容都是经过实践，请放心食用！
<!--more-->
# Effective grpc
## grpc
### 错误处理
使用`google.protobuf.Status`消息将错误报告给客户端-gRPC库应针对您的语言对这种类型进行特殊区分（例如，grpc-go具有[google.golang.org/grpc/status](https://godoc.org/google.golang.org/grpc/status))。该消息可以包含任意子消息，因此服务器可以向所有客户端提供基本错误消息，并向可以处理这些错误的客户端提供结构化错误。 有关每个错误代码的含义的详细信息，请参考[google/rpc/code.proto](https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto)；有关如何编写错误消息的详细建议，请参考[Google Cloud Error Model](https://cloud.google.com/apis/design/errors)。
>grpc-java中为[io.grpc.Status](https://grpc.github.io/grpc-java/javadoc/index.html?io/grpc/Status.html)
### 超时和截止时间
服务器端处理程序应始终传播截止时间，客户几乎应该总是设定截止时间。优先选择截止时间而不是超时，因为在跨网络边界工作时，绝对时间戳的含义不如相对时间模糊。 根据你的实现库，有可能在service schema中定义默认超时，不要这样做-schema创建者无法预测哪种行为适合所有实现或用户。
### 访问地址
始终遵循[gRPC名称解析](https://github.com/grpc/grpc/blob/master/doc/naming.md)所使用的类似URL的语法，将gRPC地址表示并存储为完整字符串。诸如`IP +端口元组`之类的限制性格式会使想要在更大的框架或集成测试中运行您的代码的用户烦恼，这些测试可能对网络地址有自己的想法。 让地址设置在命令行标志或配置文件中，以便用户可以配置它们而不必修补二进制文件。即使您确实非常确定整个世界都希望在端口80上运行服务，也要这样做。
### 流
gRPC支持单向和双向消息流，如果要传输的数据量可能很大，或者在完全接收到输入之前另一端可以有意义地处理数据，请使用流。例如，提供SHA256方法的服务可以在输入块到达时对其进行哈希处理，然后在客户端关闭请求流时将最终摘要发送回去。 与为每个块发送单独的RPC相比，流传输效率更高，但比所有块都位于重复字段中的单个RPC效率要低。流的开销可以通过使用批处理消息类型来最小化。
```proto
service Foo {
    rpc MyStream(FooRequest) returns (stream MyStreamItem);
}

message MyStreamItem {
    repeated MyStreamValue values = 1;
}
message MyStreamValue {
    // ... fields for each logical value
}
```
>WARNING：在某些实现中（例如grpc-go），流句柄不是线程安全的，即使是客户端存根也不是。与来自多个线程的流句柄进行交互可能会导致不可预测的行为，包括静默消息损坏。
### 请求/响应类型
在你的`service`中每个方法都应该有自己独有的请求和响应消息
```proto
service Foo {
    rpc Bar(BarRequest) returns (BarResponse);
}

message BarRequest  { ... }
message BarResponse { ... }
```
请不要在不同的方法中使用相同的消息，除非他们实际上是使用不同的API实现相同的方法(例如一元和流变种接收同样的响应)。即使这种情况，对于API也有可能有不同的部分，此时请新建另一个类型。
```proto
service Foo {
    rpc Bar(BarRequest) returns (BarResponse);
    rpc BarStream(BarRequest) returns (stream BarResponseStreamItem);
}

message BarRequest  { ... }
message BarResponse { ... }

message BarResponseStreamItem { ... }
```
WARNING: 请不要使用`google.protobuf.Empty`作为请求和响应类型，他的API文档中的描述[google/protobuf/empty.proto](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/empty.proto)是一种反模式。如果你使用`Empty`， 那么将字段添加到请求/响应中将使所有客户端和服务器的API发生重大变化。
## Protobuf
### 包名
使用软件包名称、项目名称、公司（如果适用）和[语义版本](https://semver.org/)控制主版本。确切的格式取决于个人喜好-流行的格式包括Java中使用的[反向域名表示法](https://en.wikipedia.org/wiki/Reverse_domain_name_notation)，或核心gRPC类型使用的`$COMPANY`.`$PROJECT`
- `om.mycompany.my_project.v1`
- `com.mycompany.MyProject.v1`
- `mycompany.my_project.v1`
尚未完全稳定的API版本应具有v1alpha，v2beta1或v3test之类的后缀。有关更详尽的指导，请参考K[ubernetes API版本控制策略](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/empty.proto)。<br>

Protobuf软件包名称用于生成的代码中，因此请避免使用内置类型或关键字（例如return或void)等常用的名称。这对于生成C ++尤为重要，因为C ++（从protobuf 3.6起）没有`FileOption`来覆盖默认的`namespace`计算名称。
### import path
尝试从原始文件的位置构建，以使导入路径与程序包名称匹配：`mycompany.my_project.v1`中的类型应与`importmycompany/my_project/v1/some_file.proto`一起导入。 Protobuf工具链不是必需的，但是确实可以帮助我们记住输入的内容。

### Next-Number 注释
在大型protobuf消息中，弄清楚应该为新字段使用哪个字段编号可能会很烦人。为了简化将来的编辑者的工作，请在消息和枚举的末尾添加注释
```proto
message MyMessage {
    // ... lots of fields here ...

    // NEXT: 42
}
```
### 枚举
枚举作用域遵循旧式C/C ++规则，因此定义的名称不限于枚举名称：
```
// symbol `FUN_LEVEL_HIGH' is of type `FunLevel'.
enum FunLevel {
    FUN_LEVEL_UNKNOWN = 0;
    FUN_LEVEL_LOW = 1;
    FUN_LEVEL_HIGH = 2;
    // NEXT: 3
}
```
> 枚举值不能重复，即使是在不同枚举类型中，你可能觉得这样很变态，但是事实就是这样的-_-
对于习惯于具有更现代范围规则的语言的用户而言，这可能很别扭。我喜欢用消息包装枚举：
```
// symbol `FunLevel::HIGH` is of type `FunLevel::Enum`.
message FunLevel {
    enum Enum {
        UNKNOWN = 0;
        LOW = 1;
        HIGH = 2;
        // NEXT: 3
    }
}
```
### 墓碑
如果某个字段已被删除，则其字段编号不得再被将来的字段添加重用。通过添加带有逻辑标记的墓碑[保留字](https://developers.google.com/protocol-buffers/docs/proto3#reserved)来防止意外的字段号重用。我总是保留字段名称和编号:
```proto
enum FunLevel {
    // removed -- too much fun
    reserved "FUN_LEVEL_EXCESSIVE"; reserved 10;
}

message MyMessage {
    reserved "crufty_old_field"; reserved 20;
}
```
## 文档
Protobuf没有用于API文档的内置生成器。在可用选项中，[protoc-gen-doc](https://github.com/pseudomuto/protoc-gen-doc)似乎最成熟，请查看在[protoc-gen-doc README](https://github.com/pseudomuto/protoc-gen-doc/blob/master/README.md)中的语法和例子
## 参数验证
Protobuf除了proto2（已在proto3中删除）所要求的之外，没有内置的验证机制。 envoyproxy的[protoc-gen-validate](https://github.com/envoyproxy/protoc-gen-validate)工具是我所知道的最好的解决方案
### 可选的自定义类型
在proto3中，删除了将标量字段（int32，字符串等）标记为可选的功能。标量字段现在始终存在，如果没有其他设置，将为默认的`零值`。当为其中`""`和`NULL`是逻辑上不同的值的系统设计架构时，这可能令人沮丧。 官方解决方法是一组在`Google/protobuf/wrappers.proto`中定义的“包装器类型”，用于定义单值消息。你的架构可以使用`google.protobuf.Int32Value`而不是`int32`来获得可选性。
```proto
import "google/protobuf/wrappers.proto";

message MyMessage {
    .google.protobuf.Int32Value some_field = 1;
}
```
另一种方法是将标量字段包装为[oneof](https://developers.google.com/protocol-buffers/docs/proto3#oneof)，而没有其他选择.并在生成的代码中添加辅助方法以检测是否设置了该字段,来迫使标量字段具有可选性。
```
message MyMessage {
    oneof oneof_some_field {
        int32 some_field = 1;
    }
}
```
# graphql与grpc集成
[grapql](https://graphql.cn/)是一种用于查询的api语言，跟protobuf一样同样的是使用schema描述，类型系统都是自己独有的。在与graphql和grpc集成过程最大问题就是类型问题。具体请查看[graphql和grpc集成](https://silencecorner.github.io/2019/08/18/graphql-grpc-in-java-world-1/)