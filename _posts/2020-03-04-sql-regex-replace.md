---
toc: true
description: How to manage messy strings when you want quality numeric data.
categories: [sql]
---

# SQL: REGEX_REPLACE

Imagine that you are working with a particularly messy dataset. In this scenario you have a field available to you that is of a string-based data type. However, the data in this field is supposed to represent some set of numeric values (e.g. price, quantity, etc.).  

This is a frequently encountered scenario when working with messy data. Assuming the values of this field are “clean,” one can simply coerce the field into the desired numeric data type with a statement similar to that of the one shown below:  

```sql
SELECT
messy_numeric_field::DOUBLE as actual_numeric_field
   FROM
   data_table
```

Now, if the values are not “clean,” then if one wants to create a numeric field, one must creatively clean the data.

There are many ways in which the data can be unclean. For this post, I want to focus on the scenario where the alleged numeric field somehow has come to possess non-numeric characters amidst the numeric characters. Below are some examples of this case.

```sql
6,000.00
$500
[33]
```

If one were to try to coerce these values to a numeric format, the SQL database would throw an error. In the first example, the SQL engine will not understand how to interpret the comma. In the second example, the SQL engine will clearly not know how to interpret the currency denomination. Lastly, in the third example, the SQL engine will not know what to do with the brackets.
One strategy for managing these uninterpretable characters would be to do a form of replacement or series of replacements using the REPLACE function. Below, I have drawn up the usage of this function that would create a field that could be successfully coerced as a numeric data type.

```sql
SELECT
REPLACE(REPLACE(REPLACE(REPLACE(messy_numeric_field,',',''),'usd',''),'[',''),']','')::DOUBLE as actual_numeric_field
   FROM
   data_table
```

As one builds this sort of fix, it becomes evident that this solution is not scalable. One will have to constantly manage this query for any new odd characters that appear in the base field.

In order to manage this problem in a scalable manner, one should instead rely on the regular expressions enabled replace function, REGEXP_REPLACE. With regular expressions, one can search for patterns within text as opposed to a specific value or sequence of values. To solve the problem that we are facing, one would want to use the query below.

```sql
SELECT
REGEXP_REPLACE(messy_numeric_field,'[^0-9.]','')::DOUBLE as actual_numeric_field
   FROM
   data_table
```

This use of the REGEXP_REPLACE function operates by searching for any characters that are not a number or not a period. Once it finds a character that fits that pattern, it replaces it with the empty string, thereby removing any unwanted characters. One can then proceed with the data type coercion with a cleansed field of values.

Based on my past experiences, one additional recommendation would be to add a NULLIF wrapper on the above usage of REGEXP_REPLACE. The addition is featured in the update query below.

```sql
SELECT
NULLIF(REGEXP_REPLACE(messy_numeric_field,'[^0-9.]',''),'')::DOUBLE as actual_numeric_field
   FROM
   data_table
```

The usage of NULLIF here adds an extra layer of cleanliness to your data clean up. It is specifically relevant due to the fact that the replace action is insert the empty string. If a messy field value consists exclusively of non-numeric characters (e.g. just a decimal) it would leave an empty string final value after our replacement operation. This is less that ideal for two reasons. For one, depending on the SQL engine, the empty string may actually not convert to a numeric in the coercion operation. More importantly though, the existence of empty string numeric values may not be perceptible if the dataset is sufficiently large. If one then goes to count non-null values under the assumption that values are either legitimate or null, one could potentially come away with an incorrect understanding of their data.