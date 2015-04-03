title: Hadoop运行包含第三方依赖库的MapReduce作业
date: 2015-04-01 17:37:52
tag: hadoop
category: 大数据
# Hadoop运行包含第三方依赖库的MapReduce作业
---
##概述
最近打算学习一下利用hadoop搭建机器学习平台，因为mahout这个机器学习库资料比较多，因此就根据《mahout in action》这本书学习了一下如何搭建hadoop+mahout的机器学习平台。
由于mahout in action只是列出了部分代码，具体的环境搭建书上写的并不多。在编写依赖于mahout的MapReduce函数时经常会出现`ClassNotFoundException: org.apache.mahout.math.Vector`等类似错误。在网上找了很多的资料，但大都是从配置服务器`HADOOP_CLASSPATH`等角度出发的。这些方法要求将第三方依赖库拷贝到对应的`HADOOP_CLASSPATH`路径下,或者更改相应的环境变量使`hadoop`能识别相应的依赖库。这种做法的问题是需要更改服务器的工作环境，当未来应用程序需要更新依赖库时需要替换服务器的jar包。而且，即使我将mahout的两个主要依赖包`mahout-core-0.9.jar`和` mahout-math-0.9.jar`加入到对应的`HADOOP_CLASSPATH`路径依然无法解决`ClassNotFoundException`问题。
<!-- more -->
注意到这两个jar包都拥有`package org.apache.mahout.math`包，里面的class却并不一样。我写的MapReduce函数中用到的`org.apache.mahout.math.Vector`和`org.apache.mahout.math.VectorWritable`分别在` mahout-math-0.9.jar`和`mahout-core-0.9.jar`定义，因此我怀疑这两个jar包有冲突，实际运行时只加载了一个jar包(不是JAVA程序员，可能理解有误)。为了验证这个猜想，我将两个jar包分别设置到`HADOOP_CLASSPATH`变量中：

 - 只加载mahout-core-0.9.jar（包含VectorWritable定义）

```shell
export HADOOP_CLASSPATH=/usr/hdp/2.2.0.0-2041/mahout/mahout-core-0.9.jar
```
然后尝试运行自己编写的MapReduce jar包：HelloHadoop.jar
```
hadoop jar HelloHadoop.jar mapred1 /user/chenbiaolong/data/mywiki.txt /user/chenbiaolong/wiki_output
```
其中`mywiki.txt`是测试数据，事先已经上传到HDFS中，具体内容将在下一小节说明。
运行这条指令，果然出现了类未找到的错误：
```shell
[hdfs@localhost wiki]$ hadoop jar HelloHadoop.jar mapred1 /user/chenbiaolong/data/mywiki.txt /user/chenbiaolong/wiki_output
15/04/01 15:46:27 INFO impl.TimelineClientImpl: Timeline service address: http://localhost:8188/ws/v1/timeline/
15/04/01 15:46:27 INFO client.RMProxy: Connecting to ResourceManager at localhost/127.0.0.1:8050
15/04/01 15:46:28 INFO input.FileInputFormat: Total input paths to process : 1
15/04/01 15:46:28 INFO mapreduce.JobSubmitter: number of splits:1
15/04/01 15:46:28 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1427799851545_0007
15/04/01 15:46:28 INFO impl.YarnClientImpl: Submitted application application_1427799851545_0007
15/04/01 15:46:28 INFO mapreduce.Job: The url to track the job: http://localhost:8088/proxy/application_1427799851545_0007/
15/04/01 15:46:28 INFO mapreduce.Job: Running job: job_1427799851545_0007
15/04/01 15:46:36 INFO mapreduce.Job: Job job_1427799851545_0007 running in uber mode : false
15/04/01 15:46:36 INFO mapreduce.Job:  map 0% reduce 0%
15/04/01 15:46:42 INFO mapreduce.Job:  map 100% reduce 0%
15/04/01 15:46:47 INFO mapreduce.Job: Task Id : attempt_1427799851545_0007_r_000000_0, Status : FAILED
Error: java.lang.ClassNotFoundException: org.apache.mahout.math.Vector
	at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
	at java.lang.Class.forName0(Native Method)
	at java.lang.Class.forName(Class.java:270)
	at org.apache.hadoop.conf.Configuration.getClassByNameOrNull(Configuration.java:2015)
	at org.apache.hadoop.conf.Configuration.getClassByName(Configuration.java:1980)
	at org.apache.hadoop.conf.Configuration.getClass(Configuration.java:2074)
	at org.apache.hadoop.mapreduce.task.JobContextImpl.getReducerClass(JobContextImpl.java:210)
	at org.apache.hadoop.mapred.ReduceTask.runNewReducer(ReduceTask.java:611)
	at org.apache.hadoop.mapred.ReduceTask.run(ReduceTask.java:389)
	at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:163)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:415)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1628)
	at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:158)
```

- 只加载mahout-math-0.9.jar(包含Vector定义)
过程类似，直接给出操作过程：
```shell
[hdfs@localhost wiki]$ export HADOOP_CLASSPATH=/usr/hdp/2.2.0.0-2041/mahout/mahout-math-0.9.jar 
[hdfs@localhost wiki]$ hadoop jar HelloHadoop.jar mapred1 /user/chenbiaolong/data/mywiki.txt /user/chenbiaolong/wiki_output
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/mahout/math/VectorWritable
	at mapred1.run(mapred1.java:59)
	at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
	at mapred1.main(mapred1.java:83)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at org.apache.hadoop.util.RunJar.run(RunJar.java:221)
	at org.apache.hadoop.util.RunJar.main(RunJar.java:136)
Caused by: java.lang.ClassNotFoundException: org.apache.mahout.math.VectorWritable
	at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
	... 9 more
```
可见，`mahout-math-0.9.jar`和`mahout-core-0.9.jar`是冲突的，HADOOP运行只能加载一个jar包。
##Mahout+Hadoop MapReduce函数编写
本文所用的例子是根据《mahout in action》第6.2节分析维基百科数据的例子来的。不过这里只是为了熟悉mahout的用法，因此对这个例子进行了大幅度简化，只实现了第一个mapreduce，并且测试数据也是自己写的小文件。
首先，给出自己写的测试文件`mywiki.txt`内容
```
100 101 102 103
101 100 105 106
102 100
103 109 101 102
104 100 103
105 106
106 110 107
107 101 100
108 101 100
109 102 103
110 108 105
```
这里每一行都是一组数据，代表意思为`userID itemID itemID`。`userID`和`itemID`均为网页编号，比如第一行`100 101 102 103`表示编号为100的网页有链接到`101` `102` `103`网页的超链接。具体的实现理论背景请参阅`mahout in action`相关章节。
这里我们先只实现第一个mapreduce函数，需要得到的reduce结果格式为
`userid:100,vector:{101:1.0,102:1.0,103:1.0}`。即将原先的itemID变为itemID:item个数的vector形式。
具体的函数编写在windows环境下用IDEA完成，环境的搭建过程参考我的[上一篇文章][1]。这里只给出`pom.xml`文件内容
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
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-examples</artifactId>
            <version>1.2.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.mahout</groupId>
            <artifactId>mahout-examples</artifactId>
            <version>0.9</version>
        </dependency>
        <dependency>
            <groupId>org.apache.mahout</groupId>
            <artifactId>mahout-integration</artifactId>
            <version>0.9</version>
            <exclusions>
                <exclusion>
                    <groupId>org.mortbay.jetty</groupId>
                    <artifactId>jetty</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.cassandra</groupId>
                    <artifactId>cassandra-all</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>me.prettyprint</groupId>
                    <artifactId>hector-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    </project>
```
具体的mapreduce函数如下
```java
package org.apache.mahout.wiki;
import java.io.IOException;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.conf.*;
import org.apache.hadoop.io.*;
import org.apache.hadoop.mapreduce.*;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat;
import org.apache.hadoop.util.ToolRunner;
import org.apache.mahout.math.RandomAccessSparseVector;
import org.apache.mahout.math.VectorWritable;
import org.apache.mahout.math.Vector;

public class mapred1 extends Configured implements org.apache.hadoop.util.Tool {
    public static class WikipediaToItemPrefsMapper
            extends Mapper<LongWritable,Text,LongWritable,LongWritable> {
        private static final Pattern NUMBERS = Pattern.compile("(\\d+)");
        public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            String line = value.toString();
            Matcher m = NUMBERS.matcher(line);
            m.find();
            LongWritable userID = new LongWritable(Long.parseLong(m.group()));
            LongWritable itemID = new LongWritable();
            while (m.find()) {
                itemID.set(Long.parseLong(m.group()));
                context.write(userID, itemID);
            }
        }
    }

    public int run(String[] args)  throws Exception {
        if (args.length < 2) {
            return 2;
        }
        Job job = new Job(getConf());
        job.setJarByClass(mapred1.class); //
        job.setMapOutputKeyClass(LongWritable.class);
        job.setMapOutputValueClass(LongWritable.class);
        job.setMapperClass(WikipediaToItemPrefsMapper.class);
        job.setReducerClass( WikipediaToUserVectorReducer.class);
        job.setInputFormatClass(TextInputFormat.class);
        job.setOutputFormatClass(TextOutputFormat.class);
        job.setOutputKeyClass(LongWritable.class);
        job.setOutputValueClass(VectorWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        job.waitForCompletion(true);
        return 0;
    }
    public static class WikipediaToUserVectorReducer extends
            Reducer<LongWritable,LongWritable,LongWritable,VectorWritable> {
        public void reduce(LongWritable userID,
                           Iterable<LongWritable> itemPrefs,
                           Context context)
                throws IOException, InterruptedException {
                
            Vector userVector = new RandomAccessSparseVector(Integer.MAX_VALUE, 100);
            for (LongWritable itemPref : itemPrefs) {
                userVector.set((int)itemPref.get(), 1.0f);
            }
            context.write(userID, new VectorWritable(userVector));
        }
    }

    public static void main(String[] args) throws Exception {
        int  res = ToolRunner.run(new Configuration(), new mapred1(), args);
        System.exit(res);
    }

    }
```
这个mapreduce函数相对比较简单，map函数给出的结果是一个`userID`和一个`itemID`的数据集，reduce函数根据`userID`设置对应`itemID`的vector。这里一个`itemID`的vector对应值只有0和1两种，因为网页链接只有有和没有两种情况。
##将mapreduce函数和mahout依赖打成jar包
这里需要解决概述小节中提到的包依赖和包冲突问题。主要参考文章[`How-to: Include Third-Party Libraries in Your MapReduce Job`][2].这篇文章中提到了用3种方案解决第三方依赖包的依赖问题，我这里用的是第二种方案。
打开IDEA的`File-->Project Structure`对话框，在`Project Setting`的`Artifacts`配置要生成的jar包属性。在左边栏中新建一个`lib`目录，将依赖的jar包文件添加到该目录里面：我们这个工程主要有`mahout-core`和`mahout-math`两个jar包依赖。具体如下图：
![此处输入图片的描述][3]:
确认后将生成的jar包上传到Hadoop服务器集群的namenode节点。这时再继续用`hadoop jar`命令运行
```shell
[hdfs@localhost wiki]$ hadoop jar HelloHadoop.jar org.apache.mahout.wiki.mapred1 /user/chenbiaolong/data/mywiki.txt /user/chenbiaolong/wiki_output
15/04/01 17:07:38 INFO impl.TimelineClientImpl: Timeline service address: http://localhost:8188/ws/v1/timeline/
15/04/01 17:07:38 INFO client.RMProxy: Connecting to ResourceManager at localhost/127.0.0.1:8050
15/04/01 17:07:39 INFO input.FileInputFormat: Total input paths to process : 1
15/04/01 17:07:39 INFO mapreduce.JobSubmitter: number of splits:1
15/04/01 17:07:39 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1427799851545_0008
15/04/01 17:07:40 INFO impl.YarnClientImpl: Submitted application application_1427799851545_0008
15/04/01 17:07:40 INFO mapreduce.Job: The url to track the job: http://localhost:8088/proxy/application_1427799851545_0008/
15/04/01 17:07:40 INFO mapreduce.Job: Running job: job_1427799851545_0008
15/04/01 17:07:47 INFO mapreduce.Job: Job job_1427799851545_0008 running in uber mode : false
15/04/01 17:07:47 INFO mapreduce.Job:  map 0% reduce 0%
15/04/01 17:07:53 INFO mapreduce.Job:  map 100% reduce 0%
15/04/01 17:08:00 INFO mapreduce.Job:  map 100% reduce 100%
15/04/01 17:08:00 INFO mapreduce.Job: Job job_1427799851545_0008 completed successfully
15/04/01 17:08:00 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=420
		FILE: Number of bytes written=227085
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=256
		HDFS: Number of bytes written=250
		HDFS: Number of read operations=6
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters 
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=4194
		Total time spent by all reduces in occupied slots (ms)=4046
		Total time spent by all map tasks (ms)=4194
		Total time spent by all reduce tasks (ms)=4046
		Total vcore-seconds taken by all map tasks=4194
		Total vcore-seconds taken by all reduce tasks=4046
		Total megabyte-seconds taken by all map tasks=12883968
		Total megabyte-seconds taken by all reduce tasks=12429312
	Map-Reduce Framework
		Map input records=11
		Map output records=23
		Map output bytes=368
		Map output materialized bytes=420
		Input split bytes=120
		Combine input records=0
		Combine output records=0
		Reduce input groups=11
		Reduce shuffle bytes=420
		Reduce input records=23
		Reduce output records=11
		Spilled Records=46
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=108
		CPU time spent (ms)=2280
		Physical memory (bytes) snapshot=1490120704
		Virtual memory (bytes) snapshot=6737338368
		Total committed heap usage (bytes)=2288517120
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters 
		Bytes Read=136
	File Output Format Counters 
		Bytes Written=250
[hdfs@localhost wiki]$ 
```
可以看出程序已经可以正常运行了，解决了相应的依赖问题。查看reduce的输出结果：
```shell
[hdfs@localhost wiki]$ hdfs dfs -cat /user/chenbiaolong/wiki_output/part-r-00000
100	{101:1.0,103:1.0,102:1.0}
101	{106:1.0,100:1.0,105:1.0}
102	{100:1.0}
103	{101:1.0,109:1.0,102:1.0}
104	{103:1.0,100:1.0}
105	{106:1.0}
106	{107:1.0,110:1.0}
107	{101:1.0,100:1.0}
108	{101:1.0,100:1.0}
109	{103:1.0,102:1.0}
110	{108:1.0,105:1.0}
[hdfs@localhost wiki]$ 
```
可以看出mapreduce函数得到了我们需要的结果。
##依赖解决原理分析
为什么通过上小节这种方式可以解决依赖问题呢？[`How-to: Include Third-Party Libraries in Your MapReduce Job`][4]这篇文章说明了原因：

>  Include the referenced JAR in the lib subdirectory of the submittable JAR: A MapReduce job will unpack the JAR from this subdirectory into ${mapred.local.dir}/taskTracker/${user.name}/jobcache/$jobid/jars on the TaskTracker nodes and point your tasks to this directory to make the JAR available to your code. If the JARs are small, change often, and are job-specific this is the preferred method.

在利用`hadoop jar`命令运行作业时，hadoop会自动解压作业jar包里`lib`文件夹的jar文件。因此当我们在`lib`文件夹加入所依赖的第三方jar包（`mahout-core`和`mahout-math`）时，hadoop将会自动将这两个jar包解压，并不会判断这两个jar包是否冲突。因为`mahout-core`和`mahout-math`虽然共同拥有`org.apache.mahout.math`这个包路径，但里面的class并没有重复冲突。因此当这两个jar包解压时得到的`org.apache.mahout.math`是`mahout-core`和`mahout-math`的合集，也就不会出现前面提到的类未找到错误了。

##参考文献
(1) [How-to: Include Third-Party Libraries in Your MapReduce Job][5]
(2) [Hadoop 实现协同过滤算法（1）][6]


  [1]: http://chenbiaolong.com/2015/03/26/%E6%90%AD%E5%BB%BAHadoop%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83/
  [2]: http://blog.cloudera.com/blog/2011/01/how-to-include-third-party-libraries-in-your-map-reduce-job/
  [3]: http://7u2qr4.com1.z0.glb.clouddn.com/blogTimLine%E6%88%AA%E5%9B%BE20150401165156.png
  [4]: http://blog.cloudera.com/blog/2011/01/how-to-include-third-party-libraries-in-your-map-reduce-job/
  [5]: http://blog.cloudera.com/blog/2011/01/how-to-include-third-party-libraries-in-your-map-reduce-job/
  [6]: http://blog.csdn.net/fansy1990/article/details/8065007