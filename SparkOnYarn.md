
### /home/spark/conf/spark-default.conf
|Property| Name|	Default|	Meaning|
|--| --|	--|	--|
|spark.yarn.jars	|(none)	|List of libraries containing Spark code to distribute to YARN containers. By default, Spark on YARN will use Spark jars installed locally, but the Spark jars can also be in a world-readable location on HDFS. This allows YARN to cache it on nodes so that it doesn't need to be distributed each time an application runs. To point to jars on HDFS, for example, set this configuration to hdfs:///some/path. Globs are allowed.|
|spark.yarn.archive|	(none)	|An archive containing needed Spark jars for distribution to the YARN cache. If set, this configuration replaces spark.yarn.jars and the archive is used in all the application's containers. The archive should contain jar files in its root directory. Like with the previous option, the archive can also be hosted on HDFS to speed up file distribution.|

### /home/hadoop/etc/hadoop/yarn-site.xml
增加日志聚合
```
    <!--是否启用日志聚合-->
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
    <!-- 设置日志保留时间,清除日志的检查周期， 单位是秒 -->
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>259200</value>
    </property>
    <property>
        <name>yarn.log-aggregation.retain-check-interval-seconds</name>
        <value>-1</value>
    </property>
```
