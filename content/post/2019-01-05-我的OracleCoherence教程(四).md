---
title: "我的OracleCoherence教程(四)"
date: 2019-01-05T11:31:35+08:00
draft: false
tags: ["Coherence", "Oracle"]
categories: ["Coherence", "Oracle"]
---

# 我的Oracle Coherence教程(四)

## Coherence Proxy

在[我的Oracle Coherence教程(一)](https://www.qiubinren.com/2018-08-03-我的oraclecoherence教程一.html/)中，我介绍了`Coherence`的全复制缓存和分布式缓存，在[我的Oracle Coherence教程(二)](https://www.qiubinren.com/2018-08-14-我的oraclecoherence教程二.html/)中，我又演示了全复制缓存的客户端的配置和编码。

从这两篇博文，相信大家也可以看出，默认情况下，应用要使用`Coherence`的服务，首先要自己成为`Coherence`集群的一个节点（不一定要给其他节点提供服务，比如分布式缓存中可以设置应用启动参数`localstorage=false`）。

这样子很明显是有问题，比如我有个应用处于不稳定的网络环境中，它想要使用`Coherence`集群上的某个全复制缓存里的内容，我们如果让它加入集群成为节点，那么势必会导致整个`Coherence`集群的速度不稳定。

`Coherence`为了避免这种情况的发生，提供了代理模式，允许有部分节点开启代理服务，成为代理节点，为客户端提供服务。

应用客户端需要访问`Coherence`数据，通过`TCP/IP`和`Proxy`实例交互，再由`Proxy`实例在集群内捞取数据后，返回给应用客户端。大体流程如图所示：

![p1](/img/2019-01-05-p1.png)

这就是所谓的`Coherence Proxy`。

<!--more-->

## 新增Proxy实例

继续使用[我的Oracle Coherence教程(三)](https://www.qiubinren.com/2018-08-29-我的oraclecoherence教程三.html/)中一路改进过来的例子。

一直以来，我们的样例集群，服务端只有一个`server`实例。说实话，也不能算是一个严格意义上的集群，我们现在给这个集群新增一个服务端实例。

新增配置文件。

``` shell
cd /home/oracle/oracle/coherence/config
cp coherence-cache-config.xml coherence-cache-config-proxy.xml
```

这边直接拷贝`coherence-cache-config.xml`文件，生成一个`proxy`实例的配置文件。

修改成下面内容，其实就是加入`proxy-scheme`标签中的内容。

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
	</proxy-config>
	<autostart>true</autostart>
     </proxy-scheme>

  </caching-schemes>
</cache-config>
```

到`bin`目录下，新建启动脚本。

``` shell
cd ../bin
cp startserver.sh startproxy.sh
vim startproxy.sh 
```

修改为以下内容。

``` shell
#!/bin/bash
nohup java -server -showversion -Xms1g -Xmx1g -Dtangosol.coherence.localstorage=true -Dtangosol.coherence.override=../config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config-proxy.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../pofconfig/pof-config-biz.xml -Dtangosol.coherence.localport=10010 -cp ../lib/coherence.jar:../app/lib/biz.jar com.tangosol.net.DefaultCacheServer > proxy.log &
```

和`server`实例的启动脚本，有三处不同，使用缓存配置不同，`proxy`实例使用的是我们新增的`coherence-cache-config-proxy.xml`文件，实例监听的端口`localport`不同，输出的日志名不同。

然后，我们启动`startserver.sh`和`startproxy.sh`两个脚本。

``` shell
./startserver.sh
./startproxy.sh
```

## 修改客户端配置文件和启动参数

修改我们客户端的启动脚本和配置，让我们的客户端使用`proxy`方式访问`coherence`集群，不再加入集群。

首先，我们要另外建立客户端的配置，不再和服务端使用同一份配置，首先拷贝一份原配置文件。

``` shell
cd ../app
mkdir config
cd config
cp ../../config/coherence-cache-config.xml coherence-cache-config-client.xml
```

修改配置成为如下内容。

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
				<address>x.x.x.x</address>
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

  </caching-schemes>
</cache-config>
```

这边定义了一个`remote-cache-scheme`远程缓存方案，其中`x.x.x.x`换成自己的`ip`地址。

对客户端来说，所有`repl-`开头的缓存方案和服务端是不一样的，服务端是全复制缓存方案，而这边是远程缓存方案，说明这个缓存是远程的，一旦访问到会根据配置的`ip`去访问`proxy`实例，再由`proxy`实例在集群内捞取后返回给我们数据。

可以修改客户端启动脚本了。

``` shell
cd ../bin/
vim gettest.sh
```

先修改`gettest.sh`，把`cacheconfig`改为我们刚才修改的内容。并且去除`-Dtangosol.coherence.override`配置，增加`-Dtangosol.coherence.tcmp.enabled=false`配置。

``` shell
#!/bin/bash
java -Dtangosol.coherence.tcmp.enabled=false -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config-client.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../../pofconfig/pof-config-biz.xml -cp ../lib/testget.jar:../../lib/coherence.jar:../lib/biz.jar testcacheget.TestCacheGet
```

再修改`puttest.sh`，修改的地方一样。

``` shell
#!/bin/bash
java -Dtangosol.coherence.tcmp.enabled=false -Dtangosol.coherence.cacheconfig=../config/coherence-cache-config-client.xml -Dtangosol.pof.enabled=true -Dtangosol.pof.config=../../pofconfig/pof-config-biz.xml -cp ../lib/testput.jar:../../lib/coherence.jar:../lib/biz.jar testcacheput.TestCachePut
```

这样修改之后，两个客户端其实已经不会加入集群，不再跟集群主节点进行`TCMP`（一种`TCP`和`UDP`混合使用的协议）通信，而转为与代理节点进行`TCP`通信。

## 修改客户端代码

最后我们还需要修改客户端`get`和`put`代码。

还记得，我们前面两个客户端代码里都包含的`CacheFactory.ensureCluster();`和`CacheFactory.shutdown();`这两句么？这两句一旦使用代理就必须删去，代理方式连接集群的客户端只能访问，无法控制集群节点的启停。

修改后，重新编译。

``` shell
javac -classpath ../../lib/coherence.jar:../lib/biz.jar TestCacheGet.java
javac -classpath ../../lib/coherence.jar:../lib/biz.jar TestCachePut.java
mv TestCacheGet.class testcacheget
jar -cvf ../lib/testget.jar testcacheget/
mv TestCachePut.class testcacheput
jar -cvf ../lib/testput.jar testcacheput/
```

然后运行。

``` shell
./puttest.sh 

2019-07-09 09:14:10.972/0.970 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence.xml"
2019-07-09 09:14:11.132/1.129 Oracle Coherence 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded operational overrides from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/tangosol-coherence-override-dev.xml"
2019-07-09 09:14:11.132/1.130 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/tangosol-coherence-override.xml" is not specified
2019-07-09 09:14:11.135/1.133 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-config.xml" is not specified
2019-07-09 09:14:11.135/1.133 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "cache-factory-builder-config.xml" is not specified
2019-07-09 09:14:11.136/1.133 Oracle Coherence 12.2.1.3.0 <D5> (thread=main, member=n/a): Optional configuration override "/custom-mbeans.xml" is not specified

Oracle Coherence Version 12.2.1.3.0 Build 68243
 Grid Edition: Development mode
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

2019-07-09 09:14:11.392/1.389 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Loaded cache configuration from "file:/home/oracle/oracle/coherence/app/config/coherence-cache-config-client.xml"; this document does not refer to any schema definition and has not been validated.
2019-07-09 09:14:11.834/1.831 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=main, member=n/a): Created cache factory com.tangosol.net.ExtensibleConfigurableCacheFactory
2019-07-09 09:14:11.967/1.964 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=ExtendTcpCacheService-REPL:TcpInitiator, member=n/a): Loaded POF configuration from "file:/home/oracle/oracle/coherence/pofconfig/pof-config-biz.xml"
2019-07-09 09:14:12.049/2.047 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=ExtendTcpCacheService-REPL:TcpInitiator, member=n/a): Loaded included POF configuration from "jar:file:/home/oracle/oracle/coherence/lib/coherence.jar!/coherence-pof-config.xml"
2019-07-09 09:14:12.629/2.626 Oracle Coherence GE 12.2.1.3.0 <Info> (thread=ExtendTcpCacheService-REPL:TcpInitiator, member=n/a): The cluster name has not been configured, a value of "oracle's cluster" has been automatically generated
写入users成功
```

一切正常，写入和读取都成功。成功使用`proxy`方式访问`coherence`缓存。