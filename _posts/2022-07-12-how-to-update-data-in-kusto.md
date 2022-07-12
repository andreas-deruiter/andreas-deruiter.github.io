# How to update data in Kusto

One of the first things you learn when using Kusto (a.k.a. Azure Data Explorer a.k.a. ADX) is: *you cannot update data*. Here I will tell you how you *can* update data. Kind of.

When you really need to update some rows of data, you could of course simply delete the rows you want to update (for example using the `.delete` command), and then replace them by ingesting replacement data back into the table.

There are several problems with that naïve approach: it is inefficient, you need to be careful with retention policies and it is not possible to make the deletion of the old rows and the ingestion of the new rows happen at the same time. There is simply no way in Kusto to do that atomically.

This is where the `.replace extents` command comes in. It allows you to replace old data with new data, and to do it atomically and as efficiently as possible.

The caveat: this command operates on extents and not on rows. Extents are parts of the table that each contain many rows; when you replace an extent, you need to replace all rows within it, also the rows with good data.

But with a bit of juggling, we can make it work. I will show you how! And as a bonus, I will also show you how to delete rows of data at the same time.

First write a KQL query that returns the rows you wish to have updated or deleted.

```
MyTable
| where someAttribute has 'bad data' // rows we wish to update
    or someAttribute has 'remove this' // rows we wish to remove
```

Then update the query to return the distinct set of extent IDs for those rows. You will need to call the `extent_id()` for this.  
So this query returns a list of extents which need to be replaced.

```
MyTable
| extend ExtentId = extent_id()
| where someAttribute has 'bad data' // rows we wish to update
    or someAttribute has 'remove this' // rows we wish to remove
| distinct ExtentId
```

Double-check that the query returns the data you expect it to before continuing.

Tag the extents to be dropped using the `.alter-merge extent tags` command, in combination with the query from the previous step. In the sample code we use a tag named 'drop_this', but it could be any name. In fact, it is a good idea to make it more specific, for example by adding today's date to the tag name.

```
.alter-merge extent tags ('drop_this') <|
MyTable
| extend ExtentId = extent_id()
| where someAttribute has 'bad data' // rows we wish to update
    or someAttribute has 'remove this' // rows we wish to remove
| distinct ExtentId
```

We are doing this because we only want to replace those extents that have 'poisoned' data in them. Often, this is only a small subset of all the extents in the table.

Next write a query that returns the corrected rows as they should be, including the good rows for the entire table. Include a filter to remove the rows you want to delete but keeps the rows you do not want to delete (so the condition is inverted compared to before).

```
MyTable
| where someAttribute !has 'remove this' // inverted condition
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
                )
```

Add a filter to the query so that it limits the results to the rows in the extents we tagged before. The query returns the replacement data for the tagged extents.

```
MyTable
| where extent_tags() has 'drop_this'
| where someAttribute !has 'remove this'
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
                )
```

Double-check that the query returns the data you expect it to before continuing.

Create a new table with the output of the query using the `.set` command. You can name the table whatever you like, you'll just need it temporarily.

One thing you need to consider at this point is what ingestion date/time you want the replacement data to have. This is important when you use a retention policy. The .set command has a `creationTime` option that you can use to back-date the ingestion date (not shown in the example below).

```
.set MyTableWithReplacementData <|
MyTable
| where extent_tags() has 'drop_this'
| where someAttribute !has 'remove this'
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
                )
```

The `.set `command above is where the heavy lifting happens. The operation might run out of resources and fail. If that happens, you need to divide the data into smaller chunks and do one chunk at a time by adding to the table using `.append`. The following sample code shows how you would do that by splitting the data into two chunks, but this method can be made to work with any number of chunks.

```
// For the first chunk we use .set to create the table 
.set MyTableWithReplacementData <|
MyTable
| where extent_tags() has 'drop_this'
| where someAttribute !has 'remove this'
| where hash(id,2) == 0 // 2 is the total number of chunks. The first one is 0 
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
                )

// In subsequent chunks we use .append to add to the existing table
.append  MyTableWithReplacementData <|
MyTable
| where extent_tags() has 'drop_this'
| where someAttribute !has 'remove this'
| where hash(id,2) == 1 // The second chunk is 1
| extend someAttribute = iif(someAttribute has 'bad data'
                ,'corrected data' // corrected data
                ,someAttribute    // leave good data unchanged
                )
```

So far, all we did was just prep-work, nothing changed our main table. Finally, we are ready for the magic to happen. The `replace extents` command accepts two predicates, the first one identifies the list of extents to be dropped, and the second one the list of replacement extents that should be added back into the table.

```
.replace extents in table MyTable <|
    {
        .show table MyTable extents where tags has 'drop'
    },
    {
        .show tables (MyTableWithReplacementData) extents 
    }
```

The `.replace extents command` returns fast because all the heavy lifting of creating the replacement extents already happened before. The actions of dropping the old extents and adding back in the replacement extents are executed simultaneously and atomically.

Do not forget to clean up the MyTableWithReplacementData table, which is now empty.

Have fun but be careful out there!