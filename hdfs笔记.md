### 上传文件夹
hadoop dfs -put /home/root/apache-hive-1.2.1-bin/lib/ /home/root/apache-hive-1.2.1-bin/

### Hadoop native异常报告
```
[ethan@ethan2 ~]$ hdfs dfs -ls /
Java HotSpot(TM) 64-Bit Server VM warning: You have loaded library /home/ethan/workspace_low/hadoop-2.7.3/lib/native/libhadoop.so which might have disabled stack guard. The VM will try to fix the stack guard now.
It's highly recommended that you fix the library with 'execstack -c <libfile>', or link it with '-z noexecstack'.
19/11/04 09:26:03 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
```
####检查so的版本
通过指令 ldd libhadoop.so.1.0.0
```
[ethan@ethan1 native]$ ldd libhadoop.so.1.0.0
	linux-vdso.so.1 =>  (0x00007ffda29b7000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007f5351877000)
	libjvm.so => /home/ethan/jdk1.8.0_221/jre/lib/amd64/server/libjvm.so (0x00007f535088e000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f53504c0000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f5351c9a000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f53501be000)
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f534ffa2000)

```

####不能加载libjvm.so库文件
最终发现这个库找没找到并不影响结果
解决办法： 
(1)在系统中查找这个文件(当然要保证系统中已经有这个.so文件，只是查找路径没有设置正确而已)：
```
[ethan@ethan2 hadoop]$ sudo find / -name libjvm.so
[sudo] ethan 的密码：
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.191-2.6.15.5.el7.x86_64/jre/lib/amd64/server/libjvm.so
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/lib/amd64/server/libjvm.so
/home/ethan/jdk1.8.0_221/jre/lib/amd64/server/libjvm.so
```
发现包含libjvm.so文件的路径不止一个，正确的路径是：/home/ethan/jdk1.8.0_221/jre/lib/amd64/server/libjvm.so 

(2)将.so文件路径的目录添加到/etc/ld.so.conf
 　　sudo vim /etc/ld.so.conf
在文件末尾新添加一行：
/home/ethan/jdk1.8.0_221/jre/lib/amd64/server/

(3)使得修改生效
sudo /sbin/ldconfig

####此环境的真实原因
这个问题是由于目录/$HADOOP_HOME/lib/native下的libhadoop.so
文件格式错误，这个文件本身的格式应该是ELF,如果从windows将配置文件上传到native目录下，所有文件格式都被转换成unix

为了避免类似事件发生，建议以后所有压缩包不要在windows系统下解压再rz上传，会造成不必要的困扰。
