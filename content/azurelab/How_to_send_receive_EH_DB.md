---
title: "How to send and receive events with EventHub in Azure Databricks"
date: 2019-09-24T08:42:11+08:00
draft: false
---
Sample code available at [kuzhao/azurePaas](https://github.com/kuzhao/azurePaas/tree/master/databricks/scala-eventhub)

### Create an EH namespace and instance  
You should already have an EH resource ready for use. For how to create one, refer to [Quickstart via Azure portal](https://docs.microsoft.com/en-in/azure/event-hubs/event-hubs-create#create-an-event-hubs-namespace)  
The next step would be to copy connStr and have it ready by your hand, as we will pass it as py mandatory argument of sample codes.  
Refer to snapshot [here](https://github.com/kuzhao/azurePaas/tree/master/eventhub/send_recv_offset/getEHconnStr.png) on where to get connStr.

###  Explanation on the code  
In the repo, I composed the two scala files based on Microsoft Azure HDInsight doc ([send-tweets-to-the-event-hub](https://docs.microsoft.com/en-us/azure/hdinsight/spark/apache-spark-eventhub-streaming)) and EventHub java SDK quickstart ([event-hubs-java-get-started](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-java-get-started-send)).  
Per the nature of Databrick UI, one needs to copy and paste the codes into notebook for execution. After finishing, do remember to replace EventHub connection string and Name \(in var `eventHubName` and `eventHubNSConnStr`\) with yours.

### Code flow & notes  
#### ehPub.scala  
* Build `connStr` with ConnectionStringBuilder, and create a new scheduled thread pool of 4 threads into `pool`  
* Init `ehClient` with `connStr` and `pool`   
* Start sending 100 events with content of "Msg #" and current timestamp  
#### ehRecv.scala  
* Build connStr the same way with ehPub  
* Set desired EventPosition timestamp. It should be the moment when the first event to be received gets enqueued.  
* Pack both connStr and `ehpos`, the EventPosition, into `customEventhubParameters` to be used to create Spark stream next  
* Create incomingStream and prepare to receive events  
* After executing events received will get printed in batch manner, each one contained in a separate table

### Sample logs  
Check Output #1 and #2 [here](https://github.com/kuzhao/azurePaas/tree/master/databricks/scala-eventhub) for detailed receiver logs.

### EventHub metrics  
You should be able to see up and down on metrics charts in Overview blade of the EH resource in Azure Portal. Refer to snapshot [here](https://github.com/kuzhao/azurePaas/tree/master/eventhub/send_recv_offset/EHmetrics.png) for an example.
