---
title: "Security practices and design priciples for implementing a data lakehouse solution in Azure Synapse"
layout: post
summary: "notes about my security learning and design practices from using Synapse to build a cost-effective data lakehouse solution"
description: notes about my security learning and design practices from using Synapse to build a cost-effective data lakehouse solution
toc: false
comments: true
image: https://azurecomcdn.azureedge.net/cvt-ee71595d3667788def73479da1629d673313a0b081e460fc596839b82f34a2df/images/page/services/machine-learning/mlops/steps/mlops-slide1-step3.svg
hide: false
search_exclude: false
categories: [datalakehouse, lakehouse,Synapse, Security, Data, SQL, Serverless SQL, Azure, VNet, Private Endpoint]
---

###  _Security practices and design priciples for implementing a data lakehouse solution in Azure Synapse_

#### Background 

<p style="font-size: 0.9rem;font-style: italic;"><img style="display: block;" src="https://live.staticflickr.com/7618/16591080240_f5b5f86100_b.jpg" alt="The Lake House "><a href="https://www.flickr.com/photos/80223459@N05/16591080240" target="_blank" rel="noopener noreferrer">"The Lake House"</a><span> by <a href="https://www.flickr.com/photos/80223459@N05" target="_blank" rel="noopener noreferrer">YellowstoneNPS</a></span>. Licensed under <a href="https://creativecommons.org/publicdomain/mark/1.0/&atype=html" style="margin-right: 5px;" target="_blank" rel="noopener noreferrer">CC PDM 1.0</a><a href="https://creativecommons.org/publicdomain/mark/1.0/&atype=html" target="_blank" rel="noopener noreferrer" style="display: inline-block;white-space: none;margin-top: 2px;margin-left: 3px;height: 22px !important;"><img style="height: inherit;margin-right: 3px;display: inline-block;" src="https://search.openverse.engineering/static/img/cc_icon.svg?media_id=66a4f356-08b3-415f-8c11-596db0912c6f" /><img style="height: inherit;margin-right: 3px;display: inline-block;" src="https://search.openverse.engineering/static/img/cc-pdm_icon.svg" /></a></p>

Synapse is a versatile data platform that supports enterprise data warehousing,  realtime data analytics, data pipeline,  time-serious data processing, machine learning and data governance. It integrates several different technologies (e.g., SQL DW, Serverless SQL, Spark, Data Pipeline, Data Explorer, Synapse ML, Purview...) to support these various capabilities. However, this also inevitably increases the complexity of  the system infrastructure. 

![](/assets/img/2022-03-15-Security-practices-and-design-priciples-for-implementing-a-data-lakehouse-solution/synaspe-solution-overview.png)
![img]({{ site.url }}{{ site.baseurl }}/assets/img/2022-03-15-Security-practices-and-design-priciples-for-implementing-a-data-lakehouse-solution/synaspe-solution-overview.png)


In this blog I would like to share the security learning after implemented a [data lakehouse](https://databricks.com/glossary/data-lakehouse) project for a international manufacture company using Synapse. [Data Lakehouse](https://databricks.com/glossary/data-lakehouse)  is a modern data management architecture that combines  data lakes's  cost-efficiency, scale, and flexibility features with data warehouses's data and transaction management capabilities. It well supports business intelligence and machine learning (ML) scenario for large amount of diverse data structure and data source.Some common use cases for the solution are IoT telemetry  analysis, commsumer activities and behaveior tracking, security log monitoring, or semi-structured data processing.   

We will focus on  the security design and implementation pratices used in the project. In the project we chose [Synapse serverless SQL](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql/on-demand-workspace-overview)  and [Synpase Spark](https://docs.microsoft.com/en-us/azure/synapse-analytics/spark/apache-spark-overview) to implemente the [data lakehouse pattern](https://blog.starburst.io/part-2-of-current-data-patterns-blog-series-data-lakehouse). Following is the high level solution design architecture.

![img](https://techcommunity.microsoft.com/t5/image/serverpage/image-id/329206iAD1B6F56B8C16E88/image-size/large?v=v2&px=999)
<figcaption align = "center">Fig.1   High level concept of the solution  <br> 
Source: <a href="https://techcommunity.microsoft.com/t5/azure-synapse-analytics-blog/the-best-practices-for-organizing-synapse-workspaces-and/ba-p/3002506" target="_blank" rel="noopener noreferrer">The best practices for organizing Synapse workspaces and lakehouse</a> </figcaption>


#### Design Focus 


We started the security design by using the [Threat Modeling tool](https://www.microsoft.com/en-us/securityengineering/sdl/threatmodeling). The tool helps us communicate with project stakeholders about the protecial risks, and define the trust boundary in the system.   Based on the thread modeling result,  <u>Identity and Access control</u>, <u>Network protection</u>, and  <u>DevOps security</u> are proirotized in the project. Based on these priorities, we implemented additional security features and changed the infrastures so we can protect the system and mitigate key secruity risks identified. These also map to __Access Control__, __Asset Protection__, and __Innovation Security__ in the  [Cloud Adoption Framework (CAF)'s security disciplines](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/secure/#security-disciplines). We will walk through the design priciples and related technologies in more detail  In the following sections.  

#### Security Design Principles and Learning

##### _Network and Asset protection Design_

One of the key security assurances principles in the Cloud Adoption Framework is the [Zero Trust principle](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/secure/#guiding-principles). When designing security for any component or system, we should reduce the risk of attackers expanding access by assuming other resources in the organization are compromised. 

Based on the threat modeling discussion result, we follow the [micro-segmentation deployment](https://docs.microsoft.com/en-us/security/zero-trust/deploy/networks#i-network-segmentation-many-ingressegress-cloud-micro-perimeters-with-some-micro-segmentation) recommendation in zero-trust and defined [several security boundaries](https://dev.azure.com/CSECodeHub/412717%20-%20Hitachi%20Vantara%20Lumada%20Manufacturing%20Insights%20Solution/_wiki/wikis/Engagement%20Wiki/13424/008-Trusted-Boundary-Planning-and-VNet-Design). VNet and [Synapse data exfiltration protection](https://docs.microsoft.com/en-us/azure/synapse-analytics/security/workspace-data-exfiltration-protection#:~:text=Azure%20Synapse%20Analytics%20workspaces%20support%20enabling%20data%20exfiltration,data%20to%20locations%20outside%20of%20your%20organization%E2%80%99s%20scope.) are the key technologies used to implement the security boundary and protect the system's data assets and critical components.

Considering Synapse is a composition of [several different technologies](https://docs.microsoft.com/en-us/azure/synapse-analytics/overview-what-is), we need to :
- <u>Identify essential  components of Synapse and related services used in the project</u>. 

  Synapse is a very versatile data platform. It can handle and fulfill many different data processing needs. First, we need to decide which components in Synapse are used in the project to plan how to protect them. Also, we need to determine what other services are communicating with these Synapse's components. In the data data lakehouse architecture, _Synapse Serverless SQL, Synapse Spark, Synpase Pipeline,Azure Data Lakes and Azure DevOps_ are the key components. 

- <u>Define the __legal communication behaviors__ between the components</u>.

    We need to define the "legal" communication behaviors between the components. For example, do we want the Synapse Spark engine to communicate with the dedicated SQL instance directly, or do we want the spark engine to communicate with the database through a proxy such as Synapse Data Integration pipeline or Data Lake?
    
    Base on the Zero trust principle, we should block the communication if there is no business need for the interaction. For example, we should block the communication if a Synapse Spark engine directly communicates with Data Lake storage in an unknown tenant.  

- <u>Chose the proper security solution that can enforce the defined communication behaviors</u>.

    In Azure, several security technologies are capable of enforcing the defined service communication behaviors. For example, in Azure Data Lake storage, you can use a white-list IP address to control its access, but you can also choose allowed VNet, Azure services, or resource instances. Each protection method provides a different security protection and needs to be selected based on the business needs and environment limitation. I will decribe the configuration we used in our project in the next section.   
    
 - <u>Add threat detection and advanced defense for critical resources</u>.   

    For critical resources, it is better to add threat detection and advanced defense. These services help identify threats and triggers alerts. So the system can notify uses about the security breach.
    
##### _Network and Asset protection Implementation in the project_

In the  data lakewarehouse solution, we designed and controlled the service's interaction behaviors based on business requirements to mitigate security threats. The following table shows the defined communication behaviors and security solutions used in the project. 

From (Client) | To (Service) | Behavior| Configuration | Notes
--- | --- | --- | --- | -- 
Internet| Azure DataLake |  Deny All | Firewall Rule - Default Deny | Default: 'Deny' | Firewall Rule - Default Deny
Synapse Pipeline/Spark| Azure DataLake |  Allow (Instance) | VNet - Managed Private EndPoint (Azure DataLake) | 
Synapse SQL | Azure DataLake | Allow (Instance) | Firewall Rule - Resource instances (Synapse SQL) | Synapse SQL needs to access Azure DataLake using Managed Identity | N/A
Azure Pipeline Agent | Azure DataLake | Allow (Instance) | * Firewall Rule - Selected Virtual networks <br> * Service Endpoint - Storage  |  For Integration Testing <br> bypass: 'AzureServices' (firewall rule)
Internet | Synapse Workspace | Deny All  | Firewall Rule | | Firewall Rule 
Azure Pipeline Agent | Synapse Workspace  | Allow (Instance) |  VNet - Private EndPoint | Requires 3 Private EndPoints (Dev, Serverless SQL, and Dedicate SQL) 
Synapse Managed VNet | Internet/ Unauthorized Azure Tenant| Deny All | VNet - Synapse Data Exfiltration Protection| 
Synaspe Pipeline/Spark| KeyVault |  Allow (Instance) | VNet - Managed Private EndPoint (KeyVault) |Default: 'Deny'  
Azure Pipeline Agent| KeyVault | Allow (Instance)  |* Firewall Rule - Selected Virtual networks <br> * Service Endpoint - KeyVault  |  bypass: 'AzureServices' (firewall rule)
Azure Functions| Synapse Serverless SQL| Allow (Instance) | VNet - Private EndPoint (Synapse Serverless SQL) 
Synaspe Pipeline/Spark|Azure Monitor |Allow (Instance) |VNet - Private EndPoint (Azure Monitor) 

The bellow diagram shows the architecture with the network and asset protection design.  
![](/assets/img/2022-03-15-Security-practices-and-design-priciples-for-implementing-a-data-lakehouse-solution/security-architecture01.png)
![img]({{ site.url }}{{ site.baseurl }}/assets/img/2022-03-15-Security-practices-and-design-priciples-for-implementing-a-data-lakehouse-solution/security-architecture01.png)



For example, the above diagram includes:
- Create a Synapse workspace with a managed virtual network. 
- Securing data egress from Synapse workspaces through [Synapse workspaces Data exfiltration protection.](https://docs.microsoft.com/en-us/azure/synapse-analytics/security/workspace-data-exfiltration-protection)
- Manage the list of approved Azure AD tenants for the Synapse workspace.
- Configure network rules to grant only traffic from selected virtual networks access to storage account and disable public network access.
- Use [Managed Private Endpoints](https://docs.microsoft.com/en-us/azure/synapse-analytics/security/synapse-workspace-managed-private-endpoints) to connect Synapse managed VNet with Data Lake. 
- Use [Resource Instance](https://docs.microsoft.com/en-us/azure/storage/common/storage-network-security?tabs=azure-portal#grant-access-from-azure-resource-instances-preview) to securely connect Synapse SQL with Data Lake 

For better Network and Asset protection, the following are additional security design considerations.

 - Deploy Perimeter Networks for Security Zones for Data Pipeline. 
 
    Because in a data pipeline, data could be loaded from external data sources. When a data pipeline workload requires access to external data and data landing zone, it is better to implement a perimeter network and separate it with a regular ETL pipeline.

- Enable [Azure Defender for all storage accounts](https://docs.microsoft.com/en-us/azure/storage/common/azure-defender-storage-configure). 

    Azure Defender provides an additional layer of security intelligence that detects unusual and potentially harmful attempts to access or exploit storage accounts. Security alerts are triggered in Azure Security Center. 
- [Lock storage account](https://docs.microsoft.com/en-us/azure/storage/common/lock-account-resource) to prevent malicious deletion or configuration changes


##### _Identity and Access control_

There are several parts in the system, each part requires different Identity and Access Management (IAM) configuration. They will need to collaborate tightly with each other to provide a streamlined user experience. Therefore, when we implement authentication and authorization control, we need to plan the following parts.

![](/assets/img/2022-03-15-Security-practices-and-design-priciples-for-implementing-a-data-lakehouse-solution/synapse-access-control.png)
![img]({{ site.url }}{{ site.baseurl }}/assets/img/2022-03-15-Security-practices-and-design-priciples-for-implementing-a-data-lakehouse-solution/synapse-access-control.png)
![](img/synapse-access-control.png)

- <u>Chose Identity type in different Access Control Layers</u>

    There are four different identity types in the system.
    - SQL Account (SQL Server)
    - Service Principal (Azure AD)
    - Managed Identity (Azure AD)
    - User AAD Account (Azure AD)

    Also, there are four different access control layers in the system.
    - Application access layer
    - Synapse access layer
    - SQL DB access layer
    - Azure Data Lake access layer

    A key part of identity and access control is choosing the right identity solution for different user roles in each access control layer. The [Well Architecture Security Design Principles](https://docs.microsoft.com/en-us/azure/architecture/framework/security/security-principles) suggests using native controls and driving simplicity. Therefore, we decided to use User AAD account in the Application, Synapse, and SQL DB access layer to leverage the native first-party IAM solutions while using Managed Identity of Synapse to access Azure Data Lake to simply the authorization process. 

- <u>Consider Least-privileged access</u>

    In Zero trust guiding principles, it suggests providing just-in-time and just-enough-access to critical resources. After discussing with Security SA, we suggested Hitachi Vantara implement Azure [AD Privileged Identity Management (PIM)](https://docs.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-configure) and enhance security deployment check in the future. 

- <u>Protect Linked Service</u>

    Linked services define the connection information needed for the service to connect to external resources. It is important to secure the linked Service configuration and access in Synapse.

    - Create [Azure Data Lake linked service with Private Links](https://docs.microsoft.com/en-us/azure/synapse-analytics/data-integration/linked-service#:~:text=In%20Azure%20Synapse%20Analytics%2C%20a%20linked%20service%20is,Manage%20tab.%20Under%20External%20connections%2C%20select%20Linked%20services.)
    - Use [Managed Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) as the authentication method in linked services
    - Use Azure Key Vault as the secret stored in Linked Service.

#### DevOps security

- <u>Use VNet enabled self-hosted pipeline agent</u>

    Default Azure DevOp pipeline agent couldn't support VNet communication because it used a very wide IP range. Therefore, we implemented Azure DevOps [self-hosted agent](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux?view=azure-devops) in VNet so the DevOps process can be protected and smoothly communicate with the whole Hitachi Data Emporium system. In addition, [VM scale sets](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview) are used to ensure the DevOps engine can scale up and down.  

![](/assets/img/2022-03-15-Security-practices-and-design-priciples-for-implementing-a-data-lakehouse-solution/devop-security-archi.png)
![img]({{ site.url }}{{ site.baseurl }}/assets/img/2022-03-15-Security-practices-and-design-priciples-for-implementing-a-data-lakehouse-solution/devop-security-archi.png)
![](img/synapse-access-control.png)
![](img/devop-security-archi.png)

- <u>Implement infrastructure security scanning & security smoke testing in CI/CD pipeline</u>

    Static analysis tool for scanning infrastructure as code (IaC) files can help detect and prevent misconfigurations that may lead to security or compliance problems. In addition, security smoke testing ensures that the vital system security measure is successfully enabled and prevents security risk due to a design fault in the deployment pipeline.

    - Use static analysis tool for scanning infrastructure as code (IaC) templates to detect and prevent misconfigurations that may lead to security or compliance problems. Use tools such as [Checkov](https://www.checkov.io/) or [Terrascan](
    https://github.com/accurics/terrascan) to detect and prevent security risks.
    - Make sure the CD pipeline correctly handles the failure of the deployment. Any deployment failure related to security features should be treated as a critical failure.  It should retry the failed action or hold the deployment.  
    - Validate the security measures in the deployment pipeline by running security smoke testing. The security smoke testing, such as validating the configuration status of deployed resources or testing cases that examine critical security scenarios, can ensure that the security design is working as expected.

#### Security Score assessment and Treat Detection 

To understand the security status of the system, we used Microsoft Defender for Cloud to assess the infrastructure security and detect the security issues. [Microsoft Defender for Cloud](https://docs.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction) is a tool for security posture management and threat protection. It can protect workloads running in Azure, hybrid, and other cloud platforms.

![img](https://docs.microsoft.com/en-us/azure/defender-for-cloud/media/defender-for-cloud-introduction/defender-for-cloud-synopsis.png)

You can enable Defender for Cloud's free plan  on all your current Azure subscriptions when you visit the Defender for Cloud pages in the Azure portal for the first time. I highly recommend to enable it so you can get your Cloud security posture evaluation and suggestions. And it'a all free, so why not ☺️. Microsoft Defender for Cloud will provide you security score and some security hardening guidance for your subscription.

![img](https://docs.microsoft.com/en-us/azure/defender-for-cloud/media/secure-score-security-controls/single-secure-score-via-ui.png)


 The information is quite strait forward and the recommendations are very good. Many of the recommendations can are easy to take action. 

![img](https://docs.microsoft.com/en-us/azure/defender-for-cloud/media/secure-score-security-controls/remediate-vulnerabilities-control.png)


 And if you need advanced security management and threat detection capabilities, which provide fetures such as suspicious activities detection and alerting. You can enable  Cloud workload protection indivually for different resources. So you have the option to choose the most cost effective way to protect the system. 

 #### Summary 

With the combination of Synapse Serverless SQL and Synpase Spark, we built a flexible,  scalable and cost-effective data lakehouse solution in the project. In this article, I tired to summarized the security design principle and practices we implemented in the solution. We start from the [Threat Modeling tool](https://www.microsoft.com/en-us/securityengineering/sdl/threatmodeling) and guidance in the  [Cloud Adoption Framework (CAF)'s security disciplines](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/secure/#security-disciplines). To harden the security protection, the project team decided to focus on Network and Asset protection, Identity and Access control, and DevOps security. We also evaluated the security score of the system and review the security suggestions provided by Microsoft Defender for cloud. Hope this learning will also helps you to  implementing a secure data lakehouse solution on Azure Synapse.


<br>
<br>
##### Reference
- [Current Data Patterns Blog Series: Data Lakehouse](https://blog.starburst.io/part-2-of-current-data-patterns-blog-series-data-lakehouse)
- [The Data Lakehouse, the Data Warehouse and a Modern Data platform architecture](https://techcommunity.microsoft.com/t5/azure-synapse-analytics-blog/the-data-lakehouse-the-data-warehouse-and-a-modern-data-platform/ba-p/2792337?msclkid=c7eddbcbb24411ecae0f0ec795c2ad28)
- [The best practices for organizing Synapse workspaces and lakehouse](https://techcommunity.microsoft.com/t5/azure-synapse-analytics-blog/the-best-practices-for-organizing-synapse-workspaces-and/ba-p/3002506)
- [Secure score in Microsoft Defender for Cloud](https://docs.microsoft.com/en-us/azure/defender-for-cloud/secure-score-security-controls#:~:text=Defender%20for%20Cloud%20continually%20assesses,lower%20the%20identified%20risk%20level.)
- [Understanding Azure Synapse Private Endpoints](https://www.thedataguy.blog/azure-synapse-understanding-private-endpoints/)
- [Azure Synapse Analytics – New Insights Into Data Security](https://dzone.com/articles/azure-synapse-analytics-new-insights-into-data-sec)
- [Azure security baseline for Azure Synapse dedicated SQL pool (formerly SQL DW)](https://docs.microsoft.com/en-us/security/benchmark/azure/baselines/synapse-analytics-security-baseline)
- [Cloud Network Security 101: Azure Service Endpoints vs. Private Endpoints](https://www.fugue.co/blog/cloud-network-security-101-azure-service-endpoints-vs.-private-endpoints)
- [How to set up access control for your Synapse workspace](https://docs.microsoft.com/en-us/azure/synapse-analytics/security/how-to-set-up-access-control)
- [Connect to Azure Synapse Studio using Azure Private Link Hubs](https://docs.microsoft.com/en-us/azure/synapse-analytics/security/synapse-private-link-hubs)

