---
title: "Lessons learned from building a large scale historical data analysis system using Azure Data Explorer - Part 2"
layout: post
summary: "notes about some lession learn from buiding a large scale historical data analysis system that has hundres of terabytes data usng Microsoft Azure Data Explorer - Part II"
description: notes about some lesson learned from building a large scale historical data analysis system that has hundreds of terabytes data using Microsoft Azure Data Explorer - Part II
toc: false
comments: true
image: https://azurecomcdn.azureedge.net/cvt-ee71595d3667788def73479da1629d673313a0b081e460fc596839b82f34a2df/images/page/services/machine-learning/mlops/steps/mlops-slide1-step3.svg
hide: false
search_exclude: false
categories: [Azure Data Explorer, Kusto, Data, KQL, Azure]
---

[_This is the part II of the topic, you can check [Part I](https://herman-wu.github.io/blogs/azure%20data%20explorer%20(kusto)/data/2020/05/21/Lession-Learn-LargeScale-ADX-part1.html) if you haven't._]

I have mentioned [5 basic lessons](https://herman-wu.github.io/blogs/azure%20data%20explorer%20(kusto)/data/2020/05/21/Lession-Learn-LargeScale-ADX-part1.html) I learned from my recent project that has hundreds of terabytes data. In this __part II__ I would like to share a few more advanced lessons.  


##### Lesson 6 Consider guiding users to submit the right queries and think about having some data process running on the client-side 

[Azure Data Explorer](https://azure.microsoft.com/en-in/services/data-explorer/)(ADX) is designed to process a huge amount of data. It can efficiently handle hundreds of terabytes data or even petabytes data. Data is a very valuable asset especially when you have a lot of them. These data can support users to find hidden business trends, detect anomaly events, identify correlation things from these data. 

However with so much data in your system's back yard, you also need to make sure users will access them in the right way because users might not have a good sense of how much data they are trying to process. So we should guide and prevent users from unconsciously trying to retrieve 100 gigabytes of data to their web browser or trying to get 1 million records to their client just for showing a line chart. Helping users to use aggregation and return data in just enough granularity for their business needs can largely reduce system workload and improve performance. 

Also you should consider using the user's client to share some data process loading of ADX. For example, users might want to sort the result by some specific columns, this can be easily implemented on the client-side. It's also easier to do data paging in client slides then resubmit queries for each page.
 
In ADX it provides some mechanisms such as [result set size](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/querylimits#limit-on-result-set-size-result-truncation), [request execution time](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/querylimits#limit-on-request-execution-time-timeout) to prevent system resource been occupied by some bad queries. You can also use these mechanisms to prevent some queries to take too many resources from the system or grant more resources to some special queries.  


##### Lesson 7 Check some advance table policy    

To further provide better query performance, ADX now provides some new more advanced configuration in Table policies such as [Data partitioning policy](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/partitioningpolicy) and [RowOrder policy](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/roworder-policy). If you need to improve query response time, they are options you can try. But using them carefully, __they will increase data ingestion time and consume more computing resources__. Also, partition has some side effect on extents management and need more resources to run the policy. 

The other consideration is the choice between putting data in different tables or different databases. This choice will impact the following areas in system design:
* Data Access Control
* Admin node workload 
* Data sharing mechanism 

Users should consider putting data in different databases when you want to have more flexibility to manage and share data. Also when the data volume is big, having multiple databases can enjoy the benefit of having different Admin nodes for each database and then have more resources to maintaining data metadata and perform transactions.    

##### Lesson 8 Check Server loading and very very carefully tuning some of them if needed. 

To handle data ingestion and query workload, ADX has several mechanisms to balance the resources used to support different operation and maintenance workload. 

For data management operations, you can use [.show capacity](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/capacitypolicy) to quickly show the resources utilization of these core capabilities. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-05-25-Lession-Learn-LargeScale-ADX-part2/show_capacity.png)

In most cases the official recommendation is __just leave the setting as is, the default setting should be able to handle most of the situations__ and they will keep each resource utilization balanced. If in some cases if you thought you really need to change these values, it's also recommended to at least check and ask in [stackoverflow](https://stackoverflow.com/questions/tagged/kusto) to have some experts to double verify it. 


Also you might also want to check the Extents number and its growth in your system. Extents are the core data allocation unit within ADX, ADX will try to organize it in the best way to fulfill both data ingestion needs and data query needs. If for some reason too much extends are created, it will increases the loading of ADX admin node and potential will impact system performance. For a large system it's a good practice to keep an eye on it. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-05-25-Lession-Learn-LargeScale-ADX-part2/total_extents.png)


##### Lesson 9 Optimize the KQL query plan for the long queries and review cache policy
Though we mentioned how different query syntax could impact query performance in  [Understand and test KQL query performance session of Part I](https://herman-wu.github.io/blogs/azure%20data%20explorer%20(kusto)/data/2020/05/21/Lession-Learn-LargeScale-ADX-part1.html), here I want to talk about KQL query plan. 

In Kusto Explorer, there is a build-in tool named Query Analyzer which provides extra information on how your KQL runs. You can use them to understand how the query actually being executed, if your query can run in parallel on all the nodes, if the query filter efficiently reduces data retrieval volumes or if the data join.   


![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-05-25-Lession-Learn-LargeScale-ADX-part2/execprofile.png)


![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-05-25-Lession-Learn-LargeScale-ADX-part2/reloptree.png)


Also by design initially (before the cluster is running) all the data are stored on cold storage (Azure Blob storage). When the cluster starts, cluster nodes will load all these hot data (defined by [Cache policy](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/cachepolicy)) data into local SSD and memory so they become hot data and can be quickly accessed. Review the cache hit rate of popular queries and check if the cache policy is properly defined can potentially have a good boost of performance. 


##### Lesson 10 Review Security Setting 

Last but not least, Security. ADX can use both Azure AD and Microsoft Account (MSAs) as the identity providers. And we also need to have a plan for user authentication and [application authentication](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/access-control/how-to-provision-aad-app). 

 There are different [security roles](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/access-control/role-based-authorization) in ADX. It's good to review the security setting in your system and make sure you have separation of concerns for your account and security planning.
 
 <br>
<br>

Azure Data Explorer is a very powerful tool for analyzing historical log or telemetries data. In many cases we saw it provide a more cost-effective and more powerful query tool to fulfill users' business needs. 

Above are 10 lessons I thought are important when implementing a large scale data analysis platform when using Azure Data Explorer. The best way to understand the detail of them is still read through the official documents. But these lessons might provide you a quick bird's-eye view of what you should check and look.    

__[Reference]__

[- kusto-high-scale-ingestion](https://github.com/Azure-Samples/kusto-high-scale-ingestion/blob/master/processing/README.md)

[- Azure Data Explorer technical white paper](https://azure.microsoft.com/en-ca/resources/azure-data-explorer/)

[- Update policies for in-place ETL in Kusto (Azure Data Explorer)](https://yonileibowitz.github.io/kusto.blog/blog-posts/update-policies.html)
