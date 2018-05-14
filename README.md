### Update May-2018
You can now have multi-master support with CosmosDB!  Please see: https://docs.microsoft.com/en-us/azure/cosmos-db/multi-region-writers.  The below article still applies, multi-master will just make your life easier.

# Azure-Dual-Region-Deployment-Approach
This shows my typically deployment utilizes two regions in Azure.  If you are deploying your code to Azure and need to ensure uptime even in the event of a regional issue, then you need to consider deploying your system to two regions. You should review which Azure Regions are paired for best performance.  The below architecture is shown for web applications, but can be adapted to other similar architectures.  I prefer an active-active approach versus a failover / disaster recovery.  I do not want to spend the time creating a DR plan and I almost never test them.  With active-active, ideally you can split your application workload across two regions, so your cost remains close to a single region deployment.  

## Overview
![alt tag](https://raw.githubusercontent.com/AdamPaternostro/Azure-Dual-Region-Deployment-Approach/master/Azure-Dual-Region-Diagram.png)

### In the above diagram:
- Traffic Manager: Global DNS that will route my traffic two either site.  This will handle when a region experiences issues.
- Azure App Services: Two services each in a different region.  
  - Each one gets its own DNS name.  
- Web Apps: This shows 5 websites being hosted on this App Service plan (you might just have 1 or 2 - frontend and REST tier).  
  - Each of the websites gets their own DNS, the above would have 10 in total.
- DocumentDB: A single DocumentDB that is globally replicated.
- Application Insights: An Application Insight per website. Only one per site since you want to see the traffic for each website regardless of the number of regions the website is deployed.
- Azure Storage: There is a local storage account in each region (local queues are created)

### How things work
#### North Central
  - Web Traffic arrives at the Web App 
  - The Web App reads and writes to local DocumentDB
  - The Web App reads and writes to local Azure Storage
  - Any Blob that is written to locally needs its Container and Filename (complete "path") written to MyAppSyncQueue (e.g. { "container" : "myfiles", "filename" : "/path/myfile.txt"})
  - Any Azure Table that is written to locally needs its Table Name, Partition Key and Row Key written to MyAppSyncQueue. (e.g. { "table" : "mytable", "partitionkey" : "mypartitionkey", "rowkey" : "myrowkey" } 
  - A web Job will then need to read the queue MyAppSyncQueue
    - This will then save to a local queue named MyAppSyncTo02 (and possibly MyAppSyncTo03, MyAppSyncTo04, etc...)
  - A Web Job will then read queue MyAppSyncTo02 and write the blob or table data to MyAppStorage02 (South Central)  
  
#### South Central
  - Web Traffic arrives at the Web App
  - The Web App reads from local DocumentDB
  - The Web App writes to **remote** North Central DocumentDB
  - The Web App reads and writes to local Azure Storage
  - Any Blob that is written to locally needs its Container and Filename (complete "path") written to MyAppSyncQueue (e.g. { "container" : "myfiles", "filename" : "/path/myfile.txt"})
  - Any Azure Table that is written to locally needs its Table Name, Partition Key and Row Key written to MyAppSyncQueue. (e.g. { "table" : "mytable", "partitionkey" : "mypartitionkey", "rowkey" : "myrowkey" }   - A web Job will then need to read the queue MyAppSyncQueue
    - This will then save to a local queue named MyAppSyncTo01 (and possibly MyAppSyncTo03, MyAppSyncTo04, etc...)
  - A Web Job will then read queue MyAppSyncTo01 and write the blob or table data to MyAppStorage01 (North Central)

### Deployment
  - Your deployment should be **exactly** the same to each region.  
  - Here are my application variables
    - Environment [This tells the application "where they are deployed"]
      - In North Central this is set to: "01"
      - In South Central this is set to: "02"
     - DocumentDBPreferredLocations [This tells the application "where you should read for DocDB" - you configure the primary write in the Azure Portal]
        - In North Central this is set to: "NorthCentral, SouthCentral" (I split the list and add to my DocumentDB: PreferredLocations)
        - In South Central this is set to: "SouthCentral, NorthCentral" (I split the list and add to my DocumentDB: PreferredLocations)
      - See: https://docs.microsoft.com/en-us/azure/documentdb/documentdb-regional-failovers
     - SyncRegions [This tells the application about all the regions you have deployed]
        - In All Regions: "01,02" - This tells me who I need to sync with (I will remove myself from the list).  So, region 01 will sync with 02 (and 03 if we had one).
     - You will have each storage account connection in all your deployments since the web job needs to sync accross regions.  By using 01, 02, etc... your code can be smart and know who it is and who is remote.
      
        
### Naming
- App Service Plan: I name these MyAppAppServicePlan01 and MyAppAppServicePlan02
- Web Apps: I name these MyAppWeb01, MyAppWeb02
- Web Jobs: I name these MyAppWebJob01, MyAppWebJob02
- Storage: I name these MyAppStorage01, MyAppStorage02
- Storage Queues: I name these MyAppSyncQueue, MyAppSyncQueue
- Storage Queues: MyAppSyncTo02 (located in MyAppStorage01) and MyAppSyncTo02 (located in MyAppStorage01)
- DocumentDB: This is global so I name MyAppDocumentDB (no number on the end)
- Traffic Manger: This global so I name MyAppTrafficManager (no number on the end)
- Resource Groups: I will create 3 resource groups in the above scenerio
  - MyAppCommon: For DocumentDB, Traffic Manager and Application Insights
  - MyApp01: For everything that ends in 01.  All the resources in this group would be in the same region.
  - MyApp02: For everything that ends in 02.  All the resources in this group would be in the same region.
- **Note**: You can place the word "Dev", "QA", "Prod" in each name to make things easier.  I personally create all my Azure resources upfront for Dev and QA and make sure my naming makes total sense and is perfectly clear to everyone.

### Selecting Azure Services
- The services I select in Azure have to meet two conditions:
  - One they must be cost effective.
  - Two they must support geo-replication.  
  - This makes my architecture easier and less I have to configure / maintain.

### Health checks
- You should have a health check URL in your Web App that reads/writes a test file to Azure Storage and reads/write to DocumentDB
- Traffic Manager will be calling the health check.  Traffic manager needs this to know when to stop routing traffic to a region.  You can also have a healthcheck URL that returns a bad health status.  This will allow you to change the URL in traffic manager to simulate a failure during development.  
- Application Insights will be calling the healthcheck.  Applications Insights should be part of your overall monitoring, so IT is alerted along with the developers.

### Failures: There are couple scenarios that can occur
- Traffic Manager might get a failure of a healthcheck.  If this happens then Traffic Manager stops sending traffic to that region.  Your web apps should auto-scale to handle the traffic in the region.
- If Azure Web Apps stop working (Azure issue), then Traffic Manager will route all traffic to the other region due to your healthcheck.  But your DocumentDB might not be down in the North Central region.  So, DocumentDB does not failover, but your traffic has been re-routed.  So, do not think a failure means every service in the region has an issue.  It just might be one tier and the other up region can still be syncing files and accessing DocumentDB.

### Recovery
- Once the incident if over
  - Your storage should sync and you should be in a consistent state.  The queue / sync job should handle this.
  - You will need to address any DocumentDB issues per this [link](https://docs.microsoft.com/en-us/azure/documentdb/documentdb-regional-failovers).

### Terms
- Region: A group of Azure data centers 
- Traffic Manager: A global DNS system that routes traffic

