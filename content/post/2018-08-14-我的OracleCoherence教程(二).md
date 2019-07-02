---
title: "我的Oracle Coherence教程(二)"
date: 2018-08-14T10:45:10+08:00
draft: false
tags: ["Coherence", "Oracle"]
categories: ["Coherence", "Oracle"]
---

## 我的Oracle Coherence教程(二)

## coherence编程模型

上一篇[博客](https://www.qiubinren.com/2018-08-03-我的oraclecoherence教程一.html/)介绍了`coherence`的安装和启动流程，今天来介绍下基于`coherence`的应用该怎么写。

<!--more-->

![p1](/img/2018-08-14-p1.png)

从单个应用的角度来看，`coherence`编程模型是一个简单的模型，可以把`coherence`看成一个类似`Map`的数据映射。但是从多个应用角度来看，多个应用通过同一个名称来访问这个映射，可以共享相同的数据。如图所示，`coherence`中的数据存储其实和`Map`非常类似，一个`key`只能找到一个`Object`。

## 配置个简单Coherence集群

首先我们需要个`coherence`集群，这边先配置必要的配置文件，新建配置文件目录。

``` shell
cd oracle/coherence
mkdir config
cd config
```

创建集群配置文件。

``` shell
touch tangosol-coherence-mycluster.xml
vim tangosol-coherence-mycluster.xml
```

输入以下内容。

``` xml
<?xml version="1.0"?>
<coherence xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://xmlns.oracle.com/coherence/coherence-operational-config">
        <cluster-config>
                 <unicast-listener>
                <well-known-addresses>
                        <socket-address id="1">
                                <address>x.x.x.x</address>
                                <port>10000</port>
                        </socket-address>
                </well-known-addresses>
                </unicast-listener>
        </cluster-config>
        <license-config>
                <edition-name system-property="tangosol.coherence.edition">GE</edition-name>
                <license-mode system-property="tangosol.coherence.mode">prod</license-mode>
        </license-config>
</coherence>
```

这边首先配置集群配置，其中`x.x.x.x`填入自己的服务器`ip`。我们采用`WKA`的方式进行节点通信，一种多节点间类似单播的通信方式。

然后配置缓存配置。

``` shell
touch coherence-cache-config.xml
vim coherence-cache-config.xml
```

输入以下内容。

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

  </caching-schemes>
</cache-config>
```

这边定义了所有缓存名前缀为`repl-`的缓存都使用`example-replicated`方案，该方案是一个全复制缓存方案，所有维护这个全复制缓存的节点的本地内存采用`unlimited-backing-map`方案，`unlimited-backing-map`就是一块不限制大小的内存块，并且这个全复制缓存配置为自启动。

进入`bin`目录，编写这个集群服务端启动脚本。

``` shell
cd ../bin
touch startserver.sh
vim startserver.sh
```

输入内容。

``` shell
#!/bin/bash
nohup java -server -showversion -Xms1g -Xmx1g -Dtangosol.coherence.localstorage=true -Dtangosol.coherence.override=../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config.xml -Dtangosol.coherence.localport=10000 -cp ../lib/coherence.jar com.tangosol.net.DefaultCacheServer > server.log &
```

赋予权限并且启动。

``` shell
chmod 777 startserver.sh
./startserver.sh
```

回车之后，即可查看目录中的`server.log`文件，查看启动日志。

``` shell
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141-b15)
Java HotSpot(TM) 64-Bit Server VM (build 25.141-b15, mixed mode)

2019-06-25 15:56:57.718/1.062 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-06-25 15:56:57.882/1.226 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-06-25 15:56:57.884/1.228 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-06-25 15:56:57.902/1.246 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-06-25 15:56:57.902/1.247 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-06-25 15:56:57.903/1.247 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-06-25 15:56:58.119/1.463 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded cache configuration from "file:/home/oracle/oracle/coherence/config/coherence-cache-config.xml"; this document does not refer to any schema definition and has not been validated.
2019-06-25 15:56:58.471/1.815 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-06-25 15:56:59.201/2.545 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-06-25 15:56:59.268/2.612 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:10000 using SystemDatagramSocketProvider
2019-06-25 15:57:02.926/6.270 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=NameService:TcpAcceptor, member=n/a): TcpAcceptor now listening for connections on VM_0_6_centos:10000.3
2019-06-25 15:57:02.928/6.272 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): NameService now listening for connections on VM_0_6_centos:7574.3
2019-06-25 15:57:02.929/6.273 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Created a new cluster "oracle's cluster" with Member(Id=1, Timestamp=2019-06-25 15:56:59.546, Address=129.211.9.207:10000, MachineId=3986, Location=process:5237, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-06-25 15:57:02.976/6.321 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:10000.41484
2019-06-25 15:57:02.999/6.343 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-06-25 15:57:03.003/6.347 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=1, Timestamp=2019-06-25 15:56:59.546, Address=129.211.9.207:10000, MachineId=3986, Location=process:5237, Role=CoherenceServer)
  OldestMember=Member(Id=1, Timestamp=2019-06-25 15:56:59.546, Address=129.211.9.207:10000, MachineId=3986, Location=process:5237, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=1
    Member(Id=1, Timestamp=2019-06-25 15:56:59.546, Address=129.211.9.207:10000, MachineId=3986, Location=process:5237, Role=CoherenceServer)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-06-25 15:56:59.546|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=0
    )
  )

TcpRing{Connections=[]}
IpMonitor{Addresses=0, Timeout=15s}

2019-06-25 15:57:03.036/6.380 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=1): Service Management joined the cluster with senior service member 1
2019-06-25 15:57:03.130/6.474 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=1): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-06-25 15:57:03.263/6.607 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=1): JMXConnectorServer now listening for connections on 129.211.9.207:37082
2019-06-25 15:57:03.302/6.646 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=1): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-06-25 15:57:03.451/6.795 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=1): Service ReplicatedCache joined the cluster with senior service member 1
2019-06-25 15:57:03.471/6.815 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=1): 
Services
  (
  ClusterService{Name=Cluster, State=(SERVICE_STARTED, STATE_JOINED), Id=0, OldestMemberId=1}
  TransportService{Name=TransportService, State=(SERVICE_STARTED), Id=1, OldestMemberId=1}
  InvocationService{Name=Management, State=(SERVICE_STARTED), Id=2, OldestMemberId=1}
  ReplicatedCache{Name=ReplicatedCache, State=(SERVICE_STARTED), Id=3, OldestMemberId=1}
  )

Started DefaultCacheServer...
```

发现这个`coherence`节点已经启动成功。

写个`coherence`交互客户端的启动脚本，测试下集群是否可用。

``` shell
touch startquery.sh
vim startquery.sh
```

输入下面内容。

``` shell
#!/bin/bash
java -server -showversion -Xms64m -Xmx64m -Dcoherence.distributed.localstorage=false -Dtangosol.coherence.override=../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config.xml -cp /home/oracle/oracle/coherence/lib/coherence.jar com.tangosol.net.CacheFactory
```

赋予权限并且启动`coherence`交互客户端。

``` shell
chmod 777 startquery.sh
./startquery.sh

java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141-b15)
Java HotSpot(TM) 64-Bit Server VM (build 25.141-b15, mixed mode)

2019-06-25 16:30:29.081/1.212 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-06-25 16:30:29.256/1.367 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-06-25 16:30:29.279/1.391 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-06-25 16:30:29.290/1.402 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-06-25 16:30:29.290/1.402 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-06-25 16:30:29.291/1.403 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-06-25 16:30:30.542/2.653 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-06-25 16:30:30.596/2.708 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:38534 using SystemDatagramSocketProvider
2019-06-25 16:30:31.010/3.121 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer) joined Cluster with senior member 1
2019-06-25 16:30:31.030/3.142 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=2, Timestamp=2019-06-25 16:30:30.799, Address=129.211.9.207:38534, MachineId=3986, Location=process:10046, Role=CoherenceConsole, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-06-25 16:30:31.294/3.406 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:38534.51804
2019-06-25 16:30:31.343/3.455 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-06-25 16:30:31.400/3.512 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=2, Timestamp=2019-06-25 16:30:30.799, Address=129.211.9.207:38534, MachineId=3986, Location=process:10046, Role=CoherenceConsole)
  OldestMember=Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=2
    Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer)
    Member(Id=2, Timestamp=2019-06-25 16:30:30.799, Address=129.211.9.207:38534, MachineId=3986, Location=process:10046, Role=CoherenceConsole)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-06-25 16:27:33.939|JOINED|12.2.1.3.0,
    2|2019-06-25 16:30:30.799|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=0
    )
  )

TcpRing{Connections=[1]}
IpMonitor{Addresses=0, Timeout=15s}

2019-06-25 16:30:31.466/3.578 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=2): Service Management joined the cluster with senior service member 1
2019-06-25 16:30:31.533/3.645 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"

Map (?): 
```

使用`cache`命令新建一个名为`repl-Test`的缓存。

``` shell
Map (?): cache repl-Test
2019-06-25 16:31:58.379/90.491 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Loaded cache configuration from "file:/home/oracle/oracle/coherence/config/coherence-cache-config.xml"; this document does not refer to any schema definition and has not been validated.
2019-06-25 16:31:58.483/90.595 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-06-25 16:31:58.546/90.658 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=2): Service ReplicatedCache joined the cluster with senior service member 1

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

Map (repl-Test): 
```

写入一个`key`为`aaa`以及`value`为`1`的条目保存在`coherence`维护的`repl-Test`的全复制缓存。

``` shell
Map (repl-Test): put aaa 1
null

Map (repl-Test): get aaa
1
```

重新开一个窗口，启动另外一个交互命令行。

``` shell
./startquery.sh 

java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141-b15)
Java HotSpot(TM) 64-Bit Server VM (build 25.141-b15, mixed mode)

2019-06-25 16:36:27.446/1.027 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-06-25 16:36:27.609/1.190 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-06-25 16:36:27.611/1.193 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-06-25 16:36:27.637/1.219 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-06-25 16:36:27.638/1.219 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-06-25 16:36:27.638/1.220 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-06-25 16:36:28.834/2.415 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-06-25 16:36:28.875/2.457 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:44844 using SystemDatagramSocketProvider
2019-06-25 16:36:29.095/2.676 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Failed to satisfy the variance: allowed=16, actual=21
2019-06-25 16:36:29.095/2.677 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): Increasing allowable variance to 17
2019-06-25 16:36:29.478/3.059 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer) joined Cluster with senior member 1
2019-06-25 16:36:29.490/3.072 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=3, Timestamp=2019-06-25 16:36:29.26, Address=129.211.9.207:44844, MachineId=3986, Location=process:10979, Role=CoherenceConsole, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-06-25 16:36:29.663/3.244 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=2, Timestamp=2019-06-25 16:30:30.799, Address=129.211.9.207:38534, MachineId=3986, Location=process:10046, Role=CoherenceConsole) joined Cluster with senior member 1
2019-06-25 16:36:29.935/3.517 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:44844.60792
2019-06-25 16:36:29.977/3.558 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-06-25 16:36:30.014/3.596 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=3, Timestamp=2019-06-25 16:36:29.26, Address=129.211.9.207:44844, MachineId=3986, Location=process:10979, Role=CoherenceConsole)
  OldestMember=Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=3
    Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer)
    Member(Id=2, Timestamp=2019-06-25 16:30:30.799, Address=129.211.9.207:38534, MachineId=3986, Location=process:10046, Role=CoherenceConsole)
    Member(Id=3, Timestamp=2019-06-25 16:36:29.26, Address=129.211.9.207:44844, MachineId=3986, Location=process:10979, Role=CoherenceConsole)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-06-25 16:27:33.939|JOINED|12.2.1.3.0,
    2|2019-06-25 16:30:30.799|JOINED|12.2.1.3.0,
    3|2019-06-25 16:36:29.26|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=0
    )
  )

TcpRing{Connections=[2]}
IpMonitor{Addresses=0, Timeout=15s}

2019-06-25 16:36:30.072/3.654 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=3): Service Management joined the cluster with senior service member 1
2019-06-25 16:36:30.112/3.694 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=3): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"

Map (?): cache repl-Test
2019-06-25 16:36:52.677/26.258 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=3): Loaded cache configuration from "file:/home/oracle/oracle/coherence/config/coherence-cache-config.xml"; this document does not refer to any schema definition and has not been validated.
2019-06-25 16:36:52.800/26.382 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=3): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-06-25 16:36:52.884/26.466 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=3): Service ReplicatedCache joined the cluster with senior service member 1

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

Map (repl-Test): get aaa
1
```

进入`repl-Test`缓存后，通过`get`直接获取`aaa`的值成功，`coherence`集群可用。

## 编写简单应用读写coherence

首先`ctrl+c`退出刚才启动的两个`coherence`交互客户端。保持只启动一个`coherence`节点，这个节点目前维护了一个名为`repl-Test`的缓存，缓存里存放着`key`为`aaa`，`value`为`1`的记录。

现在我们来写个应用来跟`coherence`交互。

建立`app/bin`，`app/lib`，`app/src`三个目录。

``` shell
cd ..
mkdir app
cd app
mkdir bin
mkdir src
mkdir lib
```

在`app/src`中写一个编辑简单的`java`应用。

``` shell
cd src
```

这边写两个`java`应用，一个往`coherence`里写，一个从`coherence`中读。

`TestCachePut.java`文件

```java
package testcacheput;

import com.tangosol.net.CacheFactory;
import com.tangosol.net.NamedCache;

public class TestCachePut {

    public static void main(String[] args) {
        String key = "key";
        CacheFactory.ensureCluster();
        NamedCache cache = CacheFactory.getCache("repl-Test");
        cache.put(key, "bbb");
        CacheFactory.shutdown();
        System.out.println("写入key成功");
    }
}
```

`TestCacheGet.java`文件

``` java
package testcacheget;

import com.tangosol.net.CacheFactory;
import com.tangosol.net.NamedCache;

public class TestCacheGet {

    public static void main(String[] args) {
        String key = "key";
        CacheFactory.ensureCluster();
        NamedCache cache = CacheFactory.getCache("repl-Test");
        cache.get(key);
        Object o = cache.get(key);
        System.out.println(o);
        System.out.println(cache.get("aaa"));
        CacheFactory.shutdown();
    }
}
```

编译成两个`.class`文件。

``` shell
javac -classpath ../../lib/coherence.jar TestCachePut.java
javac -classpath ../../lib/coherence.jar TestCacheGet.java
```

很明显，目录下多了两个`.class`文件。

``` shell
ll
total 16
-rw-rw-r-- 1 oracle oracle 807 Jun 27 14:23 TestCacheGet.class
-rw-rw-r-- 1 oracle oracle 478 Jun 27 14:16 TestCacheGet.java
-rw-rw-r-- 1 oracle oracle 819 Jun 27 14:23 TestCachePut.class
-rw-rw-r-- 1 oracle oracle 420 Jun 27 14:22 TestCachePut.java
```

接着用`jar`命令将两个文件分别打包成`jar`包。

``` shell
mkdir testcacheget
mv TestCacheGet.class testcacheget
jar -cvf ../lib/testget.jar testcacheget/

added manifest
adding: testcacheget/(in = 0) (out= 0)(stored 0%)
adding: testcacheget/TestCacheGet.class(in = 807) (out= 483)(deflated 40%)

mkdir testcacheput
mv TestCachePut.class testcacheput
jar -cvf ../lib/testput.jar testcacheput/

added manifest
adding: testcacheput/(in = 0) (out= 0)(stored 0%)
adding: testcacheput/TestCachePut.class(in = 819) (out= 507)(deflated 38%)
```

编写测试程序启动脚本。

``` shell
cd ../bin
vim puttest.sh 
```

输入下列内容。

``` shell
java -Dtangosol.coherence.override=../../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../../config/coherence-cache-config.xml -cp ../lib/testput.jar:../../lib/coherence.jar testcacheput.TestCachePut
```

赋予权限后执行。

``` shell
chmod 777 puttest.sh
./puttest.sh

2019-06-27 15:26:14.908/1.067 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-06-27 15:26:15.041/1.201 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-06-27 15:26:15.125/1.285 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-06-27 15:26:15.129/1.288 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-06-27 15:26:15.129/1.288 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-06-27 15:26:15.129/1.289 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-06-27 15:26:16.232/2.392 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-06-27 15:26:16.284/2.443 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:42592 using SystemDatagramSocketProvider
2019-06-27 15:26:16.690/2.850 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer) joined Cluster with senior member 1
2019-06-27 15:26:16.704/2.864 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=2, Timestamp=2019-06-27 15:26:16.493, Address=129.211.9.207:42592, MachineId=3986, Location=process:2712, Role=TestcacheputTestCachePut, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-06-27 15:26:17.007/3.167 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:42592.63116
2019-06-27 15:26:17.053/3.213 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-06-27 15:26:17.154/3.314 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=2, Timestamp=2019-06-27 15:26:16.493, Address=129.211.9.207:42592, MachineId=3986, Location=process:2712, Role=TestcacheputTestCachePut)
  OldestMember=Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=2
    Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer)
    Member(Id=2, Timestamp=2019-06-27 15:26:16.493, Address=129.211.9.207:42592, MachineId=3986, Location=process:2712, Role=TestcacheputTestCachePut)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-06-25 16:27:33.939|JOINED|12.2.1.3.0,
    2|2019-06-27 15:26:16.493|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=0
    )
  )

TcpRing{Connections=[1]}
IpMonitor{Addresses=0, Timeout=15s}

2019-06-27 15:26:17.219/3.379 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=2): Service Management joined the cluster with senior service member 1
2019-06-27 15:26:17.281/3.441 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-06-27 15:26:17.829/3.989 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Loaded cache configuration from "file:/home/oracle/oracle/coherence/config/coherence-cache-config.xml"; this document does not refer to any schema definition and has not been validated.
2019-06-27 15:26:17.935/4.095 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=2): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-06-27 15:26:17.992/4.152 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=2): Service ReplicatedCache joined the cluster with senior service member 1
2019-06-27 15:26:18.073/4.233 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=n/a): Service ReplicatedCache left the cluster
2019-06-27 15:26:18.079/4.239 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=n/a): Service Management left the cluster
2019-06-27 15:26:18.083/4.242 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService left the cluster
2019-06-27 15:26:18.302/4.461 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Service Cluster left the cluster
写入key成功
```

很明显，应用执行成功，写入成功，现在网格中应该有`key`为`aaa`和`key`为`key`的两个条目数据，我们再启动个应用读取一下，首先写入文件`gettest.sh`。

``` shell
#!/bin/bash

java -Dtangosol.coherence.override=../../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../../config/coherence-cache-config.xml -cp ../lib/testget.jar:../../lib/coherence.jar testcacheget.TestCacheGet
```

赋予权限后启动。

``` shell
chmod 777 gettest.sh
./gettest.sh

#!/bin/bash

java -Dtangosol.coherence.override=../../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../../config/coherence-cache-config.xml -cp ../lib/testget.jar:../../lib/coherence.jar testcacheget.TestCacheGet
[oracle@VM_0_6_centos bin]$ ./gettest.sh 
2019-06-27 15:31:38.269/1.289 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-06-27 15:31:38.451/1.471 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-06-27 15:31:38.469/1.489 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "file:/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml"
2019-06-27 15:31:38.473/1.493 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-06-27 15:31:38.473/1.494 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-06-27 15:31:38.474/1.494 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-06-27 15:31:39.881/2.902 Oracle Coherence GE 12.2.1.3.0 <Warning> (thread=main, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
2019-06-27 15:31:39.940/2.960 Oracle Coherence GE 12.2.1.3.0 <D4> (thread=main, member=n/a): TCMP bound to VM_0_6_centos:43779 using SystemDatagramSocketProvider
2019-06-27 15:31:40.410/3.430 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer) joined Cluster with senior member 1
2019-06-27 15:31:40.420/3.440 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=Cluster, member=n/a): This Member(Id=3, Timestamp=2019-06-27 15:31:40.208, Address=129.211.9.207:43779, MachineId=3986, Location=process:3437, Role=TestcachegetTestCacheGet, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1) joined cluster "oracle's cluster" with senior Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer, Edition=Grid Edition, Mode=Production, CpuCount=1, SocketCount=1)
2019-06-27 15:31:40.706/3.727 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService is bound to tmb://129.211.9.207:43779.38040
2019-06-27 15:31:40.739/3.759 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService joined the cluster with senior service member 1
2019-06-27 15:31:40.762/3.782 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Started cluster Name=oracle's cluster, ClusterPort=7574

WellKnownAddressList(
  129.211.9.207:10000
  )

MasterMemberSet(
  ThisMember=Member(Id=3, Timestamp=2019-06-27 15:31:40.208, Address=129.211.9.207:43779, MachineId=3986, Location=process:3437, Role=TestcachegetTestCacheGet)
  OldestMember=Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer)
  ActualMemberSet=MemberSet(Size=2
    Member(Id=1, Timestamp=2019-06-25 16:27:33.939, Address=129.211.9.207:10000, MachineId=3986, Location=process:9623, Role=CoherenceServer)
    Member(Id=3, Timestamp=2019-06-27 15:31:40.208, Address=129.211.9.207:43779, MachineId=3986, Location=process:3437, Role=TestcachegetTestCacheGet)
    )
  MemberId|ServiceJoined|MemberState|Version
    1|2019-06-25 16:27:33.939|JOINED|12.2.1.3.0,
    3|2019-06-27 15:31:40.208|JOINED|12.2.1.3.0
  RecycleMillis=1200000
  RecycleSet=MemberSet(Size=1
    Member(Id=2, Timestamp=2019-06-27 15:26:18.287, Address=129.211.9.207:42592, MachineId=3986)
    )
  )

TcpRing{Connections=[1]}
IpMonitor{Addresses=0, Timeout=15s}

2019-06-27 15:31:40.815/3.836 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=3): Service Management joined the cluster with senior service member 1
2019-06-27 15:31:40.861/3.882 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=3): Loaded Reporter configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/reports/report-group.xml"
2019-06-27 15:31:41.245/4.265 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=3): Loaded cache configuration from "file:/home/oracle/oracle/coherence/config/coherence-cache-config.xml"; this document does not refer to any schema definition and has not been validated.
2019-06-27 15:31:41.362/4.383 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=3): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-06-27 15:31:41.420/4.440 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=3): Service ReplicatedCache joined the cluster with senior service member 1
bbb
1
2019-06-27 15:31:41.535/4.556 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=ReplicatedCache, member=n/a): Service ReplicatedCache left the cluster
2019-06-27 15:31:41.545/4.565 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Invocation:Management, member=n/a): Service Management left the cluster
2019-06-27 15:31:41.550/4.571 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Transport:TransportService, member=n/a): Service TransportService left the cluster
2019-06-27 15:31:41.783/4.803 Oracle Coherence GE 12.2.1.3.0 <D5> (thread=Cluster, member=n/a): Service Cluster left the cluster
```

一大串输出中找到了两行`bbb`和`1`，这确实是我们上面写入`coherence`的两个`key`的`value`值，至此，最简单的网格编程已经展现出来了，下面详细说下，上面为什么这么写。

## Coherence编程基础

从上面的例子中，我们可以发现`coherence.jar`中给我们提供了一些`api`，让我们可以像操作`java.util.Map`一样来操作`coherence`来存放数据、和其他应用程序交互。

以`TestCachePut`为例，一个`coherence`进程其实跟网格有关的有`5`步操作。

1. 导入`coherence.jar`中的`CacheFactory`和`NamedCache`两个类。
2. 第二步是可选的，通过`CacheFactory.ensureCluster()`方法强制初始化集群，以防止我们要操作的`Cache`是默认或者懒加载的。
3. 使用缓存工厂通过缓存名获取一个命名缓存操作对象。
4. 像使用`java.util.Map`一样来使用这个命名缓存操作对象。
5. 第五步也是可选的，操作完成后使用`CacheFactory.shutdown()`方法释放当前所有分配的资源。这个操作执行之后，再使用上面的操作对象会报错。

一般完成上面的五步，一个最简单的操作`coherence`的网格就完成了。

有些同学可能这个时候会产生一个疑问，`java.util.Map`的实现有`HashMap`和`ConcurrentHashMap`，一个线程不安全一个线程安全，如果我们这个时候两个应用同时操作`coherence`中同一个`key`，`coherence`可以保证线程安全么？

答案是可以的，`coherence`中的`get`和`put`，以及这边没有介绍的`getAll`和`putAll`都属于原子操作，多个应用同时操作相同的`key`并不会有线程安全的问题。

至此`coherence`最简单的编程模型介绍完毕，学习完本博客，你已经有了一个`coherence`全复制缓存集群，以及两个简单的和`coherence`交互的应用程序，这对后面的教学很重要。
