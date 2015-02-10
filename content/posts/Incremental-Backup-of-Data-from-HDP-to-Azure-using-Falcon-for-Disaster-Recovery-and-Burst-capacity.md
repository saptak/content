+++
date = "2015-02-08T23:00:24-08:00"
draft = false
title = "Incremental Backup of Data from HDP to Azure using Falcon for Disaster Recovery and Burst capacity"

+++

## Introduction

Apache Falcon simplifies the configuration of data motion with: replication; lifecycle management; lineage and traceability. This provides data governance consistency across Hadoop components.

## Scenario

In this tutorial we will walk through a scenario where email data gets processed on multiple HDP 2.2 clusters around the country then gets backed up hourly on a cloud hosted cluster . In our example:

  * This cluster is hosted on Windows Azure.
  * Data arrives from all the West Coast production servers. The input data feeds are often late for up to 4 hrs.

The goal is to clean the raw data to remove sensitive information like credit card numbers and make it available to our marketing data science team for customer churn analysis.

To simulate this scenario, we have a pig script grabbing the freely available Enron emails from the internet and feeding it into the pipeline.

![](http://hortonassets.s3.amazonaws.com/tutorial/falcon/images/arch.png)  


## Prerequisite

  * A cluster with Apache Hadoop 2.2 configured
  * A cluster with Apache Falcon configured

The easiest way to meet the above prerequisites is to download the [HDP Sandbox](http://hortonworks.com/downloads)

After downloading the environment, confirm that Apache Falcon is running. Below are the steps to validate that:

  1. if Ambari is not configured on your Sandbox, go `http://127.0.0.1:8000/about/` and enable Ambari.

![](http://hortonassets.s3.amazonaws.com/falcon2/image01.png)

  1. Once Ambari is enabled, navigate to Ambari at `http://127.0.0.1:8080`, login with username and password of`admin` and `admin` respectively. Then check if Falcon is running.

![](http://hortonassets.s3.amazonaws.com/falcon2/image04.png)

  1. If Falcon is not running, start Falcon:

![](http://hortonassets.s3.amazonaws.com/falcon2/image07.png)

## Steps for the Scenario

  1. Create cluster specification XML file
  2. Create feed (aka dataset) specification XML file
    * Reference cluster specification
  3. Create the process specification XML file
    * Reference cluster specification – defines where the process runs
    * Reference feed specification – defines the datasets that the process manipulates

We have already created the necessary xml files. In this step we are going to download the specifications and use them to define the topology and submit the storm job.

### Staging the component of the App on HDFS

In this step we will stage the pig script and the necessary folder structure for inbound and outbound feeds on the HDFS:

First download this [zip file](http://hortonassets.s3.amazonaws.com/tutorial/falcon/falcon.zip) called [`falcon.zip`](http://hortonassets.s3.amazonaws.com/tutorial/falcon/falcon.zip) to your local host machine.

Navigate using your browser to the Hue – File Browser interface at [http://127.0.0.1:8000/filebrowser/](http://127.0.0.1:8000/filebrowser/) to explore the HDFS.

Navigate to `/user/ambari-qa` folder like below:  
  
![](http://hortonassets.s3.amazonaws.com/tutorial/falcon/images/file-browser.png)

Now we will upload the zip file we just downloaded:

![](http://hortonassets.s3.amazonaws.com/tutorial/falcon/images/uploadzip.png)  


This should also unzip the zip file and create a folder structure with a folder called `falcon`.

### Setting up the destination storage on Microsoft Azure

Login to the Windows Azure portal at [http://manage.windowsazure.com](http://manage.windowsazure.com/)

![](http://hortonassets.s3.amazonaws.com/falcon2/image13.png)  


Create a storage account

![](http://hortonassets.s3.amazonaws.com/falcon2/image16.png)  


Wait for the storage account to be provisioned

![](http://hortonassets.s3.amazonaws.com/falcon2/image19.png)  


Copy the access key and the account name in a text document. We will use the access key and the account name in later steps

![](http://hortonassets.s3.amazonaws.com/falcon2/image22.png)  


The other information you will want to note down is the blob endpoint of the storage account we just created

![](http://hortonassets.s3.amazonaws.com/falcon2/image25.png)  


Click on the `Containers` tab and create a new container called `myfirstcontainer`.

![](http://hortonassets.s3.amazonaws.com/falcon2/image26.png)  


### Configuring access to Azure Blob store from Hadoop

Login to Ambari – http://127.0.0.1:8080 with the credentials `admin` and `admin`.

![](http://hortonassets.s3.amazonaws.com/falcon2/image27.png)  


Then click on HDFS from the bar on the left and then select the `Configs` tab.

![](http://hortonassets.s3.amazonaws.com/falcon2/image28.png)  


Scroll down to the bottom of the page to the `Custom hdfs-site` section and click on `Add property...`

![](http://hortonassets.s3.amazonaws.com/falcon2/image31.png)  


In the `Add Property` dialog, the key name will start with `fs.azure.account.key.` followed by your blob endpoint that you noted down in a previous step. The value will be the Azure storage key that you noted down in a previous step. Once you have filled in the values click the `Add` button:

![](http://hortonassets.s3.amazonaws.com/falcon2/image34.png)  


Once you are back out of the new key dialog you will have to `Save` it by clicking on the green `Save` button:

![](http://hortonassets.s3.amazonaws.com/falcon2/image31.png)  


Then restart all the service by clicking on the orange `Restart` button:

![](http://hortonassets.s3.amazonaws.com/falcon2/image36.png)  


Wait for all the restart to complete

![](http://hortonassets.s3.amazonaws.com/falcon2/image38.png)  


Now let’s test if we can access our container on the Azure Blob Store.

SSH in to the VM:

`ssh root@127.0.0.1 -p 2222;`

The password is `hadoop`

`hdfs dfs -ls -R wasb://myfirstcontainer@saptak.blob.core.windows.net/`

Issue the command from our cluster on the SSH’d terminal

![](http://hortonassets.s3.amazonaws.com/falcon2/image40.png)  


### Staging the specifications

From the SSH session, first we will change our user to `ambari-qa`. Type:

`su ambari-qa`

Go to the users home directory:

`cd ~`

Download the topology, feed and process definitions:

`wget http://hortonassets.s3.amazonaws.com/tutorial/falcon/falconDemo.zip`

![](http://hortonassets.s3.amazonaws.com/falcon2/image10.png)  


Unzip the file:

`unzip ./falconDemo.zip`

Change Directory to the folder created:

`cd falconChurnDemo/`

Now let’s modify the `cleansedEmailFeed.xml` to point the backup cluster to our Azure Blob Store container.

Use `vi` to edit the file:

![](http://hortonassets.s3.amazonaws.com/falcon2/image42.png)  


Modify the value of `location` element of the `backupCluster`

![](http://hortonassets.s3.amazonaws.com/falcon2/image44.png)  


to look like this:

![](http://hortonassets.s3.amazonaws.com/falcon2/image46.png)  


Then save it and quit vi.

### Submit the entities to the cluster:

#### Cluster Specification

Cluster specification is one per cluster.

See below for a sample cluster specification file.

![](http://hortonassets.s3.amazonaws.com/tutorial/falcon/images/cluster-spec.png)  


Back to our scenario, lets submit the ‘oregon cluster’ entity to Falcon. This signifies the primary Hadoop cluster located in the Oregon data center.

`falcon entity -type cluster -submit -file oregonCluster.xml`

Then lets submit the ‘virginia cluster’ entity to Falcon. This signifies the backup Hadoop cluster located in the Virginia data center

`falcon entity -type cluster -submit -file virginiaCluster.xml`

If you view the XML file you will see how the cluster location and purpose has been captured in the XML file.

#### Feed Specification

A feed (a.k.a dataset) signifies a location of data and its associated replication policy and late arrival cut-off time.

See below for a sample feed (a.k.a dataset) specification file.

![](http://hortonassets.s3.amazonaws.com/tutorial/falcon/images/feed-spec.png)  


Back to our scenario, let’s submit the source of the raw email feed. This feed signifies the raw emails that are being downloaded into the Hadoop cluster. These emails will be used by the email cleansing process.

`falcon entity -type feed -submit -file rawEmailFeed.xml`

Now let’s define the feed entity which will handle the end of the pipeline to store the cleansed email. This feed signifies the emails produced by the cleanse email process. It also takes care of replicating the cleansed email dataset to the backup cluster (virginia cluster)

`falcon entity -type feed -submit -file cleansedEmailFeed.xml`

#### Process

A process defines configuration for a workflow. A workflow is a directed acyclic graph(DAG) which defines the job for the workflow engine. A process definition defines the configurations required to run the workflow job. For example, process defines the frequency at which the workflow should run, the clusters on which the workflow should run, the inputs and outputs for the workflow, how the workflow failures should be handled, how the late inputs should be handled and so on.

Here is an example of what a process specification looks like:

![](http://hortonassets.s3.amazonaws.com/tutorial/falcon/images/process-spec.png)  


Back to our scenario, let’s submit the ingest and the cleanse process respectively:

The ingest process is responsible for calling the Oozie workflow that downloads the raw emails from the web into the primary Hadoop cluster under the location specified in the rawEmailFeed.xml It also takes care of handling late data arrivals

`falcon entity -type process -submit -file emailIngestProcess.xml`

The cleanse process is responsible for calling the pig script that cleans the raw emails and produces the clean emails that are then replicated to the backup Hadoop cluster

`falcon entity -type process -submit -file cleanseEmailProcess.xml`

### Schedule the Falcon entities

So, all that is left now is to schedule the feeds and processes to get it going.

#### Ingest the feed

`falcon entity -type feed -schedule -name rawEmailFeed`

`falcon entity -type process -schedule -name rawEmailIngestProcess`

#### Cleanse the emails

`falcon entity -type feed -schedule -name cleansedEmailFeed`

`falcon entity -type process -schedule -name cleanseEmailProcess`

### Processing

In a few seconds you should notice that that Falcon has started ingesting files from the internet and dumping them to new folders like below on HDFS:

![](http://hortonassets.s3.amazonaws.com/tutorial/falcon/images/input.png)  


In a couple of minutes you should notice a new folder called processed under which the files processed through the data pipeline are being emitted:

![](http://hortonassets.s3.amazonaws.com/tutorial/falcon/images/output.png)  


We just created an end-to-end data pipeline to process data. The power of the Apache Falcon framework is its flexibility to work with pretty much any open source or proprietary data processing products out there.

