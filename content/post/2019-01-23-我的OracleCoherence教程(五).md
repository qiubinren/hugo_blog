---
title: "我的OracleCoherence教程(五)"
date: 2019-01-23T16:45:44+08:00
draft: false
tags: ["Coherence", "Oracle"]
categories: ["Coherence", "Oracle"]
---

# 我的Oracle Coherence教程(五)

## 并发读和更新问题

在[我的Oracle Coherence教程(二)](https://www.qiubinren.com/2018-08-14-我的oraclecoherence教程二.html/)中，我介绍了对`coherence`单纯的`get`和`put`操作，对整个`coherence`集群来说，操作是原子的，不会存在节点间数据不一致的情况。

但是现在的问题是，如果有两个线程并发做同一件事，先`get`同一个`key`的值，再加`1`，再`put`。

和数据库操作`select + update`并发操作产生的问题一样，如果啥都不处理肯定会有数据覆盖问题。

今天介绍`coherence`怎么解决这个问题的。

<!--more-->

## 方法一加锁

遇到并发问题，我们往往第一反应是加锁。

`coherence`支持对单个条目的加锁，提供了相关`api`。

样例：

``` java
public void updateData() {
  myCache.lock("aaa", -1);
  try {
    Object a = myCache.get("aaa");
    ...
    myCache.put("aaa", a);
  } finally {
    myCache.unlock("aaa");
  }
}
```

`coherence`的这种加锁方式开销很大，加锁后的`key`其他客户端访问必须等待释放，属于写独占锁，加锁也会增加网络通信客户端与服务端之间的往返次数。

我们一般不使用。

## 方法二EntryProcess

`coherence`其实提供了一种可扩展，无锁的并发处理方式，也就是我们口中常说的`EP`。

`Coherence`提供在服务端节点处理数据的能力，客户端通过`EntryProcessor API`可以远程发送数据到真正存储需要操作的`key`的服务端节点上，服务端节点获取客户端的数据，执行预先定义的操作。如果有多个请求更新同一对象，`coherence`会自动对它们进行排队并一个接一个地执行更新。这导致无需锁即可在系统上执行更新。

如果，客户端同时操作多个`key`的数据，调用`EP`后，存储这些`key`的节点会并发执行数据操作和更新，无需客户端启用和管理任何代理。大概流程如图所示：

![p1](/img/2019-01-23-p1.png)

调用方式有点类似`rpc`。

这种方式可以最大限度地减少争用和延迟，并提高系统吞吐量，而不会影响数据操作的容错能力。

## 例子演示

第一个加锁很简单，这边就不演示了，平时也不用。

还是使用上面几篇博文的例子。

`coherence`提供很多默认的`EP`，也可以通过扩展`AbstractProcessor`这个抽象基类来自己实现的`EP`。目前公司里没有使用任何默认`EP`，都是自己继承`AbstractProcessor`类实现的`EP`，这边也就介绍自己实现的方式。

进入`app/src`目录，新建`UsersEp.java`文件。

``` shell
cd ../app/src
vim UsersEp.java
```

代码内容如下：

``` java
package biz;

import com.tangosol.util.processor.AbstractProcessor;
import com.tangosol.util.InvocableMap.Entry;
import com.tangosol.util.BinaryEntry;
import biz.Users;

public class UsersEp extends AbstractProcessor {
    private int addAge;

    public UsersEp() {
    }

    public UsersEp(int addAge) {
        this.addAge = addAge;
    }

    @Override
    public Object process(Entry entry) {
        BinaryEntry bEntry = (BinaryEntry)entry;
        Users users = (Users)bEntry.getValue();
        if(users == null) {
            users = new Users();
            users.setUserName((String)bEntry.getKey());
            users.setAge(this.addAge);
        } else {
            users.setAge(users.getAge() + this.addAge);
        }
        bEntry.setValue(users);
        return users;
    }

    public int getAddAge() {
        return this.addAge;
    }

    public void setAddAge(int addAge) {
        this.addAge = addAge;
    }

}
```

可以看到，这边我们定义了一个`UsersEp`类，继承了`AbstractProcessor`这个抽象基类，并且实现了`process`方法，其中`process`方法里的内容，就是我们客户端调用`EP`后，想做的操作。

这边的操作含义是，网格里有这个`key`的`user`，就把他的年龄加上传入的值，如果没有，则在网格内新增一个这个`key`的值，传入值就是他的年龄。

把这个`java`文件移入`biz`目录。

``` shell
mv UsersEp.java biz
```

这边要注意，我们是不是在客户端和服务端都要用到`UsersEp`这个新增类？

很明显，服务端要运行这个类的方法，客户端要通过这个类给服务端传值，那么势必涉及序列化和反序列化，所以我们还要新增`UsersEpSerializer`类。

``` java
package biz;

import com.tangosol.io.pof.PofReader;
import com.tangosol.io.pof.PofSerializer;
import com.tangosol.io.pof.PofWriter;
import java.io.IOException;


public class UsersEpSerializer implements PofSerializer {

    @Override
    public void serialize(PofWriter pofWriter, Object o) throws IOException {
        UsersEp po = (UsersEp) o;
        pofWriter.writeInt(0, po.getAddAge());
        pofWriter.writeRemainder(null);
    }

    @Override
    public Object deserialize(PofReader pofReader) throws IOException {
        UsersEp po = new UsersEp();
        po.setAddAge(pofReader.readInt(0));
        pofReader.readRemainder();
        return po;
    }
}
```

重新编译`biz.jar`。

``` shell
mv UsersEpSerializer.java biz
javac -classpath ../../lib/coherence.jar biz/*.java
jar -cvf ../lib/biz.jar biz/*.class
```

修改`pofconfig`目录中的`pof`配置文件，加入`biz.UsersEpSerializer`的用户类型定义。

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
        <user-type>
            <type-id>1002</type-id>
            <class-name>biz.UsersEp</class-name>
            <serializer>
                <class-name>biz.UsersEpSerializer</class-name>
            </serializer>
        </user-type>
    </user-type-list>
    <enable-references>true</enable-references>
</pof-config>
```

修改`TestCacheGet.java`的代码。

``` java
package testcacheget;

import com.tangosol.net.CacheFactory;
import com.tangosol.net.NamedCache;
import biz.Users;
import biz.UsersEp;

public class TestCacheGet {

    public static void main(String[] args) {
        String key = "aaa";
        CacheFactory.ensureCluster();
        NamedCache cache = CacheFactory.getCache("repl-Test");
        System.out.println(cache.get("aaa"));
        Users o = (Users)cache.invoke(key, new UsersEp(3));
        System.out.println(o);
        System.out.println("姓名" + o.getUserName());
        System.out.println("年龄" + o.getAge());
        CacheFactory.shutdown();
    }
}
```

注意这边本来的`cache.get()`改成了`cache.invoke()`进行远程调用。

重新编译`testcacheget.jar`。

``` shell
javac -classpath ../../lib/coherence.jar:../lib/biz.jar TestCacheGet.java
mv TestCacheGet.class testcacheget/
jar -cvf ../lib/testget.jar testcacheget/
```

编码部分也已完成。启动网格服务节点，代理节点，并且写入`key`为`aaa`，年龄为`1`的`Users`对象。

``` shell
cd ../../bin/
./startserver.sh 
./startproxy.sh
cd ../app/bin
./puttest.sh
```

执行`gettest.sh`，让`get`客户端调用`EP`。

``` shell
./gettest.sh

2019-07-03 13:09:06.748/1.062 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-07-03 13:09:06.930/1.243 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-07-03 13:09:06.932/1.246 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-07-03 13:09:06.943/1.256 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-07-03 13:09:06.943/1.257 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-07-03 13:09:06.943/1.257 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-07-03 13:09:08.153/2.467 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-07-03 13:09:08.208/2.522 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:33778 using SystemDatagramSocketProvider
2019-07-03 13:09:08.650/2.964 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-07-03 13:06:14.142, Address=129.211.9.207:10000, MachineId=3986, Location=process:27849, Role=CoherenceServer) joined Cluster with senior member 1
2019-07-03 13:09:08.658/2.972 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=4, Timestamp=2019-07-03 13:09:08.443, Address=129.211.9.207:33778, MachineId=3986, Location=process:28411, Role=TestcachegetTestCacheGet, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-07-03 13:06:14.142, Address=129.211.9.207:10000, MachineId=3986, Location=process:27849, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-07-03 13:09:08.841/3.155 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=2, Timestamp=2019-07-03 13:06:18.964, Address=129.211.9.207:10010, MachineId=3986, Location=process:27876, Role=CoherenceServer) joined Cluster with senior member 1
2019-07-03 13:09:08.889/3.203 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded POF configuration from "file:/home/oracle/oracle/coherence/pofconfig/pof-config-biz.xml"
2019-07-03 13:09:08.986/3.300 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded included POF configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/coherence-pof-config.xml"
2019-07-03 13:09:09.496/3.811 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:33778.42194
2019-07-03 13:09:09.549/3.863 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-07-03 13:09:09.596/3.910 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=4, Timestamp=2019-07-03 13:09:08.443, Address=129.211.9.207:33778, MachineId=3986, Location=process:28411, Role=TestcachegetTestCacheGet)
  OldestMember=Member(Id=1, Timestamp=2019-07-03 13:06:14.142, Address=129.211.9.207:10000, MachineId=3986, Location=process:27849, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=3
    Member(Id=1, Timestamp=2019-07-03 13:06:14.142, Address=129.211.9.207:10000, MachineId=3986, Location=process:27849, Role=CoherenceServer)
    Member(Id=2, Timestamp=2019-07-03 13:06:18.964, Address=129.211.9.207:10010, MachineId=3986, Location=process:27876, Role=CoherenceServer)
    Member(Id=4, Timestamp=2019-07-03 13:09:08.443, Address=129.211.9.207:33778, MachineId=3986, Location=process:28411, Role=TestcachegetTestCacheGet)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-07-03 13:06:14.142|JOINED|12.2.1.3.0,
    2|2019-07-03 13:06:18.964|JOINED|12.2.1.3.0,
    4|2019-07-03 13:09:08.443|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=1
    Member(Id=3, Timestamp=2019-07-03 13:08:00.498, Address=129.211.9.207:39794, MachineId=3986)
    )
  )

TcpRing{Connections=[2]}
IpMonitor{Addresses=0, Timeout=15s}

2019-07-03 13:09:09.678/3.992 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=4): Service Management joined the cluster with senior service member 1
2019-07-03 13:09:09.773/4.087 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-07-03 13:09:10.182/4.496 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Loaded cache configuration from "file:/home/oracle/oracle/coherence/app/config/coherence-cache-config-client.xml"; this document does not refer to any schema definition and has not been validated.
2019-07-03 13:09:10.421/4.735 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
Users{userName='aaa', age=1}
Exception in thread "main" Portable(java.lang.ClassCastException): com.tangosol.util.InvocableMapHelper$SimpleEntry cannot be cast to com.tangosol.util.BinaryEntry
	at biz.UsersEp.process(UsersEp.java:20)
	at com.tangosol.util.InvocableMapHelper.invokeLocked(InvocableMapHelper.java:111)
	at com.tangosol.coherence.component.util.CacheHandler.invoke(CacheHandler.CDB:3)
	at com.tangosol.coherence.component.util.SafeNamedCache.invoke$Router(SafeNamedCache.CDB:1)
	at com.tangosol.coherence.component.util.SafeNamedCache.invoke(SafeNamedCache.CDB:5)
	at com.tangosol.util.ConverterCollections$ConverterInvocableMap.invoke(ConverterCollections.java:620)
	at com.tangosol.util.ConverterCollections$ConverterNamedCache.invoke(ConverterCollections.java:1104)
	at com.tangosol.coherence.component.net.extend.proxy.NamedCacheProxy.invoke$Router(NamedCacheProxy.CDB:1)
	at com.tangosol.coherence.component.net.extend.proxy.NamedCacheProxy.invoke(NamedCacheProxy.CDB:2)
	at com.tangosol.coherence.component.net.extend.messageFactory.NamedCacheFactory$InvokeRequest.onRun(NamedCacheFactory.CDB:6)
	at com.tangosol.coherence.component.net.extend.message.Request.run(Request.CDB:4)
	at com.tangosol.coherence.component.net.extend.proxy.NamedCacheProxy.onMessage(NamedCacheProxy.CDB:11)
	at com.tangosol.coherence.component.net.extend.Channel.execute(Channel.CDB:61)
	at com.tangosol.coherence.component.net.extend.Channel.receive(Channel.CDB:26)
	at com.tangosol.coherence.component.util.daemon.queueProcessor.service.Peer$DaemonPool$WrapperTask.run(Peer.CDB:9)
	at com.tangosol.coherence.component.util.DaemonPool$WrapperTask.run(DaemonPool.CDB:32)
	at com.tangosol.coherence.component.util.DaemonPool$Daemon.onNotify(DaemonPool.CDB:66)
	at com.tangosol.coherence.component.util.Daemon.run(Daemon.CDB:54)
	at java.lang.Thread.run(Thread.java:748)
	at <process boundary>
	at com.tangosol.io.pof.ThrowablePofSerializer.deserialize(ThrowablePofSerializer.java:57)
	at com.tangosol.io.pof.PofBufferReader.readAsObject(PofBufferReader.java:3618)
	at com.tangosol.io.pof.PofBufferReader.readObject(PofBufferReader.java:2889)
	at com.tangosol.io.pof.PofBufferReader.readObject(PofBufferReader.java:2876)
	at com.tangosol.coherence.component.net.extend.message.Response.readExternal(Response.CDB:20)
	at com.tangosol.coherence.component.net.extend.Codec.decode(Codec.CDB:29)
	at com.tangosol.coherence.component.util.daemon.queueProcessor.service.Peer.decodeMessage(Peer.CDB:25)
	at com.tangosol.coherence.component.util.daemon.queueProcessor.service.Peer.onNotify(Peer.CDB:54)
	at com.tangosol.coherence.component.util.Daemon.run(Daemon.CDB:54)
	at java.lang.Thread.run(Thread.java:748)
```

发现这边报了个错，`SimpleEntry cannot be cast to com.tangosol.util.BinaryEntry`，强制转换失败，我们发现报错的地方是`EP`中的`BinaryEntry bEntry = (BinaryEntry)entry;`这行，首先说明我们调用`EP`已经成功了，确实走到了服务端这边，但是为什么会转换失败呢？

根据错误提示，我们可以知道，应该是`coherence`内`key`和`value`不是二进制导致的，这边我犯了个错误，因为公司`EP`使用的都是分布式缓存，并且分布式缓存存储格式为二进制格式，所以公司的`EP`都是使用`BinaryEntry`类操作，我写的时候条件反射了，但是这边的样例是全复制缓存，并不支持二进制格式，所以报错了。

我们先改下`EP`类，不再强转`BinaryEntry`。

``` java
package biz;

import com.tangosol.util.processor.AbstractProcessor;
import com.tangosol.util.InvocableMap.Entry;
import biz.Users;

public class UsersEp extends AbstractProcessor {
    private int addAge;

    public UsersEp() {
    }

    public UsersEp(int addAge) {
        this.addAge = addAge;
    }

    @Override
    public Object process(Entry entry) {
        Users users = (Users)entry.getValue();
        if(users == null) {
            users = new Users();
            users.setUserName((String)entry.getKey());
            users.setAge(this.addAge);
        } else {
            users.setAge(users.getAge() + this.addAge);
        }
        entry.setValue(users);
        return users;
    }

    public int getAddAge() {
        return this.addAge;
    }

    public void setAddAge(int addAge) {
        this.addAge = addAge;
    }

}
```

重新编译`biz.jar`。

``` shell
javac -classpath ../../lib/coherence.jar biz/*.java
jar -cvf ../lib/biz.jar biz/*.class
```

重启网格所有节点并且执行`puttest.sh`后，启动`gettest.sh`。

``` shell
./gettest.sh 

2019-07-03 14:52:12.643/0.960 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-07-03 14:52:12.786/1.103 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-07-03 14:52:12.806/1.123 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-07-03 14:52:12.810/1.126 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-07-03 14:52:12.810/1.127 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-07-03 14:52:12.810/1.127 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-07-03 14:52:13.960/2.277 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-07-03 14:52:14.033/2.350 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:35925 using SystemDatagramSocketProvider
2019-07-03 14:52:14.470/2.787 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-07-03 14:51:44.416, Address=129.211.9.207:10000, MachineId=3986, Location=process:10673, Role=CoherenceServer) joined Cluster with senior member 1
2019-07-03 14:52:14.484/2.801 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=4, Timestamp=2019-07-03 14:52:14.272, Address=129.211.9.207:35925, MachineId=3986, Location=process:10864, Role=TestcachegetTestCacheGet, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-07-03 14:51:44.416, Address=129.211.9.207:10000, MachineId=3986, Location=process:10673, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-07-03 14:52:14.666/2.983 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=2, Timestamp=2019-07-03 14:51:49.741, Address=129.211.9.207:10010, MachineId=3986, Location=process:10706, Role=CoherenceServer) joined Cluster with senior member 1
2019-07-03 14:52:14.712/3.029 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded POF configuration from "file:/home/oracle/oracle/coherence/pofconfig/pof-config-biz.xml"
2019-07-03 14:52:14.798/3.115 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded included POF configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/coherence-pof-config.xml"
2019-07-03 14:52:15.269/3.586 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:35925.58971
2019-07-03 14:52:15.301/3.618 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-07-03 14:52:15.346/3.663 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=4, Timestamp=2019-07-03 14:52:14.272, Address=129.211.9.207:35925, MachineId=3986, Location=process:10864, Role=TestcachegetTestCacheGet)
  OldestMember=Member(Id=1, Timestamp=2019-07-03 14:51:44.416, Address=129.211.9.207:10000, MachineId=3986, Location=process:10673, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=3
    Member(Id=1, Timestamp=2019-07-03 14:51:44.416, Address=129.211.9.207:10000, MachineId=3986, Location=process:10673, Role=CoherenceServer)
    Member(Id=2, Timestamp=2019-07-03 14:51:49.741, Address=129.211.9.207:10010, MachineId=3986, Location=process:10706, Role=CoherenceServer)
    Member(Id=4, Timestamp=2019-07-03 14:52:14.272, Address=129.211.9.207:35925, MachineId=3986, Location=process:10864, Role=TestcachegetTestCacheGet)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-07-03 14:51:44.416|JOINED|12.2.1.3.0,
    2|2019-07-03 14:51:49.741|JOINED|12.2.1.3.0,
    4|2019-07-03 14:52:14.272|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=1
    Member(Id=3, Timestamp=2019-07-03 14:52:08.049, Address=129.211.9.207:36562, MachineId=3986)
    )
  )

TcpRing{Connections=[2]}
IpMonitor{Addresses=0, Timeout=15s}

2019-07-03 14:52:15.431/3.748 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=4): Service Management joined the cluster with senior service member 1
2019-07-03 14:52:15.503/3.820 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-07-03 14:52:15.970/4.287 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Loaded cache configuration from "file:/home/oracle/oracle/coherence/app/config/coherence-cache-config-client.xml"; this document does not refer to any schema definition and has not been validated.
2019-07-03 14:52:16.202/4.519 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
Users{userName='aaa', age=1}
Users{userName='aaa', age=4}
姓名aaa
年龄4
2019-07-03 14:52:16.590/4.906 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=n/a): Service Management left the cluster
2019-07-03 14:52:16.598/4.915 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService left the cluster
2019-07-03 14:52:16.668/4.985 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Service Cluster left the cluster
```

`aaa`年龄本来是`1`，现在是`4`，确实增加了`3`，再调用一次`gettest.sh`，又增加了`3`，值变为`7`了，说明此`EP`可用。

## 样例分析

从上面的样例我们可以看出，要支持`EP`其实要进行这么几步：

1. 首先定义`EP`类，实现`EP`中的`process`方法，
2. 其次要为`EP`类，实现序列化和反序列化。（教程三种提到的两种方法都可以）
3. 最后就是客户端进行`invoke`操作。

经过这几步，`EP`就已经可以使用。多个客户端远程调用同`key`修改，会在服务端节点处排队执行，并不会加锁任何客户端代码。如果不需要返回值，`EP`可以直接返回`null`。

理解`EP`的行为是很重要的，它与数据库中的事务有很大的不同。

首先`EP`是同步的。客户端执行`invoke()`或`invokeAll()`（批量调用`EP`操作）时，客户端将一直阻塞，直到要操作的`key`的所有`coherence`存储节点执行完毕，然后再继续下一行代码。

`invoke()`是一个原子操作。它要么完全成功，要么完全失败。如果失败，则一致性执行回滚。

但是，`invokeAll()`不是原子操作。如果客户端执行`invokeAll()`失败，则可能有部分成功，部分失败（分布式存储，不同`key`节点分布在不同存储节点上，各自执行各自`key`的`EP`，完全异步，`coherence`不做控制）。

如果服务端节点在`EP`的中间挂了，则`coherence`将保证`EP`在备份节点上重试。如果`EP`执行发生异常，将会不断重试。

## 调用服务

还记得我在[我的Oracle Coherence教程(一)](https://www.qiubinren.com/2018-08-03-我的oraclecoherence教程一.html/)中介绍服务的时候，说的后续会介绍的”调用服务“么？

调用服务类似于`EntryProcessor`，不同之处在于它不是针对缓存中的数据条目执行，而是针对集群的一个或多个成员执行。

概念是相同的，调用代理和`EP`一样会被序列化并发送到`Coherence`服务端以供执行。

因为它们不针对缓存中的数据条目执行，所以它们更适合于群集中的管理任务。与只能同步调用的`EP`不同，可以同步或异步执行可调用代理。

与确保每个`key`只处理成功一次且仅一次处理成功的`EP`不同，调用服务仅保证它将尝试运行一次代码。任何失败都是开发人员的责任。应用没有办法确定代码是否成功运行，因此调用服务仅适用于非关键或幂等任务。

我们公司的配置网格定时加载程序就是通过调用服务实现的。

## 调用服务示例

继续使用上面的集群。这边演示一个使用代理进行同步服务调用。

下面是使用`AbstractInvocable`类创建可调用代理的示例。实现了一个名为`MemberStatus`的类，实现`run`方法，获取该执行该任务的节点所在计算机的`cpu`数量，返回给客户端。

`src`目录`MemberStatus.java`文件。

``` java
package invoke;

import com.tangosol.net.AbstractInvocable;

public class MemberStatus extends AbstractInvocable {

    @Override
    public void run() {
        setResult(Runtime.getRuntime().availableProcessors());
    }
}
```

这个任务是客户端传给服务端的，这边很明显，又要定义序列化和反序列化了。增加`MemberStatusSerializer.java`文件。

``` java
package invoke;

import com.tangosol.io.pof.PofReader;
import com.tangosol.io.pof.PofSerializer;
import com.tangosol.io.pof.PofWriter;
import java.io.IOException;


public class MemberStatusSerializer implements PofSerializer {

    @Override
    public void serialize(PofWriter pofWriter, Object o) throws IOException {
        pofWriter.writeRemainder(null);
    }

    @Override
    public Object deserialize(PofReader pofReader) throws IOException {
        MemberStatus memberStatus = new MemberStatus();
        pofReader.readRemainder();
        return memberStatus;
    }
}
```

创建`invoke`目录，编译成`invoke.jar`包。

``` shell
mkdir invoke
mv MemberStatus.java invoke/
mv MemberStatusSerializer.java invoke/
javac -classpath ../../lib/coherence.jar invoke/*.java
jar -cvf ../lib/invoke.jar invoke/*.class
```

修改`pof-config-biz.xml `配置文件，增加`invoke`类的序列化和反序列化关系。

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
        <user-type>
            <type-id>1002</type-id>
            <class-name>biz.UsersEp</class-name>
            <serializer>
                <class-name>biz.UsersEpSerializer</class-name>
            </serializer>
        </user-type>
        <user-type>
            <type-id>1003</type-id>
            <class-name>invoke.MemberStatus</class-name>
            <serializer>
                <class-name>invoke.MemberStatusSerializer</class-name>
            </serializer>
        </user-type>
    </user-type-list>
    <enable-references>true</enable-references>
</pof-config>
```

其中`AbstractInvocable`继承`Invocable`和`Serializable`接口。

`Invocable`接口中有`run()`、`setResult()`和`getResult()`三个方法，我们继承`AbstractInvocable`之后需要实现`run`方法，把结果`setResult()`，客户端通过调用`getResult()`获取结果。

修改`TestCacheGet.java`文件。

``` java
package testcacheget;

import com.tangosol.net.CacheFactory;
import com.tangosol.coherence.component.util.safeService.SafeInvocationService;
import com.tangosol.coherence.component.net.Member;
import invoke.MemberStatus;
import java.util.Map;

public class TestCacheGet {

    public static void main(String[] args) {
        CacheFactory.ensureCluster();
        SafeInvocationService service = (SafeInvocationService)CacheFactory.getService("InvocationService");
        Map<Member, Integer> numCpuMap = service.query(new MemberStatus(), null);
        for(Map.Entry<Member, Integer> numCpu : numCpuMap.entrySet()) { 
            System.out.println("Member:" + numCpu.getKey() + " Numbers of CPUs: " + numCpu.getValue());
        }
        CacheFactory.shutdown();
    }
}
```

修改后的`TestCacheGet`类代码执行步骤如下：

1. 使用`CacheFactory.getService()`获取了名为`example-InvocationService`的服务。
2. 使用获取到的服务引用调用`query()`方法，传入的类是刚才我们新增的`MemberStatus`类，这步会导致所有注册名为`example-InvocationService`的节点同步执行这个任务。
3. 依次输出获取到的结果。

重新编译。

``` shell
javac -classpath ../../lib/coherence.jar:../lib/biz.jar:../lib/invoke.jar TestCacheGet.java
mv TestCacheGet.class testcacheget
jar -cvf ../lib/testget.jar testcacheget/
```

在服务端注册我们的调用服务。修改服务端配置文件`coherence-cache-config.xml`。

``` xml
<?xml version="1.0"?>

<!DOCTYPE cache-config SYSTEM "cache-config.dtd">

<cache-config>

  <caching-scheme-mapping>
    <cache-mapping>
      <cache-name>repl-*</cache-name>
      <scheme-name>example-replicated</scheme-name>
    </cache-mapping>
  </caching-scheme-mapping>

  <caching-schemes>
    <replicated-scheme>
        <scheme-name>example-replicated</scheme-name>
        <service-name>ReplicatedCache</service-name>

        <backing-map-scheme>
          <local-scheme>
            <scheme-ref>unlimited-backing-map</scheme-ref>
          </local-scheme>
        </backing-map-scheme>

      <autostart>true</autostart>
    </replicated-scheme>

    <local-scheme>
        <scheme-name>unlimited-backing-map</scheme-name>
    </local-scheme>

   <invocation-scheme>
         <scheme-name>example-invocation</scheme-name>
         <service-name>InvocationService</service-name>
         <thread-count>1</thread-count>
         <guardian-timeout>7200000</guardian-timeout>
         <autostart system-property="tangosol.coherence.invocation.autostart">true</autostart>
   </invocation-scheme>
  </caching-schemes>
</cache-config>
```

加入`invocation-scheme`标签内的配置。

编辑`coherence-cache-config-proxy.xml `文件。

``` xml
<?xml version="1.0"?>

<!DOCTYPE cache-config SYSTEM "cache-config.dtd">

<cache-config>

  <caching-scheme-mapping>
    <cache-mapping>
      <cache-name>repl-*</cache-name>
      <scheme-name>example-replicated</scheme-name>
    </cache-mapping>
  </caching-scheme-mapping>

  <caching-schemes>
    <replicated-scheme>
        <scheme-name>example-replicated</scheme-name>
        <service-name>ReplicatedCache</service-name>

        <backing-map-scheme>
          <local-scheme>
            <scheme-ref>unlimited-backing-map</scheme-ref>
          </local-scheme>
        </backing-map-scheme>

      <autostart>true</autostart>
    </replicated-scheme>

    <local-scheme>
        <scheme-name>unlimited-backing-map</scheme-name>
    </local-scheme>

     <proxy-scheme>
	<scheme-name>example-proxy-scheme</scheme-name>
	<service-name>example-proxy-service</service-name>
	<thread-count>5</thread-count>
	<acceptor-config>
		<tcp-acceptor>
			<local-address>
				<address system-property="coherence.proxy.address">0.0.0.0</address>
				<port system-property="coherence.proxy.port">10020</port>
			</local-address>
		</tcp-acceptor>
	</acceptor-config>
	<proxy-config>
		<cache-service-proxy>
			<enabled>true</enabled>
		</cache-service-proxy>

    <invocation-service-proxy>
			<enabled>true</enabled>
		</invocation-service-proxy>

	</proxy-config>
	<autostart>true</autostart>
     </proxy-scheme>


    <invocation-scheme>
         <scheme-name>example-invocation</scheme-name>
         <service-name>InvocationService</service-name>
         <thread-count>1</thread-count>
         <guardian-timeout>7200000</guardian-timeout>
         <autostart system-property="tangosol.coherence.invocation.autostart">true</autostart>
   </invocation-scheme>

  </caching-schemes>
</cache-config>
```

加入`cache-service-proxy`的同级标签配置`invocation-service-proxy`，设为`true`。并且在代理端节点注册`InvocationService`。

修改客户端配置文件`coherence-cache-config-client.xml`。

配置客户端本地服务`InvocationService`，配置和上面一致。

这样就有3个节点注册了`InvocationService`服务，注意这边的`service-name`一定要和客户端的`getService`中的一致。

``` xml
<?xml version="1.0"?>

<!DOCTYPE cache-config SYSTEM "cache-config.dtd">

<cache-config>

  <caching-scheme-mapping>
    <cache-mapping>
      <cache-name>repl-*</cache-name>
      <scheme-name>remote-replicated</scheme-name>
    </cache-mapping>
  </caching-scheme-mapping>

  <caching-schemes>

    <remote-cache-scheme>
	<scheme-name>remote-replicated</scheme-name>
	<service-name>ExtendTcpCacheService-REPL</service-name>
	<initiator-config>
	    <tcp-initiator>
		<remote-addresses>
			<socket-address>
				<address>129.211.9.207</address>
				<port>10020</port>
			</socket-address>
                </remote-addresses>
 	    </tcp-initiator>
                <outgoing-message-handler>
                <heartbeat-interval>15s</heartbeat-interval>
                <heartbeat-timeout>120s</heartbeat-timeout>
                <request-timeout>90s</request-timeout>
	    </outgoing-message-handler>
            <connect-timeout>45s</connect-timeout>
	</initiator-config>
    </remote-cache-scheme>
    
    <invocation-scheme>
         <scheme-name>example-invocation</scheme-name>
         <service-name>InvocationService</service-name>
         <thread-count>1</thread-count>
         <guardian-timeout>7200000</guardian-timeout>
         <autostart system-property="tangosol.coherence.invocation.autostart">true</autostart>
   </invocation-scheme>

  </caching-schemes>
</cache-config>
```

最后，修改启动脚本，把新增的`invoke.jar`加入加载包路径。

`startserver.sh`文件，加入`invoke.jar`路径：

``` shell
#!/bin/bash
nohup java -server -showversion -Xms1g -Xmx1g -Dtangosol.coherence.localstorage=true -Dtangosol.coherence.override=../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../pofconfig/pof-config-biz.xml -Dtangosol.coherence.localport=10000 -cp ../lib/coherence.jar:../app/lib/biz.jar:../app/lib/invoke.jar com.tangosol.net.DefaultCacheServer > server.log &
```

`startproxy.sh`文件，加入`invoke.jar`路径：

``` shell
#!/bin/bash
nohup java -server -showversion -Xms1g -Xmx1g -Dtangosol.coherence.localstorage=true -Dtangosol.coherence.override=../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config-proxy.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../pofconfig/pof-config-biz.xml -Dtangosol.coherence.localport=10010 -cp ../lib/coherence.jar:../app/lib/biz.jar:../app/lib/invoke.jar com.tangosol.net.DefaultCacheServer > proxy.log &
```

`gettest.sh`文件，加入`invoke.jar`路径：

``` shell
#!/bin/bash
java -Dtangosol.coherence.override=../../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config-client.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../../pofconfig/pof-config-biz.xml -cp ../lib/testget.jar:../../lib/coherence.jar:../lib/biz.jar:../lib/invoke.jar testcacheget.TestCacheGet
```

启动服务端、代理端：

``` shell
./startserver.sh
./startproxy.sh
```

执行`gettest.sh`后，观察输出。

``` shell
./gettest.sh 

2019-07-05 14:33:40.455/1.053 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-07-05 14:33:40.621/1.219 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-07-05 14:33:40.623/1.221 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-07-05 14:33:40.641/1.239 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-07-05 14:33:40.641/1.239 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-07-05 14:33:40.641/1.240 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-07-05 14:33:41.827/2.425 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-07-05 14:33:41.882/2.481 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:46601 using SystemDatagramSocketProvider
2019-07-05 14:33:42.326/2.924 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-07-05 14:30:48.776, Address=129.211.9.207:10000, MachineId=3986, Location=process:13623, Role=CoherenceServer) joined Cluster with senior member 1
2019-07-05 14:33:42.338/2.936 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=4, Timestamp=2019-07-05 14:33:42.117, Address=129.211.9.207:46601, MachineId=3986, Location=process:14453, Role=TestcachegetTestCacheGet, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-07-05 14:30:48.776, Address=129.211.9.207:10000, MachineId=3986, Location=process:13623, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-07-05 14:33:42.517/3.116 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=2, Timestamp=2019-07-05 14:30:52.983, Address=129.211.9.207:10010, MachineId=3986, Location=process:13652, Role=CoherenceServer) joined Cluster with senior member 1
2019-07-05 14:33:42.591/3.189 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded POF configuration from "file:/home/oracle/oracle/coherence/pofconfig/pof-config-biz.xml"
2019-07-05 14:33:42.673/3.271 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Loaded included POF configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/coherence-pof-config.xml"
2019-07-05 14:33:43.284/3.883 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:46601.59490
2019-07-05 14:33:43.325/3.924 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-07-05 14:33:43.379/3.977 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=4, Timestamp=2019-07-05 14:33:42.117, Address=129.211.9.207:46601, MachineId=3986, Location=process:14453, Role=TestcachegetTestCacheGet)
  OldestMember=Member(Id=1, Timestamp=2019-07-05 14:30:48.776, Address=129.211.9.207:10000, MachineId=3986, Location=process:13623, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=3
    Member(Id=1, Timestamp=2019-07-05 14:30:48.776, Address=129.211.9.207:10000, MachineId=3986, Location=process:13623, Role=CoherenceServer)
    Member(Id=2, Timestamp=2019-07-05 14:30:52.983, Address=129.211.9.207:10010, MachineId=3986, Location=process:13652, Role=CoherenceServer)
    Member(Id=4, Timestamp=2019-07-05 14:33:42.117, Address=129.211.9.207:46601, MachineId=3986, Location=process:14453, Role=TestcachegetTestCacheGet)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-07-05 14:30:48.776|JOINED|12.2.1.3.0,
    2|2019-07-05 14:30:52.983|JOINED|12.2.1.3.0,
    4|2019-07-05 14:33:42.117|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=1
    Member(Id=3, Timestamp=2019-07-05 14:31:53.907, Address=129.211.9.207:36486, MachineId=3986)
    )
  )

TcpRing{Connections=[2]}
IpMonitor{Addresses=0, Timeout=15s}

2019-07-05 14:33:43.443/4.041 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=4): Service Management joined the cluster with senior service member 1
2019-07-05 14:33:43.505/4.103 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-07-05 14:33:44.043/4.642 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Loaded cache configuration from "file:/home/oracle/oracle/coherence/app/config/coherence-cache-config-client.xml"; this document does not refer to any schema definition and has not been validated.
2019-07-05 14:33:44.336/4.934 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=4): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-07-05 14:33:44.391/4.989 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:InvocationService, member=4): Service InvocationService joined the cluster with senior service member 1
Member:Member(Id=1, Timestamp=2019-07-05 14:30:48.776, Address=129.211.9.207:10000, MachineId=3986, Location=process:13623, Role=CoherenceServer) Numbers of CPUs: 1
Member:Member(Id=2, Timestamp=2019-07-05 14:30:52.983, Address=129.211.9.207:10010, MachineId=3986, Location=process:13652, Role=CoherenceServer) Numbers of CPUs: 1
Member:Member(Id=4, Timestamp=2019-07-05 14:33:42.117, Address=129.211.9.207:46601, MachineId=3986, Location=process:14453, Role=TestcachegetTestCacheGet) Numbers of CPUs: 1
2019-07-05 14:33:44.492/5.090 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:InvocationService, member=n/a): Service InvocationService left the cluster
2019-07-05 14:33:44.505/5.103 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=n/a): Service Management left the cluster
2019-07-05 14:33:44.515/5.113 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService left the cluster
2019-07-05 14:33:44.771/5.369 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Service Cluster left the cluster
```

打了`3`行成员节点的`cpu`信息，调用成功。

这边演示了同步的远程调用服务，客户端一旦进行远程调用，就会阻塞，必须等待所有节点执行完毕才能执行下一步，`coherence`的远程调用还提供了异步的方式，需要把`service.query()`换成`service.execute()`方法，这边不再演示。