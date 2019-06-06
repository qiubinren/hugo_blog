---
title: "我的Oracle Coherence教程(一)"
date: 2018-08-03T21:17:33+08:00
draft: false
tags: ["Coherence", "Oracle"]
categories: ["Coherence", "Oracle"]
---

# 我的Oracle Coherence教程(一)

## 教程缘起

公司从项目云化开始使用`Oracle Coherence`，一直到现在应该有3年多了。这个产品说实话`Oracle`技术支持做的有点差，就一份不知道多久没有更新的官网文档。网上资料几乎没有，可以搜到的中文资料，一股浓浓的机翻味道，难以阅读。产品又不是开源项目，遇到问题只能靠猜，项目上线后因为`Coherence`使用和大大小小的`bug`导致的问题着实不少。

另外发现身边很多同事对`Coherence`的理解可以说偏差很大。大部分只是把它当成单纯的内存库数据库来用，一旦遇到问题也只会求助公司内一二熟悉`Coherence`的人，几乎束手无策。

因为我们组的项目在`coherence`使用方面有点要求，需要我在组内对组员培训一下`Coherence`的机制，既然网上也没啥资料，也在博客上分享一下。

<!--more-->

## 什么是Coherence？

国际惯例，先来介绍`Coherence`是什么。

> `Oracle Coherence`是一款基于`Java`的分布式缓存和内存数据网格。适用于需要高可用性，高可扩展性和低延迟的系统。

## 安装

首先到`Oracle`官网获取`Coherence`版本。官网最高可以下载到`12.2.1.2`版本，如果购买`Oracle`的服务，会提供更高版本。不过无所谓，我们这边就用官网版本。

![p1](/img/2018-08-03-p1.png)

这边我下载的是` Coherence Quick Install `下载后大概`91MB`左右的`zip`文件，解压它

``` shell
unzip  fmw_12.2.1.3.0_coherence_quick_Disk1_1of1.zip 

Archive:  fmw_12.2.1.3.0_coherence_quick_Disk1_1of1.zip
  inflating: fmw_12.2.1.3.0_coherence_quick.jar  
  inflating: fmw_12213_readme.htm    
```

解压出一个`jar`和`html`文件

``` shell
ll

total 178432
-rwxr-xr-x 1 root root 91159096 Jun  4 23:48 fmw_12.2.1.3.0_coherence_quick_Disk1_1of1.zip
-r-xr-xr-x 1 root root 91352357 Aug 22  2017 fmw_12.2.1.3.0_coherence_quick.jar
-rw-r--r-- 1 root root     9063 Aug  9  2017 fmw_12213_readme.htm
```

然后确保本机已经安装当前`coherence`需求的`jdk`版本，`12.2.1.3.0`这个官网版本`jdk1.6`以上都可以，如果是`oracle`提供的最新版`12.2.2.x.x`则需要升级`jdk1.8`以上才可以。这边本机`jdk`为`1.8` 。

``` shell
java -version

java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141-b15)
Java HotSpot(TM) 64-Bit Server VM (build 25.141-b15, mixed mode)
```

如果`jdk`没有安装或者版本不符自行安装，此处不做介绍。

然后创建`oracle`用户进行安装，此处注意，`root`用户无法安装此`jar`，一旦校验到用户权限是`root`就会失败。执行下面的命令安装，通过`ORACLE_HOME`指定`coherence`安装路径。

``` shell
java -jar fmw_12.2.1.3.0_coherence_quick.jar ORACLE_HOME=/home/oracle/oracle
```

等待一大串输出后，安装已经成功。

## coherence目录结构

首先查看`coherence`安装目录的目录结构。

``` shell
tree oracle/coherence/ -d -L 1

oracle/coherence/
|-- bin
|-- lib
`-- plugins
```

可以看到这边主要有`3`个目录，分别介绍下。

`bin`是二进制目录，包含启动`coherence`缓存服务器的所有命令行脚本，以及其他命令行工具脚本。

`lib`是`jar`包目录，包含所有使用`coherence`必须的`jar`文件。

`plugins`是插件目录，包含一些使用`coherence`时，可能需要用到的插件，比如`jvisualvm`。

在安装完成后，我们发现安装程序还会创建多个和`coherence`同级的顶级目录，不是很关键，不做介绍。

``` shell
tree oracle/ -d -L 1

oracle/
|-- coherence
|-- inventory
|-- OPatch
|-- oracle_common
`-- oui
```

然后介绍一下`coherence`基础脚本。`coherence`附带了一组核心脚本文件，可用于`Windows`和`Linux`。这些脚本可以安装后直接使用，用来启动`coherence`实例或者与`coherence`实例交互。脚本包括：

``` shell
tree oracle/coherence/bin/

oracle/coherence/bin/
|-- cache-server.sh
|-- coherence.sh
|-- datagram-test.sh
|-- multicast-test.sh
|-- pof-config-gen.sh
|-- query.sh
`-- readme.txt
```

`cache-server.sh`用于从命令行启动`coherence`实例。可以使用此脚本创建多个实例。

`coherence.sh`用于和正在运行的`coherence`实例交互，根据配置也可以用来启动一个`coherence`实例。

`query.sh`用于启动一个交互式会话，这个交互式会话可以与正在运行的`coherence`集群进行交互查询。

`datagram-test.sh`此脚本启动一个测试，验证当前安装是否提供所需的数据报支持。在部署应用程序之前，最好运行数据报测试来测试实际的网络速度，并确定其推送大量数据的能力。数据报测试最好以越来越多的发布者和消费者运行，因为随着发布者数量的增加，单个发布者和单个消费者看起来很好的网络可能完全崩溃。

## 启动coherence

知道了启动实例脚本后，我们就来启动一个`coherence`实例吧。

``` shell
cd oracle/coherence/bin
./cache-server.sh
```

输入命令后，会输出一大堆信息，这边先关注红色框部分信息，其余信息后续再介绍。

![p2](/img/2018-08-03-p2.png)

首先，这个`coherence`版本是`12.2.1.3.0`网格版，当前运行在开发者模式下。

然后，列出了集群的成员列表，因为目前集群只有当前节点一个成员，所以`Id`为`1`。

最后，集群以多播地址`239.192.0.0`进行节点通信。

## 简单理解coherence配置

`coherence`的配置分为两大类:操作配置和缓存配置。

操作配置是与服务器实例相关的配置。根据定义，这种配置与缓存关联的运行时环境有关，例如，使用的端口号、多播将使用的跃点数以及其他操作信息和配置。操作配置大部分默认配置即可，不需要太多自定义操作配置。

缓存配置是与缓存定义、缓存方案、后备存储、自定义处理等相关的配置。一般部署`coherence`集群前需要配置大量缓存配置。

可能有人会奇怪，刚才启动一个`coherence`实例的时候，我们什么配置都没配置，就启动成功了。这边说明一下`coherence`为做到开箱即用，根据所使用的平台的不同将大量默认配置打包在`.jar`、`.dll`、`.lib`中。

这边我们尝试解压`lib`目录中的`coherence.jar`即可看到大量`xml`配置。

``` shell
jar xvf coherence.jar
...

ls

coherence-application.xsd            coherence-operational-config-base.xsd  internal-txn-cache-config.xml    tangosol-coherence-override-dev.xml
coherence-cache-config-base.xsd      coherence-operational-config.xsd       management-config.xml            tangosol-coherence-override-eval.xml
coherence-cache-config.xml           coherence-pof-config.xml               META-INF                         tangosol-coherence-override-prod.xml
coherence-cache-config.xsd           coherence-pof-config.xsd               native                           tangosol-coherence.xml
coherence-client.xml                 coherence-report-config.xsd            pof-config.xml                   tangosol.dat
coherence-config-base.xsd            coherence-report-group-config.xsd      processor-dictionary.xml         txn-pof-config.xml
coherence-enterprise.xml             coherence-rest-config.xsd              ra.xml                           weblogic-ra.xml
coherence-federation-pof-config.xml  coherence-rtc.xml                      reports
coherence-grid.xml                   coherence-standard.xml                 tangosol-application.properties
coherence.jar                        com                                    tangosol.cer
```

如果有自定义配置（通过命令行或者自定义`xml`文件配置），`coherence`启动时就会读取命令行配置的属性、XML配置文件中的属性，来替换默认配置中的属性。

上面解压出来的文件有`4`类文件需要关注一下。

`tangosol-coherence.xml`是默认的操作配置，通常不会修改它。是`coherence`启动时第一个加载的`xml`文件。

`tangosol-coherence-override-dev.xml`、` tangosol-coherence-override-eval.xml`、`tangosol-coherence-override-prod.xml`这三个文件是指定模式配置文件，`coherence`启动时会根据指定的模式（`Development`或者`Production`）从`classpath`中读取指定的模式的配置文件，然后来覆盖默认配置。在默认操作配置之后加载。

`tangosol-coherence-override.xml`文件上面并没有，这个是可选操作重写配置文件。从`classpath`加载，包含对标准默认操作配置的重写，一般自定义配置都写在这个里面。在指定模式配置文件之后加载。

`coherence-cache-config.xml`文件是缓存部署配置文件，用于定义可以在集群中使用的各种类型的缓存。

`coherence`启动至少需要两个`xml`配置，默认情况它们位于`coherence.jar`中:

`coherence-cache-config.xml`缓存配置文件

`tangosol-coherence.xml`集群配置文件

这边不建议从`coherence.jar`中提取这两个文件修改，可以生成覆盖文件后，通过`coherence`启动时的命令行参数来指定。

``` shell
java -Dtangosol.coherence.override=/home/oracle/oracle/coherence/config/tangosol-coherence-mycluster.xml -Dtangosol.coherence.cacheconfig=/home/oracle/oracle/coherence/config/coherence-cache-config.xml ...
```

这边使用`-Dtangosol.coherence.override`和`-Dtangosol.coherence.cacheconfig`分别设置了缓存配置文件和集群配置文件的位置。

**命令行配置**

`coherence`通常以一组放在`classpath`的`xml`编写的配置文件来配置。不过`coherence`也支持使用命令行参数重写很多公共属性，方便开发人员避免修改和创建`xml`文件快速配置`coherence`。这边简单介绍几个命令行配置，更多配置参考官方文档。

`-Dtangosole.coherence.mode`用于指定开发还是生产模式(`dev`或`prod`)。

`-Dtangosol.coherence.localhost`用于指定实例本地`ip`。

`-Dtangosol.coherence.clusterport`用于指定多播默认集群侦听端口，是多播寻址的一部分（我司使用`WKA`方式，一般不会使用这个配置）。

`-Dtangosol.coherence.localport`用于指定实例启动端口。

`-Dtangosol.coherence.log.level`用于指定网格日志级别。

大家应该也发现了，命令行每个属性必须使用`-D`语法指定，并在`Java`命令行上传递。此外，所有这些属性的前缀都是`tangosol.coherence`。

这边配置部分只是简单介绍结构和几个关键配置，具体支持的配置参数太过繁杂，还是自己官网查询文档。

## 简单理解coherence服务

`coherence`中有个叫服务的概念，首先我们来看看服务的特点:

> 每个服务都具有唯一命名。
>
> 服务提供缓存和其他特定的功能，比如事件和调用。
>
> 在操作配置中定义和配置。
>
> 可以自动启动和按需启动。

`coherence`服务是整个`coherence`的基础设施，`coherence`的一切功能，包括缓存、事件、调用等都是通过服务来实现的。

其实我们可以很明显看出来，一个`coherence`实例节点就是一个`java`进程，那么`coherence`服务其本质就是一组`java`异步线程。

`coherence`启动时会启动各种配置文件中定义好的服务，这些服务在`coherence`中按功能分可以分为两类:缓存服务（`Cache services`）和调用服务（`Invocation services`）。

缓存服务提供`coherence`的基本缓存功能。

调用服务提供了一种在`coherence`各个集群节点上远程执行计算的能力。

调用服务后续介绍，缓存服务这边介绍一下。

缓存服务类型有复制缓存（`Replicated Cache`）和分布式缓存（`Distributed Cache`）两类。

**复制缓存:**

复制缓存就是集群内每个节点都要维护所有的缓存条目的缓存模式。

复制缓存`get`示例:

![p3](/img/2018-08-03-p3.png)

复制缓存`put`示例:

![p4](/img/2018-08-03-p4.png)

**分布式缓存**

分布式缓存是在集群上均匀划分缓存条目的缓存模式。

分布式缓存`get`示例:

![p5](/img/2018-08-03-p5.png)

分布式缓存`put`示例:

![p6](/img/2018-08-03-p6.png)

分布式缓存单节点不可用恢复示例:

![p7](/img/2018-08-03-p7.png)

这边图内每个缓存条目是一备一的，实际生产中可以修改配置，让分布式缓存备份数量增加，提高可用性。

分布式缓存服务还允许将某些集群节点配置为存储数据节点，而将其他节点配置为不存储数据节点。此配置被称为`local storage`。凡是`localstorage=true`的集群节点为分布式缓存提供缓存存储和备份存储。

下面就是一个分布式缓存节点设置`localstorage=true`的示例:

![p8](/img/2018-08-03-p8.png)

## coherence启动步骤

我们现在已经介绍了，`Coherence`配置文件、命令行配置、服务，那么`Coherence`启动的时候到底是什么样的顺序，来加载这些配置的呢？

官网有这么一副图可以解答我们的疑问:

![p9](/img/2018-08-03-p9.png)

从图上我们可以看到，通过启动脚本启动`coherence`后的步骤是这样的:

第一步就是加载`tangosol-coherence.xml`。

第二步加载所有的命令行参数。

第三步加载指定模式重写配置文件(`tangosol-coherence-override-{mode}.xml`)。

第四步加载指定的自定义重写配置文件，至此集群配置全部加载完毕。

第五步开始加载缓存配置文件。

第六步启动所有的配置为自启动（`Autostart`）的服务，服务未配置自启动，默认为懒加载。

第七步，至此`coherence`节点全部初始化完成。
