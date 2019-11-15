---
layout: post
title:  "pinpoint安装"
date:   2019-11-15 11:00:00
categories: pinpoint 分布式 安装 链路追踪
permalink: /archivers/pinpoint-steup
---

pinpoint是开源在github上的一款APM监控工具，它是用Java编写的，用于大规模分布式系统监控。它对性能的影响最小（官方说只增加约3％资源利用率），安装agent是无侵入式的，只需要在被测试的应用中加上3句话，打下探针，就可以监控整套程序了。

# 模块构成
pinpoint 主要有`HBase`、`pinpoint-collector`、`pinpoint-web`、`pinpoint-agent` 这四块构成,其中`Hbase` 还会依赖到 `zookeeper`
* HBase 数据主要存储
* pinpoint-collector 数据收集器,看名字也能看得出来
* pinpoint-web 前端页面UI,主要是用来看相关链路,应用信息的
* pinpoint-agent 用于收集应用端监控数据，无侵入式，只需要在启动命令中加入部分参数即可
# 安装
其实安装很简单,参考[pinpoint](https://naver.github.io/pinpoint/1.7.3/installation.html)傻瓜式操作下来就可以了
## Hbase下载安装
- [Hbase 1.2.7下载地址](http://archive.apache.org/dist/hbase/1.2.7/)  
- 下载完成后,解压,然后配置`conf/hbase-site.xml`文件,主要配置一个zk的地址和数据存储的地址,我这边配置是
~~~xml
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>file:/Users/yingbibo/java_tool/hbase-1.2.7/data</value>
  </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.regionserver.handler.count</name>
    <value>20</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>localhost</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
  </property>
  <property>
    <name>zookeeper.session.timeout</name>
    <value>200000</value>
  </property>
</configuration>
~~~
- 然后在/etc/profile 或者 ~.base_profile 下面添加habse的环境变量  
`export HBASE_HOME=/Users/yingbibo/java_tool/hbase-1.2.7/hbase-1.2.7`  
- 启动hbase, `./start-hbase.sh`  
- 添加相应的tables [hbase-create.hbase链接](https://github.com/naver/pinpoint/blob/master/hbase/scripts/hbase-create.hbase) `./hbase shell hbase-create.hbase`  
- 可以在[http://localhost:16010/](http://localhost:16010/)查看相应的信息.
## 部署Pinpoint-collector
- [下载地址](https://github.com/naver/pinpoint/releases/tag/1.8.5),下载完成后,解压放到你自己需要放置的目录
- 修改`WEB-INF⁩/lasses` 下面的 `hbase.properties`和 `pinpoint-collector.properties`
  - ~~~
    #hbase.properties
    hbase.client.host=localhost
    hbase.client.port=2181
    ~~~
  - ~~~
    #pinpoint-collector.properties
    ## 集群模式改成false
    cluster.enable=false 
    cluster.zookeeper.address=localhost
    cluster.zookeeper.sessiontimeout=30000
    cluster.listen.ip=
    cluster.listen.port=
    ~~~ 
  - 启动  
  我这边是使用config文件启动tomcat
    -  config-pinpoint-collector.conf
       ~~~xml
          <?xml version='1.0' encoding='utf-8'?>
        <Server port="19991" shutdown="SHUTDOWN">
          <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
          <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
          <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
          <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
          <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

          <Service name="Catalina">
            <Connector port="19992" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
            <Connector port="19993" protocol="AJP/1.3" redirectPort="8443" />
            <Engine name="Catalina" defaultHost="localhost">
              <Host name="localhost" unpackWARs="false" autoDeploy="false">
                <Context docBase="/Users/yingbibo/java_tool/pinpoint/pinpoint-collector-1.8.5" path="/" reloadable="false" />
                <Valve className="org.apache.catalina.valves.AccessLogValve" directory="/Users/yingbibo/java_tool/pinpoint/logs/" prefix="pinpoint-collector" suffix=".txt" pattern="%h %l %u %t &quot;%r&quot; %s %b" />
              </Host>
            </Engine>
          </Service>
        </Server>
        ~~~
      然后启动 `./startup.sh -config /Users/yingbibo/java_tool/tomcatConfig/config-pinpoint-collector.conf`

## 部署Pinpoint-web
- [下载地址](https://github.com/naver/pinpoint/releases/tag/1.8.5),下载完成后,解压放到你自己需要放置的目录

- 修改`WEB-INF⁩/lasses` 下面的 `hbase.properties`和 `pinpoint-collector.properties`,跟 collector修改方式一样

- 配置config文件,启动tomcat

- 访问对应的端口地址 `http://localhost:19998/`

## Pinpoint-Agent使用

- 我这里主要是jar启动模式,所以在启动前加上VM参数 `java -javaagent:pinpoint-bootstrap-1.8.5.jar -Dpinpoint.agentId=Test-001 -Dpinpoint.applicationName=Test -jar demo.jar`

- 注意:如果发现pinpoint-web,可以抓取到应用的信息,但是没有相应的请求数据,则需要修改agent目录下面的`pinpoint.config`配置文件中的`profiler.springboot.bootstrap.main`去指定相应的main方法
  ~~~
  profiler.springboot.enable=true
  profiler.springboot.bootstrap.main=org.springframework.boot.loader.JarLauncher, org.springframework.boot.loader.WarLauncher, org.springframework.boot.loader.PropertiesLauncher,com.tim.ying.hello.SpringbootHelloApplication
  ~~~


