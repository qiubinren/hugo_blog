---
title: "我的OracleCoherence教程(三)"
date: 2018-08-29T16:26:18+08:00
draft: false
tags: ["Coherence", "Oracle"]
categories: ["Coherence", "Oracle"]
---

# 我的Oracle Coherence教程(三)

## Coherence序列化和反序列化

上一篇[博客](https://www.qiubinren.com/2018-08-14-我的oraclecoherence教程二.html/)介绍了怎么编写和`coherence`交互的简单应用，上次写入的都是`java`的基本对象，如果这边要在`coherence`中存储一个我们的自定义对象，该怎么写？

写过`rpc`或者一些`io`程序的人，这个时候肯定会想到序列化和反序列化，这边我们就介绍下，`coherence`的序列化和反序列化。

<!--more-->

## 了解Coherence中的序列化

`Coherence`缓存值对象可以表示来自任何源的数据，内部（例如会话数据，瞬态数据等）或外部（例如数据库，大型机等）。

放置在缓存中的对象必须是可序列化的。由于序列化通常是群集数据管理中最昂贵的部分，因此Coherence提供了用于序列化/反序列化数据的不同选项：

- `com.tangosol.io.pof.PofSerializer` - 可移植对象格式（也称为`POF`）是一种与语言无关的二进制格式。`POF`的设计在空间和时间上都非常高效，是`coherence`中推荐的序列化选项。
- `java.io.Serializable` - 最简单但最慢的选择。
- `java.io.Externalizable` - 这要求开发人员手动实现序列化，但可以提供显着的性能优势。相比之下`java.io.Serializable`，可以将序列化数据大小减少两倍或更多（特别有助于分布式缓存，因为它们通常以序列化形式缓存数据）。最重要的是，`CPU`使用率大大降低。
- `com.tangosol.io.ExternalizableLite`- 这非常相似`java.io.Externalizable`，但通过使用更高效的`I/O`流实现，可以提供更好的性能和更少的内存使用。
- `com.tangosol.run.xml.XmlBean`- 默认实现`ExternalizableLite`，这边不做介绍。

## 使用POF API序列化对象

这边我们常用的是`POF`方式序列化对象，它基本上是`coherence`基石之一，时间和空间上都非常高效，并且与语言无关。

这边还是使用上一篇[博客](https://www.qiubinren.com/2018-08-14-我的oraclecoherence教程二.html/)里写的例子，在此基础上增加一个对象。

首先进入`app/src`目录，增加一个`Users`类。

``` shell
cd /home/oracle/oracle/coherence/app/src
vim Users.java
```

类的属性如下。

``` java
package biz;

public class Users {
    private String userName;
    private int age;

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
  
    public String getId() {
        return this.userName;
    }

    @Override
    public String toString() {
        return "Users{" +
                "userName='" + userName + '\'' +
                ", age=" + age +
                '}';
    }
}
```

接下来，要实现`POF`序列化的对象可以通过两个接口`com.tangosol.io.pof.PortableObject`和`com.tangosol.io.pof.PofSerializer`来实现。

第一种，修改类的结构，使用`com.tangosol.io.pof.PortableObject`接口，需要实现两个方法`public void readExternal(PofReader reader)`和`public void writeExternal(PofWriter writer)`。

这个例子里就是：

``` java
package biz;

import com.tangosol.io.pof.PofReader;
import com.tangosol.io.pof.PofWriter;
import com.tangosol.io.pof.PortableObject;
import java.io.IOException;

public class Users implements PortableObject {
    ...
    @Override
    public void readExternal(PofReader pofReader) throws IOException {
        userName = pofReader.readString(0);
        age = pofReader.readInt(1);
    }

    @Override
    public void writeExternal(PofWriter pofWriter) throws IOException {
        pofWriter.writeString(0, userName);
        pofWriter.writeInt(1, age);
    }
    ...
}
```

而如果你不想要修改原类的结构，`coherence`也提供`com.tangosol.io.pof.PofSerializer`接口，支持外部序列化。这边的例子来说就是再实现一个类`UsersSerializer`。

``` java
package biz;

import com.tangosol.io.pof.PofReader;
import com.tangosol.io.pof.PofSerializer;
import com.tangosol.io.pof.PofWriter;
import java.io.IOException;


public class UsersSerializer implements PofSerializer {

    @Override
    public void serialize(PofWriter pofWriter, Object o) throws IOException {
        Users po = (Users) o;
        pofWriter.writeString(0, po.getUserName());
        pofWriter.writeInt(1, po.getAge());
        pofWriter.writeRemainder(null);
    }

    @Override
    public Object deserialize(PofReader pofReader) throws IOException {
        Users po = new Users();
        po.setUserName(pofReader.readString(0));
        po.setAge(pofReader.readInt(1));
        pofReader.readRemainder();
        return po;
    }
}
```

其中`pofReader.readRemainder()`和`pofWriter.writeRemainder(null)`两个方法分别标记读取和写入已完成。

两种方法选择其中一种实现，都可以实现`POF`序列化。

接下来我们还是使用第二种来演示，如果类内增加了第一种的代码，请删掉后再进行后续内容。

将`Users`类打包成`jar`。

``` shell
mkdir biz
mv Users.java biz
mv UsersSerializer.java biz
javac -classpath ../../lib/coherence.jar biz/*.java
jar -cvf ../lib/biz.jar biz/*.class
```

修改`TestCacheGet.java`和`TestCachePut.java`两个文件，让他们使用`Users`类。

`TestCacheGet.java`修改后的代码

``` java
package testcacheget;

import com.tangosol.net.CacheFactory;
import com.tangosol.net.NamedCache;
import biz.Users;

public class TestCacheGet {

    public static void main(String[] args) {
        String key = "aaa";
        CacheFactory.ensureCluster();
        NamedCache cache = CacheFactory.getCache("repl-Test");
        Users o = (Users)cache.get(key);
        System.out.println("姓名" + o.getUserName());
        System.out.println("年龄" + o.getAge());
        CacheFactory.shutdown();
    }
}
```

`TestCachePut.java`修改后的代码

``` java
package testcacheput;

import com.tangosol.net.CacheFactory;
import com.tangosol.net.NamedCache;
import biz.Users;

public class TestCachePut {

    public static void main(String[] args) {
        Users users = new Users();
        users.setUserName("aaa");
        users.setAge(1);
        CacheFactory.ensureCluster();
        NamedCache cache = CacheFactory.getCache("repl-Test");
        cache.put(users.getUserName(), users);
        CacheFactory.shutdown();
        System.out.println("写入users成功");
    }
}
```

重新编译他们。

``` shell
javac -classpath ../../lib/coherence.jar:../lib/biz.jar TestCacheGet.java
javac -classpath ../../lib/coherence.jar:../lib/biz.jar TestCachePut.java
mv TestCacheGet.class testcacheget
jar -cvf ../lib/testget.jar testcacheget/
mv TestCachePut.class testcacheput
jar -cvf ../lib/testput.jar testcacheput/
```

至此，应用程序编码部分全部完成。

## 注册POF对象

`coherence`提供了`com.tangosol.io.pof.ConfigurablePofContext`序列化程序类，负责将POF序列化对象映射到匹配的序列化类（上面介绍的`PofSerializer`或`PortableObject`的实现）。

一旦你使用的类需要序列化，就需要使用`pof-config.xml`文件注册类。POF配置文件具有一个`<user-type-list>`元素，该元素包含实现`PortableObject`或`PofSerializer`与之关联的类列表。在`<type-id>`每个类都必须是唯一的，而且必须在所有群集实例（包括扩展客户端）相匹配。

这边我们首先在`coherence`目录下新建一个`pofconfig`目录

``` shell
cd ../..
mkdir pofconfig
cd pofconfig
```

创建`pof-config-biz.xml`文件。

``` xml
<?xml version="1.0"?>
<pof-config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns="http://xmlns.oracle.com/coherence/coherence-pof-config"
            xsi:schemaLocation="http://xmlns.oracle.com/coherence/coherence-pof-config coherence-pof-config.xsd">
    <user-type-list>
        <include>coherence-pof-config.xml</include>
        <user-type>
            <type-id>1001</type-id>
            <class-name>biz.Users</class-name>
            <serializer>
                <class-name>biz.UsersSerializer</class-name>
            </serializer>
        </user-type>
    </user-type-list>
    <enable-references>true</enable-references>
</pof-config>
```

这边注意一下`<include>`标签需要引用`coherence.jar`中默认的`POF`配置，不然基本类型的`POF`配置就丢了，`<type-id>`必须大于`1000`，`<enable-references>`为`true`是启用`POF`对象引用的意思，可以有效减少数据大小，避免多次编码同一对象。

修改网格启动脚本。

``` shell
cd ../bin/
vim startserver.sh
```

在启动参数中加入`-Dtangosol.pof.enabled=true `和`-Dtangosol.pof.config=../pofconfig/pof-config-biz.xml`两个参数为整个`JVM`启用`POF`和指定`POF`配置文件。

`-cp`后也要加入对象的`biz.jar`包的路径。

``` shell
nohup java -server -showversion -Xms1g -Xmx1g -Dtangosol.coherence.localstorage=true -Dtangosol.coherence.override=../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../pofconfig/pof-config-biz.xml -Dtangosol.coherence.localport=10000 -cp ../lib/coherence.jar:../app/lib/biz.jar com.tangosol.net.DefaultCacheServer > server.log &
```

启动`coherence`服务端实例。

``` shell
./startserver.sh
```

修改应用启动脚本。

``` shell
cd ../app/bin/
ll
total 8
-rwxrwxrwx 1 oracle oracle 240 Jun 27 15:30 gettest.sh
-rwxrwxrwx 1 oracle oracle 240 Jun 27 15:26 puttest.sh
```

`gettest.sh`和`puttest.sh`都要修改。修改的点和上面一样，都要加入`POF`支持，并且加入序列化和对象`jar`包路径。

``` shell
cat gettest.sh 

#!/bin/bash
java -Dtangosol.coherence.override=../../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../../config/coherence-cache-config.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../../pofconfig/pof-config-biz.xml -cp ../lib/testget.jar:../../lib/coherence.jar:../lib/biz.jar testcacheget.TestCacheGet

cat puttest.sh 

#!/bin/bash
java -Dtangosol.coherence.override=../../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../../config/coherence-cache-config.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../../pofconfig/pof-config-biz.xml -cp ../lib/testput.jar:../../lib/coherence.jar:../lib/biz.jar testcacheput.TestCachePut
```

运行`puttest.sh`。

``` shell
./puttest.sh 

2019-06-28 15:56:36.615/1.047 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-06-28 15:56:36.785/1.216 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-06-28 15:56:36.787/1.219 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-06-28 15:56:36.797/1.228 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-06-28 15:56:36.797/1.228 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-06-28 15:56:36.797/1.228 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-06-28 15:56:37.953/2.384 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-06-28 15:56:38.006/2.437 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:36900 using SystemDatagramSocketProvider
2019-06-28 15:56:38.471/2.902 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-06-28 15:52:39.695, Address=129.211.9.207:10000, MachineId=3986, Location=process:12420, Role=CoherenceServer) joined Cluster with senior member 1
2019-06-28 15:56:38.481/2.912 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=2, Timestamp=2019-06-28 15:56:38.277, Address=129.211.9.207:36900, MachineId=3986, Location=process:15080, Role=TestcacheputTestCachePut, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-06-28 15:52:39.695, Address=129.211.9.207:10000, MachineId=3986, Location=process:12420, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-06-28 15:56:38.702/3.133 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded POF configuration from "file:/home/oracle/oracle/coherence/pofconfig/pof-config-biz.xml"
2019-06-28 15:56:38.840/3.272 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:36900.63744
2019-06-28 15:56:38.866/3.297 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-06-28 15:56:39.018/3.450 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): TcpRing disconnected from Member(Id=1, Timestamp=2019-06-28 15:52:39.695, Address=129.211.9.207:10000, MachineId=3986, Location=process:12420, Role=CoherenceServer) due to a peer departure (end of stream); removing the member.
2019-06-28 15:56:39.020/3.451 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-06-28 15:56:39.019, Address=129.211.9.207:10000, MachineId=3986, Location=process:12420, Role=CoherenceServer) left Cluster with senior member 2
2019-06-28 15:56:39.023/3.454 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Member 1 left service TransportService with senior member 1
2019-06-28 15:56:39.027/3.459 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=2, Timestamp=2019-06-28 15:56:38.277, Address=129.211.9.207:36900, MachineId=3986, Location=process:15080, Role=TestcacheputTestCachePut)
  OldestMember=Member(Id=2, Timestamp=2019-06-28 15:56:38.277, Address=129.211.9.207:36900, MachineId=3986, Location=process:15080, Role=TestcacheputTestCachePut)
  ActualMemberSet=MemberSet(Size=1
    Member(Id=2, Timestamp=2019-06-28 15:56:38.277, Address=129.211.9.207:36900, MachineId=3986, Location=process:15080, Role=TestcacheputTestCachePut)
    )
  MemberId|ServiceJoined|MemberState|Version
    2|2019-06-28 15:56:38.277|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=1
    Member(Id=1, Timestamp=2019-06-28 15:56:39.019, Address=129.211.9.207:10000, MachineId=3986, Location=process:12420, Role=CoherenceServer)
    )
  )

TcpRing{Connections=[]}
IpMonitor{Addresses=0, Timeout=15s}

2019-06-28 15:56:39.076/3.507 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=2): Service Management joined the cluster with senior service member 2
2019-06-28 15:56:39.222/3.653 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-06-28 15:56:39.325/3.756 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): JMXConnectorServer now listening for connections on 129.211.9.207:41541
2019-06-28 15:56:39.348/3.780 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-06-28 15:56:39.405/3.836 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Loaded cache configuration from "file:/home/oracle/oracle/coherence/config/coherence-cache-config.xml"; this document does not refer to any schema definition and has not been validated.
2019-06-28 15:56:39.518/3.949 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-06-28 15:56:39.571/4.002 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=2): Service ReplicatedCache joined the cluster with senior service member 2
2019-06-28 15:56:39.601/4.032 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=n/a): Service ReplicatedCache left the cluster
2019-06-28 15:56:39.607/4.038 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=n/a): Service Management left the cluster
2019-06-28 15:56:39.608/4.040 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService left the cluster
2019-06-28 15:56:39.628/4.059 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Service Cluster left the cluster
写入users成功
```

很明显这边`put`成功。

再执行`get`应用，试试读取。

``` shell
./gettest.sh 

2019-06-28 17:23:40.829/1.042 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-06-28 17:23:40.984/1.196 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-06-28 17:23:40.986/1.199 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-06-28 17:23:41.004/1.217 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-06-28 17:23:41.004/1.217 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-06-28 17:23:41.004/1.217 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-06-28 17:23:42.239/2.451 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-06-28 17:23:42.284/2.497 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:39213 using SystemDatagramSocketProvider
2019-06-28 17:23:42.747/2.960 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-06-28 17:20:50.862, Address=129.211.9.207:10000, MachineId=3986, Location=process:28592, Role=CoherenceServer) joined Cluster with senior member 1
2019-06-28 17:23:42.759/2.971 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=3, Timestamp=2019-06-28 17:23:42.545, Address=129.211.9.207:39213, MachineId=3986, Location=process:29173, Role=TestcachegetTestCacheGet, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-06-28 17:20:50.862, Address=129.211.9.207:10000, MachineId=3986, Location=process:28592, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-06-28 17:23:42.997/3.209 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded POF configuration from "file:/home/oracle/oracle/coherence/pofconfig/pof-config-biz.xml"
2019-06-28 17:23:43.078/3.291 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded included POF configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/coherence-pof-config.xml"
2019-06-28 17:23:43.539/3.752 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:39213.47158
2019-06-28 17:23:43.567/3.781 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-06-28 17:23:43.587/3.800 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=3, Timestamp=2019-06-28 17:23:42.545, Address=129.211.9.207:39213, MachineId=3986, Location=process:29173, Role=TestcachegetTestCacheGet)
  OldestMember=Member(Id=1, Timestamp=2019-06-28 17:20:50.862, Address=129.211.9.207:10000, MachineId=3986, Location=process:28592, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=2
    Member(Id=1, Timestamp=2019-06-28 17:20:50.862, Address=129.211.9.207:10000, MachineId=3986, Location=process:28592, Role=CoherenceServer)
    Member(Id=3, Timestamp=2019-06-28 17:23:42.545, Address=129.211.9.207:39213, MachineId=3986, Location=process:29173, Role=TestcachegetTestCacheGet)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-06-28 17:20:50.862|JOINED|12.2.1.3.0,
    3|2019-06-28 17:23:42.545|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=1
    Member(Id=2, Timestamp=2019-06-28 17:23:31.221, Address=129.211.9.207:39443, MachineId=3986)
    )
  )

TcpRing{Connections=[1]}
IpMonitor{Addresses=0, Timeout=15s}

2019-06-28 17:23:43.647/3.860 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=3): Service Management joined the cluster with senior service member 1
2019-06-28 17:23:43.699/3.912 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=3): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-06-28 17:23:44.178/4.390 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=3): Loaded cache configuration from "file:/home/oracle/oracle/coherence/config/coherence-cache-config.xml"; this document does not refer to any schema definition and has not been validated.
2019-06-28 17:23:44.357/4.569 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=3): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-06-28 17:23:44.428/4.641 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=3): Service ReplicatedCache joined the cluster with senior service member 1
姓名aaa
年龄1
2019-06-28 17:23:44.535/4.748 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=n/a): Service ReplicatedCache left the cluster
2019-06-28 17:23:44.542/4.755 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=n/a): Service Management left the cluster
2019-06-28 17:23:44.547/4.760 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService left the cluster
2019-06-28 17:23:44.780/4.992 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Service Cluster left the cluster
```

看到输出姓名`aaa`，年龄`1`，查询成功。

现在我们把上一篇博客中，和`coherence`交互的命令行客户端的启动脚本也改下。加入`POF`配置和`jar`的相关支持。

``` shell
cd ../../bin/
cat startquery.sh 

#!/bin/bash
java -server -showversion -Xms64m -Xmx64m -Dcoherence.distributed.localstorage=false -Dtangosol.coherence.override=../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../pofconfig/pof-config-biz.xml -cp /home/oracle/oracle/coherence/lib/coherence.jar:../app/lib/biz.jar com.tangosol.net.CacheFactory
```

然后启动交互命令行，使用`list`命令查看`repl-Test`中的数据。

``` shell
./startquery.sh 

java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141-b15)
Java HotSpot(TM) 64-Bit Server VM (build 25.141-b15, mixed mode)

2019-06-28 17:24:54.271/1.284 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-06-28 17:24:54.494/1.507 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-06-28 17:24:54.496/1.510 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-06-28 17:24:54.512/1.525 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-06-28 17:24:54.512/1.525 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-06-28 17:24:54.513/1.526 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-06-28 17:24:55.994/3.007 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-06-28 17:24:56.052/3.065 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:44637 using SystemDatagramSocketProvider
2019-06-28 17:24:56.575/3.589 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-06-28 17:20:50.862, Address=129.211.9.207:10000, MachineId=3986, Location=process:28592, Role=CoherenceServer) joined Cluster with senior member 1
2019-06-28 17:24:56.586/3.599 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=4, Timestamp=2019-06-28 17:24:56.377, Address=129.211.9.207:44637, MachineId=3986, Location=process:29388, Role=CoherenceConsole, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-06-28 17:20:50.862, Address=129.211.9.207:10000, MachineId=3986, Location=process:28592, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-06-28 17:24:56.822/3.835 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded POF configuration from "file:/home/oracle/oracle/coherence/pofconfig/pof-config-biz.xml"
2019-06-28 17:24:56.909/3.922 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded included POF configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/coherence-pof-config.xml"
2019-06-28 17:24:57.520/4.533 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:44637.44370
2019-06-28 17:24:57.565/4.579 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-06-28 17:24:57.606/4.620 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=4, Timestamp=2019-06-28 17:24:56.377, Address=129.211.9.207:44637, MachineId=3986, Location=process:29388, Role=CoherenceConsole)
  OldestMember=Member(Id=1, Timestamp=2019-06-28 17:20:50.862, Address=129.211.9.207:10000, MachineId=3986, Location=process:28592, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=2
    Member(Id=1, Timestamp=2019-06-28 17:20:50.862, Address=129.211.9.207:10000, MachineId=3986, Location=process:28592, Role=CoherenceServer)
    Member(Id=4, Timestamp=2019-06-28 17:24:56.377, Address=129.211.9.207:44637, MachineId=3986, Location=process:29388, Role=CoherenceConsole)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-06-28 17:20:50.862|JOINED|12.2.1.3.0,
    4|2019-06-28 17:24:56.377|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=2
    Member(Id=2, Timestamp=2019-06-28 17:23:31.221, Address=129.211.9.207:39443, MachineId=3986)
    Member(Id=3, Timestamp=2019-06-28 17:23:44.763, Address=129.211.9.207:39213, MachineId=3986)
    )
  )

TcpRing{Connections=[1]}
IpMonitor{Addresses=0, Timeout=15s}

2019-06-28 17:24:57.680/4.693 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=4): Service Management joined the cluster with senior service member 1
2019-06-28 17:24:57.775/4.788 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"

Map (?): cache repl-Test
2019-06-28 17:26:32.025/99.038 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Loaded cache configuration from "file:/home/oracle/oracle/coherence/config/coherence-cache-config.xml"; this document does not refer to any schema definition and has not been validated.
2019-06-28 17:26:32.119/99.132 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-06-28 17:26:32.195/99.208 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=4): Service ReplicatedCache joined the cluster with senior service member 1

Cache Configuration: repl-Test
  SchemeName: example-replicated
  AutoStart: true
  ServiceName: ReplicatedCache
  ServiceDependencies
    EventDispatcherThreadPriority: 10
    ThreadPriority: 10
    WorkerThreadsMax: 2147483647
    WorkerPriority: 5
    EnsureCacheTimeout: 30000
  BackingMapScheme
    InnerScheme (LocalScheme) 
      SchemeName: unlimited-backing-map
      UnitFactor: 1

Map (repl-Test): list
aaa = Users{userName='aaa', age=1}
```

很明显，我们这个网格已经支持了对`Users`这个自定义类的存储、写入、查询支持。