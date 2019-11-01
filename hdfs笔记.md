### 上传文件夹
hadoop dfs -put /home/root/apache-hive-1.2.1-bin/lib/ /home/root/apache-hive-1.2.1-bin/

### Hadoop native异常报告

####检查so的版本
通过指令 ldd libhadoop.so.1.0.0
···
[ethan@ethan1 native]$ ldd libhadoop.so.1.0.0
	linux-vdso.so.1 =>  (0x00007ffda29b7000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007f5351877000)
	libjvm.so => /home/ethan/jdk1.8.0_221/jre/lib/amd64/server/libjvm.so (0x00007f535088e000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f53504c0000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f5351c9a000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f53501be000)
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f534ffa2000)

···

####不能加载libjvm.so库文件
