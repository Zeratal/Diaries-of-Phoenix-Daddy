 修改JavaWordCount示例，测试UnsafeShuffle以及offHeap模式。
    
    SparkSession spark = SparkSession
            .builder()
            .appName("JavaWordCount")
            .master("local")
            .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
            .config("spark.memory.offHeap.enabled", true)
            .config("spark.memory.offHeap.size", 3 *1024 *1024)
            .getOrCreate();

    JavaRDD<String> lines = spark.read().textFile("README.md").javaRDD();
    JavaRDD<String> words = lines.flatMap(s -> Arrays.asList(SPACE.split(s)).iterator());
    JavaPairRDD<String, Integer> ones = words.mapToPair(s -> new Tuple2<>(s, 1)).repartition(250);
    Long count = ones.count();
    spark.stop();
    
疑问：
1，ShuffleMapTask runTask（）流程；UnsafeShuffleWriter的write（）内，records.next()得到的是一个（int，（String，int））结构；实际上，RDD的内容应该是（String，int）。
2，
