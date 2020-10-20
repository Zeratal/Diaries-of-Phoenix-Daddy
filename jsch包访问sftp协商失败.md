
使用jsch包访问sftp，上报Algorithm negotiation fail

其原因是jsch的版本太低，0.1.42版本，其加密算法协商失败


# 参数修改一
#HostKeyAlgorithms ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,ssh-rsa,ssh-dss
KexAlgorithms ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha256
MACs hmac-sha2-256,hmac-sha2-512,hmac-sha1


# 参数修改二
https://blog.csdn.net/weixin_33897722/article/details/93885975?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.channel_param

ssh连接问题是由于主机ssh中缺少与jsch jar包匹配的加密算法导致，jsch jar包的默认加密算法貌似是diffie-hellman-group-exchange-sha1。
在目标主机ssh服务的sshd_config文件中添加下列加密算法并重启ssh服务即可解决ssh连接问题。
KexAlgorithms diffie-hellman-group1-sha1,diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha1,diffie-hellman-group-exchange-sha256

# 最终添加了如下算法：
KexAlgorithms ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha256,diffie-hellman-group1-sha1,diffie-hellman-group-exchange-sha1
