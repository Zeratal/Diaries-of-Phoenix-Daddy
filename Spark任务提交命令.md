## 任务提交
成功的提交
```
commond: /home/spark/bin/spark-submit  --name Ethan-test-1 --executor-memory 4g --num-executors 3 --master yarn --deploy-mode cluster --class org.apache.spark.examples.SparkPi --jars /home/spark/examples/jars/spark-examples_2.11-2.4.3.jar /home/spark/examples/jars/spark-examples_2.11-2.4.3.jar 
commond: /home/spark/bin/spark-submit  --name Ethan-test-2 --executor-memory 4g --total-executor-cores 3 --deploy-mode client --class org.apache.spark.examples.SparkPi --jars /home/spark/examples/jars/spark-examples_2.11-2.4.3.jar /home/spark/examples/jars/spark-examples_2.11-2.4.3.jar 
```

失败的提交
```
commond: /home/spark/bin/spark-submit  --name SqlAgent-test-2 --executor-memory 4g --total-executor-cores 3 --deploy-mode client --class com.hongshan.SqlAgent --jars /home/wujianping/sqlagent_sbt_2.11-0.1.jar /home/wujianping/sqlagent_sbt_2.11-0.1.jar 
commond: /home/spark/bin/spark-submit  --name SqlAgent-test-1 --executor-memory 4g --num-executors 3 --master yarn --deploy-mode cluster --class com.hongshan.SqlAgent --jars /home/wujianping/sqlagent_sbt_2.11-0.1.jar /home/wujianping/sqlagent_sbt_2.11-0.1.jar 
```

结论：这里失败和成功与提交无关，因为之前上传了一个sqlagent_sbt_2.11-0.1.jar在${SPARK_HOME}/jars路径下，该jar有问题，删除该jar后恢复正常
