# Reflection of my 2020 Data Projects (Part II)
### - _Lessons I learned and want to keep in mind in 2021+_

___<< Part-I___ 

### #6 Every small problem can be BIG

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/quarantine-4981010_640.jpg)
Image by <a href="https://pixabay.com/users/evgenit-4930349/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4981010">Evgeni Tcherkasski</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4981010">Pixabay</a>

When we were developing solutions for different projects, we normally start with implementing the functional logic that fulfill the design and we used small volumes of data for validate the functionality. Besides these functional development tasks,  there are a few things that deserve our additional attention because they  could impact the stability of the system when the system is running in full scale and handles production workload during peak time.

- __Cloud is a live infrastructure__: Cloud is an environment that keeps evolving over time. All cloud vendors are working hard to improve existing services and introduce new capablities in their platform. It also means the services on the cloud will be upgraded every couple months. Though cloud vendors/operators will try to prevent service interruptions and sometime reduce the interruption time to less then one second, it still impacts heavy load systems/componets that happen to have hundreds of data processing tasks at the moment. Some design pattern like 
[Claim-Check Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/claim-check) or [Queue-Based Load Leveling pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) could help mitigate the impact.  
- __Network Transient Failure__: Network is the vessel of modern system, it connects every components and transmits huge information payload in high frequency. Many services and SDKs already have build-in mechanisms to handle network transient failure, but for system that has heavy network traffice we should review how it works in the architecture and consider the needs to adding additional mechanism. 
The most common practice to handle network failure is to resend and retry the network operation. What need to be pay attention is it's better to implement a "smarter" retry strategies of system that has huge workload and heavy network operation. Some libraries like Polly (.net core) , Tenacity (python) can help us implement the retry action using advanced algorithm. 
- __Optimize Disk I/O__: 
    - Understand the distributed file storage system underneath your data solution.For example, check Sharding for Monogo DB, Extents for Azure Data Explorer, Partitioning (bucketing) for Databricks Delta table.
    -  Check how data indexing and partitioning impacts the disk I/O patterns.
    - Understand the max data access API call for your cloud file storage solution. Remember, reading one file is not an API call, it will require multiple API calls such as directory lookup, check file meta-data, read each chunk of the files. 
- __Log exploding:__  
    In most systems there will have trace log to help system administrators to monitor and debug the operation status of the system. Pay attention to how these log are been triggered and the log file size growth speed . In large system the log files could increase from KB to multiple GBs in a few hours and drag the performance of the system. Also if the system uses check point files, logging API call, they also need to be checked. 


### #7 End-to-End Testing 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/test-4092025_1920.jpg)
Image by <a href="https://pixabay.com/users/geralt-9301/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4092025">Gerd Altmann</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4092025">Pixabay</a>

Testing is a huge topic and it will/should occupy at least nearly half amount of the developing team's resources. Since projects always don't have enough time to do perfect testing, we can only try to spend our limited time on the parts that most likely go wrong and is critical to the system. _In data project, we realized tranditional unit test and system integration is not enough to verify the solution quality_.  

In our projects dev team needs to implement the code logic that retrive, move, clean up,  process and store data. To ensure each developed components meets the go-production quality, tranditonal software practices focus on unit testing and integration testing. They can verify if the code logic is correctly and thoughfully implemented. But besides ensure the code is implemented right, another equally important part is to __test and make sure the infrastructure is implement right__. For example, we want to check:
- if the infrastructure can scale properly to handle different data volume
- if the computing resource are efficeintly utilized for different workload
- if all the tasks can be finished after peak hours
- if we caculate the required system resource and make projection in a righ way  
- if the system boudary/limitation is what we understand 

To know the answer of these questions, We adopted a few techniques: 

####  <u>Mimic Real World and Edge Case Workload</u>: 
The simplest way is to deploy the whole environment, the run end-to-end testing using workload that mimic production scenario or scenario we want to test. Unlike web site load testing, this part usually takes efforts to customize and can be an small standalone project by it's own. To build the workload simulatation solution, Kubernetics is a popular choice, it's felixable and cost effective. 


####  <u>Leverage Gherkin language and BDD Tools</u>: 

"_Behavior-driven development (or BDD) is an agile software development technique that encourages collaboration between developers, QA and non-technical or business participants in a software project_" - [behave doc](https://behave.readthedocs.io/en/stable/philosophy.html)

We want to know how the system performance in different workload, and found it's easier to describe the workload criteria and expected result in a structure pattern. BDD tools like [Cucumber](https://en.wikipedia.org/wiki/Cucumber_(software)), [behave](https://behave.readthedocs.io), [behat](https://docs.behat.org/en/latest/) and their script lanaguer [Gherkin](https://behave.readthedocs.io/en/stable/philosophy.html#the-gherkin-language) (a business Readable language for behavior descriptions) match such requirements. Bussiness users can easily change the workload patterns and run the test automatically to verify if the system meet the performance requirments. 

####  <u>Provision Load Testing Environment using CD pipeline parameters </u>: 
The other pratice is to integrated load testing environment into CI/CD pipeline, and use a parameter to control the kind of environment the CD pipeline will provision. Due to the tightly integration with infrastructure and relying on features of some cloud PASS services, CI/CD pipeline will become more complicated and requires careful planning. Plan and Integrate the provision of  load testing environments in CD pipeline can reduce the number of CD pipeline and easier to keep the environment consistant. 


####  <u>Centralize Performance Metrics and Perpare Dashboard</u>: 
Since there are serveral components in the system, each components have different performance metrics which come with quite different measure unit and meaning, it's easier to have a centralize place to query and view all these metrics. 
![test](./img-2020-Data/dashboard-all.png)





### #8 Measure your Data Drift 

Data Drift is a big issue in machine learning system, it means the profile and distribution of the data changed and the previous trained/tested model might not be able to predict result on the new data with same accuracy. But Data Drift not only impacts machine learning prediction, data drift will also impact the data processing pipeline. 

When the data processing pipeline are design and tested, it also rely on the understanding of data to optimize the operation. Especially for data aggregation or classification operation, because by nature most data are not evenly distributed, the unbalance data (shew data) will introduce uneven workload that will cause low utilization of system resources. We will need to do some adjustment in order to shuffle data evenly. If data drift, it will break the optimization and could cause even worse unbanlance workload and reduce system efficeincy.  

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/perf-fine-tune.png)

Considering to add some metrics to monitor data drift if some components in the system is sensitive and will be impacted by data characteristic change. For example, we monitored the running time distribution of some Spark tasks because they are sensitive to data distribution in our scenario. The other common practice is regualrly check statistic distribution of recent data. 


### #9 Continuous learning with Chaos Engineering

_"Chaos engineering is the discipline of experimenting on a software system in production in order to build confidence in the system‘s capability to withstand turbulent and unexpected conditions"_ - <u>[Principles of Chaos Engineering](https://principlesofchaos.org)</u>


Chaos Engineering is initially proposed by Netflix in 2010 because they wanted a better way to test and improve the reliability of their distributed microservice system. They found the process to finding faults in a distributed system goes beyond the capability of standard application testing. In stead of testing single failure point, the goal of chaos engineering is to generate new knowledge about inter-connections and impacts between components in the system. 

![HowChaosEngineeringWork](https://phoenixnap.com/blog/wp-content/uploads/2020/10/how-chaos-engineering-works.jpg)
_image credit: [Chaos Engineering: How it Works, Principles, Benefits, & Tools](https://phoenixnap.com/blog/chaos-engineering)._

Though we have mentioned several key learnings and some of them learned from issues of production system, improve system resilience is a continuous learning journey and there are still a lot parts can be explored and enhanced. Through chaos engineering we can learn the system  behavior systematically and understand how it responses to the turbulence in production.  For large data project, it definite worth to include the chaos engineering tools and practices not only during system developing but also for long terms system operation. 

---


In this post I covered topics leaned from several data project I participated in 2020. It's my 2020 project reflection to reminding myself some key points I should plan and prepare for future data projects. Hoping some of them can also benefit your data projects. 

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
