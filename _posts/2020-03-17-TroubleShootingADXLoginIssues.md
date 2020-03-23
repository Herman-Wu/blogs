---
title: "Troubleshooting  ADX (Kusto) login issues"
layout: post
summary: "notes about some learning from fixing ADX (Kusto) account login issues  "
description: notes about some learning from fixing ADX (Kusto) account login issues 
toc: false
comments: true
image: https://azurecomcdn.azureedge.net/cvt-ee71595d3667788def73479da1629d673313a0b081e460fc596839b82f34a2df/images/page/services/machine-learning/mlops/steps/mlops-slide1-step3.svg
hide: false
search_exclude: false
categories: [Azure Data Explorer (Kusto),  Data]
---

##### Azure Data Explorer Introduction

[ADX (Azure Data Explorer, aka Kusto)](https://docs.microsoft.com/en-us/azure/data-explorer/data-explorer-overview) is a very powerfully log/historical data analysis platform provided by Microsoft that powers several key Azure services such as Application Insight, Azure Monitor, Time Series insight. It is designed to handle huge amounts of historical data and can ingest and process Peta-bytes of data every day with little efforts to set up the infrastructure. On February 7, 2019, Microsoft GA the service to customers. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-03-17-TroubleShootingADXLoginIssues/ADXOverview.jpg)

One of the features that I particularly like is its query language KQL (Kusto Query Language), KQL combines the concept of SQL query and data pipeline. In real project experience, it is very powerful and saved me a lot of time to get the query result in the structure that my projects need. You can check [this official tutorial](https://docs.microsoft.com/en-us/azure/kusto/query/tutorial?pivots=azuredataexplorer) that introduces KQL's key concept and following is an example about how to query data 12 days ago and do count aggregation very 30 minutes. It's a very popular bin count pattern when analyzing data on time dimension. In the query we use ["mv-expand"](https://docs.microsoft.com/en-us/azure/kusto/query/mvexpandoperator) operator to make sure there is still a record presents the 0 count even the system has no data in that 30 minutes range. "mv-expand" is also very useful when you are parsing and expanding JSON data. 

```
let StartTime=ago(12d);
let StopTime=ago(11d);
T
| where Timestamp > StartTime and Timestamp <= StopTime 
| summarize Count=count() by bin(Timestamp, 30m)
| union ( // 1
  range x from 1 to 1 step 1 // 2
  | mv-expand Timestamp=range(StartTime, StopTime, 30m) to typeof(datetime) // 3
  | extend Count=0 // 4
  )
| summarize Count=sum(Count) by bin(Timestamp, 30m) // 5
```
![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-03-17-TroubleShootingADXLoginIssues/kqlqueryresult.jpg)

You can also easily generate a graphic chart to visualize your query result through ["render"](https://docs.microsoft.com/en-us/azure/kusto/query/renderoperator?pivots=azuredataexplorer) operator. In the previous query, you can add "| render timechar" to generate the following graph which represents the trend of how many records we have every 30 mins eleven days ago. 
```
|render timechart 
```
![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-03-17-TroubleShootingADXLoginIssues/kqlquerygraph.jpg)


For data consumers ADX also provides various client tool that users can use to create different kind of query, graphic chart and dashboard. The tool that support ADX includes :
  - [ADX WebUI](https://docs.microsoft.com/en-us/azure/data-explorer/web-query-data)
  - [Kusto Desktop Client tool](https://docs.microsoft.com/en-us/azure/kusto/tools/kusto-explorer)
  - PowerBI
  - Excel 
  - Jupyter Notebook([using KQLMagic extension](https://github.com/microsoft/jupyter-Kqlmagic))
  - [Grafana](https://docs.microsoft.com/en-us/azure/data-explorer/grafana)
  - [Spark](https://github.com/Azure/azure-kusto-spark)   . 

##### Azure Data Explorer Access Control

In the access control part, ADX supports user authentication through Microsoft Accounts (MSAs) and Azure AD account. Microsoft Account is non-organizational user accounts, these accounts normally use email like hotmail.com, live.com, outlook.com. Azure AD account is created through Azure AD or another Microsoft cloud service such as Office 365. It will be tight to an Azure AD tenant and is the preferred method for authenticating to ADX.  Most enterprise users should use AAD account for authentication. Here has more information about [ADX access control](https://docs.microsoft.com/zh-tw/azure/kusto/management/access-control/). 


Like most Azure services, you can manage user accounts and their access to ADX through "Access control" function within Azure Portal. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-03-17-TroubleShootingADXLoginIssues/ADXAccessControl.jpg)

The other way to manage it is through KQL query. You can run the following command to check the accounts and roles that can access ADX database. 

```sql
.show database [Database Name]] principals 
```

You can read ADX [Security roles management](https://docs.microsoft.com/en-us/azure/kusto/management/security-roles) session to understand more about using  KQL to manage user access control. 

ADX authentication is also the part I would like to share a few troubleshooting tips which  I learned from projects. 


##### Trouble Shooting Azuer Data Explorer user authentication issue

Ideally if you login your PC with Azure AD account that in the same tenant of the Azure subscription which been used to create the ADX cluster, then everything should work good and you can just management user access in Azure Portal. How ever it's not always the case, users can come from different organizations and partners. Here are a few ways we used to check and fix the issues. 

##### The problem:
*Grant a user to access ADX in Azure Portal UI. And the user can assess ADX in Azure portal. But he/she but couldn't access it in [ADX Web UI]((https://docs.microsoft.com/en-us/azure/data-explorer/web-query-data)) and [Kusto Explorer](https://docs.microsoft.com/en-us/azure/kusto/tools/kusto-explorer), PowerBI, Excel or other client tools.*

What happened is when you grant a new user to access ADX, if the user's account comes from a different tenant, in ADX it will create a new Principle Object Id in its tenant. If the user access ADX in Azure portal, the user's account already switches to the right tenant so there is no problem accessing ADX. 

##### How to fix the issue

_**Solution A**: Force ADX WebUI and Kusto Explorer to use not ADX's tenant_

* ADX WebUI

  Default ADX WebUI has URL like 

  https://dataexplorer.azure.com/clusters/[cluster name].[data center]

  You can force it to recognize a user's login tenant by add &tenant=[tenant id] to the URL. So it will look like

  https://dataexplorer.azure.com/clusters/[cluster name].[data center]?tenant=[tenant id]

* Kusto Explorer Client

  Instead of default connection by specifying ADK cluster URL, You need to use advanced query. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-03-17-TroubleShootingADXLoginIssues/KustoExpConn.jpg)

  The connection string will be like:

```
  Data Source=https://[Cluster Name].[Data Center].kusto.windows.net;AAD Federated Security=True;AAD User ID=[User's AAD account(E-mail)];Authority Id=[User's AAD account tenant ID ]
```

<br>

_**Solution B**: Grant user using 'right' tenant id and account id_

As said by default user granted in Azure portal will using ADX's tenant id and create a new account object id. If you want to add a user from a different tenant, the suggested way will be to grant users using KQL. 

You can grant a new user with different roles like

```kql 
.add database [Database Name] [Role Name]  ('aaduser=[User AAD account id];[User AAD tenant id]') 'Notes for the account, eg. User Name'

```

* Get AAD account object id & tenant id using Azure CLI

  To get user's AAD account object id, you can use Azure CLI command :

```bash
  az ad signed-in-user show 
```

  To get the user's tenant ID, you can use Azure CLI command :

```bash
  az account list 
```

* Get AAD account object id & tenant id in Azure portal

  You can also find AAD account object id & tenant id in Azure Active Directory service of  Azure portal. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2020-03-17-TroubleShootingADXLoginIssues/AAD.jpg)


* Get AAD account object id & tenant id in ADX Web UI 

  If you login using ADX Web UI, actually the error message contains the AAD user account id and tenant id. The message will be looks like following 

  _Error message : 
action Principal 'aaduser=[__AAD accound id__];[ __AAD tenant id__ ]' is not authorized to perform operation_

  It's another easy way to get the information you needed. 

<br>
ADX is a very powerful platform that provides interactive analysis capabilities which can handle huge amount of historical log data with little infrastructure maintenance efforts. Because most enterprises use Azure AD to grant users access, helping these tips can save you sometime when you have cross tenant access requirements. 
