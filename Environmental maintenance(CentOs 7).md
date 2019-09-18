JAVA环境

在虚拟机安装了CentOS7，检查环境配置发现如下,其版本连接路径是：
/usr/bin/java   软连接至 /etc/alternatives/java
/etc/alternatives/java 软连接至 usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/java
```
[ethan@CentOS-Ethan bin]$ java -version
openjdk version "1.8.0_181"
OpenJDK Runtime Environment (build 1.8.0_181-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)
```


通过检查PATH包含各路径，在/usr/bin下可以看到java的软连接
注：这里javaws.itweb是什么，什么时候被被创建
```
[ethan@CentOS-Ethan bin]$ ll /usr/bin |grep java
lrwxrwxrwx.   1 root root          22 8月  14 10:13 java -> /etc/alternatives/java
lrwxrwxrwx.   1 root root          24 8月  14 10:15 javaws -> /etc/alternatives/javaws
-rwxr-xr-x.   1 root root        5530 4月  11 2018 javaws.itweb
```
/etc/alternatives/路径下包含大量软连接，其中：
1，java指向/usr/lib/jvm下java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64下的jre，后续分析这个路径；
2，javaws指向/usr/bin/javaws.itweb，即/usr/bin 下的javaws指向了同目录下javaws.itweb
```
[ethan@CentOS-Ethan alternatives]$ ll |grep java
lrwxrwxrwx. 1 root root 71 8月  14 10:13 java ->                /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/java
lrwxrwxrwx. 1 root root 70 8月  14 10:13 jjs ->                 /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/jjs
lrwxrwxrwx. 1 root root 62 8月  14 10:13 jre ->                 /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre
lrwxrwxrwx. 1 root root 62 8月  14 10:13 jre_1.8.0 ->           /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre
lrwxrwxrwx. 1 root root 62 8月  14 10:13 jre_1.8.0_exports ->   /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre
lrwxrwxrwx. 1 root root 62 8月  14 10:13 jre_openjdk ->         /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre
lrwxrwxrwx. 1 root root 74 8月  14 10:13 keytool ->             /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/keytool
lrwxrwxrwx. 1 root root 62 8月  14 10:13 jre_openjdk_exports -> /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre
lrwxrwxrwx. 1 root root 71 8月  14 10:13 orbd ->                /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/orbd
lrwxrwxrwx. 1 root root 74 8月  14 10:13 pack200 ->             /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/pack200
lrwxrwxrwx. 1 root root 77 8月  14 10:13 policytool ->          /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/policytool
lrwxrwxrwx. 1 root root 71 8月  14 10:13 rmid ->                /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/rmid
lrwxrwxrwx. 1 root root 78 8月  14 10:13 rmiregistry ->         /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/rmiregistry
lrwxrwxrwx. 1 root root 77 8月  14 10:13 servertool ->          /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/servertool
lrwxrwxrwx. 1 root root 76 8月  14 10:13 tnameserv ->           /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/tnameserv
lrwxrwxrwx. 1 root root 76 8月  14 10:13 unpack200 ->           /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/unpack200
lrwxrwxrwx. 1 root root 65 8月  14 10:13 jre_1.7.0 ->           /usr/lib/jvm/java-1.7.0-openjdk-1.7.0.191-2.6.15.5.el7.x86_64/jre

lrwxrwxrwx. 1 root root 21 8月  14 10:15 javaws -> /usr/bin/javaws.itweb

lrwxrwxrwx. 1 root root 37 8月  14 10:15 javaws.1.gz ->               /usr/share/man/man1/javaws.itweb.1.gz
lrwxrwxrwx. 1 root root 29 8月  14 10:13 jaxp_parser_impl ->          /usr/share/java/xerces-j2.jar
lrwxrwxrwx. 1 root root 28 8月  14 10:13 jaxp_transform_impl ->       /usr/share/java/xalan-j2.jar
lrwxrwxrwx. 1 root root 42 8月  14 10:12 servlet ->                   /usr/share/java/tomcat-servlet-3.0-api.jar
lrwxrwxrwx. 1 root root 69 8月  14 10:13 jre_1.7.0_exports ->         /usr/lib/jvm-exports/java-1.7.0-openjdk-1.7.0.191-2.6.15.5.el7.x86_64
lrwxrwxrwx. 1 root root 69 8月  14 10:13 jre_1.7.0_openjdk_exports -> /usr/lib/jvm-exports/java-1.7.0-openjdk-1.7.0.191-2.6.15.5.el7.x86_64
lrwxrwxrwx. 1 root root 66 8月  14 10:13 jre_1.8.0_openjdk_exports -> /usr/lib/jvm-exports/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64
lrwxrwxrwx. 1 root root 27 8月  14 10:15 libjavaplugin.so.x86_64 ->   /usr/lib64/IcedTeaPlugin.so

lrwxrwxrwx. 1 root root 75 8月  14 10:13 java.1.gz ->        /usr/share/man/man1/java-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 74 8月  14 10:13 jjs.1.gz ->         /usr/share/man/man1/jjs-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 78 8月  14 10:13 keytool.1.gz ->     /usr/share/man/man1/keytool-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 75 8月  14 10:13 orbd.1.gz ->        /usr/share/man/man1/orbd-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 78 8月  14 10:13 pack200.1.gz ->     /usr/share/man/man1/pack200-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 81 8月  14 10:13 policytool.1.gz ->  /usr/share/man/man1/policytool-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 75 8月  14 10:13 rmid.1.gz ->        /usr/share/man/man1/rmid-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 82 8月  14 10:13 rmiregistry.1.gz -> /usr/share/man/man1/rmiregistry-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 81 8月  14 10:13 servertool.1.gz ->  /usr/share/man/man1/servertool-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 80 8月  14 10:13 tnameserv.1.gz ->   /usr/share/man/man1/tnameserv-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz
lrwxrwxrwx. 1 root root 80 8月  14 10:13 unpack200.1.gz ->   /usr/share/man/man1/unpack200-java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64.1.gz

其中有6行热点目录的软连接也都指向/usr/lib/jvm/
lrwxrwxrwx. 1 root root 65 8月  14 10:13 jre_1.7.0 ->          /usr/lib/jvm/java-1.7.0-openjdk-1.7.0.191-2.6.15.5.el7.x86_64/jre
lrwxrwxrwx. 1 root root 60 8月  14 10:13 jre_1.7.0_openjdk ->  /usr/lib/jvm/jre-1.7.0-openjdk-1.7.0.191-2.6.15.5.el7.x86_64

lrwxrwxrwx. 1 root root 57 8月  14 10:13 jre_1.8.0_openjdk ->  /usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64
lrwxrwxrwx. 1 root root 62 8月  14 10:13 jre_openjdk ->        /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre
lrwxrwxrwx. 1 root root 62 8月  14 10:13 jre ->                /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre
lrwxrwxrwx. 1 root root 62 8月  14 10:13 jre_1.8.0 ->          /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre

```
/usr/lib/jvm有2个java版本，其仅包含jre（不包含jdk），此目录下其他软连接要么直接指向这2个目录的jre，要么通过/etc/alternatives/再连接回本路径的两个版本之一的jre
```
[ethan@CentOS-Ethan jvm]$ ll /usr/lib/jvm
总用量 0
drwxr-xr-x. 4 root root 100 8月  14 10:13 java-1.7.0-openjdk-1.7.0.191-2.6.15.5.el7.x86_64
drwxr-xr-x. 3 root root  17 8月  14 10:13 java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64
lrwxrwxrwx. 1 root root  21 8月  14 10:13 jre -> /etc/alternatives/jre
lrwxrwxrwx. 1 root root  27 8月  14 10:13 jre-1.7.0 -> /etc/alternatives/jre_1.7.0
lrwxrwxrwx. 1 root root  35 8月  14 10:13 jre-1.7.0-openjdk -> /etc/alternatives/jre_1.7.0_openjdk
lrwxrwxrwx. 1 root root  52 8月  14 10:13 jre-1.7.0-openjdk-1.7.0.191-2.6.15.5.el7.x86_64 -> java-1.7.0-openjdk-1.7.0.191-2.6.15.5.el7.x86_64/jre
lrwxrwxrwx. 1 root root  27 8月  14 10:13 jre-1.8.0 -> /etc/alternatives/jre_1.8.0
lrwxrwxrwx. 1 root root  35 8月  14 10:13 jre-1.8.0-openjdk -> /etc/alternatives/jre_1.8.0_openjdk
lrwxrwxrwx. 1 root root  49 8月  14 10:13 jre-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64 -> java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre
lrwxrwxrwx. 1 root root  29 8月  14 10:13 jre-openjdk -> /etc/alternatives/jre_openjdk
```

###环境变量设置测试
增加JAVA环境变量，如下，在不同脚本修改，效果不同
```
export JAVA_HOME=/home/ethan/jdk1.8.0_221
export PATH=$PATH:$JAVA_HOME/bin
```
加入 /ethan/.bashrc 结尾
```
  用ethan用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/bin:/usr/bin:/home/ethan/jdk1.8.0_221/bin
  用root用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
```    
加入 /etc/bashrc 结尾
``` 
  用ethan用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/bin:/usr/bin:/home/ethan/jdk1.8.0_221/bin
  用root用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/home/ethan/jdk1.8.0_221/bin
```


