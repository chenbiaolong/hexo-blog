title: Mahout in action分析维基百科数据例子(二)
date: 2015-04-07 14:53:18
tags: hadoop
category: 大数据
---
##概要
这篇文章主要论述我在实现[上一篇文章][1]所述功能时的具体操作过程。因为`hadoop`现在有两套新旧API接口，因此在实现过程中需要十分注意你import进来的class是属于新的API还是旧的API。本文的所使用的`hadoop`版本是`2.6`版本。
<!-- more -->

##工程准备

###数据准备

`mahout in action`用的是[维基百科的数据][2]，数据量较大，考虑到不便于验证我们的测试程序是否运行正确，我们这里用的是自己写的一个小数据文件`mywiki.txt`
```shell
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
每一行代表的数分别是`userID` `itemID1` `itemID2`。

###利用Intellij IDEA建立maven工程

具体流程在我[前面的文章][3]有论述。这里给出工程布局以及`pom.xml`文件:
wiki项目工程文件：

![此处输入图片的描述][4]

简单说明一下，这里` WikipediaToItemPrefsMapper`和`WikipediaToUserVectorReducer`是第一次MapReduce操作， `UserVectorToCooccurrenceMapper`和`UserVectorToCooccurrenceReducer`是第二次MapReduce操作。这两次MapReduce操作分别完成的功能同样可以参考[上一篇文章][5]。
wiki项目的`pom.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.jd.bigdata</groupId>
    <artifactId>1.0</artifactId>
    <version>1.0-SNAPSHOT</version>


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
特别提醒一下，这个`pom.xml`是根据我集群的实际环境配置的`hadoop 2.6`版本，`mahout 0.9`版本。如果你们的集群环境和我的不一样，需要进行一些调整。

##工程源码

工程源码大部分与`mahout in action`一样，根据实际情况进行了一些调整。
`WikipediaToItemPrefsMapper.java`
```java
package com.chenbiaolong.wiki;

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.mahout.math.VarLongWritable;
import java.io.IOException;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class WikipediaToItemPrefsMapper extends
        Mapper<LongWritable, Text, VarLongWritable, VarLongWritable> {
    private static final Pattern NUMBERS = Pattern.compile("(\\d+)");
    @Override
    protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {
        String line = value.toString();
        Matcher m = NUMBERS.matcher(line);
        m.find();
        VarLongWritable userID = new VarLongWritable(Long.parseLong(m.group()));
        VarLongWritable itemID = new VarLongWritable();
        while (m.find()) {
            itemID.set(Long.parseLong(m.group()));
            context.write(userID, itemID);
        }
    }
}
```

`WikipediaToUserVectorReducer.java`
```java
package com.chenbiaolong.wiki;

import org.apache.hadoop.mapreduce.Reducer;
import org.apache.mahout.math.RandomAccessSparseVector;
import org.apache.mahout.math.VarLongWritable;
import org.apache.mahout.math.Vector;
import org.apache.mahout.math.VectorWritable;
import java.io.IOException;

public class WikipediaToUserVectorReducer
        extends
        Reducer<VarLongWritable, VarLongWritable, VarLongWritable, VectorWritable> {
    public void reduce(VarLongWritable userID,
                       Iterable<VarLongWritable> itemPrefs, Context context)
            throws IOException, InterruptedException {
        Vector userVector = new RandomAccessSparseVector(Integer.MAX_VALUE, 100);
        for (VarLongWritable itemPref : itemPrefs) {
            userVector.set((int) itemPref.get(), 1.0f);
        }
        context.write(userID, new VectorWritable(userVector));
    }
}
```
第一次mapreduce操作将会将`userID itemID1 itemID2`的输入数据转变为`userID itemID1:1.0 itemID2:1.0`的形式。

` UserVectorToCooccurrenceMapper.java`
```java
package com.chenbiaolong.wiki;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.mahout.math.VarLongWritable;
import org.apache.mahout.math.Vector;
import org.apache.mahout.math.VectorWritable;
import java.io.IOException;
import java.util.Iterator;
public class UserVectorToCooccurrenceMapper extends
        Mapper<VarLongWritable, VectorWritable, IntWritable, IntWritable> {
    public void map(VarLongWritable userID, VectorWritable userVector,
                    Context context) throws IOException, InterruptedException {
        Iterator<Vector.Element> it = userVector.get().nonZeroes().iterator();
        while (it.hasNext()) {
            int index1 = it.next().index();
            Iterator<Vector.Element> it2 = userVector.get().nonZeroes().iterator();
            while (it2.hasNext()) {
                int index2 = it2.next().index();
                context.write(new IntWritable(index1), new IntWritable(index2));
            }
        }
    }
}
```
` UserVectorToCooccurrenceReducer.java`
```java
package com.chenbiaolong.wiki;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.mahout.math.RandomAccessSparseVector;
import org.apache.mahout.math.Vector;
import org.apache.mahout.math.VectorWritable;
import java.io.IOException;
public class UserVectorToCooccurrenceReducer extends
        Reducer<IntWritable, IntWritable, IntWritable, VectorWritable> {
    public void reduce(IntWritable itemIndex1,
                       Iterable<IntWritable> itemIndex2s, Context context)
            throws IOException, InterruptedException {
        Vector cooccurrenceRow = new RandomAccessSparseVector(
                Integer.MAX_VALUE, 100);
        for (IntWritable intWritable : itemIndex2s) {
            int itemIndex2 = intWritable.get();
            cooccurrenceRow.set(itemIndex2,
                    cooccurrenceRow.get(itemIndex2) + 1.0);
        }
        context.write(itemIndex1, new VectorWritable(cooccurrenceRow));
    }
}
```
接下来是job的驱动代码。我们将第一次MapReduce设置为Job0，第二次MapReduce设置为Job1. job1的输入是job0的输出，因此我们设置一个临时目录存放job0的输出。具体的配置如下面源码所示：
`wikiDriver.java`
```java
package com.chenbiaolong.wiki;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.SequenceFileInputFormat;
import org.apache.hadoop.mapreduce.lib.jobcontrol.ControlledJob;
import org.apache.hadoop.mapreduce.lib.jobcontrol.JobControl;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.mapreduce.lib.output.SequenceFileOutputFormat;
import org.apache.mahout.math.VarLongWritable;
import org.apache.mahout.math.VectorWritable;

import java.io.IOException;

public class wikiDriver {
    //hadoop的安装目录
    static final String HADOOP_HOME = "/usr/hdp/2.2.0.0-2041/hadoop";
    //HDFS的临时目录，用来放job1的输出结果
    static final String TMP_PATH = "/user/chenbiaolong/tmp/part-r-00000";
    
   //操作hdfs的文件夹
    static void OperatingFiles(String input, String output) {
        Path inputPath = new Path(input);
        Path outputPath = new Path(output);
        Path tmpPath = new Path(TMP_PATH);
        Configuration conf = new Configuration();
        conf.addResource(new Path(HADOOP_HOME+"/conf/core-site.xml"));
        conf.addResource(new Path(HADOOP_HOME+"/conf/hdfs-site.xml"));
        try {
            FileSystem hdfs = FileSystem.get(conf);
            if (!hdfs.exists(inputPath)) {
                System.out.println("input path no exist!");
            }
            if (hdfs.exists(outputPath)) {
                System.out.println("output path:"+outputPath.toString()+ " exist, deleting this path...");
                hdfs.delete(outputPath,true);
            }
            if (hdfs.exists(tmpPath)) {
                System.out.println("tmp path:"+tmpPath.toString()+ " exist, deleting this path...");
                hdfs.delete(tmpPath,true);
            }
        } catch (Exception e) {

        }

    }
    public static int main(String[] args) throws IOException {
        if (args.length!=2) {
            System.out.println("useage: <input dir> <output dir>");
            return 1;
        }
        OperatingFiles(args[0], args[1]);
        Configuration job1Conf = new Configuration();
        Job job1 = new Job(job1Conf, "job1");
        job1.setJarByClass(wikiDriver.class);
        job1.setMapperClass(WikipediaToItemPrefsMapper.class);
        job1.setReducerClass(WikipediaToUserVectorReducer.class);
        job1.setMapOutputKeyClass(VarLongWritable.class);
        job1.setMapOutputValueClass(VarLongWritable.class);
        
        //将job1输出的文件格式设置为SequenceFileOutputFormat
        job1.setOutputFormatClass(SequenceFileOutputFormat.class);
        job1.setOutputKeyClass(VarLongWritable.class);
        job1.setOutputValueClass(VectorWritable.class);
        FileInputFormat.addInputPath(job1, new Path(args[0]));
        FileOutputFormat.setOutputPath(job1, new Path(TMP_PATH));

        Configuration job2Conf = new Configuration();
        Job job2 = new Job(job2Conf, "job2");
        job2.setJarByClass(wikiDriver.class);
        job2.setMapperClass(UserVectorToCooccurrenceMapper.class);
        job2.setReducerClass(UserVectorToCooccurrenceReducer.class);
        job2.setMapOutputKeyClass(IntWritable.class);
        job2.setMapOutputValueClass(IntWritable.class);
        job2.setOutputKeyClass(IntWritable.class);
        job2.setOutputValueClass(VectorWritable.class);
        
        //将job2的输入文件格式设置为SequenceFileInputFormat
        job2.setInputFormatClass(SequenceFileInputFormat.class);
        FileInputFormat.addInputPath(job2, new Path(TMP_PATH));
        FileOutputFormat.setOutputPath(job2, new Path(args[1]));

        ControlledJob ctrlJob1 = new ControlledJob(job1.getConfiguration());
        ctrlJob1.setJob(job1);

        ControlledJob ctrlJob2 = new ControlledJob(job2.getConfiguration());
        ctrlJob2.setJob(job2);

        ctrlJob2.addDependingJob(ctrlJob1);

        JobControl JC = new JobControl("wiki job");
        JC.addJob(ctrlJob1);
        JC.addJob(ctrlJob2);

        Thread thread = new Thread(JC);
        thread.start();
        while (true) {
            if(JC.allFinished()) {
                System.out.println(JC.getSuccessfulJobList());
                JC.stop();
                System.exit(0);
            }

            if (JC.getFailedJobList().size() > 0) {
                System.out.println(JC.getFailedJobList());
                JC.stop();
                System.exit(1);
            }
        }
    }
}

```

##打包运行

将wiki工程打成jar包，上传到linux服务器运行。具体的打包过程参考我[前面写的文章][6]。
接下来将测试数据文件上传到hdfs
```shell
[hdfs@localhost wiki]$ hdfs dfs -put mywiki.txt /user/chenbiaolong/data
```
运行jar文件
```shell
hadoop jar wiki.jar com.chenbiaolong.wiki.wikiDriver  /user/chenbiaolong/data/mywiki.txt /user/chenbiaolong/wiki_output
```
这条命令指定了`user/chenbiaolong/data/mywiki.txt`为输入文件，`/user/chenbiaolong/wiki_output`为输出路径。在程序中我们指定了`/user/chenbiaolong/tmp/`为job0的输出路径。
执行的输出结果如下：
```shell
15/04/07 13:58:33 INFO impl.TimelineClientImpl: Timeline service address: http://localhost:8188/ws/v1/timeline/
15/04/07 13:58:33 INFO client.RMProxy: Connecting to ResourceManager at localhost/127.0.0.1:8050
15/04/07 13:58:34 WARN mapreduce.JobSubmitter: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
15/04/07 13:58:35 INFO input.FileInputFormat: Total input paths to process : 1
15/04/07 13:58:35 INFO mapreduce.JobSubmitter: number of splits:1
15/04/07 13:58:36 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1427799851545_0023
15/04/07 13:58:36 INFO impl.YarnClientImpl: Submitted application application_1427799851545_0023
15/04/07 13:58:36 INFO mapreduce.Job: The url to track the job: http://localhost:8088/proxy/application_1427799851545_0023/
15/04/07 13:59:01 INFO impl.TimelineClientImpl: Timeline service address: http://localhost:8188/ws/v1/timeline/
15/04/07 13:59:01 INFO client.RMProxy: Connecting to ResourceManager at localhost/127.0.0.1:8050
15/04/07 13:59:01 WARN mapreduce.JobSubmitter: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
15/04/07 13:59:03 INFO input.FileInputFormat: Total input paths to process : 1
15/04/07 13:59:03 INFO mapreduce.JobSubmitter: number of splits:1
15/04/07 13:59:03 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1427799851545_0024
15/04/07 13:59:03 INFO impl.YarnClientImpl: Submitted application application_1427799851545_0024
15/04/07 13:59:03 INFO mapreduce.Job: The url to track the job: http://localhost:8088/proxy/application_1427799851545_0024/
[job name:	job1
job id:	wiki job0
job state:	SUCCESS
job mapred id:	job_1427799851545_0023
job message:	just initialized
job has no depending job:	
, job name:	job2
job id:	wiki job1
job state:	SUCCESS
job mapred id:	job_1427799851545_0024
job message:	just initialized
job has 1 dependeng jobs:
	 depending job 0:	job1
]
```
查看输出结果，job0的输出结果：
```shell
[hdfs@localhost wiki]$ hdfs dfs -cat /user/chenbiaolong/tmp/part-r-00000/part-r-00000
```
结果输出的是乱码。这是因为我们将job0的输出格式设置为`SequenceFileOutputFormat`,这个格式便于job1读取，但无法直接用cat输出易读的字符串。
查看job1的输出结果：
```shell
[hdfs@localhost wiki]$ hdfs dfs -cat /user/chenbiaolong/wiki_output/part-r-00000
100	{106:1.0,101:2.0,103:1.0,100:5.0,105:1.0}
101	{101:4.0,103:1.0,109:1.0,100:2.0,102:2.0}
102	{101:2.0,103:2.0,109:1.0,102:3.0}
103	{101:1.0,103:3.0,100:1.0,102:2.0}
105	{106:1.0,100:1.0,108:1.0,105:2.0}
106	{106:2.0,100:1.0,105:1.0}
107	{107:1.0,110:1.0}
108	{108:1.0,105:1.0}
109	{101:1.0,109:1.0,102:1.0}
110	{107:1.0,110:1.0}
```
可以看出通过两次MapReduce操作正确得到了各个`itemID`的共现值向量。通过这个向量可以估计两个`itemID`的相似程度，便于下一步的推荐操作。


  [1]: http://chenbiaolong.com/2015/04/03/wiki1/
  [2]: http://haselgrove.id.au/wikipedia.htm
  [3]: http://chenbiaolong.com/2015/03/26/%E6%90%AD%E5%BB%BAHadoop%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83/
  [4]: http://7u2qr4.com1.z0.glb.clouddn.com/blogTimLine%E6%88%AA%E5%9B%BE20150407115152.png
  [5]: http://chenbiaolong.com/2015/04/03/wiki1/
  [6]: http://chenbiaolong.com/2015/04/01/Hadoop%E8%BF%90%E8%A1%8C%E5%8C%85%E5%90%AB%E7%AC%AC%E4%B8%89%E6%96%B9%E4%BE%9D%E8%B5%96%E5%BA%93%E7%9A%84MapReduce%E4%BD%9C%E4%B8%9A/