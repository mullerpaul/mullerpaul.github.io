## Learning the Kusto query language
Kusto is the query language used to query Azure data in Azure Data explorer and Azure monitor.  Its also used to produce data for tiles in azure dashboards, which is why we needed it.
Kusto is sometimes acronymized to KQL. Unfortunately, that acronym is already overloaded with Kibana Query Language, and Kafka Query Language!

### Similarities to SQL
People used to using SQL should be able to adjust to kusto relatively easily. Another concept which will help in learning kusto is the pipelining of unix commands. The way the output of one command becomes the input of the next command is very similar to how you build up a kusto query.

Here is a table comparing some basic kusto syntaxes to what they are called in SQL:
| **kusto** | **SQL**       |
|---------|-------------|
| project | Like a list of columns in the SELECT list.  A comma separated list of attributes which are passed to the next query step. |
| extend | Used to create a new column which is passed on to the next step. |
| summarize | Like GROUP BY, except both the grouping key AND the aggregate functions are specified here. |
| sort | Like ORDER BY |
| take (n) | Like LIMIT (n).  take 5 will return 5 rows.  These are non-deterministic unless you also specified sort |  
| top (n) | Syntactic sugar combining the sort and take commands into one |
| bin (<column>, <interval>) | Like FLOOR, or TRUNC usage in Oracle for datetimes.  This operator is very useful in summarize clauses when you want to group all data by hour or by day |
| ago(<interval>) | used for comparing a datetime field to now.  Like SYSDATE - 1 for example. |
| count() | COUNT(*) aggregate function |
| dcount(<column>) | COUNT(DISTINCT <column>) aggregate function |
| countif(<expression>) | COUNT(CASE WHEN <expression> THEN 'x' END) aggregate function |
| serialize | Conceptually like the ORDER BY clause inside the analytic clause of an analytic function.  Needed if you want to use functions like prev, next, or row_number. Incidentally, those are the only analytic functions available in the language. To do things like an analytic MAX, you'll have to self-join instead.  |


Examples:
```
customEvents |
   where timestamp >= ago(3d) |
   extend siteName = tostring(customDimensions.sitename) |
   project name, siteName |
   summarize eventCount = count() by siteName, name
```

The where line selects only rows from customEvents table where the timestamp is less than three days ago.
The extend line pulls data out of a specific attribute inside the JSON customDimentions field, casts it to a string, and names it "siteName". The project line limits the attributes to only those we need for the final result, which is a good idea before doing a memory-intensive aggregation on the next line.  You can think of the summarize line as both a `GROUP BY siteName, name` and a `COUNT(*) AS eventCount` in the SELECT.

Here is another example of a summarize clause, this time with two aggregate functions.

```
summarize eventCount = count(), eventTypeCount = dcount(eventType) by siteName
```
That groups by the siteName column, creates columns for the aggregate functions COUNT(*) and COUNT(DISTINCT eventType), and also names those columns.

Here is an example with the datatable operator and the serialize clause

```
datatable (name:string, eventTimestamp: datetime, SessionID: string )
 ["startEvent", "2020-Sep-21 09:31", "a",
  "startEvent", "2020-Sep-21 09:39", "c",
  "startEvent", "2020-Sep-21 09:41", "b",
  "startEvent", "2020-Sep-21 09:42", "a",
  "endEvent", "2020-Sep-21 09:44", "b",
  "endEvent", "2020-Sep-21 09:45", "a",
  "startEvent", "2020-Sep-21 09:47", "a",
  "endEvent", "2020-Sep-21 09:51", "a",
  "startEvent", "2020-Sep-21 09:55", "a",
  "startEvent", "2020-Sep-21 09:41", "c",
  "startEvent", "2020-Sep-21 09:44", "c",
  "endEvent", "2020-Sep-21 09:52", "d",
  "endEvent", "2020-Sep-21 09:51", "d"
 ]
| sort by SessionID, eventTimestamp asc
| serialize StartCycleFlag = iif(name == "startEvent" and ( next(name)=="endEvent" or SessionID != next(SessionID) ), 1, 0),
            CycleEndTime = case(next(name)=="endEvent", next(eventTimestamp) - eventTimestamp, time(null))
```

The datatable operator is a handy way to build your own data.  The serialize clause allows the use of order-dependent functions - in this case, `next`. Notice that the order used by `next` is the order specified by the `sort` clause.  I have not found a way to have multiple analytic functions with different order clauses.  Also notice that like Oracle and SQLserver, you can subtract two datetimes to get an interval.  Also notice the `time(null)` syntax to get a NULL of time datatype.  `iif` and `case` require that all arguments be the same datatype.

### Gotchas
+ In kusto, column names are case sensitive.  Also, operators and function names are case sensitive.  I spent quite a while trying to understand why I couldn't get a complex boolean expression with `AND` and `OR` to work before I realized they should be `and` and `or`.
+ People used to SQL might have a hard time remembering to put a pipe between clauses
+ kusto can only summarize by columns of certain data types. You might have to cast your grouping column to a different data type before the summarize clause.
+ kusto has no statistics about data sizes/cardinalities, and what passes as its optimizer is really just a rules based approach which relies on the query developer writing the queries in an optimal way. Read the best practices link below so you have a chance of writing your queries in a way that provides optimal performance.

### kusto clients
To run and develop queries, I've been using the azure portal.  I search for the App Insights resource, go to its page, and go its "logs" tab.  Its a nice web-based interface where you can submit arbitrary statements.  It has syntax checking and query history.
However, I am intrigued by another client called kusto.explorer.  Its a desktop application and it is supposed to be able to convert T-SQL queries to equivilant kusto.  I've downloaded it and installed it; but have not been able to connect to an application insights instance yet.

### Resources
Language reference:
- https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/
- https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/
- https://blog.jeremylikness.com/blog/kusto-azure-data-explorer-log-analytics/
- https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/best-practices

For kusto.explorer:
- https://docs.microsoft.com/en-us/azure/data-explorer/kusto/api/tds/t-sql
- https://docs.microsoft.com/en-us/azure/data-explorer/kusto/tools/kusto-explorer
- https://docs.microsoft.com/en-us/azure/data-explorer/query-monitor-data

