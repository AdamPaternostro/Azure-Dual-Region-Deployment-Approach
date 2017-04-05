# Azure-Dual-Region-Deployment-Approach
This shows my typically deployment utilizes two regions in Azure.

## Overview
![alt tag](https://raw.githubusercontent.com/AdamPaternostro/Azure-Dual-Region-Deployment-Approach/master/Azure-Dual-Region-Diagram.png)

If you are deploying your code to Azure and need to ensure uptime even in the event of a regional issue, then you need to consider deploying your system to two regions. 

### In the above diagram:
- There is Traffic Manager that is global 
- There are two Azure App Services that are deployed to two different regions.  Each one gets its own DNS name.  
- There are five websites being hosted on this App Service plan (typically you would have just one for large apps).  Each of the websites gets their own DNS, so I would have 10 in total.
- There is a DocumentDB that is global which has global replication with automatic failover.
- There is a local storage account in each region (local queues are created)

### How things work
- North Central
  - Web Traffic arrives at the Web App.  
  - The Web App reads and writes to local DocumentDB
  - The Web App reads and writes to local Azure Storage
  - Any Blob that is written to locally needs its Container and Filename (complete "path") written to MyAppSyncQueue (e.g. { "container" : "myfiles", "filename" : "/path/myfile.txt"})
  - Any Azure Table that is written to locally needs its Table Name, Partition Key and Row Key written to MyAppSyncQueue. (e.g. { "table" : "mytable", "partitionkey" : "mypartitionkey", "rowkey" : "myrowkey" } 
  - A web Job will then need to read the queue MyAppSyncQueue
    - This will then save to a local queue named MyAppSyncTo02 (and possibly MyAppSyncTo03, MyAppSyncTo04, etc...)
  - A Web Job will then read queue MyAppSyncTo02 and write the blob or table data to MyAppStorage02 (South Central)
  
- South Central
  - Web Traffic arrives at the Web App.  
  - The Web App reads from local DocumentDB
  - The Web App write to remote North Central DocumentDB
  - The Web App reads and writes to local Azure Storage
  - Any Blob that is written to locally needs its Container and Filename (complete "path") written to MyAppSyncQueue (e.g. { "container" : "myfiles", "filename" : "/path/myfile.txt"})
  - Any Azure Table that is written to locally needs its Table Name, Partition Key and Row Key written to MyAppSyncQueue. (e.g. { "table" : "mytable", "partitionkey" : "mypartitionkey", "rowkey" : "myrowkey" } 
  - A web Job will then need to read the queue MyAppSyncQueue
    - This will then save to a local queue named MyAppSyncTo01 (and possibly MyAppSyncTo03, MyAppSyncTo04, etc...)
  - A Web Job will then read queue MyAppSyncTo01 and write the blob or table data to MyAppStorage01 (North Central)


### Naming
- App Service Plan: I name these MyAppAppServicePlan01 and MyAppAppServicePlan02
- Web Apps: I name these MyAppWeb01, MyAppWeb02
- Web Jobs: I name these MyAppWebJob01, MyAppWebJob02
- Storage: I name these MyAppStorage01, MyAppStorage02
- Storage Queues: I name these MyAppSyncQueue, MyAppSyncQueue
- Storage Queues: MyAppSyncTo02 (located in MyAppStorage01) and MyAppSyncTo02 (located in MyAppStorage01)
- DocumentDB: This is global so I name MyAppDocumentDB (no number on the end)
- Traffic Maanger: This global so I name MyAppTrafficManager (no number on the end)
- Resouce Groups: I will create 3 resource groups in the above scenerio
  - MyAppCommon: For DocumentDB, Traffic Manager and Application Insights
  - MyApp01: For everything that ends in 01.  All the resources in this group would be in the same region.
  - MyApp02: For everything that ends in 02.  All the resources in this group would be in the same region.

### Selecting Azure Services
- The services I select in Azure have to meet two conditions.  One they must be cost effective and two they must support geo-replication.  This makes my architecture easier and less I have to configure and maintain.

### Heathchecks
- You should have a healthcheck URL in your Web App that reads/writes a test file to Azure Storage and reads/write to DocumentDB

### Failures: There are couple scenerios that can occur
- Traffic Manager might get a failure of a healthcheck.  If this happens then Traffic Manager stops sending traffic to that region.
- If your Azure Web Apps fails, then Traffic Manager will route all traffic to the other region due to your healthcheck.  But your DocumentDB might not be down in the North Central region.  So, DocumentDB does not failover, but traffic has.

### Recovery
- Once the incident if over, your DocumentDB will correct itself. You storage shoud sync and you should be okay.


### Terms
- Region - a group of Azure data centers 
- Traffic Manager - a global DNS system that routes traffic
