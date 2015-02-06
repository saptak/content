# NEW: Simulating and transporting Realtime event stream with Apache Kafka

This tutorial will show how geo-location information from trucks can be combined with sensors on roads which report real-time events like speeding, lane-departure, unsafe tailgating, and unsafe following distances. [**Apache Kafka**](http://kafka.apache.org/) can be used on the Hortonworks Data Platform to capture these data real-time events. In coming tutorials, we will persist them to different data stores/queue (HBase, HDFS, ActiveMQ) for further analysis.

[**Apache Kafka**](http://kafka.apache.org/) is an open source messaging system designed for:

  * Persistent messaging
  * High throughput
  * Distributed
  * Multi-client support
  * Real time
![Kafka Producer-Broker-Consumer](http://hortonassets.s3.amazonaws.com/mda/Screen+Shot+2014-07-08+at+9.24.54+PM.png)  
Kafka Producer-Broker-Consumer

In this tutorial, you will learn the following topics:

  1. Install and Start Kafka on [Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/).
  2. Review Kafka and ZooKeeper Configs
  3. Create Kafka topics for Truck events.
  4. Writing Kafka Producers for Truck events.

## Prerequisites

A working Hadoop cluster : the easiest way to get a pre-configured and fully functional Hadoop cluster is to download the [Hortonworks SandBox](http://hortonworks.com/products/hortonworks-sandbox/).

## Tutorial

### Step 1:

**Login to Hortonworks Sandbox.**

After downloading the Sandbox and running the VM, login to Ambari using the URL [http://127.0.0.1:8080/](http://127.0.0.1:8080/).

The username and password is `admin` and `admin`.

### Step 2:

**Setup Kafka.**

  1. After login to Ambari, select Actions -> Add Service:  
  
![](http://hortonassets.s3.amazonaws.com/mda/kafka/image01.png)

  2. Select Kafka from the list of Services and click Next:  
  
![](http://hortonassets.s3.amazonaws.com/mda/kafka/image04.png)

  3. Keep clicking `Next` with the selcted defaults until you reach the following screen:  
  
![](http://hortonassets.s3.amazonaws.com/mda/kafka/image07.png)  
  
Add logs.dir = `/tmp/kafka-logs`

  4. Deploy  
  
![](http://hortonassets.s3.amazonaws.com/mda/kafka/image10.png)

  5. Check Progress  
  
![](http://hortonassets.s3.amazonaws.com/mda/kafka/image13.png)  
  
After successful Start you may be asked to restart some dependent Services like Nagios. Please select the appropriate Services and click Restart.

4.Setting up Kafka on Sandbox with ZooKeeper.  
  
![Single Broker based Kakfa cluster](http://hortonassets.s3.amazonaws.com/mda/Screen+Shot+2014-07-08+at+10.33.50+PM.png)  
  
Kafka provides the default and simple ZooKeeper configuration file used for launching a single local ZooKeeper instance. Here, ZooKeeper serves as the coordination interface between the Kafka broker and consumers.

The important Zookeeper properties can be checked in Ambari as below:  
  
![](http://hortonassets.s3.amazonaws.com/mda/kafka/image16.png)

Ensure the following value of clientPort under Ambari  
  
clientPort=2181. By default ZooKeeper server listens on port number 2181.

Also verify if zookeeper service is running:

![](http://hortonassets.s3.amazonaws.com/mda/kafka/image19.png)  


If this port 2181 is busy or is consumed by other processes, then you could change the default port number of ZooKeeper to any other valid port number.

In case zookeeper is not running, you can start the Zookeeper service from Ambari:

![](http://hortonassets.s3.amazonaws.com/mda/kafka/image22.png)  


### Step 3:

**Verify Kafka Broker.**

Verify the ‘zookeeper.connect’ parameter to point to port number 2181.

![](http://hortonassets.s3.amazonaws.com/mda/kafka/image25.png)  


we will SSH in to follow the rest of the steps.  
  
ssh root@127.0.0.1 -p 2222;  
  
the password is hadoop  
  
Check running java processes using the “jps” command.  
  
This command should list Kafka and QuorumPeerMain

jps

![check Kafka running](http://hortonassets.s3.amazonaws.com/mda/Check+Kafka+Running.png)  
check Kafka running

### Step 4:

`cd /usr/hdp/2.2.0.0-1084/kafka`

**Create Kafka topics for truck event.**

Execute this command to create Topics for ‘truckevent':
    
    bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic truckevent
    

Note: Sometimes Kafka does not listen to localhost, you may need to use IP instead.

Check if topic ‘truckevent’ was created successfully with the following command:
    
    bin/kafka-topics.sh --list --zookeeper localhost:2181
    

![Kafka Topics](http://hortonassets.s3.amazonaws.com/mda/Kafka+Topics.png)  
Kafka Topics

### Step 5:

**Writing Kafka Producers for truck events.**

Producers are applications that create Messages and publish them to the Kafka broker for further consumption.

![Kafka Producers for truck events](http://hortonassets.s3.amazonaws.com/mda/Screen+Shot+2014-07-09+at+12.31.03+AM.png)  
Kafka Producers for truck events

In this tutorial we shall use a Java API to produce Truck events.

The Java code in `TruckEventsProducer.java` will generate data with following columns:
    
     `driver_name` string,  
     `driver_id` string,  
     `route_name` string,  
     `route_id` string,  
     `truck_id` string,  
     `timestamp` string,  
     `longitude` string,  
     `latitude` string,  
     `violation` string,  
     `total_violations` string
    

This Java Truck events producer code uses [New York City Truck Routes (kml)](http://www.nyc.gov/html/dot/downloads/misc/all_truck_routes_nyc.kml) file which defines road paths with Latitude and Longitude information.

The Truck Events Producer java code and the NYC Truck routes kml file can be downloaded with following commands.
    
    mkdir /opt/TruckEvents  
    cd /opt/TruckEvents  
    wget http://hortonassets.s3.amazonaws.com/mda/Tutorials-master.zip  
    unzip Tutorials-master.zip
    

### Step 6:

**Compiling with Maven.**

Download and extract Apache Maven as shown in the commands below and set up the environment ‘PATH’ variable.
    
    wget http://www.carfab.com/apachesoftware/maven/maven-3/3.2.2/binaries/apache-maven-3.2.2-bin.tar.gz  
    tar xvf apache-maven-3.2.2-bin.tar.gz  
    mv apache-maven-3.2.2 /usr/local/  
    export PATH=/usr/local/apache-maven-3.2.2/bin:$PATH  
    mvn -version
    

![Maven Version](http://hortonassets.s3.amazonaws.com/mda/maven+version.png)  
Maven Version

Add `export PATH=/usr/local/apache-maven-3.2.2/bin:$PATH` to the `~/.bashrc` file to auto set the values for `$PATH`.

Now lets compile and execute the code to generate Truck Events.
    
    cd /opt/TruckEvents/Tutorials-master  
    mvn clean package
    

![mvn clean pacakge](http://hortonassets.s3.amazonaws.com/mda/mvn+clean+package.png)  
mvn clean pacakge

Once the code is successfully compiled we shall see a new target directory created in the current folder. The binaries for all the Tutorials are in this target directory and the source code in src. To start the Kafka Producer we execute the following command to see the output as shown in the screenshot below.
    
    java -cp target/Tutorial-1.0-SNAPSHOT.jar com.hortonworks.tutorials.tutorial1.TruckEventsProducer localhost:9092 localhost:2181 &
    

![TruckEventsProducer Running](http://hortonassets.s3.amazonaws.com/mda/start+kafka+producer.png)  
TruckEventsProducer Running

We have now successfully compiled and have the Kafka producer publishing messages to the Kafka cluster.  
  
To verify, execute the following command:
    
    [root@sandbox ]# /usr/hdp/2.2.0.0-1084/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic truckevent --from-beginning  
    SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".  
    SLF4J: Defaulting to no-operation (NOP) logger implementation  
    SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.  
    2014-07-14 20:25:13.423|01|11|Unsafe following distance|    -74.01214432911161|40.70899264449202  
    2014-07-14 20:25:14.193|02|12|Unsafe tail distance| -74.01214432911161|40.70899264449202  
    2014-07-14 20:25:14.207|03|13|Overspeed|    -74.01214432911161|40.70899264449202  
    2014-07-14 20:25:14.22|02|12|Overspeed| -74.01150567251823|40.70295884274092  
    2014-07-14 20:25:14.236|03|13|Normal|   -74.01150567251823|40.70295884274092  
    2014-07-14 20:25:14.253|02|12|Unsafe following distance|    -74.01150567251823|40.70295884274092  
    2014-07-14 20:25:14.263|03|13|Lane Departure|   -74.01046533975128|40.71153454636318  
    2014-07-14 20:25:14.271|02|12|Lane Departure|   -74.01046533975128|40.71153454636318
    

To stop kafka, you can use the command:
    
    service kafka stop
    

## Producer Code description

We use the TruckEventsProducer.java file under the src/main/java/tutorial1/ directory to generate the Kafka TruckEvents. This uses the all_truck_routes_nyc.kml data file available from [NYC DOT](http://www.nyc.gov/html/dot/html/motorist/trucks.shtml). We use Java API’s to produce Truck Events.
    
    [root@sandbox ~]# ls /opt/TruckEvents/Tutorials-master/src/main/java/tutorial1/TruckEventsProducer.java  
    [root@sandbox ~]# ls /opt/TruckEvents/Tutorials-master/src/main/resources/all_truck_routes_nyc.kml
    

The java file contains 3 functions

  * **public class TruckEventsProducer**

We configure the Kafka producer in this function to serialize and send the data to Kafka Topic ‘truckevent’ created in the tutorial. The code below shows the Producer <k, v="" style="box-sizing: border-box;">class used to generate messages.
    
    String TOPIC = "truckevent";  
    ProducerConfig config = new ProducerConfig(props);  
    Producer<String, String> producer = new Producer<String, String>(config);
    

The properties of the producer are defined in the ‘props’ variable. The events, truckIds and the driverIds data is selected with random function from the array variables.
    
    Properties props = new Properties();  
    props.put("metadata.broker.list", args[0]);  
    props.put("zk.connect", args[1]);  
    props.put("serializer.class", "kafka.serializer.StringEncoder");  
    props.put("request.required.acks", "1");
    
    
    
    String[] events = {"Normal", "Normal", "Normal", "Normal", "Normal", "Normal", "Lane Departure", "Overspeed", "Normal", "Normal", "Normal", "Normal", "Lane Departure","Normal", "Normal", "Normal", "Normal",  "Unsafe tail distance", "Normal", "Normal", "Unsafe following distance", "Normal", "Normal", "Normal", "Normal", "Overspeed", "Normal", "Normal", };  
    String[] truckIds = {"01", "02", "03"};  
    String[] driverIds = {"11", "12", "13"};
    

KeyedMessage class takes the topic name, partition key, and the message value that needs to be passed from the producer as follows:

**class KeyedMessage[K, V](http://hortonworks.com/hadoop-tutorial/simulating-transporting-realtime-events-stream-apache-kafka/val%20topic:%20String,%20val%20key:%20K,%20val%20message:%20V)**
    
    KeyedMessage<String, String> data = new KeyedMessage<String, String>(TOPIC, finalEvent);
    

The Kafka producer events with timestamps are created by selecting the data from above arrays and geo location from the all_truck_routes_nyc.kml file.
    
    KeyedMessage<String, String> data = new KeyedMessage<String, String>(TOPIC, finalEvent);  
    LOG.info("Sending Messge #:" + i +", msg:" + finalEvent);  
    producer.send(data);  
    Thread.sleep(1000);'
    

To transmit the data we now build an array using the GetKmlLangList() and getLatLong() function.

  * **private static String getLatLong**

This function returns coordinates in Latitude and Longitude format.
    
     if (latLong.length == -1)  
     {
        return latLong[1].trim() + "|" + latLong[0].trim();  
     }
    

  * **public static String[] GetKmlLanLangList**

This method is reading KML file which is an XML file. This xml file is loaded in File fXmlFile variable.
    
    File fXmlFile = new File(urlString);
    

Which will parse this file by running through each node (Node.ELEMENT_NODE) in loop. The XML element "coordinates" has array of two items lat, long. The function reads the lat, long and returns the values in array.

This tutorial gives you brief glimpse of how to use Apache Kafka to transport real-time events data.
