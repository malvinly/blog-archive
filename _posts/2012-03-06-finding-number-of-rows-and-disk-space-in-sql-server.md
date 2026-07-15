---
layout: post
title: "Finding number of rows and disk space in SQL Server"
date: 2012-03-06 06:24:55 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/03/06/finding-number-of-rows-and-disk-space-in-sql-server/
---

To see how many rows a table contains or how much disk space a table is using, we can use the system stored procedure [`sp_spacedused`](http://msdn.microsoft.com/en-us/library/ms188776.aspx "sp_spaceused") to quickly return information about a table or the database.

For a single table, we can pass the table name to the stored procedure:

[![]({{ site.baseurl }}/assets/images/2012/03/table.png "table")]({{ site.baseurl }}/assets/images/2012/03/table.png)

We can execute the stored procedure without passing a table name to see how much disk space the entire database is using:

[![]({{ site.baseurl }}/assets/images/2012/03/database.png "database")]({{ site.baseurl }}/assets/images/2012/03/database.png)

We can combine this system stored procedure with the undocumented stored procedure `sp_msforeachtable` to view the number of rows and disk space usage for all tables in the current database:

[![]({{ site.baseurl }}/assets/images/2012/03/all_tables.png "all_tables")]({{ site.baseurl }}/assets/images/2012/03/all_tables.png)

However, we have to keep in mind that the numbers being reported by `sp_spaceused` may not be immediately accurate after an index or table has been dropped, a table has been truncated, etc... since the information is not maintained in real time. The command [`dbcc updateusage`](http://msdn.microsoft.com/en-us/library/ms188414.aspx) may be execute to update the information for a single table or the entire database. Similarly, the `sp_spaceused` stored procedure may be executed with an additional parameter to update the information for a single table or the entire database:

{% raw %}
```sql
-- Single table
sp_spaceused @objname = 'Person.Address', @updateusage = 'true'

-- Entire database
sp_spaceused @updateusage = ‘true’
```
{% endraw %}

Although we've executed the update commands above, the row counts may still be inaccurate. The commands above only updates the disk space usage information, but not the row count information stored in the table `sysindexes`. We may execute the command `dbcc updateusage` with the `count_rows` argument to update `sysindexes`:

{% raw %}
```sql
-- Single table
dbcc updateusage ('AdventureWorks', 'Person.Address') with count_rows

-- Entire database
dbcc updateusage (0) with count_rows
```
{% endraw %}
