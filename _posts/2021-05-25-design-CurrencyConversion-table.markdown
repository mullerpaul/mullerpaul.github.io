# Why do we need an exchange rates table?
In order for our app to show money amounts and rates in currencies other than the original currency, we need a table which stores currency exchange rates.  We are as yet unsure if we'll persist the converted amounts in the DB, or join to this table during the data pull (NOT persisting the converted amounts; but instead compute them at extract time), or just send this exchange rate table to the front end so we can join to it there in Power BI memory.  But whichever way we choose, we'll need this Exchange Rates table and the data in it.

# Design considerations
## What data source can we use?
We have access to a utility database containing a table with daily conversion rates from various currencies to and from USD.  This is populated by an existing SSIS package from a 3rd party source, and its read into our client's OLTP databases by a scheduled job in each client environment.  We can pull data from there.
The conversion rate table is laid out like this:

|currency|date|toUSDrate|fromUSDrate|
|--|--|--|--|
|EUR|01-01-2021|1.2230|.81766|
|EUR|01-02-2021|1.2242|.81686|

When I saw the FromUSD and ToUSD rates separated like that I worried that this data shows a currency conversion service's bid-ask spread and that by using this data we were misrepresenting rates to our customers.  However, some checking the data revealed that in _nearly_ all the rows in the source table, the toUSD and FromUSD are in fact reciprocals of each other and are truly just different representations of the same thing.


## Which dates to use?
While the source data has a row for each non USD currency every day; we have been told that we can use monthly rates.  That is, we can use one rate for a whole month.  We discussed averaging the rate across all days in the month and then using that rate for any conversions in that month; but decided against it as its one more layer of complexity.  Instead, we decided to use the rate which exists on the first of the month for the whole duration of the month.  Its not going to be _exact_; but then again, no system which uses one rate for the whole month will be exact!  Using the rate on the first should be much easier to explain to customers and to troubleshoot than using a per-month average, max, or min.

## Which currencies to convert from?
We don't know which currencies any individual client will be using.  We could build a list of all the distinct currencies in use; but it will be simpler to just have data from all currencies which we have data for.  Since this data is the same source as the OLTP systems, clients can't be using currencies other than the ones included in this set anyways.

## Which currencies to convert to?
We have been provided a list of eight currencies our front end will use.  We need to be able to convert amounts in any currency to all of these eight currencies.

## Historical rates
We've settled on loading four years worth of conversion data.  We have three years of data in the system, and the extra time is for any dates which fall outside of the data range.  (data inclusion based on the 3 year window _may not_ be based on the same date attribute as we use for currency conversion.

## Will missing rows be OK?
NO!  Wherever in our process we end up using this data, we'll nearly certainly be doing some sort of "inner join" to get the conversion rate.  If we end up missing a row for a certain key (date, from currency, to currency), the results cannot include any data which may have existed for that key.  This is wrong results and would be a serious bug.  If our source data is incomplete for some key, I'd much rather have a row with a NULL conversion rate than no row at all.  If nothing else, its an easier problem to find and troubleshoot.  This issue could come up due to (usually erronious) future invoice dates or dates more than three years in the past, or by source currencies which are not in the conversion data - though I onder if that could happen.

## Data size
If we decide to do the join in PowerBI; it might be important to keep the amount of data as small as possible.  For example, to achieve real-time performance, PowerBI puts a lot of data into memory - and this can potentaily increase our cloud bill since we are billed by PowerBI memory usage.
Using only monthly data helps a lot here.  There will be only 12 rows per combination of year, from currency, and to currency instead of 365.  Also, converting to only 8 currencies will cut down the number of rows quite a bit.
Given that there are about 210 different from currencies in our data, we can expect to see around 80K rows in our table.

```
Rowcount = month count * from currency count * to currency count

48 * 200 * 8 = 76,800
```

With only four columns, this table will be pretty thin.  A date, two currency codes, and a number - I'd estimate 50 or 60 bytes per row.  Given SQL Server's 8Kb page size, we will probably get about 130-140 rows per page (assuming unindexed heap table - this may be a bad assumption) and that means the whole table could fit in 550-600 pages, under 5Mb.  Given the amount of data we are putting into PowerBI already, an additional 5Mb shouldn't pose too much of a problem!

