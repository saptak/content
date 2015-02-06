+++
date = "2015-02-06T11:58:26-08:02"
draft = false
title = "Faster Pig with Tez"

+++

### What is Pig?

Pig is a high level scripting language that is used with Apache Hadoop. Pig excels at describing data analysis problems as data flows. Pig is complete in that you can do all the required data manipulations in Apache Hadoop with Pig. In addition through the User Defined Functions(UDF) facility in Pig you can have Pig invoke code in many languages like JRuby, Jython and Java. Conversely you can execute Pig scripts in other languages. The result is that you can use Pig as a component to build larger and more complex applications that tackle real business problems.

A good example of a Pig application is the ETL transaction model that describes how a process will extract data from a source, transform it according to a rule set and then load it into a datastore. Pig can ingest data from files, streams or other sources using the User Defined Functions(UDF). Once it has the data it can perform select, iteration, and other transforms over the data. Again the UDF feature allows passing the data to more complex algorithms for the transform. Finally Pig can store the results into the Hadoop Data File System.

Pig scripts are translated into a series of MapReduce jobs or a Tez DAG that are run on the Apache Hadoop cluster. As part of the translation the Pig interpreter does perform optimizations to speed execution on Apache Hadoop. We are going to write a Pig script that will do our data analysis task.

### What is Tez?

Tez – Hindi for “speed” provides a general-purpose, highly customizable framework that creates simplifies data-processing tasks across both small scale (low-latency) and large-scale (high throughput) workloads in Hadoop. It generalizes the [MapReduce paradigm](http://en.wikipedia.org/wiki/MapReduce) to a more powerful framework by providing the ability to execute a complex DAG ([directed acyclic graph](http://en.wikipedia.org/wiki/Directed_acyclic_graph)) of tasks for a single job so that projects in the Apache Hadoop ecosystem such as Apache Hive, Apache Pig and Cascading can meet requirements for human-interactive response times and extreme throughput at petabyte scale (clearly MapReduce has been a key driver in achieving this).

### Our data processing task

We are going to read in a baseball statistics file. We are going to compute the highest runs by a player for each year. This file has all the statistics from 1871-2011 and it contains over 90,000 rows. Once we have the highest runs we will extend the script to translate a player id field into the first and last names of the players.

### Downloading the data

The data file we are using comes from the site [www.seanlahman.com](http://www.seanlahman.com/). We will SSH into the VM.

`ssh root@127.0.0.1 -p 2222;`

the password is `hadoop`

In case you are not running the Sandbox using VirtualBox, you may have to replace the 127.0.0.1 IP address with the actual IP of the VM.

You can download the data file in csv zip using the command below:

`wget http://hortonassets.s3.amazonaws.com/pig/lahman591-csv.zip`

![](http://hortonassets.s3.amazonaws.com/pig-tez/1.png)

After the file gets downloaded, `unzip lahman591-csv.zip`

![](http://hortonassets.s3.amazonaws.com/pig-tez/2.png)

### Uploading into HDFS

Let's change into the directory with `cd lahman591-csv` and use the list command `ls` to check all the files we downloaded:

![](http://hortonassets.s3.amazonaws.com/pig-tez/3.png)

We are going to upload the `Batting.csv` file using the command `hadoop fs -put ./Batting.csv /user/guest/`:

![](http://hortonassets.s3.amazonaws.com/pig-tez/4.png)

Let's check if the files are on HDFS, with the command `hadoop fs -ls /user/guest/`:

![](http://hortonassets.s3.amazonaws.com/pig-tez/5.png)

### Running Pig on MapReduce

We will run first Pig without Tez.

So, first let's create the pig script with the command `vi 1.pig`:

![](http://hortonassets.s3.amazonaws.com/pig-tez/6.png)

Press `i` in vi to enable the insert mode. Then copy the pig script below and paste it in vi:
    
    batting = LOAD '/user/guest/Batting.csv' USING PigStorage(',');
    raw_runs = FILTER batting BY $1>0;
    runs = FOREACH raw_runs GENERATE $0 AS playerID, $1 AS year, $8 AS runs;
    grp_data = GROUP runs BY (year);
    max_runs = FOREACH grp_data GENERATE group as grp, MAX(runs.runs) AS max_runs;
    join_max_runs = JOIN max_runs BY ($0, max_runs), runs BY (year, runs);
    join_data = FOREACH join_max_runs GENERATE $0 AS year, $2 AS playerID, $1 AS runs;
    DUMP join_data;
    

Then hit `esc` button and type `:wq` to save the file:

![](http://hortonassets.s3.amazonaws.com/pig-tez/7.png)

Now we can execute the pig script using the MapReduce engine by simply typing `pig 1.pig`:

![](http://hortonassets.s3.amazonaws.com/pig-tez/8.png)

It typically takes a little more than two minutes to finish on our single node pseudocluster. Note the time it took on our machine after it completes:

![](http://hortonassets.s3.amazonaws.com/pig-tez/9.png)

### Running Pig on Tez

Let's run the same Pig script with Tez using the command `pig -x tez 1.pig`:

![](http://hortonassets.s3.amazonaws.com/pig-tez/10.png)

This time note the time after the execution completes:

![](http://hortonassets.s3.amazonaws.com/pig-tez/11.png)

On our machine it took around 58 seconds with Pig using the Tez engine. That is more than 2X faster than Pig using MapReduce even without any specific optimization in the script for Tez.

Tez definitely lives up to it's name.
