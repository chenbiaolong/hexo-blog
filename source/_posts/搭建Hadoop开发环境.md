title: Intellij IDEA 搭建Hadoop开发环境
date: 2015-03-26 16:01:30
tags: hadoop
category: 大数据
---
##概述
在linux环境下编译调试java代码比较麻烦，为了方便，我在windows环境下利用 IDEA搭建了Hadoop的开发环境。只要在IDEA环境中导入Hadoop的核心包，就可以在IDEA环境下编译MapReduce程序，配置生成相应的jar包。通过将该jar包导入到linux服务器的hadoop集群，就可以运行相应的MapReduce程序了。在这里我以官方的WordCount程序作为例子。
<!-- more -->
##IDEA新建maven项目
首先在IDEA新建一个maven项目。具体新建过程参考http://guoze.me/2014/08/27/intellij-maven-develop-hadoop/ 
我新建的工程为HelloHadoop，具体的pom.xml如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.jd.bigdata</groupId>
    <artifactId>bigdata</artifactId>
    <version>1.0-SNAPSHOT</version>
    <repositories>
        <repository>
            <id>apache</id>
            <url>http://maven.apache.org</url>
        </repository>
    </repositories>

    <dependencies>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-common</artifactId>
        <version>2.6.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-core</artifactId>
        <version>1.2.1</version>
    </dependency>

    </dependencies>

</project>
```
我们使用的Hadoop测试集群Hadoop版本为2.6.0，具体的pom.xml配置需要根据你集群的具体发行版本修改。如果版本不一致将会出现hadoop集群无法正常运行MapReduce程序的情况。开发一个普通的Hadoop项目，我们一般需要hadoop-common、hadoop-core两组依赖；如果需要读取HDFS上的文件内容，则需要hadoop-hdfs和hadoop-client另外两组依赖；如果需要读取HBase的数据，则需要再加入hbase-client。这里我们只导入了hadoop-common和hadoop-core。
##编写WordCount MapReduce程序
这里直接使用了官方的[代码][1]
```java
import java.io.IOException;
import java.util.*;

import org.apache.hadoop.fs.Path;
import org.apache.hadoop.conf.*;
import org.apache.hadoop.io.*;
import org.apache.hadoop.mapreduce.*;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;

public class WordCount {

    public static class Map extends Mapper<LongWritable, Text, Text, IntWritable> {
        private final static IntWritable one = new IntWritable(1);
        private Text word = new Text();

        public void map(LongWritable key, Text value, Context context) 
            throws IOException, InterruptedException {
            
            String line = value.toString();
            StringTokenizer tokenizer = new StringTokenizer(line);
            while (tokenizer.hasMoreTokens()) {
                word.set(tokenizer.nextToken());
                context.write(word, one);
            }
        }
    }

    public static class Reduce extends Reducer<Text, IntWritable, Text, IntWritable> {

        public void reduce(Text key, Iterable<IntWritable> values, Context context)
                throws IOException, InterruptedException {
            int sum = 0;
            for (IntWritable val : values) {
                sum += val.get();
            }
            context.write(key, new IntWritable(sum));
        }
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();

        Job job = new Job(conf, "wordcount");
        job.setJarByClass(WordCount.class); //注意，必须添加这行，否则hadoop无法找到对应的class

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        job.setMapperClass(Map.class);
        job.setReducerClass(Reduce.class);

        job.setInputFormatClass(TextInputFormat.class);
        job.setOutputFormatClass(TextOutputFormat.class);

        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        job.waitForCompletion(true);
    }

}
```
需要注意的是需要在官方代码中加入`job.setJarByClass(WordCount.class);`这一行，具体解释可以参考[这里][2]。
##生成HelloHadoop jar包
生成jar包的过程也比较简单，
1.选择菜单File->Project Structure，弹出Project Structure的设置对话框。
2.选择左边的Artifacts后点击上方的“+”按钮
3.在弹出的框中选择jar->from moduls with dependencies..
4.选择要启动的类，然后 确定
5.应用之后，对话框消失。在IDEA选择菜单Build->Build Artifacts,选择Build或者Rebuild后即可生成，生成的jar文件位于工程项目目录的out/artifacts下。
##运行HelloHadoop jar包
将生成的HelloHadoop.jar传送到hadoop集群的Name node节点上。
处于测试目的，简单写了一个测试数据文本wctest.txt
```
this is hadoop test string
hadoop hadoop
test test
string string string
```
将该测试文本传到HDFS
```shell
[hdfs@172-22-195-15 data]$ hdfs dfs -mkdir /user/chenbiaolong/wc_test_input
[hdfs@172-22-195-15 data]$ hdfs dfs -put wctest.txt /user/chenbiaolong/wc_test_input
```
cd 到jar包对应的目录，执行HelloHadoop jar包
```
[hdfs@172-22-195-15 code]$ cd WorkCount/
[hdfs@172-22-195-15 WorkCount]$ ls
HelloHadoop.jar  
[hdfs@172-22-195-15 WorkCount]$ hadoop jar HelloHadoop.jar WordCount /user/chenbiaolong/wc_test_input /user/chenbiaolong/wc_test_output
15/03/26 15:54:19 INFO impl.TimelineClientImpl: Timeline service address: http://172-22-195-17.com:8188/ws/v1/timeline/
15/03/26 15:54:19 INFO client.RMProxy: Connecting to ResourceManager at 172-22-195-17.com/172.22.195.17:8050
15/03/26 15:54:20 WARN mapreduce.JobSubmitter: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
15/03/26 15:54:20 INFO input.FileInputFormat: Total input paths to process : 1
15/03/26 15:54:21 INFO mapreduce.JobSubmitter: number of splits:1
15/03/26 15:54:21 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1427255014010_0005
15/03/26 15:54:21 INFO impl.YarnClientImpl: Submitted application application_1427255014010_0005
15/03/26 15:54:21 INFO mapreduce.Job: The url to track the job: http://172-22-195-17.com:8088/proxy/application_1427255014010_0005/
15/03/26 15:54:21 INFO mapreduce.Job: Running job: job_1427255014010_0005
15/03/26 15:54:28 INFO mapreduce.Job: Job job_1427255014010_0005 running in uber mode : false
15/03/26 15:54:28 INFO mapreduce.Job:  map 0% reduce 0%
15/03/26 15:54:34 INFO mapreduce.Job:  map 100% reduce 0%
15/03/26 15:54:41 INFO mapreduce.Job:  map 100% reduce 100%
15/03/26 15:54:42 INFO mapreduce.Job: Job job_1427255014010_0005 completed successfully
15/03/26 15:54:43 INFO mapreduce.Job: Counters: 49
        File System Counters
                FILE: Number of bytes read=150
                FILE: Number of bytes written=225815
                FILE: Number of read operations=0
                FILE: Number of large read operations=0
                FILE: Number of write operations=0
                HDFS: Number of bytes read=210
                HDFS: Number of bytes written=37
                HDFS: Number of read operations=6
                HDFS: Number of large read operations=0
                HDFS: Number of write operations=2
        Job Counters 
                Launched map tasks=1
                Launched reduce tasks=1
                Rack-local map tasks=1
                Total time spent by all maps in occupied slots (ms)=4133
                Total time spent by all reduces in occupied slots (ms)=4793
                Total time spent by all map tasks (ms)=4133
                Total time spent by all reduce tasks (ms)=4793
                Total vcore-seconds taken by all map tasks=4133
                Total vcore-seconds taken by all reduce tasks=4793
                Total megabyte-seconds taken by all map tasks=16928768
                Total megabyte-seconds taken by all reduce tasks=19632128
        Map-Reduce Framework
                Map input records=4
                Map output records=12
                Map output bytes=120
                Map output materialized bytes=150
                Input split bytes=137
                Combine input records=0
                Combine output records=0
                Reduce input groups=5
                Reduce shuffle bytes=150
                Reduce input records=12
                Reduce output records=5
                Spilled Records=24
                Shuffled Maps =1
                Failed Shuffles=0
                Merged Map outputs=1
                GC time elapsed (ms)=91
                CPU time spent (ms)=3040
                Physical memory (bytes) snapshot=1466998784
                Virtual memory (bytes) snapshot=8678326272
                Total committed heap usage (bytes)=2200961024
        Shuffle Errors
                BAD_ID=0
                CONNECTION=0
                IO_ERROR=0
                WRONG_LENGTH=0
                WRONG_MAP=0
                WRONG_REDUCE=0
        File Input Format Counters 
                Bytes Read=73
        File Output Format Counters 
                Bytes Written=37
[hdfs@172-22-195-15 WorkCount]$ 
```
结果被输出到`/user/chenbiaolong/wc_test_output`
```shell
[hdfs@172-22-195-15 WorkCount]$ hdfs dfs -ls /user/chenbiaolong/wc_test_output
Found 2 items
-rw-r--r--   3 hdfs hdfs          0 2015-03-26 15:54 /user/chenbiaolong/wc_test_output/_SUCCESS
-rw-r--r--   3 hdfs hdfs         37 2015-03-26 15:54 /user/chenbiaolong/wc_test_output/part-r-00000
[hdfs@172-22-195-15 WorkCount]$ hdfs dfs -cat /user/chenbiaolong/wc_test_output/part-r-00000
hadoop  3
is      1
string  4
test    3
this    1
[hdfs@172-22-195-15 WorkCount]$ 
```
可以看出我们已经顺利得到正确结果。


  [1]: http://wiki.apache.org/hadoop/WordCount
  [2]: http://stackoverflow.com/questions/21373550/class-not-found-exception-in-mapreduce-wordcount-job
