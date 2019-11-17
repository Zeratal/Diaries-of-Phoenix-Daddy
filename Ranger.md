Ranger
-------------------

[TOC]

## 编译

### 前置软件
```
JAVA 
MAVEN
PYTHON
GIT
 ...
```
### 编译命令
```
mvn compile package assembly:assembly install -DskipTests -Drat.skip=true -X
mvn -DskipTests=false clean compile package install assembly:assembly 
注意：第二行是官网命令，但是其实-DskipTests=false会导致编译不过，去掉后面的=false
```
## Admin安装

由于选择postgres为ranger数据库，官网仅描述支持，并没有对应的配置修改文档。自己摸索吧(DB_FLAVOR和SQL_CONNECTOR_JAR配置)
### pg预设
```
PG安装好后，其实不需要手动创建ranger的库，第一次部署，是因为配置没有localhost所以导致pg连接不上，setup的报错信息不准确。
```

### 配置修改
修改主目录下install.properties文件
- PYTHON_COMMAND_INVOKER 参数在后续setup.sh里没有使用，而是直接使用了"python"，建议修改该脚本引用python的地方为$PYTHON_COMMAND_INVOKER，这样可以在安装多python版本时指定对应的版本
- db_host 配置理论上应该带port，但是这里没带也正常工作了，后续进一步测试加上port
- audit_store相关配置没有配，后续研究其作用再补充
```
#------------------------- DB CONFIG - BEGIN ----------------------------------
# Uncomment the below if the DBA steps need to be run separately
setup_mode=SeparateDBA
PYTHON_COMMAND_INVOKER=python
DB_FLAVOR=POSTGRES

# Location of DB client library (please check the location of the jar file)
SQL_CONNECTOR_JAR=/usr/share/java/postgresql.jar

# DB password for the DB admin user-id
# **************************************************************************
# ** If the password is left empty or not-defined here,
# ** it will try with blank password during installation process
# **************************************************************************
#db_host=host:port              # for DB_FLAVOR=MYSQL|POSTGRES|SQLA|MSSQL       #for example: b_host=localhost:3306
db_root_user=postgres
db_root_password=postgres
db_host=ethan

#
# DB UserId used for the Ranger schema
#
db_name=ranger
db_user=ranger
db_password=ranger

#此配置文档介绍要配置，但本次实践没有配置，可能admin UI偶尔报audit无法获取有关
#audit_store=db
#audit_db_name=ranger_audit
#audit_db_user=root
#audit_db_password=root

#
# ------- UNIX User CONFIG ----------------
#
unix_user=ranger
unix_user_pwd=ranger
unix_group=ranger

#
# ------- PolicyManager CONFIG ----------------
#
policymgr_external_url=http://localhost:6080
#policymgr_http_enabled=true
```
### setup.sh解析

```


```

##  hdfs plugins
研究半天，发现走入一个误区，这些插件是应该在组件节点启动的，而不是admin节点。
另外，目测是不需要root权限的。需要特别关注2个问题：
1. 修改脚本，规避默认的文件配置(/etc/ranger/)这个路径。需要观察谁在使用，如何使用这个路径，理论上就是hdfs启动plugin功能时会用到，如果该路径可修改，则改到当前用户目录即可。
2. 另外enable脚本多次修改目录及文件的拥有者权限，如果我们在指定目录下安装，配置，是不需要root权限的（如果安装用户和配置用户不一致，才需要root用户安装，并将所有者修改为配置用户）

### 配置修改
修改主目录下install.properties文件
**注意：此文件增加 COMPONENT_INSTALL_DIR_NAME配置**

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

#**注意：此行重要，这里其实配置的是 $HADOOP_HOME**
COMPONENT_INSTALL_DIR_NAME=/home/ethan/workspace/release/hadoop-2.7.3
```

### 版本拷贝
需要注意的是，plugin的版本需要拷贝到组件所在节点，如本实践，要拷贝到hdfs的namenode所在的节点。准确的说，plugin本身并不会自己启动。而是通过enable脚本，往hadoop驻入自己提供的库，配置，并修改hdfs组件的鉴权参数以及鉴权实例为自己。然后hdfs重启后，在内部启动鉴权相关的服务。
```
 scp -r ranger-2.0.1-SNAPSHOT-hdfs-plugin/ ethan1:/home/ethan/workspace/release/
 scp -r ranger-2.0.1-SNAPSHOT-hdfs-plugin/ ethan2:/home/ethan/workspace/release/
```
### 软连接hadoop库和配置（过时）
  这个操作来源于Aanger官网的wiki，但其对应版本是ranger-0.5，已经过时。
  在ranger-2.0版本里，由于我们之前在install.properties里配置了**COMPONENT_INSTALL_DIR_NAME**，所以后面enable-hdfs-plugin.sh会根据组件路径，正确的找到组件的配置路径和共享库路径。
(https://cwiki.apache.org/confluence/display/RANGER/Ranger+Installation+Guide#RangerInstallationGuide-Install/ConfigureRangerHDFSPlugin)
这里的软连接其实就是为了enable-hdfs-plugin.sh能找到组件安装目录，在配置和库路径增加修改plugin相关内容
（创建 ${PROJ_INSTALL_DIR}/lib/*  到 ${HCOMPONENT_INSTALL_DIR}/share/hadoop/hdfs/lib的连接）
```
ln -s $HADOOP_HOME/share/hadoop/hdfs/lib ./hadoop/lib
ln -s $HADOOP_HOME/etc/hadoop/ ./hadoop/conf
```
### enable-hdfs-plugin.sh解析
#### 运行环境检查
1，通过判断当前用户是否有root权限（为什么要root？仅仅是因为后面“SSL密钥库和信任库生成凭据文件”默认是保存在/etc下。另外，多次调用了chown和chmod）
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
组件名称是脚本用到的一个变量，从脚本名称解析出来的。所以如果要开发自己的插件，写启动（其实应该是安装或者嵌入）脚本时，注意脚本名称要保持风格。
```
COMPONENT_NAME=`basename $0 | cut -d. -f1 | sed -e 's:^disable-::' | sed -e 's:^enable-::'`

echo "${COMPONENT_NAME}" | grep 'plugin' > /dev/null 2>&1
if [ $? -ne 0 ]
then
	echo "$0 : is not applicable for component [${COMPONENT_NAME}]. It is applicable only for ranger plugin component; Exiting ..."
	exit 0 
fi
/*这些脚本必须携带plugin字样，否则会被认为是异常*/

HCOMPONENT_NAME=`echo ${COMPONENT_NAME} | sed -e 's:-plugin::'`

CFG_OWNER_INF="${HCOMPONENT_NAME}:${HCOMPONENT_NAME}"
/*后续会该权限?改成组件名称的用户组和用户？
这里其实会被后面指定的custom用户和组替换掉*/

if [ "${HCOMPONENT_NAME}" = "hdfs" ]
then
	HCOMPONENT_NAME="hadoop"
fi
```
5，获取action类型
因为插件脚本，enable和disable脚本除了名字，内容完全一样（好作坊风），此处根据脚本名字判断当前是打开还是关闭plugin
```
basename $0 | cut -d. -f1 | grep '^enable-' > /dev/null 2>&1

if [ $? -eq 0 ]
then
	action=enable
else
	action=disable
fi
```
#### 环境变量设置
```
PROJ_INSTALL_DIR=`(cd ${basedir} ; pwd)`          // /home/ethan/workspace/release/ranger-2.0.1-SNAPSHOT-hdfs-plugin
SET_ENV_SCRIPT_NAME=set-${COMPONENT_NAME}-env.sh     //set-hdfs-plugin-env.sh
SET_ENV_SCRIPT_TEMPLATE=${PROJ_INSTALL_DIR}/install/conf.templates/enable/${SET_ENV_SCRIPT_NAME} //对应文件不存在
     DEFAULT_XML_CONFIG=${PROJ_INSTALL_DIR}/install/conf.templates/default/configuration.xml   //default都不存在
  PROJ_INSTALL_LIB_DIR="${PROJ_INSTALL_DIR}/install/lib"
           PROJ_LIB_DIR=${PROJ_INSTALL_DIR}/lib
          INSTALL_ARGS="${PROJ_INSTALL_DIR}/install.properties"   //关键配置文件
COMPONENT_INSTALL_ARGS="${PROJ_INSTALL_DIR}/${COMPONENT_NAME}-install.properties"   //hdfs-plugin-install.properties不存在

PLUGIN_DEPENDENT_LIB_DIR=lib/"${PROJ_NAME}-${COMPONENT_NAME}-impl"   //   lib/ranger-hdfs-plugin-impl/
    PROJ_LIB_PLUGIN_DIR=${PROJ_INSTALL_DIR}/${PLUGIN_DEPENDENT_LIB_DIR}  //目录存在
```
#### 参数获取
用户权限
```
/*一整段的CUSTOM家族配置，最后为了得到一个CFG_OWNER_INF*/
CUSTOM_USER=$(getInstallProperty 'CUSTOM_USER')
CUSTOM_USER=${CUSTOM_USER// }/* 看不懂，在后面加个‘/’？也没加啊 */
CUSTOM_GROUP=$(getInstallProperty 'CUSTOM_GROUP')
CUSTOM_GROUP=${CUSTOM_GROUP// }/* 看不懂，在后面加个‘/’？也没加啊 */
CUSTOM_GROUP_STATUS=${CUSTOM_GROUP};
CUSTOM_USER_STATUS=${CUSTOM_USER};
egrep "^$CUSTOM_GROUP" /etc/group >& /dev/null
if [ $? -ne 0 ]
then
	CUSTOM_GROUP_STATUS=""
fi
id -u ${CUSTOM_USER} > /dev/null 2>&1
if [ $? -ne 0 ]
then
	CUSTOM_USER_STATUS=""
fi

if [ ! -z "${CUSTOM_USER_STATUS}" ] && [ ! -z "${CUSTOM_GROUP_STATUS}" ]
then
  echo "Custom user and group is available, using custom user and group."
  CFG_OWNER_INF="${CUSTOM_USER}:${CUSTOM_GROUP}"
elif [ ! -z "${CUSTOM_USER_STATUS}" ] && [ -z "${CUSTOM_GROUP_STATUS}" ]
then
  echo "Custom user is available, using custom user and default group."
  CFG_OWNER_INF="${CUSTOM_USER}:${HCOMPONENT_NAME}"
elif [ -z  "${CUSTOM_USER_STATUS}" ] && [ ! -z  "${CUSTOM_GROUP_STATUS}" ]
then
  echo "Custom group is available, using default user and custom group."
  CFG_OWNER_INF="${HCOMPONENT_NAME}:${CUSTOM_GROUP}"
else
  echo "Custom user and group are not available, using default user and group."
  CFG_OWNER_INF="${HCOMPONENT_NAME}:${HCOMPONENT_NAME}"
fi
```
#### 组件依赖和配置

从脚本解析过程看，我们应该把HCOMPONENT_INSTALL_DIR_NAME这个参数，配置为对应组件的安装路径，比如hadoop为$HADOOP_HOME.那么就没有后续一堆软连接的操作了，而且组件安装路径未配置的情况下，默认路径很诡异，软连接很容易设置错误。
```
/*关键配置*/
HCOMPONENT_INSTALL_DIR_NAME=$(getInstallProperty 'COMPONENT_INSTALL_DIR_NAME')
/*意料之外的关键参数，主要是后续对默认值的设置诡异莫测*/

if [ "${HCOMPONENT_INSTALL_DIR_NAME}" = "" ]
then
    if [ "${HCOMPONENT_NAME}" = "knox" ];
    then
        HCOMPONENT_INSTALL_DIR_NAME=$(getInstallProperty 'KNOX_HOME')
    fi
    if [ "${HCOMPONENT_INSTALL_DIR_NAME}" = "" ]    /*冗余的判断条件，放在上面的else if 就可以了*/
    then
	    HCOMPONENT_INSTALL_DIR_NAME=${HCOMPONENT_NAME}
    fi
fi

/*判断是否是'/'开始的绝对路径，如果是不是，则在工程目录的上一级目录创建，诡异*/
firstletter=${HCOMPONENT_INSTALL_DIR_NAME:0:1}
if [ "$firstletter" = "/" ]; then
    hdir=${HCOMPONENT_INSTALL_DIR_NAME}
else
    hdir=${PROJ_INSTALL_DIR}/../${HCOMPONENT_INSTALL_DIR_NAME}
fi
```
组件插件需要的配置和依赖，这里明显hadoop需要的配置和库指向上面的hdir，但是上面把这个目录创建在此plugin的父目录是几个意思。
```
HCOMPONENT_INSTALL_DIR=`(cd ${hdir} ; pwd)`

/*构造HCOMPONENT_LIB_DIR，对hadoop来说，是安装目录下 share/hadoop/hdfs/lib/
HCOMPONENT_LIB_DIR=${HCOMPONENT_INSTALL_DIR}/lib
               ................
elif [ "${HCOMPONENT_NAME}" = "hadoop" ] ||
     [ "${HCOMPONENT_NAME}" = "yarn" ]; then
    HCOMPONENT_LIB_DIR=${HCOMPONENT_INSTALL_DIR}/share/hadoop/hdfs/lib
               ...............

/*构造HCOMPONENT_CONF_DIR，对hadoop来说，是安装目录下 etc/hadoop
HCOMPONENT_CONF_DIR=${HCOMPONENT_INSTALL_DIR}/conf
               ...............
elif [ "${HCOMPONENT_NAME}" = "hadoop" ]; then
    HCOMPONENT_CONF_DIR=${HCOMPONENT_INSTALL_DIR}/etc/hadoop
elif [ "${HCOMPONENT_NAME}" = "yarn" ]; then
    HCOMPONENT_CONF_DIR=${HCOMPONENT_INSTALL_DIR}/etc/hadoop
               ...............

#这两个变
HCOMPONENT_ARCHIVE_CONF_DIR=${HCOMPONENT_CONF_DIR}/.archive    /*不存在 $HADOOP_HOME/etc/hadoop/.archive*/
SET_ENV_SCRIPT=${HCOMPONENT_CONF_DIR}/${SET_ENV_SCRIPT_NAME}   /*不存在  $HADOOP_HOME/etc/hadoop/set-hadoop-plugin-env.sh   */
```
#### 检查组件安装目录，组件配置目录，组件库目录
```
if [ ! -d "${HCOMPONENT_INSTALL_DIR}" ]
then
	echo "ERROR: Unable to find the install directory of component [${HCOMPONENT_NAME}]; dir [${HCOMPONENT_INSTALL_DIR}] not found."
	echo "Exiting installation."
	exit 1
fi

if [ ! -d "${HCOMPONENT_CONF_DIR}" ]
then
	echo "ERROR: Unable to find the conf directory of component [${HCOMPONENT_NAME}]; dir [${HCOMPONENT_CONF_DIR}] not found."
	echo "Exiting installation."
	exit 1
fi

if [ ! -d "${HCOMPONENT_LIB_DIR}" ]
then
    mkdir -p "${HCOMPONENT_LIB_DIR}"
    if [ ! -d "${HCOMPONENT_LIB_DIR}" ]
    then
        echo "ERROR: Unable to find the lib directory of component [${HCOMPONENT_NAME}];  dir [${HCOMPONENT_LIB_DIR}] not found."
        echo "Exiting installation."
        exit 1
    fi
fi
```

#### 预处理
- set-hdfs-plugin-env.sh(不存在)
这段脚本比较古怪，但是其实它不会执行，因为它判断了 ./install/conf.templates/enable/set-hdfs-plugin-env.sh 存在与否，而安装包没有携带此脚本
```
./install/conf.templates/enable/set-hdfs-plugin-env.sh
1,  判断./install/conf.templates/enable/set-hdfs-plugin-env.sh是否存在
    是 2， 判断$HADOOP_HOME/etc/hadoop/set-hadoop-plugin-env.sh是否存在
           是  判断目录 $HADOOP_HOME/etc/hadoop/.archive 是否存在，如果不在则创建改目录
               把$HADOOP_HOME/etc/hadoop/set-hadoop-plugin-env.sh移动到$HADOOP_HOME/etc/hadoop/.archive
       3， 判断是否action
           是  把 ./install/conf.templates/enable/set-hdfs-plugin-env.sh 拷贝到 $HADOOP_HOME/etc/hadoop/set-hadoop-plugin-env.sh
	       4， 判断是否 $HAD OOP_HOME/libexec/hadoop-config.sh 存在
	           是  $HAD OOP_HOME/libexec/hadoop-config.sh 原地备份 $HADOOP_HOME/libexec/.hadoop-config.sh.data 
	               删除 hadoop-config.sh 里的 ‘xasecure-.*-env.sh’ 行
                       判断hadoop-config.sh里是否包含了set-hadoop-plugin-env.sh行，如果是则**，如果不是则报告hadoop-config.sh包含了hadoop-env.sh（不理解）
```
#### enable|disable配置处理

```
1,  判断 ${PROJ_INSTALL_DIR}/install/conf.templates/${action} 是否存在（必然存在）
    是  2，  ${PROJ_INSTALL_DIR}/install/lib 下所有文件赋值给INSTALL_CP
             3，  判断action是否是enable 
	          是  生成 $HADOOP_HOME/etc/hadoop/ranger-security.xml，并修改文件持有者和权限
		      遍历  ${PROJ_INSTALL_DIR}/install/conf.templates/${action} 下xml文件。拷贝到$HADOOP_HOME/etc/hadoop，并修改文件持有者和权限（如果已经存在则原地备份后拷贝修改）
		  否  如果存在，则备份删除$HADOOP_HOME/etc/hadoop/ranger-security.xml
   
   获取REPOSITORY_NAME 给REPO_NAME
   构造POLICY_CACHE_FILE_PATH（/etc/ranger/hadoopdev/policycache）
   构造CREDENTIAL_PROVIDER_FILE（/etc/ranger/hadoopdev/cred.jceks）
   如果不存在则创建构造POLICY_CACHE_FILE_PATH，修改该路径所有节点的权限为a+rx
   修改POLICY_CACHE_FILE_PATH的所有者为指定客户和群组
   遍历  ${PROJ_INSTALL_DIR}/install/conf.templates/${action} 下cfg文件，如果存在文件则：（不可能不存在，刚从目录得到的）
     获取cfg文件全名，并替换“-changes.cfg”为“.xml”（其实得到hdfs-site.xml和此目录下其他xml。）
     查找$HADOOP_HOME/etc/hadoop下同名xml（hdfs-site.xml本身存在，后3个xml本来就是从此目录拷贝过去的）
     	4，判断$HADOOP_HOME/etc/hadoop下同名文件是否存在（必然存在）
	   否  查找${PROJ_INSTALL_DIR}/install/conf.templates/default/configuration.xml文件（不存在）
	         是  拷贝configuration.xml为$HADOOP_HOME/etc/hadoop下对应xml，并改权限和所有者
		 否  报错退出
           是  备份该xml文件（archivefn）到原地，备份失败则报错退出
	       通过org.apache.ranger.utils.install.XmlConfigChanger比较conf文件，和备份文件（入参为${PROJ_INSTALL_DIR}/install.properties，-cp参数为${PROJ_INSTALL_DIR}/install/lib/*）
	       合并后的文件覆盖原xml
```
#### 创建库连接
```
1,  如果action是enable
    是  遍历${PROJ_INSTALL_DIR}/lib。
        判断${HCOMPONENT_INSTALL_DIR}/share/hadoop/hdfs/lib路径是否有同名文件或者文件夹
          是  原地mv到备份（这里有点问题，如果原地mv，则此文件丢失。！没有问题，因为后面再次判断时候存在同名文件或文件夹）
	判断${HCOMPONENT_INSTALL_DIR}/share/hadoop/hdfs/lib路径是否有同名文件或者文件夹
          否  创建${PROJ_INSTALL_DIR}/lib/* 到 ${HCOMPONENT_INSTALL_DIR}/share/hadoop/hdfs/lib的连接
```
#### 创建凭证文件路径
```
继承上文的 如果action是enable
  2,  获取CREDENTIAL_PROVIDER_FILE文件路径（/etc/ranger/hadoopdev/cred.jceks），如果无效路径则报错退出
      创建该文件路径（如果不存在），并修改权限为a+rx      
```

#### 为SSL密钥库和信任库生成凭据提供程序文件和凭据
```
继承上文的 如果action是enable
  3,  从配置文件获取SSL_KEYSTORE_PASSWORD和SSL_TRUSTSTORE_PASSWORD；分别和sslKeyStore，sslTrustStore调用create_jceks写入cred.jceks（上节）
      create_jceks调用org.apache.ranger.credentialapi.buildks，完成cred.jceks的写入（入参就是key-value,-cp参数是${PROJ_INSTALL_DIR}/install/lib）
      修改cred.jceks的权限和所有者
```

#### 后续处理
knox, storm, atlas，sqoop，kylin，和presto有些附加操作，但是hdfs则没有，这里直接打印提示重启hadoop，然后正常退出

