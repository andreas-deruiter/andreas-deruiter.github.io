# Quick Kusto intro for software developers

Kusto, also known as Azure Data Explorer or ADX, is best known for its ability to query and analyze logs. That sounds a bit boring (as does the 'Azure Data Explorer' name), but I found it to be one of those hidden gems that has become one of my favorite Microsoft technologies, right up there with VS Code, Typescript and Excel. Seriously!

At its core, Kusto is just another database system. How does it work, and how does it fundamentally differ from other databases? What makes it so awesome for certain workloads?

# Kusto as a cloud-based database

Kusto is essentially a cloud-based database hosted in Azure. In that sense, it's like Cosmos DB or Azure SQL Server.

You can go to the Azure portal, and with a few clicks create a new Kusto cluster and a database. A Kusto cluster is the equivalent of an Azure SQL Server. It's called a cluster because Kusto is from the ground up designed to run in parallel on many VMs, which are called nodes. Because it's a PaaS service, you don't need to manage those VMs yourself. A Kusto administrator can, however, easily scale up by adding nodes to the cluster.

In case you already use Application Insights or Log Analytics, you already have a Kusto cluster you can work with! These are Microsoft owned services that run on top of Kusto, and that you can query the same way that you can with any Kusto cluster. Of course, you don't have access to the management commands, but you can run queries the same way as you would on your own cluster.

# What makes Kusto different from other databases

Every database system needs to make trade-offs in its design. Kusto is very good at rapidly ingesting and querying huge amounts of both structured and unstructured data. It's primary use case is to analyze logging data. That data needs to be ingested into the Kusto database before it can be queried. This often happens in real-time, something known as streaming ingestion.

There are also things which Kusto does not do very well, or at all. Most importantly, you can't update data as it's an 'append-only' database. (There is an indirect way of updating data as you can read in [How to update data in Kusto](https://andreas-deruiter.github.io/2022/07/12/how-to-update-data-in-kusto.html)).

Kusto also doesn't support transactions. Knowing that you can't update data, which shouldn't come as a surprise.

You can remove data, although that can sometimes require more steps that you would like. See [There are many ways to remove data in Kusto. When to use which?](https://andreas-deruiter.github.io/2022/07/12/there-are-many-ways-to-delete-data-in-kusto.html).

These tradeoff means that Kusto doesn't compete with the traditional SQL or non-SQL databases for many typical workloads. Instead, you'd use it in scenarios where you might otherwise use a technology like Apache Spark. As with Spark, Kusto's ability to handle huge amounts of data largely stems from the fact that the work is spread across many nodes (computers) in the backend.

A secret sauce in Kusto is the column store technology which also made it to other Microsoft products, including SQL Server and Power BI. This technology seams ideal for Kusto, given how data never changes.

The trade-offs made in Kusto's design make it ideal for querying logs, the main workload. However, the technology can also be used in other creative ways. One is to use it together with a 'normal' database to create a time machine. I will cover this in more detail in

# KQL

When I first learned that Kusto has its own query language, instead of SQL, I was quite upset having to learn yet another query language after having spent countless years building my SQL skills. After spending a bit of time with KQL though, I quickly got to appreciate it, and found it to be superior (and fun to use!) compared to SQL.

One thing I like most about KQL is its fluent syntax. SQL's language has a very specific structure, for example the WHERE clause needs to come after the SELECT and FROM clause, and before GROUP BY and HAVING. HAVING awkwardly is just another WHERE clause for filtering on aggregated data. KQL on the other hand allows you to use any operator at any point in the query. Any operator is simply a transformation that takes what becomes before it and produces an output. If you know C\# LINQ, this must sound familiar! This makes KQL queries highly composable and you don't need a clumsy construct like Common Table Expressions to perform a series of transformations.

Another benefit of KQL compared to SQL is the much better IntelliSense for it in tooling. Anyone writing SQL queries in SSMS, or another editor will understand what I mean!

While SQL was designed by scientists, KQL appears to be designed by engineers. There are lots of practical features that are clearly based on actual experience using it. Just a small example is that in Kusto, when using 'sort by' or 'order by', data is by default sorted in descending order. This may seem counter-intuitive at first, but in Kusto you often find yourself sorting data by time and sorting it in descending order just seems to be the right thing most of the time.

A minor downside of KQL – perhaps also the result of being designed by engineers – is that language elements don't always follow a consistent naming convention, and some statements exist with multiple names. For example, 'sort by' and 'order by' are the same thing. (One of the two is deprecated, but I always forget which one).

# Interacting with Kusto

You can work with Kusto interactively, just like you can with SQL and SSMS. In fact, there are multiple tools to interact with Kusto this way.

My favorite one tool is the browser via https://dataexplorer.azure.com/. You get an amazingly rich experience, including syntax highlighting and IntelliSense, like what you would expect from an IDE. It remembers your sessions, so you can close the browser and open it again later, all your tabs with queries are still there. The best feature is the Share button, which lets you create URLs to queries that you can share.

There are also several client tools you can use. I sometimes like using Azure Data Studio, because of its support for Jupyter notebooks in which you can use KQL. You can also use it together with Python, which is a nice way to create scripts to run sequences of KQL commands.

As a developer you can access Kusto from your application through a REST API. You can use that to query data from a Kusto database, or to ingest data to one. There's also a connector which you can use with Power Automate (Logic Apps). You can combine this with KQL's ability to render the output into a chart, which the Power Automate app can then send as an email.
