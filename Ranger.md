## 编译
```
mvn compile package assembly:assembly install -DskipTests -Drat.skip=true -X
mvn -DskipTests=false clean compile package install assembly:assembly 
注意：第二行是官网命令，但是其实-DskipTests=false会导致编译不过，去掉后面的=false
```
### 前置
```
JAVA PYTHON GIT ...
```

##安装

由于选择postgres为ranger数据库，官网仅描述支持，并没有对应的配置修改文档。自己摸索吧
### pg预设
```

```
