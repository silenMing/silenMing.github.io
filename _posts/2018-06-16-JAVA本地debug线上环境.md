---
layout:     post
title:      JAVA Spring debug线上环境
subtitle:   
date:       2018-06-16
author:     BY
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - JAVA 
    - Spring
---

####jar 命令开启远程调试
java -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=8000,suspend=n -jar demo.jar


当我们运行一个项目的时候，一般都是在本地进行debug。但是如果是一个分布式的微服务，这时候我们选择远程debug是我们开发的利器。
环境
apache-tomcat-8.5.16
Linux
如何启用远程调试
tomcat开启远程调试
方法
切换到你的tomcat的bin目录/apache-tomcat-8.5.16/bin 
下，执行:
./catalina.sh jpda start  

执行上面的命令就可以开启远程debug了，如果想配置一些信息，比如端口号什么的，请参考下面的说明。
参数说明

####

    #   JPDA_TRANSPORT  (Optional) JPDA transport used when the "jpda   start"
    #                   command is executed. The default is "dt_socket".
    #
    #   JPDA_ADDRESS    (Optional) Java runtime options used when the "jpda start"
    #                   command is executed. The default is localhost:8000.
    #
    #   JPDA_SUSPEND    (Optional) Java runtime options used when the "jpda start"
    #                   command is executed. Specifies whether JVM should suspend
    #                   execution immediately after startup. Default is "n".
    #
    #   JPDA_OPTS       (Optional) Java runtime options used when the "jpda start"
    #                   command is executed. If used, JPDA_TRANSPORT, JPDA_ADDRESS,
    #                   and JPDA_SUSPEND are ignored. Thus, all required jpda
    #                   options MUST be specified. The default is:
    #
    #                   -agentlib:jdwp=transport=$JPDA_TRANSPORT,
    #                       address=$JPDA_ADDRESS,server=y,suspend=$JPDA_SUSPEND

####操作说明
所以如果想修改配置，则如下操作：
在catalina.sh中进行配置：
JPDA_TRANSPORT=dt_socket  
JPDA_ADDRESS=5005  
JPAD_SUSPEND=n  

或者通过JPDA_OPTS进行配置：
JPDA_OPTS='-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005’

####springboot开启远程调试
远程调试maven设置
The run goal forks a process for the boot application. It is possible to specify jvm arguments to that forked process. The following configuration suspend the process until a debugger has joined on port 5005

```
<project>
  ...
  <build>
    ...
    <plugins>
      ...
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>1.1.12.RELEASE</version>
        <configuration>
          <jvmArguments>
            -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=5005
          </jvmArguments>
        </configuration>
        ...
      </plugin>
      ...
    </plugins>
    ...
  </build>
  ...
</project>

```
These arguments can be specified on the command line as well, make sure to wrap that properly, that is:
mvn spring-boot:run -Drun.jvmArguments="-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=5005"

####jar 命令开启远程调试
在执行jar的时候，添加上参数。如下：
java -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=8000,suspend=n -jar demo.jar

如果想深入了解Java调试，那么去看一下这个吧。深入Java调试体系
问题
如果出现Connection refused。
首先检查一下端口8000的使用情况:
use:~/tomcat/logs # netstat -an|grep 8000
cp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN


可见当前16808端口服务被绑定了回环地址，外部无法访问。即本地调试。
办法：
修改catalina.sh中一个参数。
if [ -z "$JPDA_TRANSPORT" ]; then
    JPDA_TRANSPORT="dt_socket"
  fi
  if [ -z "$JPDA_ADDRESS" ]; then
    JPDA_ADDRESS="0.0.0.0:8000"
  fi
  if [ -z "$JPDA_SUSPEND" ]; then
    JPDA_SUSPEND="n"
  fi

对JPDA_ADDRESS="localhost:8000" 把默认值（localhost:8000）改成0.0.0.0:8000。默认是本地ip调试也就是无法远程调试，0.0.0.0表示所有ip地址都能调试。
远程连上后再看端口情况：

```
root@VM-198-217-ubuntu:/opt/apache-tomcat-8.5.16/bin# netstat -an | grep 8000
tcp        0      0 10.133.198.217:8000     60.177.99.27:49998      ESTABLISHED

```
断开后是这样的：

```
root@VM-198-217-ubuntu:/opt/apache-tomcat-8.5.16/bin# netstat -an | grep 8000
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN
tcp        0      0 10.133.198.217:8000     60.177.99.27:49998      TIME_WAIT
```

idea连接远程端口进行远程debug
我已经在服务器上开启了远程调试，idea连接的步骤，直接上图。

![](http://silenblog.oss-cn-beijing.aliyuncs.com/709000652417.png)

edit configurations

远程调试配置 

![](http://silenblog.oss-cn-beijing.aliyuncs.com/709000822983.png)

参数配置 

![](http://silenblog.oss-cn-beijing.aliyuncs.com/709000850533.png)

将红框内的地址和端口号改成自己的。

![](http://silenblog.oss-cn-beijing.aliyuncs.com/709000929536.png)

启动远程调试 

![](http://silenblog.oss-cn-beijing.aliyuncs.com/709001035942.png)

成功界面

![](http://silenblog.oss-cn-beijing.aliyuncs.com/709001108492.png)

请求一下试试

![](http://silenblog.oss-cn-beijing.aliyuncs.com/709001136707.png)

> 本文引用 http://blog.csdn.net/WSYW126



