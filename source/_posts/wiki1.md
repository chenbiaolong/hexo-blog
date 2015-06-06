title: Mahout in action分析维基百科数据例子(一)
date: 2015-04-03 16:34:27
tags: hadoop
category: 大数据
---
##概要

本文主要根据mahout in action第六章分析维基百科链接数据的例子编写。大部分内容是直接翻译的mahout in action，不过不是逐字翻译，加入了一些个人理解。关于本文的前提背景可以参考其他博主翻译的文章：
[Mahout in action 中文版-6.分布式推荐计算-6.1][1]
[Mahout in action 中文版-6.分布式推荐计算-6.2][2]

<!-- more -->

##6.3.2 使用MapReduce：生成用户向量

在这个例子中，我们主要目的是利用`item-base`的推荐算法实现维基百科网页的推荐。基础理论就是：如果网页A和网页B同时被多个网页同时引用，那么我们就可以推测网页A和网页B的内容相近。当一个用户看了网页A时，我们可以考虑将网页B推荐给它。计算的输入文件是维基百科的链接数据文件，这个文件的每一行数据并不是按照`userId`,`itemId`,`preference`格式排列的，而是按照`userID:itemID1 itemID2 itemID3 ...`格式排列。这个格式代表的意义是ID为`userID`的网页有指向ID为`itemID1`,`itemID2...`网页的超链接。可以在[这里][3]获得所需要的维基百科链接数据文件。将这个文件传送到HDFS文件系统，以便Hadoop集群使用该文件。
为了实现网页推荐功能，我们需要2次的MapReduce操作。

第一次的MapReduce操作将生成用户向量:

 - 在MapReduce框架中，输入的文件将会被看成（Long，String）的键值对，Key是Long类型，它表示值在文件中的位置。String是该行具体的数据内容。比如`239/98955:590 22 9059`，`239`表示数据行位于的文件位置，`95955:590 22`是该行的内容
 - 每一行将会被一个map函数解析为一个`user ID`和若干个`item ID`。这个map函数将会生成新的key/value对：一个`user ID`关联一个`item ID`,比如`98955/590`
 - 框架收集所有的被`user ID`关联的`item ID`
 - reduce函数输出`user ID`和与该用户对各个item的喜好向量。用户喜好向量只能为0或者1（因为链接只有有和没有2种情况）。比如`95955/[590:1.0,22:1.0,9059:1.0]`表示网页`95955`有链接到`590`,`22`,`9059`的超链接。需要注意的是这里`userID`和`itemID`只是为了表示方便，实际上`userID`和`itemID`均表示的是各个网页的ID值。
这些思路的具体实现可以参考代码6.1和6.2.代码实现了Hadoop的mapper和reducer接口。
列表6.1 解析维基百科链接数据的mapper函数
```java
public class WikipediaToItemPrefsMapper
    extends Mapper<LongWritable,Text,VarLongWritable,VarLongWritable> {
  private static final Pattern NUMBERS = Pattern.compile("(\\d+)");
  public void map(LongWritable key,
                  Text value,
                  Context context) 
        throws IOException, InterruptedException {
    String line = value.toString();
    Matcher m = NUMBERS.matcher(line);
    m.find();
    VarLongWritable userID = 
    new VarLongWritable(Long.parseLong(m.group()));
    VarLongWritable itemID = new VarLongWritable();
    while (m.find()) {
      itemID.set(Long.parseLong(m.group()));
      context.write(userID, itemID);
    }
   }
}
```

列表6.2 Reducer函数
```java
public static class WikipediaToUserVectorReducer extends 
    Reducer<VarLongWritable,VarLongWritable,VarLongWritable,VectorWritable> {
  public void reduce(VarLongWritable userID,
            Iterable<VarLongWritable> itemPrefs,
            Context context) 
        throws IOException, InterruptedException {
    Vector userVector = new RandomAccessSparseVector(
    Integer.MAX_VALUE, 100); 
    for (VarLongWritable itemPref : itemPrefs) {
    userVector.set((int)itemPref.get(), 1.0f);
    }
    context.write(userID, new VectorWritable(userVector));
    }
}
```
可以看出，实际上第一次MapReduce操作只是将`varLongWritable`的`itemID`转换成`vectorWritable`类型的`userVector`。

##6.3.3 使用MapReduce：计算共现值

下一个MapReduce操作将根据第一个Mapreduce的结果计算出各个item的共现值(co-occurrence)。这里对共现值简单做一个解释：比如网页A和网页B同时被网页C、D、E链接，那么网页A和网页B的共现值为3。第二个MapReduce函数

 1. 输入是用户ID与用户喜好向量键值对——即第一次MapReduce结果。比如`98955/[590:1.0,22:1.0,9059:1.0]`
 2. 根据第一次mapreduce的结果计算出各个item的共现值。比如第一次mapreduce的一个结果`98955/[590:1.0,22:1.0,9059:1.0]`，先忽略`userID:98955`,这个用户ID对应的各个`itemID`都同时被`userID:98955`链接，因此各个`itemID`两两之间的共现值应该加1。因此我们的第二个job的map操作可以用`itemID1:itemID2`这种格式表示`itemID1`和`itemID2`同时被某个`userID`网页链接，共现值加1.比如用`590/22`表示网页590和22的共现值需要加1.
 3. reduce操作就是统计`itemID`之间的总共现值。在map操作后拥有共现关系（`itemID1`和`itemID2`同时被某网页链接表示`itemID1`和`itemID2`拥有共现关系）的各个`itemID`都组成了`itemID1:itemID2`形式的key-value对。reduce操作需要统计各个网页之间的共现关系。


假设map操作后我们得到以下结果：
```
100 101
100 105
100 110
100 101
100 105
100 101
```
reduce操作计算100与其他各个ID的共现值，得到另外一组`long:vector`形式的输出结果。上述例子将输出
`100/[101:3.0, 105:2.0, 110:1.0]`的形式。这个共现值有什么用呢？简单来说，两个网页的共现值越大，意味着这两个网页越相似，我们也就可以根据这个特性进行网页推荐了。

Listing 6.3 Mapper component of co-occurrence computation
```java
public class UserVectorToCooccurrenceMapper extends
		Mapper<VarLongWritable, VectorWritable, IntWritable, IntWritable> {

	public void map(VarLongWritable userID, VectorWritable userVector,
			Context context) throws IOException, InterruptedException {
		Iterator<Vector.Element> it = userVector.get().iterateNonZero();
		while (it.hasNext()) {
			int index1 = it.next().index();
			Iterator<Vector.Element> it2 = userVector.get().iterateNonZero();
			while (it2.hasNext()) {
				int index2 = it2.next().index();
				context.write(new IntWritable(index1), new IntWritable(index2));
			}
		}
	}
}
```


Listing 6.4 Reducer component of co-occurrence computation
```
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

##小结

本文主要解释了实现维基百科网页推荐的基础原理：通过网页之间的共现值判断网页之间的是否相似。本质上就是`item-base`类型的推荐算法。更详细的内容可以参考`mahout in action`的相关章节。由于`mahout in action`只是给出了mapreduce的实现算法，具体利用`hadoop`运行这两个mapreduce操作时会出现一些问题：比如有一些class在新版本的`hadoop`已经作了调整;这里的两个mapreduce操作存在相互依赖如何利用`hadoop`接口解决等。在下一篇文章中我将给出实现这两个mapreduce操作的相应工程代码。


  [1]: http://www.cnblogs.com/colorfulkoala/archive/2012/12/16/2820855.html
  [2]: http://www.cnblogs.com/colorfulkoala/archive/2012/12/16/2820866.html
  [3]: http://haselgrove.id.au/wikipedia.htm