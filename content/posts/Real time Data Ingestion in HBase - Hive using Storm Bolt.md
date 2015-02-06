+++
date = "2015-02-06T11:58:26-08:35"
draft = false
title = "Real time Data Ingestion in HBase & Hive using Storm Bolt"

+++


## Overview

In this tutorial, we will build a solution to ingest real time streaming data into HBase and HDFS.

In previous tutorial we have explored generating and processing streaming data with [Apache Kafka](http://hortonworks.com/hadoop-tutorial/simulating-transporting-realtime-events-stream-apache-kafka/)and [Apache Storm](http://hortonworks.com/hadoop-tutorial/ingesting-processing-real-time-events-apache-storm/). In this tutorial we will create HDFS Bolt & HBase Bolt to read the streaming data from the Kafka Spout and persist in Hive & HBase tables. 

### About HBase

HBase provides near real-time, random read and write access to tables (or to be more accurate ‘maps’) storing billions of rows and millions of columns.

In this case once we store this rapidly and continuously growing dataset from Internet of Things (IoT), we will be able to do super fast lookup for analytics irrespective of the data size.

### About Storm

Apache Storm is an Open Source distributed, reliable, fault – tolerant system for real time processing of large volume of data. Spout and Bolt are the two main components in Storm, which work together to process streams of data.

  * Spout: Works on the source of data streams. In the "Truck Events" use case, Spout will read data from Kafka topics.
  * Bolt: Spout passes streams of data to Bolt which processes and passes it to either a data store or another Bolt.

In this tutorial, you will learn the following topics:

  * To configure Storm Bolt.
  * Store Persisting data in HBase and Hive.
  * Verify data in HDFS and HBase.

## Prerequisites

[Tutorial #1](http://hortonworks.com/hadoop-tutorial/simulating-transporting-realtime-events-stream-apache-kafka/) & [Tutorial #2](http://hortonworks.com/hadoop-tutorial/ingesting-processing-real-time-events-apache-storm/) should be completed successfully with a functional Storm and Kafka Bolt reading data from the Kafka Queue.

### Step 1:

**Smoke Test running Services.**

  * **Check NameNode and DataNode Service.**

The _‘jps’_ command will show all java process that are currently running as seen in the screenshot below.  
  
Verify that the _NameNode_ and the _DataNode_ processes are running.  
  
`  
[root@sandbox ~]# jps  
`  
  
![jps](http://hortonassets.s3.amazonaws.com/mda/tut3/jps.png)

  * **Check that HBase is running.**

If **HMaster** and **HRegionServer** are missing in the ‘jps’ command output list, as seen above, we need to restart the HBase services from Ambari.

‘jps’ command output list should now show that the **HMaster** and **HRegionServer** services are running.

![jps : verify hbase services running](http://hortonassets.s3.amazonaws.com/mda/tut3/jps+hbase+running.png)  
  
jps : verify hbase services running

To smoke test HBase, we need to login to _HBase_ user account and start the HBase shell to verify the status of the services.
    
    [root@sandbox ~]# su hbase  
    [hbase@sandbox root]$ hbase shell
    
    hbase(main):001:0> status
    
    hbase(main):002:0> exit
    
    [hbase@sandbox root]$ exit  
    [root@sandbox ~]# 
    

The output is as shown in the screenshot below.

![hbase running](http://hortonassets.s3.amazonaws.com/mda/tut3/hbase+running.png)  
  
hbase running

  * **Smoke test that Hive is Running.**

You can smoke test Hive by either using the command line or by browser.

To smoke test using the command line, login as a Hive user with _su hive_ command and follow the instructions shown below:
    
    [root@sandbox ~]# su hive  
    [hive@sandbox root]$ hive
    
    hive> show databases;
    
    hive> exit;  
    [hive@sandbox root]$ exit
    
    [root@sandbox ~]# 
    

![smoketest hive](http://hortonassets.s3.amazonaws.com/mda/tut3/smoketest+hive.png)  
  
smoketest hive

To smoke test Hive by using a browser, open the url ‘http://localhost:8000/beeswax/’ from any browser on your local machine.  
  
Execute the query "**show databases;**" in the query editor window.  
  
Click on ‘Execute’ button to see an output as shown in the screenshot below.  
  
![Query Editor](http://hortonassets.s3.amazonaws.com/mda/tut3/show+databases.png)  
  
Hive queries can be tested and saved using the Query Editor.

![default database](http://hortonassets.s3.amazonaws.com/mda/tut3/default+databases.png)  
  
default database

### Step 2:

**Persist data in HDFS & HBase.**

  * **Creating HBase tables**

We work with have 2 Hbase tables in this tutorial.  
  
The first table stores all events generated and the second stores the ‘driverId’ and non-normal events count.  
  
As with Hive, we can execute HBase queries via a browser.  
  
Use the following url to open a HBase shell: http://localhost:8000/shell/create?keyName=hbase. 
    
    hbase(main):001:0> create 'truck_events', 'events'  
    hbase(main):002:0> create 'driver_dangerous_events', 'count'  
    hbase(main):003:0> list  
    hbase(main):004:0> 
    

![hbase create tables](http://hortonassets.s3.amazonaws.com/mda/tut3/hbase+create+tables.png)  
  
hbase create tables

Next, we will create Hive tables.

  * **Creating Hive tables**

Open the URL http://localhost:8000/beeswax/ in a browser and copy the below script into the query editor:
    
    create table truck_events_text_partition  
    (driverId string,  
     truckId string,  
     eventTime timestamp,  
     eventType string,  
     longitude double,  
     latitude double)  
    partitioned by (date string)  
    ROW FORMAT DELIMITED  
    FIELDS TERMINATED BY ',';
    

This script creates the Hive table to persist all events generated. This table is partitioned by date.  
  
The table created can be viewed at this URL: http://localhost:8000/beeswax/table/default/truck_events_text_partition

![Hive table](http://hortonassets.s3.amazonaws.com/mda/tut3/Screen+Shot+2014-08-15+at+12.00.48+AM.png)  
  
Hive table

Verify that the table has been properly created by clicking Tables and selecting truck_events_text_partiiton.

![](http://hortonassets.s3.amazonaws.com/mda/storm2/image22.png)  
![truck_events_text_partition](http://hortonassets.s3.amazonaws.com/mda/tut3/truck_events_text_partition+table+created.png)  
truck_events_text_partition

  * **Creating ORC ‘truckevent’ Hive tables**

The Optimized Row Columnar (ORC) file format provides a highly efficient way to store Hive data. It was designed to overcome limitations of the other Hive file formats. Using ORC files improves performance when Hive is reading, writing, and processing data.

Syntax for ORC tables:

CREATE TABLE … STORED AS ORC  
  
ALTER TABLE … [PARTITION partition_spec] SET FILEFORMAT ORC  
  
_Note: This statement only works on partitioned tables. If you apply it to flat tables, it may cause query errors._

SET hive.default.fileformat=Orc

Let us create the ‘truckevent’ table as per the above syntax:  
  
`  
create table truck_events_text_partition_orc  
(driverId string,  
truckId string,  
eventTime timestamp,  
eventType string,  
longitude double,  
latitude double)  
partitioned by (date string)  
ROW FORMAT DELIMITED  
FIELDS TERMINATED BY ','  
stored as orc tblproperties ("orc.compress"="NONE");  
`

The data in ‘truck_events_text_partition_orc’ table can be stored with ZLIB, Snappy, LZO compression options. This can be set by changing `tblproperties ("orc.compress"="NONE")`option in the query above. 

### Step 3:

**Updating Tutorials-master Project**

  * Copy /etc/hbase/conf/hbase-site.xml to src/main/resources/ directory and recompile the Maven project.

[root@sandbox ~]# cd /opt/TruckEvents/Tutorials-master/  
  
[root@sandbox Tutorials-master]# cp /etc/hbase/conf/hbase-site.xml src/main/resources/  
  
[root@sandbox Tutorials-master]# mvn clean package

![update project](http://hortonassets.s3.amazonaws.com/mda/tut3/mvn+clean+package.png)  
  
In case, mvn is not in your path, you can use the command `export PATH=/usr/local/apache-maven-3.2.2/bin:$PATH` to include it in your path.

  * Deactivate & Kill the Storm topology using the Storm UI as shown in the screenshot below:  
  
![storm UI](http://hortonassets.s3.amazonaws.com/mda/tut3/storm+UI.png)  
  
storm UI![deactivate and kill](http://hortonassets.s3.amazonaws.com/mda/tut3/Deactivate+and+Kill.png)  
  
deactivate and kill![Deactivate](http://hortonassets.s3.amazonaws.com/mda/tut3/deactivate.png)  
  
Deactivate![kill](http://hortonassets.s3.amazonaws.com/mda/tut3/kill.png)  
  
kill

  * **Loading new Storm topology.**

Execute the Storm ‘jar’ command to create a new Topology from **Tutorial# 3** after the code has been compiled.
    
    [root@sandbox Tutorials-master]# storm jar target/Tutorial-1.0-SNAPSHOT.jar com.hortonworks.tutorials.tutorial3.TruckEventProcessingTopology
    

![Topology Summary](http://hortonassets.s3.amazonaws.com/mda/tut3/Topology+Summary.png)  
  
Topology Summary

### Step 4:

**Verify Data in HDFS and HBase.**

  * Start the **‘TruckEventsProducer’** Kafka Producer and verify that the data has been persisted by using the Storm Topology view.

[root@sandbox Tutorials-master]# java -cp target/Tutorial-1.0-SNAPSHOT.jar com.hortonworks.tutorials.tutorial1.TruckEventsProducer localhost:9092 localhost:2181

![Verify Storm UI](http://hortonassets.s3.amazonaws.com/mda/tut3/Verify+data+in+Storm+UI.png)  
  
Verify Storm UI

  * Verify that the data is in HBase by executing the following commands in HBase shell:

hbase(main):001:0> list

hbase(main):002:0> count ‘truck_events’  
  
366 row(s) in 0.3900 seconds

=> 366  
  
hbase(main):003:0> count ‘driver_dangerous_events’  
  
3 row(s) in 0.0130 seconds

=> 3  
  
hbase(main):004:0> exit

The ‘driver_dangerous_events’ table is updated upon every violation.

![Verify data in HBase](http://hortonassets.s3.amazonaws.com/mda/tut3/Verify+data+in+HBase.png)  
  
Verify data in HBase

  * Verify that the data is in HDFS by browsing to this URL: http://localhost:8000/filebrowser/view/truck-events-v4/staging  
  
We should see the files been injested in HDFS now.  
  
With the default settings for HDFS, users might see the data written to HDFS once in every 5 minutes.  
  
![Verify data in HDFS](http://hortonassets.s3.amazonaws.com/mda/tut3/Verify+Data+in+HDFS.png)  
  
Verify data in HDFS

This completes the tutorial #3. 

In the next tutorial we will integrate with a UI and visualize the events in real time. 

### Code Description

1.**BaseTruckEventTopology.java**
    
    topologyConfig.load(ClassLoader.getSystemResourceAsStream(configFileLocation));
    

This is the base class, where the topology configuration is initialized from the /resource/truck_event_topology.properties files.

2.**FileTimeRotationPolicy.java**

This implements the file rotation policy after a certain duration.

“`  
  
public FileTimeRotationPolicy(float count, Units units) {  
  
this.maxMilliSeconds = (long) (count * units.getMilliSeconds());  
  
}
    
    @Override  
    public boolean mark(Tuple tuple, long offset) {  
        // The offsett is not used here as we are rotating based on time  
        long diff = (new Date()).getTime() - this.lastCheckpoint;  
        return diff >= this.maxMilliSeconds;  
    }
    

3.**LogTruckEventsBolt.java**

LogTruckEvent Spout logs the Kafka messages received from the Kafka Spout to the log files under /var/log/storm/worker-*.log
    
    public void execute(Tuple tuple)  
     {
     LOG.info(tuple.getStringByField(TruckScheme.FIELD_DRIVER_ID) + "," +  
     tuple.getStringByField(TruckScheme.FIELD_TRUCK_ID) + "," +  
     tuple.getValueByField(TruckScheme.FIELD_EVENT_TIME) + "," +  
     tuple.getStringByField(TruckScheme.FIELD_EVENT_TYPE) + "," +  
     tuple.getStringByField(TruckScheme.FIELD_LATITUDE) + "," +  
     tuple.getStringByField(TruckScheme.FIELD_LONGITUDE));  
     }
    

4.**TruckScheme.java**

This is the deserializer provided to the Kafka Spout to deserialize Kafka’s byte message streams to Values objects.
    
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
    

5.**HiveTablePartitionAction.java**

This creates Hive partitions based on timestamp and loads the data by executing the Hive DDL statements.
    
    public void loadData(String path, String datePartitionName, String hourPartitionName )  
        {
    
            String partitionValue = datePartitionName + "-" + hourPartitionName;
    
            LOG.info("About to add file["+ path + "] to a partitions["+partitionValue + "]");
    
            StringBuilder ddl = new StringBuilder();  
            ddl.append(" load data inpath ")  
                .append(" '").append(path).append("' ")  
                .append(" into table ")  
                .append(tableName)  
                .append(" partition ").append(" (date='").append(partitionValue).append("')");
    
            startSessionState(sourceMetastoreUrl);
    

The data is stored in the partitioned ORC tables using the following method.
    
    String ddlORC = "INSERT OVERWRITE TABLE " + tableName + "_orc SELECT * FROM " +tableName;
    
    
    
    
        try {  
            execHiveDDL("use " + databaseName);  
            execHiveDDL(ddl.toString());  
            execHiveDDL(ddlORC.toString());  
        } catch (Exception e) {  
            String errorMessage = "Error exexcuting query["+ddl.toString() + "]";  
            LOG.error(errorMessage, e);  
            throw new RuntimeException(errorMessage, e);  
        }
    } 
    

6.**TruckEventProcessingTopology.java**

This creates a connection to HBase tables and access data within the prepare() function.
    
    public void prepare(Map stormConf, TopologyContext context,  
     OutputCollector collector)  
     {
     ...  
     this.connection = HConnectionManager.createConnection(constructConfiguration());  
     this.eventsCountTable = connection.getTable(EVENTS_COUNT_TABLE_NAME);  
    
     this.eventsTable = connection.getTable(EVENTS_TABLE_NAME);  
     } 
    
    
    
    ...  
    }
    
    
    
    Data to be stored is prepared in the constructRow() function using put.add().
    
    
    
    private Put constructRow(String columnFamily, String driverId, String truckId,  
     Timestamp eventTime, String eventType, String latitude, String longitude)  
     {
    
        String rowKey = consructKey(driverId, truckId, eventTime);  
        ...  
        put.add(CF_EVENTS_TABLE, COL_DRIVER_ID, Bytes.toBytes(driverId));  
        put.add(CF_EVENTS_TABLE, COL_TRUCK_ID, Bytes.toBytes(truckId));
    
        ...  
    }
    

This executes the getInfractionCountForDriver() to get the count of events for a driver using driverID and stores the data in HBase with constructRow() function.
    
    public void execute(Tuple tuple)  
     {
    
        ...  
        long incidentTotalCount = getInfractionCountForDriver(driverId);
    
        ...
    
            Put put = constructRow(EVENTS_TABLE_NAME, driverId, truckId, eventTime, eventType,  
                                latitude, longitude);  
            this.eventsTable.put(put);
    
        ...  
                incidentTotalCount = this.eventsCountTable.incrementColumnValue(Bytes.toBytes(driverId), CF_EVENTS_COUNT_TABLE,  
                                                                                               ...  
    }
    

7.**TruckEventProcessingTopology.java**

HDFS and HBase Bolt configurations created within configureHDFSBolt() and configureHBaseBolt() respectively. 
    
    public void configureHDFSBolt(TopologyBuilder builder)  
    {
    
        HdfsBolt hdfsBolt = new HdfsBolt()  
                         .withFsUrl(fsUrl)  
                 .withFileNameFormat(fileNameFormat)  
                 .withRecordFormat(format)  
                 .withRotationPolicy(rotationPolicy)  
                 .withSyncPolicy(syncPolicy)  
                 .addRotationAction(hivePartitionAction);
    
    }  
    public void configureHBaseBolt(TopologyBuilder builder)  
    {
        TruckHBaseBolt hbaseBolt = new TruckHBaseBolt(topologyConfig);  
        builder.setBolt(HBASE_BOLT_ID, hbaseBolt, 2).shuffleGrouping(KAFKA_SPOUT_ID);  
    }
    

  

