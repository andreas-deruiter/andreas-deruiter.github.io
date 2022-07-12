---
layout: post
title: "There are many ways to remove data in Kusto. When to use which?"
description: "Kusto provides many ways to delete data, including .delete, .purge, .drop table,.drop extents or using a retention policy. This article helps you underdtand how data removal in Kusto works, and which method to use in which situation."
date: 2022-07-12
tags: Kusto delete remove purge drop clean clear retention policy extents GDPR ADX Azure Data Explorer 
---
# There are many ways to remove data in Kusto. When to use which?

Kusto, also known as Azure Data Explorer or ADX, is best known for its ability to query and analyze logs. The essence is that it allows you to capture, retain and analyze lots of data.

This is great, but how do you get data *out* of Kusto again? There are many reasons you may want to remove data, including…

-   Performance
-   Cost
-   Cleaning up bad data you ingested
-   Complying with privacy laws such as GDPR

In Kusto, you can set up a retention policy, delete or purge rows from a table, drop entire tables and more exotic options such as dropping and replacing extents.

This can be a bit bewildering, until you understand how Kusto handles data removal internally.

I’ll first discuss how data removal works in Kusto internally, and then cover all the removal methods and their trade-offs.

# How Kusto removes data

Kusto is optimized for ingesting and querying vast amounts of data very rapidly. The way it can accomplish that is at odds with something that is very easy in other types of databases, namely the ability to update and delete data. In fact, Kusto doesn’t even have a way to update data, unless by removing it adding it back again.

Key to understanding why this is, is the concept of extents. Each table is under divided into extents, each of which has a subset of the data. Those extents are managed by multiple computers (nodes) in the data center, allowing Kusto to efficiently distribute the load and achieve a high level of performance.

It’s important to understand that those extents are immutable. After an extent has been created while ingesting data, with will never be updated again.

This means that data is eventually removed at the extent level. The extent contains many rows of data, and all of them eventually get removed together. There are two commands in Kusto which seem to contradict this, .delete and .purge, but we’ll discuss those later.

Extents don’t get removed in one step though. They first get dropped, which means that they are not visible anymore. They still exist though, and sometimes you can get them back if needed.

Dropped extents can hang around for a while, even if you don’t see them.

Eventually dropped extents are erased. Think of this as garbage collection. You have some control over that process though, as you will see later.

With all of this in mind, let’s look at the various commands. We’ll start with the most common ones, and then the more advanced ones, until we end with the most exotic ones.

# Setting a retention policy

The most convenient way of deleting old data is by defining a retention policy on a table, materialized view or database in Kusto. By doing so, Kusto automatically takes care of removing data beyond a certain age.

See also: [Kusto retention policy controls how data is removed - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/retentionpolicy)

In the policy you define a SoftDeletePeriod. Microsoft guarantees that the data will remain available for this period after ingestion but gives no guarantee how soon it will remove the data once this age has been reached.

Supposedly, this is used to trigger when the extent to which the data exists is dropped. Remember that the extent contains many rows of data, so it will not be dropped before all the rows within it have expired. However, keep in mind that all the rows were ingested during the same day or so, and all the rows have the same retention policy since they belong to the same table. Therefore, the expiration dates of all the rows usually lie within a short time span.

Even after an extent data has been dropped, it still exists for an undefined period in the Microsoft data center.

Setting a data retention policy is a good way to limit the resources and costs, which would increase endlessly if you would never delete any data.

# Deleting records using `.delete`

While I mentioned earlier that Kusto handles the deletion of data at the extent level, you can still delete individual rows using the `.delete` command.

The `.delete` command does this by remembering which rows were deleted (think of it as a row-level delete flag) and applying an extra filter behind the scenes whenever someone runs a query.

See also: [Data soft delete - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/data-soft-delete)

The main use case for the `.delete` command is to get rid of corrupted data. Say for example you’re using Kusto to analyze IoT data, you could use the `.delete` command to quickly remove data from a faulty device.

According to Microsoft’s documentation, the deletion process is final and irreversible. It’s therefore a bit misleading that `.delete` is sometimes referred to as “soft-delete”, given that the data cannot be undeleted.

Note however that the extent to which the deleted rows lived will not be dropped.

The benefits of using the .delete command are that it can be performed on individual rows. It’s very fast and does not require a lot of computing resources.

On the other hand, it does not actually free up any resources, so the total size of the data occupied by your cluster won’t decrease. Moreover, if it’s a concern that the data still exists somewhere in Microsoft’s data center, this command is not for you.

# Purging records using `.purge`

Like .delete, .purge is a way to remove individual records from Kusto.

The `.purge` command differs from `.delete` in that it is meant to permanently delete rows from a table. The most important use case for this is if you need to comply to GDPR or other privacy laws.

See also: [Data purge - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/data-purge)

Remember that data removal in Kusto really happens at the extent level. This means that when you use `.purge` to remove a couple of rows from a table, Kusto first identifies to which extents they are located. Then it will copy all the rows from those extents – except the ones that need to be purged – to a new set of extents. Then it drops the ‘poisoned’ extents and replaces them with the new ones. Finally, it triggers a cleanup of the dropped extents so they will be entirely removed from the Microsoft data center. This last step happens after 5 days at the earliest, and 30 days at latest.

Purging data uses a lot of system resources because large portions of the table will need to be rebuilt. You should expect a significant performance impact and additional usage costs. Microsoft advises you to use this command sparingly.

The `.purge` command does not initially mark the affected rows as deleted, as `.delete` does. Therefore, queries run immediately after you issue the `.purge` command can still return those rows.

See also: [Data purge - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/data-purge)

The key reason to use this command is when you have user data, and you need to comply with privacy laws like GDPR. It uses many system resources, so it negatively affects cost and performance. The fact that the data is permanently removed might eventually result in a lower cost, but if that’s your main objective, you can better rely on other data removal methods.

# Dropping tables using `.drop table` and `.drop tables`

As its name suggests, you can use `.drop table`(s) to remove entire tables.

See also [.drop table and .drop tables - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/drop-table-command)

The most obvious use case is if you have old data in a table which you don’t need anymore. Of course, this can occasionally happen when system requirements change, or you want to remove old tables you’ve used for system development and testing. It’s a simple and fast way to remove data.

When you drop a table, it also drops the extents within the table. As was mentioned earlier, those extents will continue to exist for a period in the data center (which can be a long time because it depends on the original retention policy).

You can in fact undo dropping a table, which will also recover the extents back, assuming they still exist in the data center.

See also: [.undo drop table - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/undo-drop-table-command)

# Clearing table data using `.clear table`

Kusto provides the `.clear table `command to remove all the data from a given table, without removing the table itself.

[.clear table data - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/clear-table-data-command)

This is like using TRUNCATE TABLE in SQL Server.

According to the documentation, `.clear table` clears the data of an existing table, *including streaming ingestion data*. I’m not sure what it means by this. Perhaps that data still sitting in the streaming ingestion pipeline is also flushed?

# Dropping extents using `.drop extents`

You can explicitly drop extents from a table. This essentially means that all the rows in those extents are removed, which is a quick operation.

This command can be useful in several ways:

-   You can use it to quickly remove all the data from a table but retain the empty table itself. This appears like what .`clear table` does, although there might be a difference in whether they remove data in the ingestion pipeline as well.
-   You can also use it to get rid of the oldest data in a table by dropping extents by age.

Keep in mind that those dropped extents will continue to exist for a period in the Microsoft data center.

Unlike extents that were dropped as part of `.drop` table, and which can be recovered using `.undo drop table`, I don’t know of a way to recover dropped extents which were dropped using `.drop extents`. But there’s a work-around for this, as you’ll see in the next section.

# How to drop extents and still be able to recover them

As I mentioned above, I don’t know of a direct way to recover extents which were removed using `.drop extents.` There is an indirect approach however!

Instead of using .`drop extents`, you can do the following:

-   First create a table, with a schema based on the table from which you wish to delete the extents. You can use the `.create table` `based-on` command for this. See [.create table based-on - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/create-table-based-on-command)
-   Then, use the `.move extents` command to move the extents you wish to remove from the main table to the one you just created. See [.move extents - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/move-extents)
-   Then use `.drop table` to remove the table with the extents to be removed.

You can later recover that table again using `.undo drop table`, which also brings back the extents.

# Updating and deleting data with `.replace extents`

One of the first things you learn when using Kusto is: *you can’t update data*. Well, with `.replace extents` you can, and you can also use it to remove data.

I wrote a separate blog article on this:

[How to update data in Kusto](https://andreas-deruiter.github.io/2022/07/12/how-to-update-data-in-kusto.html)

The ability to update data in Kusto is exciting, but you can also use this method to remove rows of data, which makes it an third way of removing rows of data, next to using `.delete` and `.purge`.

The method is similar to `.purge` in that the data is not just flagged as deleted, but you remove rows from the extents by rewriting the poisoned extents. Using `.replace` gives you a bit more control in exchange for a bit more complexity because there are more steps for you to take.

Unlike `.purge`, `.replace` does not control when the dropped data is physically removed in the Microsoft data center.

# Physically deleting dropped extents in the Microsoft data center

With the exception of `.purge`, none of the methods discussed so far by themselves give any guarantees on when Microsoft physically removes the deleted data from the data center. Often, it’s good enough not to know about how garbage collection works behind the scenes, but there can be exceptions, such as when you must comply by law to ensure every copy of certain data is removed according to a given timeline or schedule.

`.purge` is designed to give that guarantee. Kusto, however, also provides another way to influence when the physical data is removed using `.clean databases extentcontainers.`

Like `.purge`, when you use `.clean databases extentcontainers` Kusto waits at least 5 days before physically removing the data. Unlike .purge though, Microsoft’s documentation doesn’t currently state a maximum number of days it might wait.

Another difference is that .purge works at the table level, and `.clean databases extentcontainers` works on an entire database.

You can track the progress of a `.clean databases extentcontainers` operation via `.show database <database> extentcontainers clean operations`.` .purge `commands do not show up here. This shows that, while there are similarities between both commands, they appear to be independent from each other.

# Putting it together

As you saw, there are many ways to remove data in Kusto. It depends on your situation which method to use.

| **Scenario**                                           | **What to use**                    |
|--------------------------------------------------------|------------------------------------|
| Quickly delete faulty rows of data                     | `.delete`                          |
| Sporadically delete rows of data for legal reasons.    | `.purge`                           |
| Updating and deleting rows of data                     | `.replace extents`                 |
| Get rid of old tables                                  | `.drop table`                      |
| Get rid of all data in a table                         | `.clear table`                     |
| Get rid of old data in a table, manually               | `.drop extents`                    |
| Get rid of old data in a table, automatically          | Define a retention policy          |
| Physically remove data from dropped tables and extents | `.clean database extentcontainers` |
