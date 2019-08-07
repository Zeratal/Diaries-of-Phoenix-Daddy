### Intellij idea項目中无法创建scala class的解决办法

在本项目的iml文件的component标签中添加以下标签内容
```
<orderEntry type="library" name="scala-sdk-2.10.6" level="application" />
```

其实应该是，在
```
  Project Structure -> Add to Modules ... -> 当前工程
  Project窗口 -> src\main 右键 -> new -> Directory -> 创建scala目录
  scala目录右键 -> Make Directory as ... -> Sources Root
```

### Maven仓库
标准库地址：http://repo1.maven.org/maven2/

在maven的setting添加镜像库:C:\Users\$username\.m2\settings.xml的mirrors标签增加：
```
         <!--阿里云仓库-->
    <mirror>
      <id>alimaven</id>
      <mirrorOf>central</mirrorOf>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
    </mirror>
```
