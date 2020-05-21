---
title: "Lessions learned from buiding a large scale historical data analysis system using Azure Data Explorer - Part 1"
layout: post
summary: "notes about some lession learn from buiding a large scale historical data analysis system that has hundres of terabytes data usng Microsoft Azure Data Explorer"
description: notes about some lession learn from buiding a large scale historical data analysis system that has hundres of terabytes data  usng Microsoft Azure Data Explorer
toc: false
comments: true
image: https://azurecomcdn.azureedge.net/cvt-ee71595d3667788def73479da1629d673313a0b081e460fc596839b82f34a2df/images/page/services/machine-learning/mlops/steps/mlops-slide1-step3.svg
hide: false
search_exclude: false
categories: [Azure Data Explorer (Kusto),  Data]
---

Recently I had a opportunity to participate another data project that also uses [Azure Data Explorer](https://azure.microsoft.com/en-in/services/data-explorer/)(ADX) as the core data process and store engine. In this project we used ADX to ingest and process more than half of petabytes data. Like most projects we were under some time and resources constrains, and also encountered a few unexpected technical challenges due to the contrains . Though we couldn't implement the system using the best optimized architecture (it will take too much time than the project was allowed), we still managed to achieve the project goal. It's a exciting and fun journey and here are a few lesions we learned. 

##### Lesson 1 Select proper SKU

There are couple different [SKU options for ADX](https://docs.microsoft.com/en-us/azure/data-explorer/manage-cluster-choose-sku), some are more CPU optimized like D-Series (D-series, Ds-series) VM, they have more powerful CPU; some are more storage optimized like Ls-Serious VM, they are equipped with larger SSD to achieve more I/O performance. _Have a testing plan to test the key user query patterns on these different type of VMs and check which one is best for your query workload can benefit the project in the long run._

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-05-21-Lession-Learn-LargeScale-ADX-part1/ADX_SKU.JPG)


##### Lesson 2 Check different ingestion options. 

In Azure Data Explorer, it supports several different ingestion solutions, the decision will depend on the purpose and stage of your development. You can check [here](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/data-ingestion/#ingestion-methods) for detail information of these solutions.

It's better to read through the official document, understand their differences before make a decision. Mean while, here are a few thumb rules:  

* For query testings, verifying scripts, tables, you can use __Inline ingestion (push)__
* For adhoc feature engineering, data cleaning, you can use __Ingest from query__
* For ingestion testing, create some volumes of data, you can use __Ingest from storage (pull)__
* For production ingestion pipeline testiong, I normally will use __Queued ingestion__. ADX recently added 

In addition to above basic ingestion options, you can also check following options based on your scenario and environment. 

* [Ingest from storage using Event Grid subscription](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/data-ingestion/eventgrid)
* [Ingest from Event Hub](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/data-ingestion/eventhub)
* [Ingest from IoT Hub](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/data-ingestion/iothub)

And there is a new ingestion opition [One-Click Ingestion](https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-one-click) which are just been announced in [Microsoft Build 2020](https://mybuild.microsoft.com/). 

Eeek, a lot of choices. :p


##### Lesson 3 Have a clean ingestion pipeline 

Normally when trying to ingest data into data repository, we might need to do some data pre-process such as check file format, clean dirty data, do a few data transformation etc. Azure Data Explorer provides some of these capabilities through [User-defined function](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/functions/user-defined-functions) and [update policy](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/update-policy). You can use these mechanism to quickly perform some data pre-process tasks within ADX without setup extra computing services to handle it. 

While these are convenient ways to massage data, it will occupy ADX's resources and potentially introduce more data fragmentation. There are complex mechanisms within ADX to optimize the resources it has, maintain and organize data in it's storage and keep system in healthy status. 

Under the condition that data ingestion volume is big, these data pipeline activities could impact the resource available for ADX to handle queries or do internal housekeeping tasks. You might need to carefully monitor ADX status and do a few fine tune on it's configuration. While I did use these mechanisms for other smaller scale projects and love it, _it is still a better practice just to keep the data pre-process tasks outside of ADX in large scale project, at least before you are very familiar each internal mechanisms within ADX.)_  

Data Factory, Databricks, Azure Functions/App servicesc, AKS, HDInsight provide good funtational capabilities to pre-process data. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-05-21-Lession-Learn-LargeScale-ADX-part1/Ingest.png)



In [my git project](https://github.com/Herman-Wu/ADXAutoFileIngestion) I shared some codes that I used Azure Funtions to do Queued ingestion. 

##### Lesson 4 Evaluate the horizontal scale (# of servers) needed  

When planing system roll out, an solid estimation of the number of servers needed and a well estimated expansion plan for the future is important. It can also help saving cost by preventing under-utilization and provide valuable information for system design. System [scale out/ horizontal scaling](https://docs.microsoft.com/en-us/azure/data-explorer/manage-cluster-horizontal-scaling) is one of the core capabilities that we can make sure the system can be adaptive to the workload and provide just enough resource to the users. In our test, one of key ADX features that users love is it can provide almost linear performance growth when scale out. ADX also provides non-destructive services when scale out.

![img](https://docs.microsoft.com/en-us/azure/data-explorer/media/manage-cluster-horizontal-scaling/manual-scale-method.png)


_It's suggested that you should run a few workload simulation and test about how the scale out can increase your system capabilities._

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-05-21-Lession-Learn-LargeScale-ADX-part1/QueryPerf01.png)


ACI and AKS can be helpful if you want to simulate system workload. 


##### Lesson 5 Understand and test KQL query performance   

One of the key strength of ADX is it's powerful query language [Kusto Query Language](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/) (KQL). Like most of other data query languages, some query operators in KQL are similar, give you same result but could have huge different in performance. 

_It's always good the validate what's your key query scenarios and try how different query syntax impact the query performance._ 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-05-21-Lession-Learn-LargeScale-ADX-part1/ADX_SKU.JPG)

You can review the query performance using [Kusto.Explorer tool](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/tools/kusto-explorer) or [ADX Web UI](https://dataexplorer.azure.com/). You can also use [.show queries](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/queries) operator to review performance of  historical queries. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-05-21-Lession-Learn-LargeScale-ADX-part1/ShowQueries.png)



... to be continued 


##### Lesson 6 Consider the query run in client side. 

ADK is designed to process hundredes of gigbybte data or terabaybe data, when consider to fullfill query requirments for Application, we can consider to share some of workload to the Appication side. Especialy workload like sorting.  

But for some workload like aggregation, you should consider how to leverage it. 






##### Lesson 6 Check some advance table schema    

ADX provide some new more advance configuration in Table schema like partition and row ordering. You need more performance, they are options you can try. But use them carefully, they will increase data ingestion time and consume more computing resources. Also partition has some side effect on extents management. 

Also users should also consider to put data in differnt database difference. 

##### Lesson 7 Check Server loading

There a few core components/capablities in ADX. You can check them by using .show capacity.  

##### Lesson 8 Review Security Setting 

There are different security roles in ADX. It's good to review the security setting in your system.  ADX supports managed Identity.  
Also 