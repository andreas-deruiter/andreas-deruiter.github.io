# Implementing data retention in Kusto

Kusto, also known as Azure Data Explorer or ADX, is best known for its ability to query and analyze logs. The essence is that it allows you to capture, retain and analyze lots of data. While powerful, society now expects organizations to be able to *forget* about them. This is a core tenant in GDPR and other privacy laws world-wide. Another benefit of removing old data is that it reduces costs.

Kusto supports multiple ways of forgetting (deleting) data, which we’ll cover in more detail below.

# Setting a retention policy

The first thing to consider is defining a retention policy on a table, materialized view or database in Kusto. By doing so, Kusto automatically takes care of removing data beyond a certain age.

There are two parts to this policy:

-   SoftDeletePeriod. This works like an extra filter on your queries, so anything older than the given period is automatically excluded.
-   Recoverability, which is a Boolean value. If set to true, the data is recoverable for 14 days after the SoftDeletePeriod, although this requires you to create a support ticket with Microsoft.

Hence this means that data is physically retained at least for the SoftDeletePeriod + 14 days, but it could be longer. Microsoft currently provides no guarantees on how much longer it might physically retain the data. This might be a cause of concern if you want to use this to enforce compliance. It’s telling that Microsoft’s documentation says you might use this feature to reduce costs but does not mention compliance. You should therefore consult with a legal expert to verify that you can use this method in your situation if your objective is to be compliant with GDPR or other privacy laws.

# Using `.delete` and `.purge`

Kusto supports two commands to delete a set of rows: `.delete` and `.purge`. Both methods prevent deleted records from being recovered, regardless of any retention or recoverability settings. The deletion process is final and irreversible. It’s therefore a bit misleading that `.delete` is sometimes referred to as “soft-delete”, given that the data cannot be undeleted.

The `.delete` command acts like an extra filter to all your queries. It is therefore a fast operation that requires few system resources. Microsoft’s documentation is a bit unclear if, and when, the data will be purged from the underling storage. The main use case for the `.delete` command is to get rid of corrupted data. Say for example you’re using Kusto to analyze IoT data, you could use the `.delete` command to quickly remove data from a faulty device.

The `.purge` command on the other hand is meant to permanently delete rows from a table. Purge also doesn’t do that instantly, Microsoft’s documentation states that it may take “a couple of days”. Purging data requires a lot of system resources because large portions of the database will need to be rebuilt. You should expect a significant performance impact and additional usage costs.

See also: [Data purge - Azure Data Explorer \| Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/data-purge)

From a compliance perspective, `.delete` is like the retention policy mentioned above. You should consult with a legal expert before relying on this feature for compliance. Microsoft on the other hand clearly states that the `.purge` command is specifically designed with GDPR in mind.

The fact that `.purge` can take “a couple of days” to complete sounds a bit vague. The `.purge` command can however compute an estimation of the duration before you start the purge operation itself.


# Conclusion and recommendations

If you want Kusto to forget old data for compliance reasons, the safest bet – legally – is to rely on the `.purge` command. If you want to use `.delete` or rely on a retention policy, you should first consult with a legal expert.

Due to the performance and cost impact of `.purge`, you should use it sparingly, for example once every three months.

Best practices:

-   Design your ingestion process such that it already filters or obfuscates privacy-sensitive data. For example, if you use the Kusto SDK to write your own ingestion process, you can use a regular expression find and replace email addresses with something else. This way, you also don’t need to worry about removing this data later.
-   Consider designing your Kusto tables such that the privacy data is stored in separate tables from the other data. This way, you can keep the non-sensitive data for a much longer period, and fewer system resources are needed to purge rows from the tables with the sensitive data.

Make sure you thoroughly review Microsoft’s documentation on purging data.
