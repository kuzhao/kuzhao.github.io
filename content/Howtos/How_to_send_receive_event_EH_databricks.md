---
title: "How to send and receive events via Azure EventHub"
date: 2019-09-15T09:47:11+08:00
draft: false
---
Sample code available at [kuzhao/azurePaas](https://github.com/kuzhao/azurePaas/tree/master/eventhub/send_recv_offset)

### Create an EH namespace and instance  
You should already have an EH resource ready for use. For how to create one, refer to [Quickstart via Azure portal](https://docs.microsoft.com/en-in/azure/event-hubs/event-hubs-create#create-an-event-hubs-namespace)  
The next step would be to copy connStr and have it ready by your hand, as we will pass it as py mandatory argument of sample codes.  
Refer to snapshot [here](https://github.com/kuzhao/azurePaas/tree/master/eventhub/send_recv_offset/getEHconnStr.png) on where to get it.

###  Basic use with Python
I composed both py files based on the example [azure-sdk-for-python](https://github.com/Azure/azure-sdk-for-python/tree/master/sdk/eventhub/azure-eventhubs/examples).

You ought to pass both connStr obtained in above section, along with the name of EventHub to use, as the 1st and 2nd arg respectively.  
For send.py specifically, you can pass a timestamp in the 3rd arg, for EH offset so that the script reads event since that particular datetime. Note that the format should be **DDMMYYYY HH:MM:SS** in **UTC**.  
Example:

* send.py "\<connStr\>" "\<EHname\>"
* recv.py "\<connStr\>" "\<EHname\>" "12/11/2018 09:15:32"

#### Code flow & notes
* Init EHClient via `from_connection_string` class method of `EventHubClient` class  
* Spawn EH sender/receiver via `add_receiver/sender` method of the client, with EH connection params  
* `client.run()` to start client, establishing EH connection  
* Perform send/receive actions. I coded in blocking mode rather than async for simplicity as we only need PoC on functionality here  
* During the send/receive process, event details will get printed to terminal. You can see for each event the "offset" and "seqNum" values of event metadata, along with event body at the end  
* Note that per Azure recommendation [here](https://docs.microsoft.com/en-in/azure/event-hubs/event-hubs-features#consumer-groups), there should be only 1 consumer on one partition for a clientApp. Meanwhile, all partitions should be listened on in order to capture all published events. So there are **2 consumers** in recv.py, and statistics is printed separately at the end of each consumer reception  
* In practical scenario, a more dynamic mechanism would be necessary to handle partition listening/reception  

#### Sample logs
Check [here](https://github.com/kuzhao/azurePaas/tree/master/eventhub/send_recv_offset/terminalLog) for detailed send/recv logs.

### Use EH in Databricks
I composed the two scala files based on Microsoft Azure HDInsight doc ([send-tweets-to-the-event-hub](https://docs.microsoft.com/en-us/azure/hdinsight/spark/apache-spark-eventhub-streaming)) and EventHub java SDK quickstart ([event-hubs-java-get-started](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-java-get-started-send)).  
Per the nature of Databrick UI, one needs to copy and paste the codes into notebook for execution. After finishing, do remember to replace EventHub connection string and Name \(in var `eventHubName` and `eventHubNSConnStr`\) with yours.

#### Code flow & notes  
ehPub.scala:  
* Build `connStr` with ConnectionStringBuilder, and create a new scheduled thread pool of 4 threads into `pool`  
* Init `ehClient` with `connStr` and `pool`   
* Start sending 100 events with content of "Msg #" and current timestamp  
ehRecv.scala:  
* Build connStr the same way with ehPub  
* Set desired EventPosition timestamp. It should be the moment when the first event to be received gets enqueued.  
* Pack both connStr and `ehpos`, the EventPosition, into `customEventhubParameters` to be used to create Spark stream next  
* Create incomingStream and prepare to receive events  
* After executing events received will get printed in batch manner, each one contained in a separate table

#### Sample logs  
Check Output #1 and #2 [here](https://github.com/kuzhao/azurePaas/tree/master/databricks/scala-eventhub) for detailed receiver logs.

### EventHub metrics
You should be able to see up and down in metric chart, Overview of EH namespace blade through Azure Portal. Refer to snapshot [here](https://github.com/kuzhao/azurePaas/tree/master/eventhub/send_recv_offset/EHmetrics.png) for an example.

