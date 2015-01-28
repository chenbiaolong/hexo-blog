title: google Protocol Buffer 入门 
date: 2015-01-22 14:35:11
category: 第三方软件
tags: protobuf
---

###简介
Google Protocol Buffer( 简称 Protobuf) 是 Google 公司内部的混合语言数据标准，目前已经正在使用的有超过 48,162 种报文格式定义和超过 12,183 个 .proto 文件。他们用于 RPC 系统和持续数据存储系统。
Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。目前提供了 C++、Java、Python 三种语言的 API。

<!-- more -->

###入门
下载安装protobuf后编写hello.proto文件
```
package lm;
message helloworld
{
	required int32 id=1;
	required string str=2;
	optional int32 opt=3;
}
```
.proto文件定义需要进行序列化和反序列化数据的格式。具体参考[官方文档](https://developers.google.com/protocol-buffers/docs/proto)。其中的required，optional的官方说明如下

> ###Specifying Field Rules
You specify that message fields are one of the following:
>###required: 
a well-formed message must have exactly one of this field.
>###optional:
a well-formed message can have zero or one of this field (but not more than one).
>###repeated: 
this field can be repeated any number of times (including zero) in a well-formed message. The order of the repeated values will be preserved.

定义中的
```
required int32 id=1;
```
1是id在message helloword中的数字标签，这个标签在一个message中是唯一的，一旦定义后在使用时就不能再改变。
>each field in the message definition has a unique numbered tag. These tags are used to identify your fields in the message binary format, and should not be changed once your message type is in use.

###编译 .proto 文件
写好 proto 文件之后就可以用 Protobuf 编译器将该文件编译成目标语言了。本例中我们将使用 C++。
假设proto 文件存放在 SRC_DIR 下面，生成的文件放在DST_DIR目录下，则可以使用如下命令：
```
 protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/hello.proto
```

###编写writer.cpp
```cpp
#include "hello.pb.h"
#include <fstream>
#include <iostream>
#include <string>
#include <google/protobuf/text_format.h>
using namespace google::protobuf;
using std::string;
using std::fstream;
using std::cout;
using std::endl;
using std::ios;
int main(int argc, char const *argv[])
{
	lm::helloworld msg1;
	string hellostring;
	string outputstring;
	msg1.set_id(101);
	msg1.set_str("hello");
	fstream output("./log", ios::out | ios::trunc | ios::binary); 
	if (!msg1.SerializeToOstream(&output)) { 
      cout << "Failed to write msg." << endl; 
      return -1; 
  }       
  TextFormat::PrintToString(msg1, &outputstring);
  cout << outputstring << endl;
return 0;
}
```
编译&运行
```shell
[root@localhost protobuf]# g++ hello.pb.cc writer.cpp -lprotobuf -o hello_writer
[root@localhost protobuf]# ./hello_writer 
id: 101
str: "hello"

[root@localhost protobuf]# 
```
当前路径下保存了log文件，reader可以从该文件解析出数据原来的内容。
###编写reader.cpp
```cpp
#include "hello.pb.h"
#include <fstream>
#include <iostream>
using std::fstream;
using std::cout;
using std::endl;
using std::ios;

void ListMsg(const lm::helloworld & msg) { 
  cout << msg.id() << endl; 
  cout << msg.str() << endl; 
 } 
 int main(int argc, char* argv[]) { 
    lm::helloworld msg1; 
    fstream input("./log", ios::in | ios::binary); 
    if (!msg1.ParseFromIstream(&input)) { 
      cout << "Failed to parse address book." << endl; 
      return -1; 
    } 
  ListMsg(msg1); 
 }
```
编译&运行
```shell
[root@localhost protobuf]# g++ hello.pb.cc reader.cpp -lprotobuf -o hello_reader
[root@localhost protobuf]# ./hello_reader 
101
hello
[root@localhost protobuf]# 
```
可见reader成功从log文件解析出message。