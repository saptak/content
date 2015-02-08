+++
date = "2015-02-06T11:58:26-08:25"
draft = false
title = "Ingesting and processing Real-time events with Apache Storm"

+++


Trucking business is a high-risk business where truck drivers venture into remote areas, often despite harsh weather conditions and chaotic traffic on a daily basis. Using this solution illustrating Modern Data Archtecture with Hortonworks Data Platform, we have developed a centralized management system that can help reduce risk and lower the total cost of operations. This system can take into consideration adverse weather conditions, the driver’s driving patterns, current traffic conditions and other criteria to alert and inform the management staff and the drivers themselves when risk factors run high.

In a [previous tutorial](http://hortonworks.com/hadoop-tutorial/simulating-transporting-realtime-events-stream-apache-kafka/) we learned to collect this data using Apache Kafka.

In this tutorial we will use [**Apache Storm**](http://hortonworks.com/labs/storm/) on the Hortonworks Data Platform to capture these data events and process them in real time for further analysis.

### Technologies Used

Hadoop, HDFS, Hive, HBase, Kafka, Storm, Falcon, Leafletjs.

### Data Sets Used

  * New York City Truck Routes from NYC DOT.
  * Truck Events Data generated using a custom simulator.
  * Weather Data, collected using APIs from Forcast.io.
  * Traffic Data, collected using APIs from MapQuest.

_All data sets used in these tutorials are real data sets but modified to fit these use cases._

### About Storm

Apache Storm is an Open Source distributed, reliable, fault tolerant system for real time processing of data at high velocity.

It’s used for:

  * Real time analytics
  * Online machine learning
  * Continuous statics computations
  * Operational Analytics
  * And, to enforce Extract, Transform, and Load (ETL) paradigms.

Spout and Bolt are the two main components in Storm, which work together to process streams of data.

  * Spout: Works on the source of data streams. In the "Truck Events" use case, Spout will read data from Kafka “truckevent” topics.
  * Bolt: Spout passes streams of data to Bolt which processes and passes it to either a data store or another Bolt.

For details on Storm, [click here](http://hortonworks.com/labs/storm/).

In this tutorial, you will learn the following topics:

  1. Managing Storm on HDP.
  2. Creating a Storm spout to consume the Kafka ‘truckevents’ generated in [Tutorial #1](http://hortonworks.com/hadoop-tutorial/simulating-transporting-realtime-events-stream-apache-kafka/).

## Prerequisites

[Tutorial #1 should be completed successfully.](http://hortonworks.com/hadoop-tutorial/simulating-transporting-realtime-events-stream-apache-kafka/)

### Step 1: **Configure Storm.**

Verify if Apache Storm is installed and started by login into Ambari

![](http://hortonassets.s3.amazonaws.com/mda/storm2/image01.png)  


Note: If you do not see Storm listed under Services, please follow click on Action->Add Service and select Storm and deploy it.

#### Check Storm configurations on the Sandbox by login into Ambari.

  * Zookeeper configuration:  
  
Ensure storm.zookeeper.servers is set to `sandbox.hortonworks.com`

![](http://hortonassets.s3.amazonaws.com/mda/storm2/image04.png)

  * Check the local directory configuration:  
  
Ensure storm.local.dir is set to `/hadoop/storm`

![](http://hortonassets.s3.amazonaws.com/mda/storm2/image07.png)

  * Check the nimbus host configuration:  
  
Ensure nimbus.host is set to `sandbox.hortonworks.com`

![](http://hortonassets.s3.amazonaws.com/mda/storm2/image10.png)

  * Check the slots allocated:  
  
Ensure supervisor.slots.ports is set to `[6700, 6701]`

![](http://hortonassets.s3.amazonaws.com/mda/storm2/image13.png)

  * Check the UI configuration port:  
  
Ensure ui.port is set to `8744`

![](http://hortonassets.s3.amazonaws.com/mda/storm2/image16.png)

#### Check the Storm UI from the Quick Links

![](http://hortonassets.s3.amazonaws.com/mda/storm2/image19.png)  


Now you can see the UI:

![Storm UI](http://hortonassets.s3.amazonaws.com/storm-truck/Storm+UI.png)  
Storm UI

### Step 2. **Creating a Storm Spout to consume the Kafka truck events generated in Tutorial #1.**

#### New York City truck routes:

The required [New York City truck routes](http://www.nyc.gov/html/dot/downloads/misc/all_truck_routes_nyc.kml) KML file is included in the master.zip file. If required you can download the latest copy of the file with the following command. 
    
    [root@sandbox ~]# wget http://www.nyc.gov/html/dot/downloads/misc/all_truck_routes_nyc.kml --directory-prefix=/opt/TruckEvents/Tutorials-master/src/main/resources/
    

Compile the code using Maven after downloading a new data file or on completing any changes to the code under `/opt/TruckEvents/Tutorials-master/src` directory.
    
    [root@sandbox ~]# cd /opt/TruckEvents/Tutorials-master/  
    [root@sandbox ~]# export PATH=/usr/local/apache-maven-3.2.2/bin:$PATH  
    [root@sandbox ~]# mvn clean package
    

![mvn clean package](http://hortonassets.s3.amazonaws.com/storm-truck/mvn+clean+package.png)  
mvn clean package![mvn build success](http://hortonassets.s3.amazonaws.com/storm-truck/Build+Success.png)  
mvn build success

We now have a successfully compiled the code.  
  
Look through the Java code under ‘src/main/java’ directory.

#### Verify that Kafka process is running

After successful completion of Tutorial #1, we should now have a running instance of Kafka. To verify execute ‘jps’ command to find Kafka in the running Java processes.

![Kafka Running](http://hortonassets.s3.amazonaws.com/storm-truck/Kafka+Running.png)  
Kafka Running

If Kafka is not running, you can start by login into Ambari (Follow instruction in the previous tutorial)

#### Loading Storm Topology

We now have ‘supervisor’ daemon and Kafka processes running.  
  
The command below will start a new Storm Topology for TruckEvents.
    
    [root@sandbox ~]# [root@sandbox ~]# storm jar target/Tutorial-1.0-SNAPSHOT.jar com.hortonworks.tutorials.tutorial2.TruckEventProcessingTopology
    

![storm new topology](http://hortonassets.s3.amazonaws.com/storm-truck/storm+new+topology.png)  
storm new topology

Refresh the browser to see new Topology ‘truck-event-processor’ in the browser.

![truck event processor new topology](http://hortonassets.s3.amazonaws.com/storm-truck/truck+event+topology.png)  
truck event processor new topology

#### Generationg TruckEvents

The TruckEvents producer can now be executed as we did in Tutorial #1.
    
    java -cp target/Tutorial-1.0-SNAPSHOT.jar com.hortonworks.tutorials.tutorial1.TruckEventsProducer  localhost:9092 localhost:2181
    

![Truck Events Producer](http://hortonassets.s3.amazonaws.com/storm-truck/TruckEvents+Producer.png)  
Truck Events Producer

Refresh the browser and you can see that messages are processed in real time by Spout.

![kafkaSpout count](http://hortonassets.s3.amazonaws.com/storm-truck/Storm+spout+messages+count.png)  
kafkaSpout count

### Code description

Let us review the code used in this tutorial. The source files are under the `/opt/TruckEvents/Tutorials-master/src/main/java/tutorial2` folder. 
    
    [root@sandbox Tutorials-master]# ls -l src/main/java/tutorial2/  
    total 16  
    -rw-r--r-- 1 root root  861 Jul 24 23:34 BaseTruckEventTopology.java  
    -rw-r--r-- 1 root root 1205 Jul 24 23:34 LogTruckEventsBolt.java  
    -rw-r--r-- 1 root root 2777 Jul 24 23:34 TruckEventProcessingTopology.java  
    -rw-r--r-- 1 root root 2233 Jul 24 23:34 TruckScheme.java  
    [root@sandbox Tutorials-master]# 
    

#### BaseTruckEventTopology.java
    
    topologyConfig.load(ClassLoader.getSystemResourceAsStream(configFileLocation));
    

Is the base class, where the topology configurations is initialized from the /resource/truck_event_topology.properties files.

#### TruckEventProcessingTopology.java

This is the storm topology configuration class, where the Kafka spout and LogTruckevent Bolts are initialized. In the following method the Kafka spout is configured.
    
    private SpoutConfig constructKafkaSpoutConf()  
        {
     …  
     SpoutConfig spoutConfig = new SpoutConfig(hosts, topic, zkRoot, consumerGroupId);  
    …
            spoutConfig.scheme = new SchemeAsMultiScheme(new TruckScheme());
    
    return spoutConfig;  
        }
    

A logging bolt that prints the message from the Kafka spout was created for debugging purpose just for this tutorial.  
  
`  
public void configureLogTruckEventBolt(TopologyBuilder builder)  
{  
LogTruckEventsBolt logBolt = new LogTruckEventsBolt();  
builder.setBolt(LOG_TRUCK_BOLT_ID, logBolt).globalGrouping(KAFKA_SPOUT_ID);  
}  
`

The topology is built and submitted in the following method;  
  
`  
private void buildAndSubmit() throws Exception  
{  
...  
StormSubmitter.submitTopology("truck-event-processor",  
conf, builder.createTopology());  
}  
`

#### TruckScheme.java

Is the deserializer provided to the kafka spout to deserialize kafka byte message stream to Values objects.
    
    public List<Object> deserialize(byte[] bytes)  
            {
            try  
                    {
                String truckEvent = new String(bytes, "UTF-8");  
                String[] pieces = truckEvent.split("\\|");
    
                Timestamp eventTime = Timestamp.valueOf(pieces[0]);  
                String truckId = pieces[1];  
                String driverId = pieces[2];  
                String eventType = pieces[3];  
                String longitude= pieces[4];  
                String latitude  = pieces[5];  
                return new Values(cleanup(driverId), cleanup(truckId),  
                                        eventTime, cleanup(eventType), cleanup(longitude), cleanup(latitude));
    
            }  
                    catch (UnsupportedEncodingException e)  
                    {
                        LOG.error(e);  
                        throw new RuntimeException(e);  
            }
    
        }
    

#### LogTruckEventsBolt.java

LogTruckEvent spout logs the kafka message received from the kafka spout to the log files under /var/log/storm/worker-*.log  
  
`  
public void execute(Tuple tuple)  
{  
LOG.info(tuple.getStringByField(TruckScheme.FIELD_DRIVER_ID) + "," +  
tuple.getStringByField(TruckScheme.FIELD_TRUCK_ID) + "," +  
tuple.getValueByField(TruckScheme.FIELD_EVENT_TIME) + "," +  
tuple.getStringByField(TruckScheme.FIELD_EVENT_TYPE) + "," +  
tuple.getStringByField(TruckScheme.FIELD_LATITUDE) + "," +  
tuple.getStringByField(TruckScheme.FIELD_LONGITUDE));  
}  
`

In this tutorial we have learned to capture data from Kafka Producer into Storm Spout. This data can now be processed in real time. In our next Tutorial, using Storm Bolt we will learn to store data into multiple sources to persist.
