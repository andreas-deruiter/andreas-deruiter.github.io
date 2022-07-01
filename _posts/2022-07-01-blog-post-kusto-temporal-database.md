# Creating a temporal database with help of Kusto

Kusto, also known as Azure Data Explorer or ADX, is best known for its ability to query and analyze logs. That sounds a bit boring (as does the 'Azure Data Explorer' name), but I found it to be one of those hidden gems that has become one of my favorite Microsoft technologies, right up there with VS Code, Typescript and Excel. Seriously!

The technology powering Kusto is awesome and can be used for more than just analyzing logs. Another application, which I will discuss in this article, is that you can pair Kusto to another database, such as Cosmos DB, and create a temporal database. A temporal database is one which retains historic data, so that you can easily query data as it was at any moment in the past, like having a time machine! This is great when you need the ability to audit or troubleshoot changes to data over time.

In this article I'll explain how you can implement a temporal database with Cosmos DB and Kusto, but you can also use the same approach together with other databases, and even with SaaS applications like Dynamics CRM. 

We have a lot of ground to cover, and in this article, I'll focus on the high-level concepts.

# *Kusto explained*

If you're already familiar with Kusto, you can skip this part.

Kusto is essentially a cloud-based database in Azure. In that sense, it's like Cosmos DB or Azure SQL Server. Every database system makes certain trade-offs. Kusto is very good at ingesting and querying huge amounts of data, both structured and unstructured. However, it's very bad and updating existing data. In fact, you can't even update existing data at all!

This tradeoff means that Kusto doesn't compete with the traditional SQL or non-SQL databases. Instead, you'd use it in scenarios where you might otherwise use Apache Spark. Just like with Spark, Kusto's ability to handle huge amounts of data largely stems from the fact that the work is spread across many nodes (computers) in the backend. The benefit of Kusto is that you don't need to worry about managing any infrastructure. It's very simple to create and manage a Kusto cluster in the Azure Portal.

A secret sauce in Kusto is the magnificent column store technology which also made it to other Microsoft products, including SQL Server and Power BI. Intuitively, it feels to me as if Kusto from the ground up is built for this technology, whereas in SQL Server half of the time I don't understand why column store indexes don't perform as I would expect.

The result is that you can query large datasets in Kusto, including those based on unstructured or semi-structured data, very fast. In fact, Microsoft also uses Kusto as the backbone for Application Insights and Log Analytics!

## KQL

When I first learned that Kusto has its own query language, instead of SQL, I was quite upset having to learn yet another syntax to do the same kind of thing. After spending a bit of time with KQL though, I have become very fond of it, and found it to be superior to SQL in many ways.

What I like most about KQL is its fluent syntax. In SQL, the WHERE clause needs to come after the SELECT and FROM clause, and before GROUP BY and HAVING. HAVING awkwardly is just another WHERE clause for filtering on aggregated data. KQL on the other hand allows you use any clause at any point in the query. Any clause is simply a transformation that takes what becomes before it and produces an output (like C# LINQ). This makes KQL queries highly composable and you don't need ugly clumsy constructs like CTEs or nested queries to perform a series of transformations.

While SQL was designed by scientists, KQL appears to be designed by engineers. There are lots of practical features that are clearly based on actual experience using it. Just a small example is that in Kusto, when using 'sort by' or 'order by', data is by default sorted in descending order. This may seem counter-intuitive at first, but in Kusto you often find yourself sorting data by time and sorting it in descending order just seems to be the right thing to do 9 out of 10 times.

A minor downside of KQL – perhaps also the result of being designed by engineers – is that language elements don't always follow a consistent naming convention, and some statements exist with multiple names. For example, 'sort by' and 'order by' are the same thing. (One of the two is deprecated, but I always forget which one).

For me, a bonus for learning KQL is that I can also leverage this skill to build powerful Application Insights and Log Analytics queries for my applications, and no longer rely solely on the built-in telemetry charts.

# How to use Kusto as a temporal database, conceptually

By now it should be obvious that Kusto will not replace the existing transactional database in your application, which in this article we'll assume to be a Cosmos DB collection. You'll keep using the transactional database as you did in the past, but whenever a row in the transactional database is inserted or updated, you log the new version of it in the Kusto database. So, while your transactional database always has the latest version of each row, the Kusto database contains all versions of all rows.

Of course, you should keep track of deletions too in Kusto. The way to do this is to maintain an extra row attribute in Kusto called 'isActive' which you set to true upon ingestion when rows are inserted or updated in the source database. When a row is deleted from the source database, you treat this as a new version of the object, with isActive as false.

So far, this approach allows you to have a temporal database, provided you start with an empty source database. However, it's more likely you want to implement a temporal database for an existing database. In that case, you should just start by ingesting the existing transactional data to Kusto as inserts, and once that's completed process any further changes and mentioned before. It goes without saying that your historic queries in Kusto can't go back to the time before you started recording changes to Kusto!

# Implementing a temporal database with Cosmos DB

We now know at a high-level how to implement a temporal database with Kusto. Let's take a deeper look at how to specifically implement this with Cosmos DB.

## Creating a Kusto table to track changes to the Cosmos DB collection

Cosmos DB doesn't explicitly manage schema. Every row in Cosmos DB is just a Json object (albeit with a couple of built-in system attributes like \_ts). Your application manages the schema implicitly through code.

Kusto has a more traditional schema approach, where a database contains tables, and tables have attributes. Luckily string attributes can be up to 1MB in size, so you can store the entire object payload as Json in an attribute called something like 'data'. The benefit of this approach is that it's generic, and you don't need to update the Kusto schema every time your application schema changes.

For practical reasons, you'll want to add several additional attributes to the Kusto table.

You should add the `id` attribute as a separate attribute to your Kusto table. Keep in mind that while those IDs are unique to Cosmos DB, they won't be unique in Kusto. That's fine, Kusto doesn't even support the concept of a unique primary key.

As mentioned before, we also need an `isActive` Boolean attribute on the table to track deletions.

Chances are that you already have an attribute in your Cosmos DB database collection called something like 'type', 'entity' or 'class'. With this attribute, your application distinguishes between various types of objects stored in the collection. You should also add this attribute explicitly on the Kusto table. We'll assume this attribute is named `type`.

If you have other generic attributes that exist on all your objects in Kusto, such as `createdBy` and `modifiedBy`, I recommend you also add those as separate attributes to the Kusto table.

Finally, you need a `timestamp` attribute on your Kusto table which is the date/time when this row was inserted, updated or soft-deleted.

By adding these attributes, you will be able pre-filter many queries without the need of having Kusto parse the data attribute first. This is important for performance.

Putting it all together, we'll be creating a Kusto table like so:

```
.create table MyCollectionTable (
    ['id']: string,
    ['type']: string
    isActive: bool,
    timestamp: datetime,
    ['data']: string)
```

(I assume you already have created a Kusto cluster in the Azure Portal. My favorite tool for running queries and commands is the web-based portal at <https://dataexplorer.azure.com/> )

## Updating the Cosmos DB application

Ideally, we could use Kusto with an existing application and transactional database without needing to make any adjustments to it. There are however a couple of caveats with Cosmos DB which requires us to make some modifications to the Cosmos DB application and its schema first.

### Update the Cosmos DB application to support soft-delete

What I will describe in this section is a common pattern when using Cosmos DB, so you might already have this in place. In that case, you're in luck!

As I'll explain later, we'll rely on a Cosmos DB feature called “Change Feed” to update our Kusto table whenever anything changes in Cosmos DB. One caveat with Change Feed is that it only triggers when data is created or updated, not when objects are deleted.

Therefore, instead of hard deleting data from your Cosmos DB collection, you should implement a soft-delete mechanism by adding a Boolean attribute to the Json schema in Cosmos DB, which I will suggest you call “isActive”, just like the equivalent attribute in the Kusto table.

Whenever your application creates an object, `isActive` should be initialized to True, and when it needs to delete the object, your application should set this attribute to False rather than hard deleting the object. Your application should also clear any other attributes in the payload upon deletion, except of course `id` and the partition key, both of which are immutable attributes in Cosmos DB.

You'll need to update all the queries in your application and add an additional filter so that only objects where `isActive` does not equal False are returned (this is more robust than returning objects where isActive equals True).

## Implement a timestamp attribute in your Cosmos DB application

When ingesting data to Kusto, it will need to know *when* the data was last modified so it can initialize the timestamp attribute accordingly. To this end, I recommend you add a `timestamp` attribute to your data in Cosmos DB if you don't already have one.

Instead of implementing your own timestamp attribute, however, you could also rely on the \_ts system attribute, which is already part of every Cosmos DB object. The downside of \_ts however is that it has a 1-second granularity, which can be limiting when you later want to use Kusto to analyze what happened during a chain of events that occurred within a short time span.

### Update the data in your Cosmos database

If you implemented soft-delete and/or a timestamp attribute as mentioned above, you need to run a batch process to update the existing data in Cosmos DB to initialize those attributes. The process should set `isActive` to true for all objects, and you can use the value of \_ts to initialize the timestamp attribute for all the already existing objects.

## Implement data ingestion of the Cosmos DB collection to Kusto

I mentioned earlier that to use Kusto as a temporal database, you'll first need to ingest the already existing data in Cosmos DB to Kusto, and from then onwards all the objects as they change. The good news is that using Cosmos DB's Change Feed, you can do both at once, and doing it in a very reliable way, while the Cosmos DB application remains online!

Cosmos DB change feed works a bit differently than you might expect and understanding this is important!

Let me first explain what Change Feed is *not*: it's not a message or event queue, and it's not a “push” mechanism.

Fundamentally, Change Feed is just a special way to query the Cosmos DB database. This query returns a list of all objects, ordered by modification time, oldest first. The result set is limited to a batch of max 1000 object per call, and to get the next batch, you need to call it again with a continuation token. You could do all of that using an ordinary Cosmos DB query!

What's special is that when you call it with a continuation token for the next batch, rather than returning the next 1000 objects from the original results, the remaining results are refreshed. It will then return fresh values starting where it left off the last time.

By continuously pulling results this way, you eventually get all objects in the collection, and from that moment on, the result set will just be empty, assuming nothing changes.

It gets interesting when data *does* change. If an object changes, it will reappear on the list of objects to be returned to the client, even if a previous version of it was already returned before. Since the object was the last one to change, it will be the last on the list. However, once the change feed processor gets around to process it, other objects may have also changed in Cosmos DB, which will then appear after it.

This mechanism allows us to implement a process that keeps another system in sync with the Cosmos DB collection, and that's exactly what we want to do with Kusto!

There are some interesting implications that you need to keep in mind:

-   Objects that are deleted from Cosmos DB will not appear in the feed. This makes sense now that you know that Change Feed is just a query, and a query only returns objects that currently exist in the database. Now you understand why we needed to implement soft-delete in the Cosmos DB application!
-   The objects returned are always the current version. Again, this should be obvious now that you know that Change Feed is just a query and returns data as it exists now.
-   The time Change Feed lags the source database can vary. Assume there is a huge spike changes to objects in Cosmos DB. The Change Feed client will receive and process those in batches of up to 1000 objects at a time. The lag of how long it takes for those changes to become visible downstream (in Kusto) will increase as objects change faster than the ingestion process can keep up. Later, when the load on Cosmos DB lessons, the lag decreases again as the change feed processes catches up.
-   A nice consequence of the previous point is that the downstream system is shielded from the spike in Cosmos DB. It just keeps processing a steady stream of objects, max 1000 at a time.
-   You are not guaranteed to receive all changes to an object. If an object in Cosmos DB is changed, and rapidly changes again before the change feed client received the first change, the first change will appear to disappear. This is important to keep in mind when you query the data in Kusto, though I am yet to encounter a situation where this was a problem.
-   Changes to objects in Cosmos DB are not guaranteed to arrive in the order in which they occurred. However, within a Cosmos DB partition they do arrive in the right order.

See also: [Working with the change feed support in Azure Cosmos DB \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/cosmos-db/change-feed)

There are several ways to implement a change feed processor. I recommend creating an Azure function with a Cosmos DB trigger, which abstracts a lot of the complexity so that your code just needs to listen for changes. If you wouldn't know better, you'd think that change feed has a push mechanism that sends you change events!

Here's some skeleton code for such an Azure function:

```
[Singleton(Mode = SingletonMode.Listener)]
[FunctionName(nameof(KustoIngestorAsync))]
public async Task KustoIngestorAsync(
    [CosmosDBTrigger(
        databaseName: "%CosmosStoreSettings:DatabaseName%",
        collectionName: "%CosmosStoreSettings:CollectionName%",
        ConnectionStringSetting = "CosmosDbConnectionString",
        LeaseCollectionName = "KustoIngestor",
        LeasesCollectionThroughput = 2000,
        CreateLeaseCollectionIfNotExists = true,
        StartFromBeginning = true,
        MaxItemsPerInvocation = 1000,
        FeedPollDelay = 500,
        LeaseCollectionPrefix = "KustoIngestor")]IReadOnlyList<Document> input)
{
    if (input != null && input.Count > 0)
    {
        try
        {
            await ProcessDocumentsForKustoIngestion(input).ConfigureAwait(false);
        }
        catch (Exception ex)
        {	// Exception handling logic
        }
    }
}
```

The ProcessDocumentsForKustoIngestion() method would to the actual legwork of calling the Kusto's ingestion libraries to ingest the data. While ingesting the data, it needs to set the attributes on the Kusto table we created before accordingly:

-   `id` is initialized to the Cosmos DB ID of the object
-   The `type` attribute is set to the type/entity/class of the object as you manage it in your Cosmos DB schema.
-   The `timestamp` is set to the timestamp you added to the Cosmos DB database as I suggested before. If you already had an attribute with the modification date/time, you could of course use that to set the timestamp on the Kusto table.  
    To be on the safe side, your ingestion code could check if the `timestamp` attribute in Cosmos DB has a valid value, and if not, initialize it based on \_ts (which is a Unix time) instead.
-   The `isActive` attribute needs to be set based on the attribute you added to the Cosmos database to support soft delete.
-   Finally, the `data` attribute should be set to the Json payload. Keep in mind that the max size is 1MB. If your Cosmos DB collection has objects that exceed that size, you could consider stripping some low-value attributes from the payload as part of the ingestion process.

Implementing ingestion in a robust way is not entirely trivial. For example, when an exception occurs, the data to be ingested needs to be temporarily saved to a persistent store such as Azure Blob Storage, so that ingestion can be retried later. This means you need another Azure Function that periodically checks for items which previously failed and try again.

About the code:

We're running the function as a singleton. This will make it more robust as it essentially throttles the ingestion of data and prevents concurrency troubles.

Note that we specified StartFromBeginning as true. This means that when we deploy the function, it will first start processing all the existing data in the collection. This is essential as we want to use Kusto as a temporal database.

# Querying the temporal database

Assuming everything is in place and data is being ingested to Kusto, we can start querying it.

My favorite tool for this is the web-based Azure Data Explore site at <https://dataexplorer.azure.com/>. It supports nice editing features, such as context highlighting and auto complete, and it allows you to easily share links to queries.

I'll show some useful queries on our temporal MyCollectionTable table.

To get a sense of what the data looks like, try the following query, which returns 10 objects somewhat randomly:

```
MyCollectionTable
| take 10
```

You can count the number of rows in Kusto like this:

```
MyCollectionTable
| count
```

Keep in mind that the object count above is expected to be higher than the number of objects in Cosmos DB, because there can be multiple versions of each object in Kusto. Inactive (deleted) records are also counted.

To query the number of “current”, non-deleted objects in Kusto, use this query:

```
MyCollectionTable
| summarize arg_max(timestamp, \*) by id // Keep only the newest version of each object
| where isActive // Keep only records which aren't deleted
| count
```

This function should return the same number of objects as Cosmos DB does, not counting those where isActive is false. The actual count might differ slightly, only because there is a time lag of a couple of minutes from Cosmos DB to Kusto.

The line where we summarize data using the arg_max function is where the magic happens to filter out previous versions of each object. This is a demanding call though, and if possible, you should filter the data already prior to making that call.

A nice bonus is that you can now easily query the data in Cosmos DB using KQL, without being limited by the Cosmos DB's SQL query language, which supports a very limited SQL dialect.

The following query counts the number of active objects where the type is 'Company'

```
MyCollectionTable
| where type == 'Company'
| summarize arg_max(timestamp, \*) by id // Keep only the newest version of each object
| where isActive // Keep only records which aren't deleted
| count
```

Now let's try our first historic query. This query returns the number of companies we had in the Cosmos DB database exactly one day ago:

```
MyCollectionTable
| where timestamp \< ago(1d)
| where type == 'Company'
| summarize arg_max(timestamp, \*) by id // Keep only the newest version of each object
| where isActive // Keep only records which aren't deleted
| count
```

As you see we have the same query as before, except we now also filter out all data from the last day. Knowing that existing data in Kusto is never updated, this query returns the same results as if we had run it one day ago!

Now suppose we want to know which Company object has changed most often:

```
MyCollectionTable
| where type == 'Company'
| summarize n=count() by ['id']
| sort by n // default sort order is descending
| take 1
```

Result:

| **Id**                           | **n** |
|----------------------------------|-------|
| 6f2a11e5041a9fd34184c8aab4e5718a | 288   |

As you can see, we have a company object that has changed 288 times!

To see all versions of this company object:

```
MyCollectionTable
| where id == '6f2a11e5041a9fd34184c8aab4e5718a'
| sort by timestamp // returns newest version on top
```

To see graphically when changes to this object were made:

```
MyCollectionTable  
| where id == '6f2a11e5041a9fd34184c8aab4e5718a'  
| make-series n=count() default=0 on timestamp step 7d  
| render timechart
```

![Graphical user interface, application Description automatically generated](media/5cf212b0f3215673ec7d1f623ae0e1af.png)

As you can see, something happened in March of the previous year that caused most changes to this company object. With the historic records in Kusto we could further investigate in detail what caused this to happen!

Now for something more sophisticated, let's see how many active (i.e., non-deleted) Company objects we had in the Cosmos DB database over time:

```
MyCollectionTable  
| where type == 'Company'  
| sort by id, timestamp asc   
| extend v=row_number(1,id != prev(id))  
| extend delta = iff(v==1,toint(isActive),toint(isActive)-toint(prev(isActive)))  
| sort by timestamp asc  
| extend total = row_cumsum(delta)  
| summarize max(total) by bin(timestamp, 1d)  
| render timechart
```

![Graphical user interface Description automatically generated](media/bfc2c791d8ec5e3f0a8d74249c93ce96.png)

Until now, we haven't really done anything with the payload yet. With the extract_json and todynamc functions in KQL you can extract data from the Json payload, which in our case is loaded as a string in the data attribute.

For my Cosmos DB schema, the 'createdBy' value can be retrieved from the payload using the Json path '\$.data.createdBy.v'. To retrieve a list of people who created company objects and display the number of company objects each of them created, you can use this query:

```
MyCollectionTable
| where type == 'Company'
| summarize arg_max(timestamp, \*) by id // Keep only the newest version of each object
| where isActive // Keep only records which aren't deleted
| extend createdBy = extract_json('\$.data.createdBy.v', data, typeof(string))
| summarize n=count() by createdBy
| sort by createdBy asc
```

As before, we could also run this query at a moment in the past by adding a time filter.

Finally, let's select all the company objects that were created by me:

```
MyCollectionTable
| where type == 'Company'
| summarize arg_max(timestamp, \*) by id // Keep only the newest version of each object
| where isActive // Keep only records which aren't deleted
| where data has 'Andreas de Ruiter' // pre-filter the data
| extend createdBy = extract_json('\$.data.createdBy.v', data, typeof(string))
| where createdBy == 'Andreas de Ruiter'
```

Note that the query pre-filters the data before the line where we call extract_json. By leveraging Kusto's highly efficient full-text search this way, fewer objects need to be parsed by extract_json, thereby speeding up this query.

# Final thoughts

I've used Kusto as a temporal database in projects for several years and having a lens to the past feels like having super-powers. I'll end by mentioning a couple of things that I haven't touched upon yet, but also need careful consideration.

## Compliance with privacy laws and policies

As a business or enterprise, you need to be able to comply with privacy laws. Moreover, your customers, business partners and employees deserve their privacy to be respected. Effectually this means that applications and databases need to be able to 'forget' data after a given period. In some cases, you want to go even further and prevent sensitive data from being ingested into your Kusto in the first place. Having sensitive data in Kusto means that fewer people may access it, which limits its usefulness.

Philosophically, Kusto is designed so you can remember as much as possible, and this seems at odds with the privacy requirements of needing to forget. That said, Kusto supports multiple mechanisms to forget data, including automated retention policies.

The problem is that these retention mechanisms don't always play well when using Kusto as a temporal database. Perhaps this is the reason why Microsoft doesn't actively promote using Kusto as a temporal database. There are ways to deal with this though, just keep in mind it's not trivial. (Let me know if you want me to write an article about that).

## Beware of bug fixes that require database updates!

Our Cosmos DB has attributes like timestamp and isActive which are essential for the temporal database queries in Kusto to behave correctly. The Cosmos DB application code consistently sets these attributes, so what could go wrong here?

Let's say a bug is found in the Cosmos DB application which has nothing to do with the attributes Kusto relies on, such as timestamp. The engineering team goes ahead and implements a code fix. So far so good, but sometimes a fix also requires data be updated in the Cosmos DB database. To do that, the engineering team creates a script to update the data directly in the Cosmos DB database, bypassing the code which normally updates attributes like timestamp and isActive.

Change feed will see those changes and Kusto will be updated accordingly. However, if the developer of the script didn't think of updating timestamp and isActive flags in Cosmos DB, or if the script hard deletes records in the database all together, the temporal queries won't return accurate results anymore. Now we have a lot of work to find the discrepancies and fix the data in Kusto, which is not simple know that Kusto doesn't even have a command equivalent to UPDATE in SQL.

If have seen this happen it the past, especially when the Cosmos DB application developers are not the same people who implemented the Kusto integration.
