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

###  _Lessons I learned and want to keep in mind in 2021+_


![img](./img-2020-Data/new-year-4768119_1280.jpg)
Image by <a href="https://pixabay.com/users/syaibatulhamdi-13452116/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4768119">Syaibatul Hamdi</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4768119">Pixabay</a>


2020 has sunsetted. Looking back over it, I had three projects that needed to process multiple terabytes of data regularly (one of them needs to process hundreds of terabytes of data every day) and one project that wanted to apply good [MLOps](https://blogs.nvidia.com/blog/2020/09/03/what-is-mlops/) practices on GitHub. Some projects have gone production and are running on data centers in different continents to serve worldwide customers. 

In this blog post, I want to __summarize the key practices learned from these projects__ which I want to remember and keep in mind in 2021. I want to read them each time before a new data project kicks off and make a better plan. 


_Note! Even for similar workflow, small/medium/large scale projects require different architecture and plan differently. This post focus on large scale data project._

### #1 _Prepare for Data Schema Change_

This is a hard fact, but __real-world data always evolve and data schema will change overtime__. Years ago, it was a common practice that we spent a lot of time negotiating the data schema/contract with data providers and tried to define the best schema that can live forever. But we are in a world that data is exponentially growing; more and more data comes from non-traditional OLTP systems. In some projects (especially projects with over several gigabytes of data every day), it's not realistic to assume we can have a stable data format/schema over time. The requests to support varying data schema keeps pop up in system requirements in my 2020 data projects. 

There are several techniques that can help to handle data schema changes, such as:
-  [Bronze, Silver, and Gold architecture](https://docs.microsoft.com/en-us/learn/modules/describe-azure-databricks-delta-lake-architecture/2-describe-bronze-silver-gold-architecture)
<br>
![Bronze, Silver, and Gold](https://databricks.com/wp-content/uploads/2019/08/Delta-Lake-Multi-Hop-Architecture-Bronze.png)
- [Extract, load, transform](https://en.wikipedia.org/wiki/Extract,_load,_transform#Common_storage_options) <br>
![Extract, load, transform](https://blog.panoply.io/hubfs/Blog_images/http%20images%20to%20https/etl-processes-described---x----1000-420x---%20(1).jpg)
- __Using data format that supports dynamic schema__ such as JSON, Parquet, and  __leveraging capabilities of NoSQL solutions__ such as MongoDB, COSMOS DB, Elastic Search, Azure Data Explorer, Delta Lake, Big Query, DynamoDB...
- __Schema versioning__. You can adopt some techniques like [Temporal Patterns](https://martinfowler.com/eaaDev/timeNarrative.html), [Artifacts are version controlled with application code](https://martinfowler.com/articles/evodb.html#AllDatabaseArtifactsAreVersionControlledWithApplicationCode), [Data Mapper](https://docs.microsoft.com/en-us/archive/msdn-magazine/2009/april/design-patterns-for-data-persistence), [Document Versioning Pattern](https://www.mongodb.com/blog/post/building-with-patterns-the-document-versioning-pattern) to implement data versioning.

_Most of the techniques share similar principles and can complement each other in handling data schema change_. The key point is paying some extra attention to it. If someone assumes that they will have a stable schema, it's better to discuss more on this assumption and validate again. The requirement to handle schema change will impact the foundation of the whole system design. 


### #2 _Design for Query & Charting_

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/financial-2860753_1280.jpg) <br>
Image by <a href="https://pixabay.com/users/6689062-6689062/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2860753">David Schwarzenberg</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=2860753">Pixabay</a>

_A good data system should at least fulfill the team's basic query and charting requirements_. It is very common that  __data collectors__ (such as data engineers, people who collect and ingest the data) and __data consumers__ (BI reporter, data analyzer, ML researcher.. ) _are from different teams, and they don't understand each others' languages in projects_.  Considering the skill sets required for different roles and different groups also have different business goals, this is very normal. However, bringing the promised benefits of the whole data system requires deep engagement from both parties. A couple of times, we saw that technologies are chosen and built based on one side's requirements, but the solution poorly meets the other side's needs.  

It is usually easier to figure out the data collector requirements than data consumers' needs because data collectors' requirements are mostly related to the [3Vs of Big Data](https://www.zdnet.com/article/volume-velocity-and-variety-understanding-the-three-vs-of-big-data/) - __Volume, Variant, and Velocity__.  And we can quantify them with specific numbers in the requirements specification. The data consumers' requirements often depend on the business context, problem domain, and size, skill set of the data consumers team; it requires some more digging to figure out.  

Here are some common factors to consider: 
- __Visualization Tools__:  Some teams may have strong Excel skills; they are fluent in using PowerPivot and Excel functions to generate in-depth interactive reports. Some teams like the advanced graphical presentation provided by dashboard/reporting tools such as PowerBI, Tableau. Some groups like the powerful programmability in Jupyter Notebook or Shinny. Also, most of the data platforms will provide their build-in tools to visualize data. For example, Azure Data Explore provides very handy and powerful charting capabilities for historical log data and time-series data analysis.  
![AzureDataExplorer-Chart](https://docs.microsoft.com/en-us/azure/data-explorer/media/time-series-analysis/time-series-operations.png)
<br>
_Residual time series chart in Azure Data Explorer_ 

- __Data Post Processing/Query Tools and Languages__: Data consumers will need to further process data to fulfill their analytic requirements. Languages like SQL, R, and Python (ok, you can also add Julia) are very common and powerful ones that data analyzers use to further "[__massage data__](https://stackoverflow.com/questions/577892/what-does-data-massage-mean)". But each analysis domain has it's own unique data processing pattern. Based on different design purposes,  _different data platforms also provides their own data query syntax and query engine to support and simplify data post-processing for specific domains_. Understanding each platform's strength and choosing the best-fit one is critical to the project's success. For example, it's easy to implement complex text filtering and parsing in Elastic Search, but it's hard to implement complex data join and aggregation in it. The other example is Azure Data Explorer, which focuses on historical log data analytics,  provides powerful [Kusto Query Lanaguage](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/) to simply log parsing, aggregation, data join, and semi-structure data processing. It is one of my favorite query languages.  

- __Performance goal and Limitation to retrieve insights from "Big Data"__: Processing large volumes of data is not easy; different solution applies different strategies to achieve their performance goal. They use different index algorithms, in-memory computing, cluster resource allocation, file storage structure, data compression type, etc., to archive their performance goal. These strategies will also come with some limitations like "limitation of concurrent users", "unbalance computing resource allocation", "required huge memory caches", "data paging support"...etc. Review the pro and cons of different solutions and test them in the project's main data query scenarios. 

Both data collectors and data consumers are critical stakeholders of a data project. In the solution design phase, it's helpful to have deep engagement with both two teams. Spending a little more time on how the data query pattern looks like and considering the data analysis tools suitable for users, you will have a bright outlook for the system.   

### #3 Balance the workload of each resources  

![startwar-balance](https://www.denofgeek.com/wp-content/uploads/2017/04/obi_wan.jpg)
<br>
_image credit: [Rob Leane -Den of Geek UK](https://www.denofgeek.com/movies/star-wars-the-last-jedi-and-the-balance-of-the-force/)._

When we worked on small or medium-sized data projects, we implemented the solution using one or two key technologies such as Databricks or Azure Data Explorer. These technologies come with some default capabilities to ingest, manage, and query data. We focused on providing proper configuration on these platforms. 

However, when we worked on large scale projects, instead of relying on built-in capabilities,  we found we need to __re-architect the system to further distribute the computation workloads to different components in the system__. We need to spend more time thinking about how to __make the data more "swallowable" for the data platform__. "Swallowable" could mean things like reducing the data fragmentation, easier to build index, remove dirty data, remove duplicated data... etc. 

The workload distribution process is also a process to find the balance of each component. Here are some points that might help:
- Size of your batch and how it impacts latency and memory consumption
- Choose different data formats, consider it's the impact to data serialize/un-serialize time and disk I/O
- Decide where to filter, split, aggregate data so the next component can reduce workload and simplify management and configuration complexity.
- It is also related to the maximum scale-out/in capabilities of each component. All the data input and output velocity of these components can then be within its connected components' bandwidth. 

For a large scale project, the right solution will be the tasks are well distributed among all critical components, and the workload is balanced.  So the designed system can have room growth its capacity, handle peak-time traffic, and prepare for future business growth. 

### #4 Wow, there are duplicated data

This is another common issue when we process a large amount of data. We could have duplicated data for dozens of reasons. It could be the data source sent twice because of a network issue; it could be some parts of the data processing pipeline partially failed and the system is trying to resume from the latest checkpoint; it could be the underneath message service only guarantee at-least-once, etc. 

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/window-941625_1280.jpg)
<br>
Image by <a href="https://pixabay.com/users/martinharry-1411929/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=941625">Martin Pyško</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=941625">Pixabay</a>

We need to evaluate the business impact and the cost we want to pay for mitigating the issue. Common strategies to handle duplicated data are:

- __Ingestion Time Check__: We can have ingestion time check by adding data processing logs or checkpoint files. Then all the ingestion operations need to verify the log/checkpoint file before process the data. 
- __Settling Basin Filter__: In this strategy, we store data in a temporary storage/table. Then, we compared the data with recently ingested data to make sure they are not duplicated data. 
- __Query Time Check__: In some systems, a few duplicated data can be ignored in most use cases (e.g., website user login log for analyzing user demographic/ geographical distribution), and only a few of its use cases require duplicated data check. For such a scenario, we can check duplicated data at query time for only required use cases; then we can reduce the system cost for de-duplicated data check.   
- __Under Some Threshold, Ignore It__: It's the best solution if the business context allows it. De-duplication operations usually occupy a certain amount of system resources and impact system performance. In a large data processing system, it's very costly to implement it. It will be helpful if we can ignore it when it doesn't impact the business goad.     

Besides the above strategies to handle duplicated data, one other point is understanding where the duplicated data comes from. Which part of the system or infrastructure services provides at-least-once and which part can support exactly-once. If the system use checkpoint to resume the failed operation, try to understand how it works. 


### #5 Understanding of Core Technologies

![img]({{ site.url }}{{ site.baseurl }}/assets/img/2021-01-01-Reflection-of-my-2020-Data-Projects-part1/book-4126483_1280.jpg)
<br>
Image by <a href="https://pixabay.com/users/geralt-9301/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4126483">Gerd Altmann</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=4126483">Pixabay</a>

This point should be needless to say, but I add it here not just because it's fundamental but also it could take consider mount of time to understand the design philosophy and available configuration parameters of the different data platforms. Many settings could impact each other; we should keep in mind that planning enough time to learn, test, and verify the best configuration setting is required.  

For example, in the core technologies I used in these projects, here are some key configurations that need to check, calculate, and test:

- __Databricks__
    - Number of Cores
    - Number of Memory
    - Duration of Job, Stage, Tasks
    - Fair scheduler Pool
    - Structured Steaming : max-files, max-size per trigger
    - Shuffle Partitions
    - Shuffle Spill (memory)
- __Azure Data Explorer__
    - Ingestion
        - MaximumBatchingTimeSpan
        - MaximumNumberOfItems
        - MaximumRawDataSizeMB
    - Capacity Policy
        - Ingestion capacity
        - Extents merge capacity
        - Extents purge rebuild capacity
        - Export capacity
        - Extents partition capacity
        - Materialized views capacity policy
- __Azure Functions (Python)__
    - FUNCTIONS_WORKER_PROCESS_COUNT
    - Max instances when Functions scale out
    - Queue Trigger
        - batchSize
        - newBatchThreshold
    - HTTP Trigger  
        - maxConcurrentRequests



___[>> Part-II](https://herman-wu.github.io/blogs/azure%20data%20explorer/kusto/data/azure/data%20pipeline/reflections/2021/01/01/Reflection-of-my-2020-Data-Projects-part2.html)___ 




## References
- [Best Practices for Building Robust Data Platform with Apache Spark and Delta](https://databricks.com/session_na20/best-practices-for-building-robust-data-platform-with-apache-spark-and-delta)
- [Analytics Challenges — Big Data Management & Governance](https://medium.com/analytics-vidhya/big-data-management-governance-bce5f72821c1)
- [An overview of ETL and ELT architecture](https://www.sqlshack.com/an-overview-of-etl-and-elt-architecture/)
- [ETL Vs ELT: The Difference Is In The How](https://blog.panoply.io/etl-vs-elt-the-difference-is-in-the-how)
- [Intro to Chaos Engineering (Video)](https://www.youtube.com/watch?v=qHykK5pFRW4)
- [Evolutionary Database Design](https://martinfowler.com/articles/evodb.html#AllDatabaseArtifactsAreVersionControlledWithApplicationCode)
-[Chaos Engineering: How it Works, Principles, Benefits, & Tools
](https://phoenixnap.com/blog/chaos-engineering)
