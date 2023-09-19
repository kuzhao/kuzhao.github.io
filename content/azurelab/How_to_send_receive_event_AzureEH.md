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

###  Explanation on the code
I composed both py files based on the example [azure-sdk-for-python](https://github.com/Azure/azure-sdk-for-python/tree/master/sdk/eventhub/azure-eventhubs/examples).

You ought to pass both connStr obtained in above section, along with the name of EventHub to use, as the 1st and 2nd arg respectively.  
For send.py specifically, you can pass a timestamp in the 3rd arg, for EH offset so that the script reads event since that particular datetime. Note that the format should be **DDMMYYYY HH:MM:SS** in **UTC**.  
Example:

* send.py "\<connStr\>" "\<EHname\>"
* recv.py "\<connStr\>" "\<EHname\>" "12/11/2018 09:15:32"

### Code flow & notes
* Init EHClient via `from_connection_string` class method of `EventHubClient` class  
* Spawn EH sender/receiver via `add_receiver/sender` method of the client, with EH connection params  
* `client.run()` to start client, establishing EH connection  
* Perform send/receive actions. I coded in blocking mode rather than async for simplicity as we only need PoC on functionality here  
* During the send/receive process, event details will get printed to terminal. You can see for each event the "offset" and "seqNum" values of event metadata, along with event body at the end  
* Note that per Azure recommendation [here](https://docs.microsoft.com/en-in/azure/event-hubs/event-hubs-features#consumer-groups), there should be only 1 consumer on one partition for a clientApp. Meanwhile, all partitions should be listened on in order to capture all published events. So there are **2 consumers** in recv.py, and statistics is printed separately at the end of each consumer reception  
* In practical scenario, a more dynamic mechanism would be necessary to handle partition listening/reception  

### Sample logs
Check [here](https://github.com/kuzhao/azurePaas/tree/master/eventhub/send_recv_offset/terminalLog) for detailed send/recv logs.

### EventHub metrics
You should be able to see up and down in metric chart, Overview of EH namespace blade through AzurePortal. Refer to snapshot [here](https://github.com/kuzhao/azurePaas/tree/master/eventhub/send_recv_offset/EHmetrics.png) for an example.

