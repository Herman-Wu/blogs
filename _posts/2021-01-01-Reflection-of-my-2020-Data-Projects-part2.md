---
title: "Reflections on my 2020 Data Projects  - Part I"
layout: post
summary: "Last year, I participated in projects which process and analyze terabytes of data daily. We managed to have the system went production successfully and it is now processing data from different continents 24x7. Here are some learnings on the journey."
description: Last year, I participated in projects which process and analyze terabytes of data daily. We managed to have the system went production successfully and it is now processing data from different continents 24x7. Here are some learnings on the journey.
toc: false
comments: true
image: 
hide: false
search_exclude: false
categories: [Azure Data Explorer, Kusto, Data, Azure, Data Pipeline, Reflections]
---

####  _Lessons I learned and want to keep in mind in 2021+_

___[<< Part-I](https://herman-wu.github.io/blogs/2021/01/01/Reflection-of-my-2020-Data-Projects-part1.html)___ 

### #6 Every small problem can be BIG

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/quarantine-4981010_640.jpg)
<br>
Image by <a href="https://pixabay.com/users/evgenit-4930349/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4981010">Evgeni Tcherkasski</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4981010">Pixabay</a>

When we were developing solutions for different projects, the dev team usually started by implementing the functional logic that fulfills the design. We used small volumes of data to validate the functionality. During these functional development tasks, there are a few things that deserve our additional attention because they could impact the system’s stability when the system is running in full scale and handles production workload during peak time.

- __Cloud is a live infrastructure__: Cloud is an environment that keeps evolving over time. All cloud vendors are working hard to improve existing services and introduce new capabilities in their platform. It also means the services on the cloud will be upgraded every couple of months. Though cloud vendors/operators will try to prevent service interruptions and sometimes reduce the interruption time to less than one second, it still impacts heavy load systems/components that happen to have hundreds of data processing tasks at the moment. Some design pattern like 
[Claim-Check Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/claim-check) or [Queue-Based Load Leveling pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) could help mitigate the impact.  
- __Network Transient Failure__: Network is the vessel of a modern system. It connects every component and transmits a massive information payload in high frequency. Many services and SDKs already have build-in mechanisms to handle transient network failure. However, for a system with heavy network traffic, we should review how it works in the architecture and consider adding additional mechanisms. 
The most common practice to handle network failure is to resend and retry the network operation. What needs to pay attention is it's better to implement a "smarter" retry strategy for a system that has a huge workload and heavy network operation. Some libraries like Polly (.net core), Tenacity (python) can help us implement the retry action using an advanced algorithm. 
- __Optimize Disk I/O__: 
    - Understand the distributed file storage system underneath your data solution. For example, check Sharding for Mongo DB, Extents for Azure Data Explorer, Partitioning (bucketing) for Databricks Delta table.
    -  Check how data indexing and partitioning impacts the disk I/O patterns.
    - Understand the max data access API call for your cloud file storage solution. Remember, reading one file is not an API call; it will require multiple API calls such as directory lookup, check file meta-data, read each chunk of the files. 
- __Log exploding:__  
    In most systems, there will be trace logs to help system administrators monitor and debug the system's operation status. Pay attention to how these logs are being triggered and the file size growth speed of logs. In a large system, the log files could increase from KB to multiple GBs in a few hours and drag the system's performance. If the system uses checkpoint files, logging API calls, they also need to be checked. 


### #7 End-to-End Testing 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/test-4092025_1920.jpg)
<br>
Image by <a href="https://pixabay.com/users/geralt-9301/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4092025">Gerd Altmann</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4092025">Pixabay</a>

Testing is a vast topic. It will/should occupy at least nearly half of the developing team's resources. Since projects always don't have enough time to do perfect testing, we can only try to spend our limited time on the parts that most likely go wrong and is critical to the system. _In data project, we realized the traditional unit test and system integration are not enough to verify the solution quality_.  

In our projects, dev teams need to implement the code logic that retrieves, move, clean up,  process, and store data. Traditional software practices focus on unit testing and integration testing. They can verify if the code logic is correctly and thoughtfully implemented. But to ensure each developed components meet go-production quality, besides ensuring the code is implemented right, another equally important part is to __test and make sure the infrastructure is also implemented right__. For example, we want to check:
- If the infrastructure can scale adequately to handle different data volume
- If the computing resources are efficiently utilized for diverse workload
- If all the tasks can be finished after peak hours
- If we calculate the required system resource and make projections in the right way  
- If the system boundary/limitation is what we understand 

To know the answer to these questions, We adopted a few techniques: 

####  <u>Mimic Real World and Edge Case Workload</u>: 
The simplest way is to deploy the whole environment and run end-to-end testing using workloads that mimic the production scenario or scenario we want to test. Unlike website load testing, this part usually takes efforts to customize and can be a small standalone project on its own. To build the workload simulation solution, Kubernetes is a popular choice. It's flexible and cost-effective. 


####  <u>Leverage Gherkin language and BDD Tools</u>: 

"_Behavior-driven development (or BDD) is an agile software development technique that encourages collaboration between developers, QA and non-technical or business participants in a software project_" - [behave doc](https://behave.readthedocs.io/en/stable/philosophy.html)

We want to know how the system performs in different workloads and found it easier to describe the workload criteria and expected results in a structured pattern. BDD tools like [Cucumber](https://en.wikipedia.org/wiki/Cucumber_(software)), [behave](https://behave.readthedocs.io), [behat](https://docs.behat.org/en/latest/) and their script language [Gherkin](https://behave.readthedocs.io/en/stable/philosophy.html#the-gherkin-language) (a business Readable language for behavior descriptions) match such requirements. Business users can easily change the workload patterns and automatically run the test to verify if they meet the performance requirements. 

####  <u>Provision Load Testing Environment using CD pipeline parameters </u>: 
The other practice is integrating the load testing environment into the CI/CD pipeline and using a parameter to control the kind of environment the CD pipeline will provide. Due to the tight integration with infrastructure and relying on some cloud PASS services, the CI/CD pipeline will become more complicated and requires careful planning. Plan and Integrate the provision of load testing environments in the CD pipeline can reduce the CD pipeline number and make it easier to keep the environment consistent. 


####  <u>Centralize Performance Metrics and Perpare Dashboard</u>: 
Since there are several components in the system, each part has different performance metrics, with an entirely different measuring unit and business meaning. It's easier to have a centralized place to query and view all these metrics. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/dashboard-all.png)


### #8 Measure your Data Drift 

Data Drift is a big issue in machine learning systems. It means the profile and distribution of the data changed, and the previous trained/tested model might not be able to predict the result on the new data with the same accuracy. But not only impacts machine learning prediction, data drift will also affect the data processing pipeline. 

When we design and test the data processing pipeline, it relies on understanding data to optimize the operation. Especially for data aggregation or classification operation, because most data are not evenly distributed, the unbalanced data (skew data) will introduce an uneven workload that will cause low utilization of system resources. We will need to make some adjustments to shuffle data evenly. If data drift, it will break the optimization and cause an even worse unbalanced workload and reduce system efficiency.  

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/perf-fine-tune.png)

Consider adding some metrics to monitor data drift if some components in the system are sensitive and will be impacted by data characteristic change. For example, we watched some Spark tasks' running time because they are sensitive to data distribution in our scenario. The other common practice is checking the statistical distribution of recent data regularly. 


### #9 Continuous learning with Chaos Engineering

_"Chaos engineering is the discipline of experimenting on a software system in production in order to build confidence in the system‘s capability to withstand turbulent and unexpected conditions"_ - <u>[Principles of Chaos Engineering](https://principlesofchaos.org)</u>


Chaos Engineering is initially proposed by Netflix in 2010 because they wanted a better way to test and improve their distributed microservice system's reliability. They found that finding faults in a distributed system goes beyond the capability of standard application testing. Instead of testing the single failure point, chaos engineering aims to generate new knowledge about inter-connections and impacts between components in the system. 

![HowChaosEngineeringWork](https://phoenixnap.com/blog/wp-content/uploads/2020/10/how-chaos-engineering-works.jpg)
<br>
_image credit: [Chaos Engineering: How it Works, Principles, Benefits, & Tools](https://phoenixnap.com/blog/chaos-engineering)._

Improve system resilience is a continuous learning journey. Though we have mentioned several key learnings, and some of them learned from the production system issues, there are still many parts that can be explored and enhanced. We can systematically learn the system behavior through chaos engineering and understand how it responds to production turbulence.  For a large data project, it is definitely worth including the chaos engineering tools and practices during system development and long-term system operation.  

---


This post covered topics learned from several data projects I participated  in 2020. I will use it to remind me of some key points that should be considered and prepared for future data projects. Hoping some of them can also benefit your data projects. 

Goodbye 2020 and Welcome 2021. 





## References
- [Best Practices for Building Robust Data Platform with Apache Spark and Delta](https://databricks.com/session_na20/best-practices-for-building-robust-data-platform-with-apache-spark-and-delta)
- [Analytics Challenges — Big Data Management & Governance](https://medium.com/analytics-vidhya/big-data-management-governance-bce5f72821c1)
- [An overview of ETL and ELT architecture](https://www.sqlshack.com/an-overview-of-etl-and-elt-architecture/)
- [ETL Vs ELT: The Difference Is In The How](https://blog.panoply.io/etl-vs-elt-the-difference-is-in-the-how)
- [Intro to Chaos Engineering (Video)](https://www.youtube.com/watch?v=qHykK5pFRW4)
- [Evolutionary Database Design](https://martinfowler.com/articles/evodb.html#AllDatabaseArtifactsAreVersionControlledWithApplicationCode)
-[Chaos Engineering: How it Works, Principles, Benefits, & Tools
](https://phoenixnap.com/blog/chaos-engineering)
