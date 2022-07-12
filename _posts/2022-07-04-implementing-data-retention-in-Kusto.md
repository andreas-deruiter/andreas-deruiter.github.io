# There are many ways to remove data in Kusto. When to use which?

Kusto, also known as Azure Data Explorer or ADX, is best known for its ability to query and analyze logs. The essence is that it allows you to capture, retain and analyze lots of data.

This is great, but how do you get data *out* of Kusto again? There are many reasons you may want to remove data, including…

-   Performance
-   Cost
-   Cleaning up bad data you ingested
-   Complying with privacy laws such as GDPR

In Kusto, you have the ability to setup a retention policy, delete or purge rows from a table, drop entire tables and also more exotic options such as dropping and replacing extents.

This can be a bit bewildering, until you understand how Kusto handles data removal internally.

I’ll first discuss how data removal works in Kusto internally, and then cover all the removal methods and their trade-offs.

# How Kusto removes data

Kusto is optimized for ingesting and querying vast amounts of data very rapidly. The way it is able to accomplish that is at odds with something that is very easy in other types of databases, namely the ability to update and delete data. In fact, Kusto doesn’t even have a way to update data, unless by removing it adding it back again.

Key to understanding why this is, is the concept of extents. Each table is under divided into extents, each of which has a subset of a the data. Those extents managed by multiple computers (nodes) in the data center, allowing Kusto to efficiently distribute the load and achieve a high level of performance.

It’s important to understand that those extents are immutable. After an extent has been created while ingesting data, with will never be updated again.

This means that data is eventually removed at the extent level. The extent contains many rows of data, and all of them eventually get removed together. There are two commands in Kusto which seem to contradict this, .delete and .purge, but we’ll discuss those later.

Extents don’t get removed in one step though. They first get dropped, which means that they are not visible anymore. They still exist though, and sometimes you can get them back if needed.

Dropped extents can hang around for a while, even if you don’t see them.

Eventually dropped extents are erased. Think of this as garbage collection. You have some control over that process though, as you will see later.

With all of this in mind, let’s take a look at the various commands. We’ll start with the most common ones, and then the more advanced ones, until we end with the most exotic ones.

# Setting a retention policy

The most convenient way of deleting old data is by defining a retention policy on a table, materialized view or database in Kusto. By doing so, Kusto automatically takes care of removing data beyond a certain age.

See also: [Kusto retention policy controls how data is removed - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/retentionpolicy)

In the policy you define a SoftDeletePeriod. Microsoft guarantees that the data will remain available for this period of time after ingestion, but gives no guarantee how soon it will remove the data once this age has been reached.

Supposedly, this is used to trigger when the extent in which the data exists is dropped. Remember that the extent contains many rows of data, so it will not be dropped before all of the rows within it have expired. However, keep in mind that all the rows were ingested during the same day or so, and all the rows have the same retention policy since they belong to the same table. Therefor, the expiration dates of all the rows usually lie within a short time span.

Even after an extent data has been dropped, it still exists for an undefined period of time in the Microsoft data center.

Setting a data retention policy is a good way to limit the resources and costs, which would increase endlessly if you would never delete any data.

# Deleting records using `.delete`

While I mentioned earlier that Kusto handles the deletion of data at the extent level, you can still delete individual rows using the `.delete` command.

The `.delete` command does this by remembering which rows were deleted (think of it as a row-level delete flag) and applying an extra filter behind the scenes whenever someone runs a query.

See also: [Data soft delete - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/data-soft-delete)

The main use case for the `.delete` command is to get rid of corrupted data. Say for example you’re using Kusto to analyze IoT data, you could use the `.delete` command to quickly remove data from a faulty device.

According to Microsoft’s documentation, the deletion process is final and irreversible. It’s therefore a bit misleading that `.delete` is sometimes referred to as “soft-delete”, given that the data cannot be undeleted.

Note however that the extent in which the deleted rows lived will not be dropped.

The benefits of using the .delete command are that it can be performed on individual rows. It’s very fast and does not require a lot of computing resources.

On the other hand, it does not actually free up any resources, so the total size of the data occupied by your cluster won’t decrease. Moreover, if it’s a concern that the data still exists somewhere in Microsoft’s data center, this command is not for you.

# Purging records using `.purge`

Like .delete, .purge is a way to remove individual records from Kusto.

The `.purge` command differs from `.delete` in that it is meant to permanently delete rows from a table. The most important use case for this is if you need to comply to GDPR or other privacy laws.

See also: [Data purge - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/data-purge)

Remember that data removal in Kusto really happens at the extent level. This means that when you use `.purge` to remove a couple of rows from a table, Kusto first identifies in which extents they are located. Then it will move all of the rows in those extents – except the ones that need to be purged – to a set of new extents. Then it drops the ‘poisoned’ extents and replaces then with the new ones. Finally it triggers a clean up of the dropped extents so they will be entirely removed from the Microsoft data center. This last step happen after 5 days at earliest, and 30 days at latest.

Purging data uses a lot of system resources because large portions of the database will need to be rebuilt. You should expect a significant performance impact and additional usage costs. Microsoft wants you to use this command sparingly.

The .purge command does not initially mark the affected rows as deleted, as .delete does. Therefore, queries run immediately after you issues the .purge command can still return those rows.

See also: [Data purge - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/data-purge)

The key reason to use this command is when you have user data and you need to be compliant with privacy laws like GDPR. It uses many system resources, so it negatively affects cost and performance. The fact that the data is permanently removed might eventually result in a lower cost, but if that’s your main objective, you can better rely on other data removal methods.

# Dropping tables using `.drop table `and` .drop tables`

As its name suggests, you can use `.drop table`(s) to drop entire tables.

See also [.drop table and .drop tables - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/drop-table-command)

The most obvious use case is if you have old data sitting in a table you don’t need anymore. Of course, this can occasionally happen when system requirements change or you want to remove old tables you’ve used for system development and testing. It’s a simple and fast way to remove data.

When you drop a table, it also drops the extents within the table. As was mentioned earlier, those extents will continue to exist for a period of time in the data center (which can be a long time because it depends on the original retention policy).

You can in fact undo dropping a table, which will also recover the extents back, assuming they still exist in the data center.

See also: [.undo drop table - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/undo-drop-table-command)

# Dropping extents using `.drop extents`

You can explicitly drop extents from a table. This essentially means that all the rows in those extents are removed, which is a quick operation.

This command can be useful in several ways:

-   You can use it to quickly remove all the data from a table, but keep empty table itself. This is similar to using TRUNCATE TABLE in TSQL.
-   You can also use it to get rid of the oldest data in a table by dropping extents by age.

Keep in mind that those dropped extents will continue to exist for a period of time in the Microsoft data center.

Unlike extents that were dropped as part of .drop table, and which can be recovered using `.undo drop table`, I don’t know of a way to recover dropped extents which were dropped using `.drop extents`. But there’s a work-around for this, as you’ll see in the next section.

How to drop extents and still be able to recover them

As I mentioned above, I don’t know of a direct way to recover extents which were removed using `.drop extents.` There is an indirect approach however!

Instead of using .drop extents, you can do the following:

-   First create a table, with a schema based on the table from which you wish to delete the extents. You can use the `.create table` `based-on` command for this. See [.create table based-on - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/create-table-based-on-command)
-   Then, use the `.move extents` command to move the extents you wish to remove from the main table to the one you just created. See [.move extents - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/move-extents)
-   The use `.drop table` to remove the table with the extents to be removed.

You can later recover that table again using `.undo drop table`, which also brings back the extents.

# For ninja’s only: updating data with `.replace extents `

One of the first things you learn when using Kusto is: *you can’t update data*. Well, with `.replace extents `you can. Kind of.

If you really want to update some data, you could think of a way to delete the rows you want to update (for example using the .delete command), and then replace it by ingesting the updated data into the table.

There are several problems with that naïve approach: not only is it’s inefficient, it’s also not possible to time the deletion of the old rows and the ingestion of the new rows such that they happen at the same time. There’s no way to do that atomically.

This is where `.replace extents` comes in. It allows you to replace old data with new data, and do it atomically and as efficiently-as-possible.

The caveat: this command operates on extents and not on rows. But with your ninja super powers you can overcome this. Here’s how.

1.  First write a KQL query that returns the rows you wish to have updated.
2.  Then update the query to return the distinct set of extent IDs for those rows. You will need to call the extent_id() for this.  
    So now you have a list of extents which will need to be replaced.
3.  Next write a query that returns the corrected data as it should be, for the entire table. So this query should also return the rows that you don’t want to update.
4.  Add a filter to that query of the previous step, so that it only returns the rows which are in the extent IDs we identified before. Keep in mind that all rows to be updated will be part of the result set, but also a subset of the rows which don’t need to be updated will be in the result set.
5.  Tag the extents to be dropped using `.alter-merge extent `tags command, in combination with the query from the previous step. See also: [Kusto query ingestion (set, append, replace) - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/data-ingestion/ingest-from-query)
6.  Use the `.set` command in combination with the query from the previous step to create a new table with the contents of that query.  
    This is where the heavy lifting happens. This operation might run out of resources and fail. If that happens, you need to divide the data into chunks and do it in multiple steps by adding to the table using `.append`.  
    See also: [Kusto query ingestion (set, append, replace) - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/data-ingestion/ingest-from-query)
7.  Finally, use the .replace extents to make it all happens. You can pass to predicates to this command: one that is a list of all the

# Conclusion and recommendations

If you want Kusto to forget old data for compliance reasons, the safest bet – legally – is to rely on the `.purge` command. If you want to use `.delete` or rely on a retention policy, you should first consult with a legal expert.

Due to the performance and cost impact of `.purge`, you should use it sparingly, for example once every three months.

Best practices:

-   Design your ingestion process such that it already filters or obfuscates privacy-sensitive data. For example, if you use the Kusto SDK to write your own ingestion process, you can use a regular expression find and replace email addresses with something else. This way, you also don’t need to worry about removing this data later.
-   Consider designing your Kusto tables such that the privacy data is stored in separate tables from the other data. This way, you can keep the non-sensitive data for a much longer period, and fewer system resources are needed to purge rows from the tables with the sensitive data.

Make sure you thoroughly review Microsoft’s documentation on purging data.
