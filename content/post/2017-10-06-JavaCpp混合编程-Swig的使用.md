---
title: "Java Cpp混合编程 Swig的使用"
date: 2017-10-06T15:46:30+08:00
draft: false
tags: ["Cpp", "Java", "Jni", "Swig"]
categories: ["Cpp", "Java", "Jni", "Swig"]
---

# JavaCpp混合编程 Swig的使用

## 江苏移动服开系统云化遇到的问题

江苏移动`BOSS`系统云化风风火火进行中，目前轮到服务开通子系统了。

服务开通子系统接收前台办理业务请求，或者信控系统触发的停复机工单，生成实际的设备物理指令发往真正的网元设备。`C++`老系统为这些不同厂商不同协议的网元设备实现了不同通信协议的动态库，涉及到几十个大大小小不同通信协议的设备，部分老设备几乎处于联系不到厂商负责人的状态。

云化直接使用`Java`重写这类通信动态库，风险比较大，而且也面临外围厂商联调配合的问题。所以想复用这部分设备通信动态库，首先把动态库代码，从`AIX`迁移到`X86`主机重新编译，这部分不做介绍，主要介绍`Java`侧如何调用`C++`动态库的问题，很容易就会想到写`Jni`来实现`Java`和`Cpp`混合编程，这边我们选择`Swig`自动生成`Jni`代码。

<!--more-->

## Swig
**简介**
下面摘自[SWIG官网](http://www.swig.org/)

> SWIG是个帮助使用C或者C++编写的软件能与其它各种高级编程语言进行嵌入联接的开发工具。SWIG能应用于各种不同类型的语言包括常用脚本编译语言例如Perl, PHP, Python, Tcl, Ruby and PHP。支持语言列表中 也包括非脚本编译语言，例如C#, Common Lisp (CLISP, Allegro CL, CFFI, UFFI), Java, Modula-3, OCAML以及R，甚至是编译器或者汇编的计划应用（Guile, MzScheme, Chicken）。SWIG普遍应用于创建高级语言解析或汇编程序环境，用户接口，作为一种用来测试C/C++或进行原型设计的工具。SWIG还能够导出 XML或Lisp s-expressions格式的解析树。SWIG可以被自由使用，发布，修改用于商业或非商业中。

**安装**

到官网下载最新版`swig`

```shell
wget https://jaist.dl.sourceforge.net/project/swig/swig/swig-3.0.12/swig-3.0.12.tar.gz
```

安装`gcc`、`g++`(已有则跳过此步)

```
yum -y install gcc automake autoconf libtool make
```

安装PCRE

``` shell
wget https://jaist.dl.sourceforge.net/project/pcre/pcre/8.39/pcre-8.39.tar.gz
tar -zxvf pcre-8.39.tar.gz
cd pcre-8.39
./configure
make && make install
```

解压`swig`并且编译

``` shell
tar -zxvf swig-3.0.12.tar.gz
cd swig-3.0.12
./configure
make && make install
```

查看安装是否成功

``` shell
swig -version
```

**来一个Swig的Hello World**

现在就可以写一个`Swig`版的`Hello World`例子，`Java`调用`C++`动态库。程序结构和调用关系大致如下图所示，最左边一列是`C++`程序的常见结构，中间一列是`Swig`在编码时扮演的角色，以及其生成的接口代码调用关系，最右边就是我们的`Java`应用了。

![p2](/img/2017-10-06-p2.png)

**首先编写一个C++动态库**

目录结构

``` shell
tree test
test
├── inc
│   ├── HelloAPI.h
│   ├── HelloFactoryImpl.h
│   └── HelloImpl.h
├── src
│   ├── HelloFactoryImpl.cpp
│   ├── HelloImpl.cpp
│   └── Makefile
└── swig
    └── hello4j.i
```

`HelloAPI.h`代码

``` 
#ifndef __HELLO_API_H__  
#define __HELLO_API_H__

class Hello;
class HelloFactory;
 
class Hello
{
public:
        Hello(){}
        virtual ~Hello(){}
public:
        virtual const char* sayHello() = 0;
};

class HelloFactory
{
public:
        HelloFactory(){}
        virtual ~HelloFactory(){}
public:
        virtual Hello* CreateHello() = 0;
};
   
extern "C"
{
        HelloFactory* getHelloFactoryInstance();
}

#endif
```

`HelloImpl.h`代码

``` c++
#ifndef __GEOMETRY_IMPL_H__  
#define __GEOMETRY_IMPL_H__  
 
#include "HelloAPI.h"  
 
class HelloImpl : public Hello  
{  
public:  
        HelloImpl();  
        virtual ~HelloImpl();  
public:  
        virtual const char* sayHello();  
};  
#endif
```

`HelloFactoryImpl.h`代码

``` c++
#ifndef __HELLO_FACTORY_IMPL_H__  
#define __HELLO_FACTORY_IMPL_H__  
  
#include "HelloImpl.h"  
#include "stddef.h"
  
class HelloFactoryImpl : public HelloFactory  
{  
public:  
        HelloFactoryImpl();  
        virtual ~HelloFactoryImpl();  
public:  
        virtual Hello* CreateHello();  
};  
#endif 
```

`HelloImpl.cpp`代码

``` c++
#include "HelloImpl.h"  
   
HelloImpl::HelloImpl()  
{  
}  
HelloImpl::~HelloImpl()  
{  
}  
const char* HelloImpl::sayHello()  
{  
    return "Hello World!";  
}
```

`HelloFactoryImpl.cpp`代码

``` c++
#include "HelloFactoryImpl.h"  
  
HelloFactoryImpl::HelloFactoryImpl()  
{  
}  
HelloFactoryImpl::~HelloFactoryImpl()  
{  
}  

Hello* HelloFactoryImpl::CreateHello()
{
	return new HelloImpl();
}
 
HelloFactory* getHelloFactoryInstance()
{
	static HelloFactory* helloFactory;
	if(helloFactory == NULL)
 	{
   		helloFactory = new HelloFactoryImpl();
   	}
	return helloFactory;
}
```

`Makefile`写法

``` makefile
OBJS=HelloFactoryImpl.o \
     HelloImpl.o  
INCLUDE=-I../inc   
TARGET=libhello.so  
CPPFLAG=-shared
CC=g++  
LDLIB=  
$(TARGET) : $(OBJS)  
	$(CC) $(CPPFLAG) $(INCLUDE) -o $(TARGET) $(OBJS) $(LDLIB)  
$(OBJS) : %.o : %.cpp  
	$(CC) -c -fPIC $(INCLUDE) $< -o $@  
clean:  
	-rm -f $(OBJS)  
install:  
	cp $(TARGET) ../lib
```

切换到`src`目录下执行`gmake`命令，编译`libhello.so`动态库

![p1](/img/2017-10-06-p1.png)

这个动态库里包含虚基类，纯虚函数，继承，C函数，静态变量等概念，几乎模拟了大部分动态库的情况。

**使用swig将C++接口转为Java接口**

编写`.i`文件(`hello4j.i`)

`module`是模块名。`SWIG`将C函数通过`Java`的JNI转换为`JAVA`方法，这些方法都以静态方法的方式封装到一个与模块名同名的`Java`类中。  

``` c++
%module hello4j
%{
#include "HelloAPI.h"
%}
%include "HelloAPI.h"
```

执行`Swig`命令

``` shell
swig -c++ -java -package com.test -outdir ./ -I../inc hello4j.i
```

> 参数说明：
>  -c++ -java
> 	告诉swig将C++接口转换为java接口。如果是将C接口转换为java接口，就不需要-c++，直接写 swig -java就可以。 
> -package
> 	生成的java类的包的名称 
> -I
> 	hello4j.i中include的.h文件的路径 
> hello4j.i 
> 	swig的.i文件

执行这条命令后，将在`swig`路径下生成几个文件

> hello4j_wrap.cxx 
>
> ​	C++文件，包装器文件。它将C++类的方法转换为C的函数。
>
> hello4j.java 
>
> ​	这是与刚才定义的module同名的一个类。
>
> hello4jJNI.java
>
> ​	打开这个文件可以看到，C++类的方法都转化为Java的静态方法。
>
> 其他与C++类同名的Java类
>
> ​	每一个C++类都被转化为与之对应的Java类，并且类名，方法明完全一样。

编写`Makefile`编译`hello4j_wrap.cxx`文件为`.so`动态库

``` makefile
OBJS=hello4j_wrap.o
INCLUDE=-I../inc \
        -I/usr/local/jdk1.8.0_141/include \
           -I/usr/local/jdk1.8.0_141/include/linux
TARGET=libhello4j.so
CPPFLAG=-shared
CC=g++
LDLIB=-L../lib -lhello
$(TARGET) : $(OBJS)
        $(CC) $(CPPFLAG) $(INCLUDE) -o $(TARGET) $(OBJS) $(LDLIB)  
$(OBJS) : %.o : %.cxx
        $(CC) -c -fPIC $(INCLUDE) $< -o $@  
clean:  
        -rm -f $(OBJS)  
install:  
        cp $(TARGET) ../lib
```

编译`hello4j_wrap.cxx`需要用到`jni`的头文件`jni.h`，在`JAVA_HOME`中即可找到。`-lhello`链接刚才上面编译好的动态库。

到目前为止我们的操作都是为了生成`libhello4j.so`动态库，`jni`就是通过这个库来调用我们真正想调用的`C++`动态库中的类和方法的。

**编译Swig生成的java并且尝试调用**

在`swig`目录中执行下面的命令生成`.class`文件。

``` shell
javac *.java
```

刚才我们执行`swig`命令时，设置的包名是`com.test`，所以我们建立目录`com/test`，把`class`文件全部移入后打包。

``` shell
mkdir -p com/test
mv *.class com/test/
```

然后打包成jar文件

``` shell
jar -cvf hello4j.jar ./com
```

将刚才的`libhello4j.so`和`libhello.so`都放在`java`的`library path`下。可以通过运行以下代码查看有哪些路径。

``` java
class test{  
    public static void main(String[] argv){  
        System.out.println(System.getProperty("java.library.path"));  
    }  
}
```

也可以通过系统`LD_LIBRARY_PATH`环境变量，将动态库本身的路径设置为`java`的`library path`。

``` shell
export LD_LIBRARY_PATH=/home/qiubinren/test/lib
```

编写`java`应用程序。

``` java
import com.test.*;

class test{
    static {
        System.loadLibrary("hello4j");
    }

    public static void main(String argv[]){
        HelloFactory helloFactory = null;
        helloFactory = hello4j.getHelloFactoryInstance();
        if(helloFactory==null){
            System.out.println("null HelloFactory");
            return;
        }
        Hello hello = null;
        hello = helloFactory.CreateHello();
        if(hello==null){
            System.out.println("null hello");
            return;
        }
        System.out.println(hello.sayHello());
    }
}
```

编译以及执行`test`程序

``` shell
javac -cp hello4j.jar test.java
java -cp ./hello4j.jar:. test
```

输出结果为：

``` shell
Hello World!
```

至此，可以看出我们已经成功使用了`Java`调用了`C++`动态库了。


