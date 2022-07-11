# How to update data in Kusto

One of the first things you learn when using Kusto (a.k.a. Azure Data Explorer a.k.a. ADX) is: *you can’t update data*. Here I will tell you how you *can* update data. Kind of.

If you really need to update some data, you could of course simply delete the rows you want to update (for example using the .delete command), and then replace them by ingesting replacement data back into the table.

There are several problems with that naïve approach: it’s inefficient, you need to be careful with retention policies, and finally, it’s not possible to make the deletion of the old rows and the ingestion of the new rows happen at the same time. There’s simply no way in Kusto to do that atomically.

This is where the `.replace extents` command comes in. It allows you to replace old data with new data, and to do it atomically and as efficiently as possible.

The caveat: this command operates on extents and not on rows. Extents are parts of the table that each contain many rows; when you replace an extent, you need to replace all rows within it, also the rows with good data.

But with a bit of juggling, we can make it work how we wish. I’ll show you how!

First write a KQL query that returns the rows you wish to have updated.

```
MyTable
| where someAttribute has 'bad data'
```

Then update the query to return the distinct set of extent IDs for those rows. You will need to call the `extent_id()` for this.  
So this query returns a list of extents which will need to be replaced.

```
MyTable
| extend ExtentId = extent_id()
| where someAttribute has 'bad data'
| distinct ExtentId
```

Tag the extents to be dropped using the `.alter-merge extent tags` command, in combination with the query from the previous step. In the sample code we use a tag named ‘drop’, but it could be any name. In fact, it’s a good idea to make it more specific, for example by adding today’s date to the tag name.

```
.alter-merge extent tags ('drop') <|
MyTable
| extend ExtentId = extent_id()
| where someAttribute has 'bad data'
| distinct ExtentId
```

Next write a query that returns the corrected data as it should be, for the entire table. So, this query should also return the rows that you don’t want to update.

```
MyTable
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
                )
```

Add a filter to that query so that it limits the results to the rows in the extents we tagged before.

```
MyTable
| where extent_tags() has 'drop'
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
```

Use the `.set` command to create a new table with the output of the query.

One thing you need to consider at this point is what ingestion date/time you want the replacement data to have. This is important when you use a retention policy. The .set command has a `creationTime` option that you can use for that (not shown in the example below).

```
.set MyTableWithReplacementData <|
MyTable
| where extent_tags() has 'drop'
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
                )
```

The `.set `command above is where the heavy lifting happens. The operation might run out of resources and fail. If that happens, you need to divide the data into smaller chunks and do it in multiple steps by adding to the table using `.append`. The following sample code shows how you would do that by splitting the data into two chunks, but this method can be made to work with any number of chunks.

```
// For the first chunk we use .set to create the table 
.set MyTableWithReplacementData <|
MyTable
| where extent_tags() has 'drop'
| where hash(id) % 2 == 0 // 2 is the total number of chunks. The first one is 0 
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
                )

// In subsequent chunks we use .append to add to the existing table
.append  MyTableWithReplacementData <|
MyTable
| where extent_tags() has 'drop'
| where hash(id) % 2 == 1 // The second chunk is 1
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
                )
```

Finally, we’re ready for the magic to happen. The `replace extents` command accepts two predicates, the first one identifies the list of extents to be dropped, and the second one the list of replacement extents that should be added back into the table.

```
.replace extents in table MyTable <|
    {
        .show table MyTable extents where tags has 'drop'
    },
    {
        .show tables (MyTableWithReplacementData) extents 
    }
```

The `.replace extents command` returns fast because all the heavy lifting of *creating* the replacement extents happened before. Both actions of dropping the old extents and adding back in the replacement extents are executed atomically.

Don’t forget to clean up the MyTableWithReplacementData table, which is now empty.

Have fun but be careful out there!
