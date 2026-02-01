---
title: Prioritizing Databases by Business Impact
date: 2026-02-01 17:35:00 +0100
categories: ['product','tech']
tags: ['database','categorization','tech']
math: true
---

<a data-flickr-embed="true" href="https://www.flickr.com/photos/97713098@N06/55071599932/in/dateposted-public/" title="Layers"><img src="https://live.staticflickr.com/65535/55071599932_4c1b167592_c.jpg" width="332" height="500" alt="Layers"/></a>

## Introduction

&emsp;Recently I had an interesting categorization problem in front of me. The company I work for has certain peak months when the usage of its application skyrockets. During this time, the applications handle more than 4 times their usual number of requests. Also, during this season the application process large number of database intensive reporting operations. To prepare for the peak season, I wanted to come up with an approach for optimizing the infrastructure of the platform, specifically the databases. During previous peak season we observed our databases struggling a high influx of requests and heavy (IO heavy) user queries.

&emsp;The first thing I wanted to do was to categorize the databases based on impact on business if they have performance issues. Among the tens of databases our application connects to, I wanted to specifically know which databases we need to focus on. I worked on a categorization logic based on extracted business attributes for databases and in this post, I will try to explain my approach. 

## The Sample Application

&emsp;Let’s take an example application - an Inventory Management Platform. Companies (Tenants) can create their company space on the platform where they can create one or many shops and store inventory information about the shop. There can be companies with few shops (local shops) or companies with many shops (chain store). The shops in turn can be big shops like a supermarket (more inventory items) or small shops like a local bakery shop (less inventory items). Below is a sample representation of the schema structure.

![](/assets/img/posts/db_schema_design.png)

Every month inventory information is updated for shops by running background jobs. Annually January is a peak season when inventory items of all shops are carried over (updated) from the previous year to the current year. Along with this, a lot of reports for auding purposes are generated which in turn generate database intensive queries. Each database can hold hundreds of logically separate tenants. Most of the databases have a mixture of big, medium, and small tenants based on the number of shops and inventory items. We have a single web server that redirects queries to the appropriate database based on an internal mapping. In short, we have is a [Sharding pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding) database architecture.

<img title="" src="https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/media/storage-data/sharding.png" alt="" data-align="center">

## Determining Business Impact Drivers

&emsp;Even though all the databases are important, we still need to know which databases have high business impact so that appropriate scaling measures can be taken in case of performance issues. What we need is a way to categorize all the databases into medallions like Gold, Silver and Bronze tiers to determine their business impact. 

&emsp;Below are some of the business attributes for a database that can be taken into for the prioritization

1. Amount of data (`db_load`) - This represents the total number of inventory items that a database holds. This value can be considered as a proxy for the number of transactions the database needs to make frequently and specifically in January. Higher number of inventory items would mean more load on the system, which can lead to performance issues.

2. Number of large shops (`large_shops_count`) - This is a derived attribute which signifies how many *large shops* are present in a database. For example, a shop with more than 5,000 inventory items can be categorized as a large shop and less than 100 items as a small shop. SQL queries generated for a small shop with 100 inventory items will be far less expensive compared to a large shop with 5,000 inventory items. If a database has more large shops, there is good chance of [noisy neighbor](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/noisy-neighbor/noisy-neighbor) situation happening where high transactions of one company (especially during peak month) can negatively affect another company's performance.

3. Count of tenants by size (`tenant_small_count`, `tenant_medium_count`, `tenant_large_count`) - It’s a derived attribute which can be calculated by assigning companies to buckets based on the total number of inventory items processed across all their shops. 
   
&emsp;For example, if a company processes less than 500 inventory items across all shops, then it is assigned to the small company bucket. If a company processes more than 2500 inventory items, it is assigned to the large company bucket. Based on these threshold values, we can find the _count_ of small, medium, and large companies in a database.

> These attributes can change based on database design, application architecture, and business requirements of the platform. The key is to identify business attributes that can affect a database's performance. 

## Calculating the Business Impact Score

&emsp;How do we use the business attributes to categorize the databases? For this we need to come up with a single score for each database - the Business Impact Score (BCS) - based on which we can create a prioritized list of the databases. For calculating the Business Impact Score, we will follow the below steps.

1. Collect business data

2. Normalize the data

3. Defining attribute weights

4. Calculate Business Impact Score

**1. Collect Business data**

&emsp;The first step is to collect data about the business attributes for all the databases. We can get such information by directly querying the databases or from in-house Data teams. Below is the sample data for a few databases:

| db_name      | db_load | large_shops_count | tenant_small_count | tenant_medium_count | tenant_large_count |
| ------------ | ------- | :---------------: | :----------------: | :-----------------: | :----------------: |
| SQLDB01-Live | 165,127 |         6         |        919         |         156         |         7          |
| SQLDB02-Live | 103,417 |         4         |         5          |          9          |         14         |
| SQLDB03-Live | 130,365 |         2         |         80         |         66          |         5          |
| SQLDB04-Live | 138,443 |         4         |        341         |         124         |         10         |
| SQLDB05-Live | 28,106  |         0         |        128         |         17          |         5          |

&emsp;As one can clearly see in this example, the data is not spread uniformly across the databases. This is a more realistic view of an application which has grown organically over the years with data being split across multiple databases when the load becomes too much for one to manage.

**2. Normalize the data**

&emsp;Normalization is a statistical concept that means adjusting values measured on different scales to a common scale [1]. As the scales of various measures like `db_load` and `large_shop_count` can be quite different, we need to normalize them to a single scale. We can use the basic [Min-max normalization][1] to convert them into a number between 0 and 1. Below is the formula:

$$
x_{norm} = {x - min(x) \over max(x)- min(x)}
$$

For example:

$$
db-load_{138,443} = {138,443 - 103,417 \over 165,127-103,417} = 0.56
$$

&emsp;We normalize all the attribute values to to values between 0 and 1 to bring them on the same scale. Below is the normalized data:

| db_name      | db_load | large_shops_count | tenant_small_count | tenant_medium_count | tenant_large_count |
| ------------ | :-----: | :---------------: | :----------------: | :-----------------: | :----------------: |
| SQLDB01-Live |    1    |         1         |         1          |          1          |        0.22        |
| SQLDB02-Live |  0.37   |       0.67        |         0          |          0          |         1          |
| SQLDB03-Live |  0.64   |       0.33        |        0.08        |        0.39         |         0          |
| SQLDB04-Live |  0.73   |       0.67        |        0.37        |        0.78         |        0.56        |
| SQLDB05-Live |    0    |         0         |        0.14        |        0.05         |         0          |

**3. Defining attribute weights**

&emsp;While we have the normalized values for all the business attributes, not all these values have same contribution to application's performance or impact on business. Some affect them more than others. We need to define weights to these attributes based on our understanding of the system. We need to answer questions like -

- How does our database perform with expensive queries run? 

- How do our databases perform for high transactions?  

- How have the databases performed in the past - what were the issues? 

- Is this database historically prone to performance issues? 

- How confident are we that all our application queries are optimized for peak load? 

&emsp;Based on the answers to these questions, we can assign weights for the business attributes. Below are a few weight scenarios:

| Business Attributes | Balanced weights | Transaction focused weights | Reporting focused weights |
| ------------------- | :--------------: | :-------------------------: | :-----------------------: |
| db_load             |       0.20       |            0.55             |           0.25            |
| large_shops_count   |       0.20       |            0.15             |           0.50            |
| tenant_small_count  |       0.20       |            0.10             |           0.03            |
| tenant_medium_count |       0.20       |            0.10             |           0.07            |
| tenant_large_count  |       0.20       |            0.20             |           0.15            |

&emsp;Balanced strategy is a neutral baseline and can be used when we do not want any one attribute to overshadow the other. In case we know the databases to struggle with high number of queries transactions, we can go for Transaction focused weights where we assign more weights for inventory items in the database.

&emsp;During the peak months, companies generate heavy reports like transactions for the previous year or inventory projection for the next year. Such reports generate expensive queries and when multiple tenants generate similar reports, they can affect each other’s performance. Also, a database having multiple large shops will fare far worse in terms of performance bottlenecks because of concurrency than a database with more small shops. For such scenarios, Reporting focused weights can be used where more weight is provided to large shops.

&emsp;Weight strategies can be based on other attributes and scenarios as well. We can have a customer complaint attribute that represents the number of complaints received from a particular company. Weights can then be distributed accordingly. The defined weights too need to be experimented with and adjusted iteratively.

**4. Calculate Business Impact Score**

&emsp;Finally, we calculate the Business Impact Score by multiplying the weights with the normalized attribute values and adding them all together. For example:

$$
BIS_{SQLDB04-Live} = 0.73*0.25 + 0.67*0.50 + 0.37*0.03 + 0.78*0.07 + 0.56*0.15 = 0.66
$$

&emsp;We calculate the scores for all the other databases. Once done, we can assign medallions to the databases based on this score - Gold (top 75th percentile), Silver (top 50th percentile) and Bronze (remaining) tiers. The database list will finally look like the below: 

| db_name      | bis_score | medallion |
| ------------ | --------- | :-------: |
| SQLDB01-Live | 0.88      |   Gold    |
| SQLDB04-Live | 0.66      |  Silver   |
| SQLDB02-Live | 0.57      |  Bronze   |
| SQLDB03-Live | 0.35      |  Bronze   |
| SQLDB05-Live | 0.01      |  Bronze   |

## Conclusion

&emsp;This process of categorizing databases may look a bit long, but this process can be automated. The business attribute data should be periodically ingested from source using an automated job and the calculations should performed behind the scenes. The databases and their medallions should be the final overview that is displayed on a reporting platform like Excel, Power BI or a custom solution. 

&emsp;I implemented a similar solution, and I have been satisfied with it as of now. I feel more confident about my understanding of the databases we have. When reviewing performance issues, we know where to start investigating. Also, having real-time data about business attributes helps us decide how much to scale.

&emsp;In a future post, I will write more about how the categorization can be used to help predict performance issues and perform pro-active scaling for the peaks seasons. Until then, thanks for reading!😄

[1]: https://en.wikipedia.org/wiki/Normalization_(statistics)