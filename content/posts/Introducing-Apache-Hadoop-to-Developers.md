+++
date = "2015-02-06T11:58:26-08:20"
draft = false
title = "Introducing Apache Hadoop to Developers"

+++


# Introducing Apache Hadoop to Developers

Apache Hadoop is a community driven open-source project goverened by the [Apache Software Foundation](http://apache.org/).

It was originally implemented at Yahoo based on papers published by Google in 2003 and 2004. Hadoop committers today work at several different organizations like Hortonworks, Microsoft, Facebook, Cloudera and many others around the world.

Since then Apache Hadoop was matured and developed to become a data platform for not just processing humongous amount of data in batch but with the advent of [YARN](http://hortonworks.com/labs/yarn/) it now supports many diverse workloads such as Interactive queries over large data with [Hive on Tez](http://hortonworks.com/labs/stinger/), Realtime data processing with [Apache Storm](http://hortonworks.com/labs/storm/), super scalable NoSQL datastore like [HBase](http://hortonworks.com/hadoop/hbase/), in-memory datastore like [Spark](http://hortonworks.com/hadoop/spark/) and the list goes on.

![](http://hortonworks.com/wp-content/uploads/2014/09/5-boxes.png)

For this introductory tutorial for Hadoop Developers we are going to focus on the basics, like:

  * The core concepts of Apache Hadoop
  * Writing a MapReduce program

We have many [tutorials](http://hortonworks.com/tutorials) which you can use with the Hortonworks Sandbox to learn about a rich and diverse set of components of the Hadoop platform.

### Core of Apache Hadoop

  * The Hadoop Distributed File System (HDFS)
  * MapReduce

A set of machines running HDFS and MapReduce is known as a Hadoop Cluster. Individual machines are known as nodes. A cluster can have as few as one node to as many as several thousands. For most application scenarios Hadoop is linearly scalable, which means you can expect better performance by simply adding more nodes.

### MapReduce

MapReduce is a method for distributing a task across multiple nodes. Each node processes data stored on that node to the extent possible.

A running Map Reduce job consists of various phases such as `Map -> Sort -> Shuffle -> Reduce`

The primary advantages of abstracting your jobs as MapReduce running over a distributed infrastructure like CPU and Storage are:

  * Automatic parallelization and distribution of data in blocks across a distributed, scale-out infrastructure.
  * Fault-tolerance against failure of storage, compute and network infrastructure
  * Deployment, monitoring and security capability
  * A clean abstraction for programmers

Most MapReduce programs are written in Java. It can also be written in any scripting language using the Streaming API of Hadoop. MapReduce abstracts all the low level plumbing away from the developer such that developers can concentrate on writing the Map and Reduce functions.

#### The MapReduce Concepts and Terminology

MapReduce jobs are controlled by a software daemon known as the `JobTracker`. The JobTracker resides on a 'master node'. Clients submit MapReduce jobs to the JobTracker. The JobTracker assigns Map and Reduce tasks to other nodes on the cluster.

These nodes each run a software daemon known as the `TaskTracker`. The TaskTracker is responsible for actually instantiating the Map or Reduce task, and reporting progress back to the JobTracker

A `job` is a program with the ability of complete execution of Mappers and Reducers over a dataset. A `task` is the execution of a single Mapper or Reducer over a slice of data.

There will be at least as many task attempts as there are tasks. If a task attempt fails, another will be started by the JobTracker. Speculative execution can also result in more task attempts than completed tasks.

#### MapReduce: The Mapper

Hadoop attempts to ensure that Mappers run on nodes which hold their portion of the data locally, to minimize network traffic. Multiple Mappers run in parallel, each processing a portion of the input data.

The Mapper reads data in the form of key/value pairs. It outputs zero or more key/value pairs
    
    map(in_key, in_value) -> (inter_key, inter_value) list
    

The Mapper may use or completely ignore the input key. For example, a standard pattern is to read a line of a file at a time. The key is the byte offset into the file at which the line starts. The value is the contents of the line itself. Typically the key is considered irrelevant. If the Mapper writes anything out, the output must be in the form of key/value pairs.

#### MapReduce: The Reducer

After the Map phase is over, all the intermediate values for a given intermediate key are combined together into a list. This list is given to a Reducer. There may be a single Reducer, or multiple Reducers, this is specified as part of the job configuration. All values associated with a particular intermediate key are guaranteed to go to the same Reducer.

The intermediate keys, and their value lists, are passed to the Reducer in sorted key order. This step is known as the 'shuffle and sort'. The Reducer outputs zero or more final key/value pairs. These are written to HDFS. In practice, the Reducer usually emits a single key/value pair for each input key.

It is possible for some Map tasks to take more time to complete than the others, often due to faulty hardware, or underpowered machines. This might cause a bottleneck as all mappers need to finish before any reducers can kick-off. Hadoop uses speculative execution to mitigate against such situations. If a Mapper appears to be running sluggishly than the others, a new instance of the Mapper will be started on another machine, operating on the same data. The results of the first Mapper to finish will be used. Hadoop will kill off the Mapper which is still running.

### Writing a MapReduce Program

In this section you will learn how to use the Hadoop API to write a MapReduce program in Java

Each of the portions (RecordReader, Mapper, Partitioner, Reducer, etc.) can be created by the developer. The developer is expected to atleast write the Mapper, Reducer, and driver code.

#### The MapReduce Example

**WordCount** example reads text files and counts how often words occur. The input is text files and the output is text files, each line of which contains a word and the count of how often it occured, separated by a tab.

Each mapper takes a line as input and breaks it into words. It then emits a key/value pair of the word and 1. Each reducer sums the counts for each word and emits a single key/value with the word and sum.

As an optimization, the reducer is also used as a combiner on the map outputs. This reduces the amount of data sent across the network by combining each word into a single record.

To run the example, the command syntax is
    
    hadoop jar hadoop-*-examples.jar wordcount [-m <#maps>] [-r <#reducers>] <in-dir> <out-dir>
    

All of the files in the input directory are read and the counts of words in the input are written to the output directory. It is assumed that both inputs and outputs are stored in HDFS. If your input is not already in HDFS, but is rather in a local file system somewhere, you need to copy the data into HDFS using a command like this:
    
    hadoop dfs -copyFromLocal <local-dir> <hdfs-dir>
    

Below is the standard wordcount example implemented in Java:
    
    
        package org.myorg;
    
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
    
            public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
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
    

Every MapReduce job consists of three portions

  * The driver code
  * Code that runs on the client to configure and submit the job
  * The Mapper
  * The Reducer

Before we look at the code, we need to cover some basic Hadoop API concepts

#### Mapper reading data from HDFS

The data passed to the Mapper is specified by an InputFormat. The InputFormat is specified in the driver code. It defines the location of the input data like a file or directory on HDFS. It also determines how to split the input data into input splits.

Each Mapper deals with a single input split. InputFormat is a factory for RecordReader objects to extract (key, value) records from the input source.

FilelnputFormat is the base class used for all file-based InputFormats. TextlnputFormat is the default FilelnputFormat. It treats each \n-terminated line of a file as a value. The Key is the byte offset within the file of that line. KeyValueTextlnputFormat maps \n-terminated lines as 'key SEP value'. By default, separator is a tab. SequenceFilelnputFormat is a binary file of (key, value) pairs with some additional metadata. SequenceFileAsTextlnputFormat is similar, but maps (key.toString( ), value.toString( )).

Keys and values in Hadoop are objects. Values are objects which implement the writable interface. Keys are objects which implement writableComparable.

#### Writable

Hadoop defines its own 'box classes' for strings, integers and so on:

  * IntWritable for ints
  * LongWritable for longs
  * FloatWritable for floats
  * DoubleWritable for doubles
  * Text for strings
  * Etc.

The writable interface makes serialization quick and easy for Hadoop. Any value's type must implement the writable interface.

#### WritableComparable

A WritableComparable is a Writable which is also Comparable. Two writableComparables can be compared against each other to determine their 'order'. Keys must be WritableComparables because they are passed to the Reducer in sorted order.

Note that despite their names, all Hadoop box classes implement both Writable and WritableComparable, for example, intwritable is actually a WritableComparable

#### Driver

The driver code runs on the client machine. It configures the job, then submits it to the cluster.

### Streaming API

Many organizations have developers skilled in languages other than Java, such as

  * C#
  * Ruby
  * Python
  * Perl

The Streaming API allows developers to use any language they wish to write Mappers and Reducers as long as the language can read from standard input and write to standard output.

The advantages of the Streaming API are that there is no need for non-Java coders to learn Java. So it results in faster development time and the ability to use existing code libraries.

#### How Streaming Works

To implement streaming, write separate Mapper and Reducer programs in the language of your choice. They will receive input via stdin. They should write their output to stdout.

If TextinputFormat (the default) is used, the streaming Mapper just receives each line from the file on stdin where no key is passed. Streaming Mapper and streaming Reducer's output should be sent to stdout as key (tab) value (newline) and the Separators other than tab can be specified.

In Java, all the values associated with a key are passed to the Reducer as an iterator. Using Hadoop Streaming, the Reducer receives its input as (key, value) pairs, one per line of standard input.

Your code will have to keep track of the key so that it can detect when values from a new key start appearing launching a Streaming Job .To launch a Streaming job, use e.g.,:
    
    hadoop jar $HADOOP_HOME/contrib/streaming/hadoop-streaming*.jar \
    -input mylnputDirs \ -output myOutputDir \
    -mapper myMap.py \
    -reducer myReduce.py \ -file myMap.py \ -file myReduce.py
    

### Hive and Pig: Motivation

MapReduce code is typically written in Java. Although it can be written in other languages using Hadoop

Streaming Requires a programmer who understands how to think in terms of MapReduce, who understands the problem they're trying to solve and who has enough time to write and test the code.

Many organizations have only a few developers who can write good MapReduce code

Meanwhile, many other people want to analyze data

  * Data analysts
  * Business analysts
  * Data scientists
  * Statisticians

So we needed a higher-level abstraction on top of MapReduce providing the ability to query the data without needing to know MapReduce intimately. Hive and Pig address these needs.

See the following tutorial for more on Hive and Pig:

  * [Process Data with Apache Hive](http://hortonworks.com/hadoop-tutorial/how-to-process-data-with-apache-hive/)
  * [Process Data with Apache Pig](http://hortonworks.com/hadoop-tutorial/how-to-process-data-with-apache-pig/)
  * [Get Started with Cascading on Hortonworks Data Platform](http://hortonworks.com/hadoop-tutorial/cascading-log-parsing/)
  * [Interactive Query for Hadoop with Apache Hive on Apache Tez](http://hortonworks.com/hadoop-tutorial/supercharging-interactive-queries-hive-tez/)
  * [Exploring Data with Apache Pig from the Grunt shell](http://hortonworks.com/hadoop-tutorial/exploring-data-apache-pig-grunt-shell/)
