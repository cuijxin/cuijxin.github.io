---
layout: post
title: Hadoop3.3.1 版本 Wordcount 教程翻译
subtitle: 大数据
date: 2021-11-19
author:
header-img: img/Big-Data.jpg
catalog: true
tags:
  - 大数据
---

根据 Hadoop 文档中的 MapReduce 教程，结合 Hadoop-3.3.1 版本，学习 Map/Reduce 框架。

## 先决条件

我安装的是单机版的 Hadoop-3.3.1，仅供学习使用。

## 概述

Hadoop Map/Reduce 是一个使用简易的软件框架。

一个 Map/Reduce 作业（job）通常会把输入的数据集切分为若干独立的数据块，由 map 任务（task）以完全并行的方式处理它们。框架会对 map 的输出先进行排序，然后把结果输入给 reduce 任务。通常作业的输入和输出都会被存储在文件系统中。整个框架负责任务的调度和监控，以及重新执行已经失败的任务。

通常，Map/Reduce 框架和分布式文件系统是运行在一组相同的节点上的，也就是说，计算节点和存储节点通常在一起。这种配置允许框架在哪些已经存好数据的节点上高效地调度任务，这可以使整个集群的网络带宽被非常高效地利用。

Map/Reduce 框架由一个单独的 master JobTracker 和每个集群节点一个 slave TaskTracker 共同组成。master 负责调度构成一个作业的所有任务，这些任务分布在不同的 slave 上，master 监控它们的执行，重新执行已经失败的任务。而 slave 仅负责执行由 master 指派的任务。

应用程序至少应该指明输入/输出的位置（路径），并通过实现合适的接口或抽象类型供 map 和 reduce 函数。再加上其他作业的参数，就构成了作业配置（job configuration）。然后，Hadoop 的 job client 提交作业（jar 包/可执行程序等）和配置信息给 JobTracker，后者负责分发这些软件和配置信息给 slave、调度任务并监控它们的执行，同时提供状态和诊断信息给 job-client。

## 输入与输出

Map/Reduce 框架运转在 <key, value> 键值对上，也就是说，框架把作业的输入看为是一组 <key, value> 键值对，同样也产出一组 <key, value> 键值对作为作业的输出，这两组键值对的类型可能不同。

框架需要对 key 和 value 的类（classes）进行序列化操作，因此，这些类需要实现 Writable 接口。另外，为了方便框架执行排序操作，key 类必须实现 WritableComparable 接口。

一个 Map/Reduce 作业的输入和输出类型如下所示：

(input)<k1, v1> -> map -> <k2, v2> -> combine -> <k2, v2> -> reduce -> <k3, v3> (output)

## 例子：WordCount v1.0

在深入细节之前，让我们先看一个 Map/Reduce 的应用示例，以便对它们的工作方式有一个初步认识。

WordCount 是一个简单应用，它可以计算出指定数据集中每一个单词出现的次数。

这个应用适用于 单机模式、伪分布式模式或完全分布式模式三种 Hadoop 安装方式。

我这里使用的是单机模式的简单例子。

### 源代码

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

### 用法

假设环境变量配置为如下情况：

```
export JAVA_HOME=/usr/java/jdk1.8.0_202
export PATH=${JAVA_HOME}/bin:${PATH}
export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
```

编译 WordCount.java 源文件并将其打包成 jar 包：

```
$ hadoop com.sun.tools.javac.Main WordCount.java
$ jar cf wc.jar WordCount*.class
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

### 解释

WordCount 应用程序非常的直截了当。

```java
public void map(Object key, Text value, Context context
                ) throws IOException, InterruptedException {
  StringTokenizer itr = new StringTokenizer(value.toString());
  while (itr.hasMoreTokens()) {
    word.set(itr.nextToken());
    context.write(word, one);
  }
}
```

map 方法通过指定的 FileInputFormat 一次处理一行。然后，它通过 StringTokenizer 以空格为分隔符将一行切分为若干 tokens，之后，输出 <<word>, 1> 形式的键值对。

对于示例中的第一个输入，map 输出的是：

```
<Hello, 1>
<World, 1>
<Bye, 1>
<World, 1>
```

第二个输入，map 输出的是：

```
<Hello, 1>
<Hadoop, 1>
<Goodbye, 1>
<Hadoop, 1>
```

关于组成一个指定作业的 map 数目的确定，以及如何以更精细的方式去控制这些 map，我们将在教程的后续部分学习到更多的内容。

WordCount 还指定了一个 combiner。因此，每次 map 运行之后，会对输出按照 key 进行排序，然后把输出传递给本地的 combiner（按照作业的配置与 Reducer 一样），进行本地聚合。

```java
job.setMapperClass(TokenizerMapper.class);
job.setCombinerClass(IntSumReducer.class);
job.setReducerClass(IntSumReducer.class);
```

第一个 map 的输出是：

```
<Bye, 1>
<Hello, 1>
<Hello, 2>
```

第二个 map 的输出是：

```
<Goodbye, 1>
<Hadoop, 2>
<Hello, 1>
```

```java
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
```

reduce 方法仅是将每个 key（本例中就是单词）出现的次数求和。

因此这个作业的输出就是：

```
<Bye, 1>
<Goodbye, 1>
<Hadoop, 2>
<Hello, 2>
<World, 2>
```

代码中的 main 方法中的 job 对象指定了作业的几个方面，例如：通过命令传递过来的输入/输出路径、key/value 的类型、输入/输出的格式等等。随后调用 job.waitForCompletion 来提交作业并监控它的执行。

我们接下来会学习更多关于 Job，InputFormat，OutputFormat 等接口和类型的知识。

## Map/Reduce - 用户界面

这部分文档为用户将会面临的 Map/Reduce 框架中的各个环节提供了适当的细节。这应该会帮助用户更细粒度地去实现、配置和调优作业。然而，请注意每个类/接口的 javadoc 文档提供最全面的文档；本文只是想起到指南的作用。

我们会先看看 Mapper 和 Reducer 接口。应用程序通常通过提供 map 和 reduce 方法来实现它们。

然后，我们会讨论其他得核心接口，其中包括：Job，Partitioner，InputFormat，OutputFormat 等等。

最后，我们将通过讨论框架中一些有用的功能点（例如：DistributedCache，IsolationRunner 等等）来收尾。

### 核心功能描述

应用程序通常会通过提供 map 和 reduce 方法来实现 Mapper 和 Reducer 接口，它们组成作业的核心。

#### Mapper

Mapper 将输入键值对（key/value pair）映射到一组中间格式的键值对集合。

Map 是一类将输入记录集转换为中间格式记录集的独立任务。这种转换的中间格式记录集不需要与输入记录集的类型一致。一个给定的输入键值对可以映射成 0 个或多个输出键值对。

Hadoop Map/Reduce 框架为每一个 InputSplit 产生一个 map 任务，而每个 InputSplit 是由该作业的 InputFormat 产生的。

概括地说，mapper 的实现逻辑是通过 Job.setMapperClass(Class) 方法传递给作业的。然后，框架为这个任务的 InputSplit 中的每个键值对调用一次 map(WritableComparable, Writable, Context) 操作。应用程序可以通过重写 Closeable.close() 方法来执行相应的清理工作。

输出键值对不需要与输入键值对的类型一致。一个给定的输入键值对可以映射成 0 个或多个输出键值对。通过调用 context.write(WritableComparable, Writable) 方法可以收集输出的键值对。

应用程序可以使用 Counter 来报告其统计数据。

框架随后会把与一个特定的 Key 关联的所有中间过程的值（value）分成组，然后把它们传给 Reducer 以产出最终的结果。用户可以通过 Job.setGroupingComparatorClass(Class) 方法来指定具体负责分组的 Comparator。

Mapper 的输出被排序后，就被划分给每个 Reducer。分块的总数目和一个作业的 reduce 任务的数目是一样的。用户可以通过实现自定义的 Partitioner 来控制哪个 Key 被分配给具体哪个 Reducer。

用户可以选择性的通过 Job.setCombinerClass(Class) 方法来指定一个 combiner，它负责对中间过程的输出进行本地额聚集，这会有助于降低从 Mapper 到 Reducer 的数据传输量。

这些被排好序的中间过程的输出结果保存的格式是 (key-len, key, value-len, value)，应用程序可以通过 Configuration 对象来控制是否对这些中间结果进行压缩以及使用哪种 CompressionCodec 来进行压缩。

#### 需要多少个 Map？

Map 的数据通常是由输入数据的大小决定的，一般就是所有输入文件的总块（block）数。

Map 正常的并行规模大致是为每个节点（node）分配大约 10 到 100 个 map，对于 CPU 消耗较小的 map 任务可以设置到 300 个左右。由于每个任务初始化需要一定的时间，因此，比较合理的情况是 map 的执行时间至少超过 1 分钟。

这样，如果你输入 10 TB 的数据，每个块（bloak）的大小是 128 MB，你将需要大约 82,000 个 map 了，来完成任务，除非使用 Configuration.set(MRJobConfig.NUM_MAPS, int) （注意：这里仅仅是对框架进行了一个提示（hint））将这个数值设置的更高。

#### Reducer

Reducer 将与一个 key 关联的一组中间数据集规约（reduce）为一个更小的数据集。

用户可以通过 Job.setNumReduceTasks(int) 设定一个作业中 reduce 任务的数据。

概括地说，Reducer 的实现者需要通过 Job.setReducerClass(Class) 方法来将 Reducer 的逻辑实现类传递给任务，来完成 Reducer 实现类的初始化工作。然后，框架为成组的输入数据中的每个 <key,(list of values)> 对调用一次 reduce(WritableComparable, Iterable<Writable>, Context) 方法。之后应用程序可以通过重写 cleanup(Context) 方法来执行相应的清理工作。

Reducer 有 3 个主要阶段：shuffle、sort 和 reduce。

**Shuffle**

Reducer 的输入就是 Mapper 已经排好序的输出。在这个阶段，框架通过 HTTP 为每个 Reducer 获得所有 Mapper 输出中与之相关的分块。

**Sort**

这个阶段，框架将按照 key 的值对 Reducer 的输入进行分组（因为不同 mapper 的输出中可能会有相同的 key）。

Shuffle 和 Sort 两个阶段是同时进行的；map 的输出也是一边被取回一边被合并的。

**Secondary Sort**

如果有中间过程对 key 的分组规则和 reduce 前对 key 的分组规则不同的需求，那么可以通过调用
Job.setSortComparatorClass(Class) 来指定一个 Comparator。再加上调用 Job.setGroupingComparatorClass(Class) 可用于控制中间过程的 key 如何被分组，所以结合两者可以实现按值得二次排序。

**Reduce**

在这个阶段，框架为以分组得输入数据中的每个 <key,(list of values)> 对调用一次 reduce(WritableComparable, Iterable<Writable>, Context) 方法。

Reduce 任务的输出通常是通过调用 Context.write(WritableComparable, Writable) 方法写入文件系统的。

应用程序可以使用 Counter 来报告 Reduce 任务的执行进度。

Reducer 的输出是没有经过排序的。

#### 需要多少个 Reduce ？

Reduce 的数目建议是 0.95 或 1.75 乘以（<no. of nodes> \* <no. of maximum containers per node>）。

用 0.95，所有 reduce 可以在 maps 一完成时就立刻启动，开始传输 map 的输出结果。用 1.75，速度快的节点可以在完成第一轮 reduce 任务后，可以开始第二轮，这样可以得到比较好的负载均衡的效果。

增加 reduce 的数目会增加整个框架的开销，但可以改善负载均衡，降低由于执行失败带来的负面影响。

上述比例因子比整体数目稍小一些是为了给框架中的推测性任务（speculative-tasks） 或失败的任务预留一些 reduce 的资源。

#### Reducer NONE

如果没有归约的需求，那么将 reduce 任务的数据设置为 0 是允许的。

这种情况下，map 任务的输出会直接被写入由 FileOutputFormat.setOutputPath(Job, Path) 指定的输出路径。框架在把它们写入 FileSystem 之间没有对它们进行排序。

#### Partitioner

Partitioner 用于划分键值空间（key space）。

Partitioner 负责控制 map 输出结果中 key 的分割。Key（或者一个 key 子集）被用于产生分区，通常使用的是 Hash 函数。分区的数目与一个作业的 reduce 任务的数目是一样的。因此，它控制将中间过程的 key（也就是这条记录）发送给 reduce 任务中的具体的哪一个来进行 reduce 操作。

HashPartitioner 是默认的 Partitioner。

#### Counter

Counter 扮演着 MapReduce 应用程序报告其统计数据的角色。

Mapper 和 Reducer 的实现者可以使用 Counter 来报告它们的统计数据。

Hadoop Map/Reduce 框架附带了一个包含许多实用型的 mapper、reducer 和 partitioner 的类库。

### 作业配置

Job 代表着一个 Map/Reduce 作业的配置。

Job 是用户向 Hadoop 框架描述一个 Map/Reduce 作业该如何执行的主要接口。框架会尝试按照这个 Job 描述的信息去执行这个作业，然而：

- 一些参数可能会被管理者标记为 final，这意味着它们不能被更改。
- 一些作业的参数可以被直接设置（比如：Job.setNumReduceTasks(int)），而另一些参数则与框架或者作业的其他参数之间微妙的相互影响，并且设置起来比较复杂（比如：Configuration.set(JobContext.NUM_MAPS, int)）。

通常，Job 会指明 Mapper、Combiner（如果有的话）、Partitioner、Reducer、InputFormat 和 OutputFormat 的具体实现。Job 还能指定一组输入文件（FileInputFormat.setInputPaths(Job, Path…)/ FileInputFormat.addInputPath(Job, Path)) and (FileInputFormat.setInputPaths(Job, String…)/ FileInputFormat.addInputPaths(Job, String)），以及输出文件应该写在哪儿（FileOutputFormat.setOutputPath(Path)）。

Job 可以选择性的对作业设置一些高级选项，比如：设置使用的 Comparator；选择放到 DistributedCache 上的文件；中间结果或者作业输出结果是否需要压缩以及怎么压缩；作业是否允许预防性（speculative）任务的执行 (setMapSpeculativeExecution(boolean))/ setReduceSpeculativeExecution(boolean))，每个任务最大的尝试次数 (setMaxMapAttempts(int)/ setMaxReduceAttempts(int)) 等等。

当然，用户可以通过使用 Configuration.set(String, String)/ Configuration.get(String) 方法来配置或者获取应用程序需要的任意参数。然而，DistributedCache 是用于针对大规模数据量的。

### 任务的执行和环境

MRAppMaster 是在一个单独的 jvm 上以子进程的形式执行 Mapper/Reducer 任务（Task）的。

子任务会继承父 MRAppMaster 的环境。用户可以通过 Job 中的 mapreduce.{map|reduce}.java.opts 配置参数来设定子 jvm 上的附加选项，例如：通过 -Djava.library.path=<> 将一个非标准路径设为运行时的链接用以搜索共享库，等等。如果 mapreduce.{map|reduce}.java.opts 参数中包括一个 @taskid@ 的符号，它将会被替换为 map/reduce 的 taskid 的值。

下面是一个包含多个参数和替换的例子，其中包括：记录 jvm GC 日志； JVM JMX 代理程序以无密码的方式启动，这样它就能连接到 jconsole 上，从而可以查看子进程的内存和线程，得到线程的 dump；还把子 jvm 的最大堆尺寸设置为 512MB 或者 1024MB， 并为子 jvm 的 java.library.path 添加了一个附加路径。

```xml
<property>
  <name>mapreduce.map.java.opts</name>
  <value>
  -Xmx512M -Djava.library.path=/home/mycompany/lib -verbose:gc -Xloggc:/tmp/@taskid@.gc
  -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false
  </value>
</property>

<property>
  <name>mapreduce.reduce.java.opts</name>
  <value>
  -Xmx1024M -Djava.library.path=/home/mycompany/lib -verbose:gc -Xloggc:/tmp/@taskid@.gc
  -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false
  </value>
</property>
```

#### Memory Management

用户/管理员还可以通过设置 mapreduce.{map|reduce}.memory.mb 来指定使用 MapReduce 启动的子项，以及使用 MapReduce 递归启动的任何子进程的最大虚拟内存。注意，此处设置的值是每个进程的限制。mapreduce.{map|reduce}.memory.mb 的单位是兆字节（mb）。而且该值必须大于或等于传递给 JavaVM 的 -Xmx，否则 VM 可能无法启动。

注意：mapreduce.{map|reduce}.java.opts 仅用于配置从 MRAppMaster 启动的子任务。配置守护进程的内存选项请参见配置 Hadoop 守护进程的环境。

框架的某些部分可用的内存也是可配置的。在 map 和 reduce 任务中，通过调整影响操作并发性的参数和数据命中磁盘的频率，可能会影响性能。监视作业的文件系统计数器---特别是相对于从 map 到 reduce 的字节计数---对于这些参数的调优是非常宝贵的。

#### Map Parameters

从 map 发出的记录将被序列化到缓冲区中，元数据被存储到记账缓冲区。如以下选项所述，当序列化缓冲区或元数据超过阈值时，缓冲区的内容将被排序并在后台写入磁盘，同时 map 继续输出记录。如果任意一个缓冲区溢出过程中完全填满，则 map 线程将阻塞。当 map 过程执行完毕，所有剩余的记录将被写入磁盘，所有磁盘上的段将被合并到一个文件中。最小化磁盘溢出的数量可以减少 map 时间，但是较大的缓冲区也会减少 map 可用的内存。

| Name                             | Type  | Description                                                           |
| -------------------------------- | ----- | --------------------------------------------------------------------- |
| mapreduce.task.io.sort.mb        | int   | 存储从 map 发出的记录的序列化和记帐缓冲区的累积大小，以兆字节为单位。 |
| mapreduce.map.sort.spill.percent | float | 序列化缓冲区中的软限制。一旦到达，线程将开始在后台将内容溢出到磁盘。  |

其他注意事项：

- 如果在进行溢出时超过了溢出阈值，则将继续收集，直到溢出完成。例如，如果 mapreduce.map.sort.spill.percent 被设置为 0.33，并且在溢出运行时缓冲区的剩余部分被填满，那么下一次溢出将包含所有收集到的记录，或者缓冲区的 0.66，并且不会生成额外的溢出。换句话说，阈值是定义触发器，而不是阻塞。

- 大于序列化缓冲区的记录将首先触发溢出，然后溢出到单独的文件中。没有定义该记录是否将首先通过合并器。
