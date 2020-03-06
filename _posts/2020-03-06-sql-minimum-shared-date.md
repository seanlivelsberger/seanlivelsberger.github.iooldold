---
toc: true
description: How to find the minimum shared date between overlapping date ranges.
categories: [sql,snowflake,vertica,postgres]
---

# SQL: Minimum Shared Date Between Date Ranges

I recently encountered this scenario at work and thought the solution was worth highlighting.

In this scenario, I had two tables that held information on Users. One table contained the start and end dates of ranges for which the user had active saved payment information. The other table contained the start and end date ranges for which the user had an active shipping address saved. I needed to find the earliest date at which each user had both active payment information and an active shipping address. The user could be identified by a user ID field in each table.

The starting thought was to construct a table of all date values in the relevant range. One would then cross join this table to both tables looking for dates where the payment information and shipping address are both saved, then taking the minimum of those dates for each user. This could be a viable strategy for a small dataset, but when performed on millions of members and date ranges, the cross join operation would be inefficient. Also, maintaining the list of “relevant” dates is not something that can necessarily be automated. As such, if you wanted to perform this operation repeatedly in the future, you would have to manually maintain the date list.

However, we have a much more elegant and efficient solution to this problem shown in the query below.

```sql
SELECT
COALESCE(p.user_id,a.user_id) AS user_id
,MIN(CASE
         WHEN p.start_date BETWEEN a.start_date AND a.end_date
              OR a.start_date BETWEEN p.start_date AND p.end_date         
              THEN GREATEST(a.start_date,p.start_date)
END)
    FROM
    data_schema.saved_payment_date_ranges p
    INNER JOIN
    data_schema.saved_address_date_ranges a
    ON a.user_id = p.user_id
GROUP BY 1
```

The way this solution does most of the work in the MIN function statement. Specifically, the query joins all possible date ranges for each user. Through the case statement, it then scans each pairing looking for ones where there is some overlap in the date ranges. For that pairing, it then extracts the greatest of the two range start dates (the start of the overlap). For each user, it then uses the MIN aggregation function to pick their earliest overlap date.
