## 编译
```
mvn compile package assembly:assembly install -DskipTests -Drat.skip=true -X
mvn -DskipTests=false clean compile package install assembly:assembly 
注意：第二行是官网命令，但是其实-DskipTests=false会导致编译不过，去掉后面的=false
```
### 前置
```
JAVA MAVEN PYTHON GIT ...
```

## 安装

由于选择postgres为ranger数据库，官网仅描述支持，并没有对应的配置修改文档。自己摸索吧
### pg预设
```
PG安装好后，其实不需要手动创建ranger的库，第一次部署，是因为配置没有localhost所以导致pg连接不上，setup的报错信息不准确
```

### hdfs plugins
研究半天，发现走入一个误区，这些插件是应该在组件节点启动的，而不是admin节点。
目测是不需要root权限的，普通权限下部署测试
#### vim install.properties
```
POLICY_MGR_URL=http://ethan:6080
REPOSITORY_NAME=hadoopdev
XAAUDIT.DB.IS_ENABLED=true
XAAUDIT.DB.FLAVOUR=POSTGRES
XAAUDIT.DB.HOSTNAME=localhost
XAAUDIT.DB.DATABASE_NAME=ranger
XAAUDIT.DB.USER_NAME=postgres
XAAUDIT.DB.PASSWORD=postgres

# Custom component user
# CUSTOM_COMPONENT_USER=<custom-user>
# keep blank if component user is default
CUSTOM_USER=ethan

#
# Custom component group
# CUSTOM_COMPONENT_GROUP=<custom-group>
# keep blank if component group is default
CUSTOM_GROUP=ethan
```

#### 版本拷贝
```
 scp -r ranger-2.0.1-SNAPSHOT-hdfs-plugin/ ethan1:/home/ethan/workspace/release/
 scp -r ranger-2.0.1-SNAPSHOT-hdfs-plugin/ ethan2:/home/ethan/workspace/release/
```
#### 软连接hadoop库和配置
```
ln -s $HADOOP_HOME/share/hadoop/hdfs/lib ./hadoop/lib
ln -s $HADOOP_HOME/etc/hadoop/ ./hadoop/conf
```
#### enable-hdfs-plugin.sh解析

1，通过判断当前用户是否有root权限（为什么要root）
```
if [ ! -w /etc/passwd ]
then
    echo "ERROR: $0 script should be run as root."
    exit 1
fi
```
2，检查环境变量JAVA_HOME
3，获取当前文件夹，后面需要赋值给PROJ_INSTALL_DIR
```
basedir=`dirname $0`
if [ "${basedir}" = "." ]
then
    basedir=`pwd`
elif [ "${basedir}" = ".." ]
then
    basedir=`(cd .. ;pwd)`
fi
/*诡异的写法，这段代码冗余，当不是脚本当前目录执行或者子目录执行，basedir就是一个相对路径，后面复制的时候依然用了相对路径，如果这期间有cd更改目录，这段代码就挂了，还不如直接basedir = "$(cd "`dirname "$0"`"; pwd)"*/
.......

PROJ_INSTALL_DIR=`(cd ${basedir} ; pwd)`
```
4，获取组件名称
```
COMPONENT_NAME=`basename $0 | cut -d. -f1 | sed -e 's:^disable-::' | sed -e 's:^enable-::'`

echo "${COMPONENT_NAME}" | grep 'plugin' > /dev/null 2>&1

if [ $? -ne 0 ]
then
	echo "$0 : is not applicable for component [${COMPONENT_NAME}]. It is applicable only for ranger plugin component; Exiting ..."
	exit 0 
fi

HCOMPONENT_NAME=`echo ${COMPONENT_NAME} | sed -e 's:-plugin::'`

CFG_OWNER_INF="${HCOMPONENT_NAME}:${HCOMPONENT_NAME}"

if [ "${HCOMPONENT_NAME}" = "hdfs" ]
then
	HCOMPONENT_NAME="hadoop"
fi
```
```
sudo ln -s $HADOOP_HO

ME/etc/hadoop/* ./hadoop/conf/

cp /home/ethan/workspace/release/ranger-2.0.1/ranger-hdfs-plugin/lib/ranger-hdfs-plugin-impl/*.jar /home/ethan/workspace/release/hadoop-2.7.3/share/hadoop/hdfs/lib/
sudo mkdir -p /opt/ranger-hdfs-plugin/hadoop/lib/
sudo ln -s /home/ethan/workspace/release/hadoop-2.7.3/share/hadoop/hdfs/lib/*.jar /opt/ranger-hdfs-plugin/hadoop/lib/

 scp /home/ethan/workspace/release/ranger-2.0.1/ranger-hdfs-plugin/lib/ranger-hdfs-plugin-impl/*.jar ethan1:/home/ethan/workspace/release/hadoop-2.7.3/share/hadoop/hdfs/lib/
 scp /home/ethan/workspace/release/ranger-2.0.1/ranger-hdfs-plugin/lib/ranger-hdfs-plugin-impl/*.jar ethan2:/home/ethan/workspace/release/hadoop-2.7.3/share/hadoop/hdfs/lib/
 scp /home/ethan/workspace/release/ranger-2.0.1/ranger-hdfs-plugin/lib/ranger-hdfs-plugin-impl/*.jar ethan3:/home/ethan/workspace/release/hadoop-2.7.3/share/hadoop/hdfs/lib/


```
