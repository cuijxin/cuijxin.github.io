---
layout: post
title: Hadoop3.3.1版本 MapReduce wordcount 示例编译
subtitle: 大数据
date: 2021-11-19
author:
header-img: img/Big-Data.jpg
catalog: true
tags:
  - 大数据
---

根据 Hadoop 文档中的 MapReduce 教程，结合 Hadoop-3.3.1 版本，学习 Map/Reduce 框架。

**先决条件**

我安装的是单机版的 Hadoop-3.3.1，仅供学习使用。

**概述**

Hadoop Map/Reduce 是一个使用简易的软件框架。

一个 Map/Reduce 作业（job）通常会把输入的数据集切分为若干独立的数据块，由 map 任务（task）以完全并行的方式处理它们。框架会对 map 的输出先进行排序，然后把结果输入给 reduce 任务。通常作业的输入和输出都会被存储在文件系统中。整个框架负责任务的调度和监控，以及重新执行已经失败的任务。

通常，Map/Reduce 框架和分布式文件系统是运行在一组相同的节点上的，也就是说，计算节点和存储节点通常在一起。这种配置允许框架在哪些已经存好数据的节点上高效地调度任务，这可以使整个集群的网络带宽被非常高效地利用。

Map/Reduce 框架由一个单独的 master JobTracker 和每个集群节点一个 slave TaskTracker 共同组成。master 负责调度构成一个作业的所有任务，这些任务分布在不同的 slave 上，master 监控它们的执行，重新执行已经失败的任务。而 slave 仅负责执行由 master 指派的任务。

应用程序至少应该指明输入/输出的位置（路径），并通过实现合适的接口或抽象类型供 map 和 reduce 函数。再加上其他作业的参数，就构成了作业配置（job configuration）。然后，Hadoop 的 job client 提交作业（jar 包/可执行程序等）和配置信息给 JobTracker，后者负责分发这些软件和配置信息给 slave、调度任务并监控它们的执行，同时提供状态和诊断信息给 job-client。

**输入与输出**

Map/Reduce 框架运转在 <key, value> 键值对上，也就是说，框架把作业的输入看为是一组 <key, value> 键值对，同样也产出一组 <key, value> 键值对作为作业的输出，这两组键值对的类型可能不同。

框架需要对 key 和 value 的类（classes）进行序列化操作，因此，这些类需要实现 Writable 接口。另外，为了方便框架执行排序操作，key 类必须实现 WritableComparable 接口。

一个 Map/Reduce 作业的输入和输出类型如下所示：

(input)<k1, v1> -> map -> <k2, v2> -> combine -> <k2, v2> -> reduce -> <k3, v3> (output)

**例子：WordCount v1.0**

在深入细节之前，让我们先看一个 Map/Reduce 的应用示例，以便对它们的工作方式有一个初步认识。

WordCount 是一个简单应用，它可以计算出指定数据集种每一个单词出现的次数。

这个应用适用于 单机模式、伪分布式模式或完全分布式模式三种 Hadoop 安装方式。

我这里使用的是单机模式的简单例子。

**源代码**

```java
import java.io.IOException;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

  public static class TokenizerMapper
       extends Mapper<Object, Text, Text, IntWritable>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
      }
    }
  }

  public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values,
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

**用法**

假设环境变量配置为如下情况：

```
export JAVA_HOME=/usr/java/jdk1.8.0_202
export PATH=${JAVA_HOME}/bin:${PATH}
export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
```

编译 WordCount.java 源文件并将其打包成 jar 包：

```
$ hadoop com.sun.tools.javac.Main WordCount.java
$ jar cf.wc.jar WordCount*.class
```

假设：

输入文件的路径为：

- /root/bigdata/wordcount/input

输出文件的路径为：

- /root/bigdata/wordcount/output

文本文件的输入示例：

```
$ hadoop fs -ls /root/bigdata/wordcount/input/
/root/bigdata/wordcount/input/file01
/root/bigdata/wordcount/input/file02

$ hadoop fs -cat /root/bigdata/wordcount/input/file01
Hello World Bye World

$ hadoop fs -cat /root/bigdata/wordcount/input/file02
Hello Hadoop Goodbye Hadoop
```

执行应用程序：

```
$ hadoop jar wc.jar WordCount /root/bigdata/wordcount/input /root/bigdata/wordcount/output
```

执行结果验证：

```
$ hadoop fs -cat /root/bigdata/wordcount/output/part-r-00000
Bye     1
Goodbye 1
Hadoop  2
Hello   2
World   2
```
