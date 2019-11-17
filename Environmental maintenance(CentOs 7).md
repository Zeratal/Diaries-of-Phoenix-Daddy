-------------------

[TOC]


## JAVA环境

### 环境默认JAVA
在虚拟机安装了CentOS7，检查环境配置发现如下,其版本连接路径是：
/usr/bin/java   （软连接至 /etc/alternatives/java）

/etc/alternatives/java 软连接至 usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/java
```
[ethan@CentOS-Ethan bin]$ java -version
openjdk version "1.8.0_181"
OpenJDK Runtime Environment (build 1.8.0_181-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)
```


###默认JAVA软连接和实际位置 
- 通过检查PATH包含各路径，在/usr/bin下可以看到java的软连接链向/etc/alternatives/
注：这里javaws.itweb是什么，什么时候被被创建
```
[ethan@CentOS-Ethan bin]$ ll /usr/bin |grep java
lrwxrwxrwx.   1 root root          22 8月  14 10:13 java -> /etc/alternatives/java
lrwxrwxrwx.   1 root root          24 8月  14 10:15 javaws -> /etc/alternatives/javaws
-rwxr-xr-x.   1 root root        5530 4月  11 2018 javaws.itweb
```
- /etc/alternatives/路径下包含大量软连接，其中：
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
-  /usr/lib/jvm有2个java版本，其仅包含jre（不包含jdk），此目录下其他软连接要么直接指向这2个目录的jre，要么通过/etc/alternatives/再连接回本路径的两个版本之一的jre
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

## 环境变量设置测试
### 增加JAVA环境变量，如下，在不同脚本修改，效果不同
```
export JAVA_HOME=/home/ethan/jdk1.8.0_221
export PATH=$PATH:$JAVA_HOME/bin
```
#### 加入 /ethan/.bashrc 结尾
| Item      |    non-login| xshell login|
| :-------- | --------:| :--: |
| root| /usr/local/sbin<br>/usr/local/bin<br>/usr/sbin<br>/usr/bin |  /usr/local/sbin<br>/usr/local/bin<br>/usr/sbin<br>/usr/bin<br>/root/bin   |
| ethan|/usr/local/bin<br>/usr/bin<br><br><br>/home/ethan/jdk1.8.0_221/bin|  /usr/local/bin<br>/usr/bin<br>/usr/local/sbin<br>/usr/sbin<br>/home/ethan/jdk1.8.0_221/bin<br>/home/ethan/.local/bin<br>/home/ethan/bin|


```
  用ethan用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/bin:/usr/bin:/home/ethan/jdk1.8.0_221/bin
  用root用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

  用ethan用户通过xshell login登录，直接echo $PATH返回结果：  
    /usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:
    /home/ethan/jdk1.8.0_221/bin:/home/ethan/.local/bin:/home/ethan/bin
  用root用户通过xshell login登录，直接echo $PATH返回结果：
    /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```    

#### 加入 /root/.bashrc 结尾
| Item      |    non-login| xshell login|
| :-------- | --------:| :--: |
| root| /usr/local/sbin<br>/usr/local/bin<br>/usr/sbin<br>/usr/bin<br>/home/ethan/jdk1.8.0_221/bin |  /usr/local/sbin<br>/usr/local/bin<br>/usr/sbin<br>/usr/bin<br>/home/ethan/jdk1.8.0_221/bin<br>/root/bin  |
| ethan|/usr/local/bin<br>/usr/bin|  /usr/local/bin<br>/usr/bin<br>/usr/local/sbin<br>/usr/sbin<br>/home/ethan/.local/bin<br>/home/ethan/bin|
```
  用ethan用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/bin:/usr/bin
  用root用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/home/ethan/jdk1.8.0_221/bin

  用ethan用户通过xshell login登录，直接echo $PATH返回结果：  
    /usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/ethan/.local/bin:/home/ethan/bin
  用root用户通过xshell login登录，直接echo $PATH返回结果：
    /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/home/ethan/jdk1.8.0_221/bin:/root/bin
```

#### 加入 /etc/bashrc 结尾
| Item      |    non-login| xshell login|
| :-------- | --------:| :--: |
| root| /usr/local/sbin<br>/usr/local/bin<br>/usr/sbin<br>/usr/bin<br>/home/ethan/jdk1.8.0_221/bin |  /usr/local/sbin<br>/usr/local/bin<br>/usr/sbin<br>/usr/bin<br>/home/ethan/jdk1.8.0_221/bin<br>/root/bin  |
| ethan|/usr/local/bin<br>/usr/bin<br><br><br>/home/ethan/jdk1.8.0_221/bin|/usr/local/bin<br>/usr/bin<br>/usr/local/sbin<br>/usr/sbin<br>/home/ethan/jdk1.8.0_221/bin<br>/home/ethan/.local/bin<br>/home/ethan/bin|
``` 
  用ethan用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/bin:/usr/bin:/home/ethan/jdk1.8.0_221/bin
  用root用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/home/ethan/jdk1.8.0_221/bin

  用ethan用户通过xshell login登录，直接echo $PATH返回结果：  
    /usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:
    /home/ethan/jdk1.8.0_221/bin:/home/ethan/.local/bin:/home/ethan/bin
  用root用户通过xshell login登录，直接echo $PATH返回结果：
    /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/home/ethan/jdk1.8.0_221/bin:/root/bin
```

#### 加入 /etc/profile 结尾
| Item      |    non-login| xshell login|
| :-------- | --------:| :--: |
| root| /usr/local/sbin<br>/usr/local/bin<br>/usr/sbin<br>/usr/bin|  /usr/local/sbin<br>/usr/local/bin<br>/usr/sbin<br>/usr/bin<br>/home/ethan/jdk1.8.0_221/bin<br>/root/bin  |
| ethan|/usr/local/bin<br>/usr/bin|/usr/local/bin<br>/usr/bin<br>/usr/local/sbin<br>/usr/sbin<br>    /home/ethan/jdk1.8.0_221/bin<br>/home/ethan/.local/bin<br>/home/ethan/bin|
```
  用ethan用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/bin:/usr/bin
  用root用户通过 non-login 交互式访问（使用idea中用例，通过ssh2的client提交命令），返回结果：
    result: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

  用ethan用户通过xshell login登录，直接echo $PATH返回结果：  
    /usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:
    /home/ethan/jdk1.8.0_221/bin:/home/ethan/.local/bin:/home/ethan/bin
  用root用户通过xshell login登录，直接echo $PATH返回结果：
    /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/home/ethan/jdk1.8.0_221/bin:/root/bin
```

## 开发环境搭建

### gcc
```
yum install gcc
gcc --version
```
### cmake
- 卸载原有通过 yum 安装的 cmake
yum remove cmake

- 下载并解压缩，然后增加环境变量
```
export CMAKE_HOME=/home/ethan/cmake-3.16.0-rc3-Linux-x86_64
export PATH=$PATH:$CMAKE_HOME/bin
```
- 检查版本
```
cmake -version
```

## yum本地搭建
### 使用本地yum源

#### 关闭默认repo
修改 /etc/yum.repos.d/CentOS-Base.repo
```
#所有项目全部设置为关闭
enabled=0
```
#### 挂载镜像
- 光驱场景，加载ISO至光驱,如：mount /dev/cdrom /root/CentOS-7-x86_64-DVD-1810.iso /ISO
- ISO文件场景，则需指定挂载，如下：
```
[root@ethan-mini yum.repos.d]# mkdir /ISO
[root@ethan-mini yum.repos.d]# mount -o loop -t iso9660 /root/CentOS-7-x86_64-DVD-1810.iso /ISO
mount: /dev/loop0 写保护，将以只读方式挂载
[root@ethan-mini yum.repos.d]# cd /ISO/
[root@ethan-mini ISO]# ll
总用量 686
-rw-rw-r--. 1 root root     14 11月 26 2018 CentOS_BuildTag
drwxr-xr-x. 3 root root   2048 11月 26 2018 EFI
-rw-rw-r--. 1 root root    227 8月  30 2017 EULA
-rw-rw-r--. 1 root root  18009 12月 10 2015 GPL
drwxr-xr-x. 3 root root   2048 11月 26 2018 images
drwxr-xr-x. 2 root root   2048 11月 26 2018 isolinux
drwxr-xr-x. 2 root root   2048 11月 26 2018 LiveOS
drwxrwxr-x. 2 root root 663552 11月 26 2018 Packages
drwxrwxr-x. 2 root root   4096 11月 26 2018 repodata
-rw-rw-r--. 1 root root   1690 12月 10 2015 RPM-GPG-KEY-CentOS-7
-rw-rw-r--. 1 root root   1690 12月 10 2015 RPM-GPG-KEY-CentOS-Testing-7
-r--r--r--. 1 root root   2883 11月 26 2018 TRANS.TBL

```

#### 启用本地repo
修改 /etc/yum.repos.d/CentOS-Media.repo，将baseurl指向挂载的ISO目录，enabled为1
```
[c7-media]
name=CentOS-$releasever - Media
baseurl=file:///root/ISO
#        file:///media/cdrom/
#        file:///media/cdrom/
#        file:///media/cdrecorder/
gpgcheck=0
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

```
#### 检查版本
通过'yum list'获取新的软件仓库列表
```
[root@ethan-mini yum.repos.d]# yum clean all
已加载插件：fastestmirror
正在清理软件源： c7-media
Cleaning up list of fastest mirrors
[root@ethan-mini yum.repos.d]# yum list|wc -l
4084

```

### 搭建本地yum服务
#### 安装httpd
```
yum install httpd
```
#### 同步repos
- 检查本地repos
```
#这里得到c7-media库，就是前步骤安装的
[root@ethan-mini Packages]# yum repolist
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * c7-media: 
源标识              源名称              状态
c7-media     CentOS-7 - Media        4,021
repolist: 4,021
```
- 创建http的html目录
```
mkdir /var/www/html/new_centos_repos
cd  /var/www/html/new_centos_repos/
```
- 同步repos
```
# reposync r 仓库名（一般为base） -p 目标目录。这里由于当前目录就是目标目录，所以省略参数
reposync --repoid=c7-media
#如果reposync命令没有找到，则先安装yum-utils
yum install yum-utils

#这时 /vaw/www/html/new_centos_repos里面已经有很多包了，但只有软件包
[root@ethan-mini new_centos_repos]# ll /var/www/html/new_centos_repos/c7-media/Packages/ |wc -l
4022

```

- 创建repodate清单
```
createrepo /var/www/html/new_centos_repos/

#如果reposync命令没有找到，则先安装createrepo
yum -y install createrepo

#这时 /vaw/www/html/new_centos_repos里面有repodate清单了
[root@ethan-mini new_centos_repos]# ll /var/www/html/new_centos_repos/repodata |wc -l
8
```
#### 启动yum服务
- 启动http服务
```
[root@ethan-mini new_centos_repos]# service httpd restart
Redirecting to /bin/systemctl restart httpd.service
```

- 创建rops文件

```
 vim /etc/yum.repos.d/new_centos_repos.repo

[new_centos_repos]
name=new_centos_repos
baseurl=http://ip/new_centos_repos
enabled=1
gpgcheck=0
```

- 使用私有的yum
如果其他节点需要添加该yum源，只要在yum.repos.d目录添加以上配置文件，并更新yum cache即可
```
[root@test yum.repos.d]# yum makecache
```
