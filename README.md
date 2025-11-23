# SQL tips and tricks

[![Stand With Ukraine](https://raw.githubusercontent.com/vshymanskyy/StandWithUkraine/main/badges/StandWithUkraine.svg)](https://stand-with-ukraine.pp.ua)

[![Ceasefire Now](https://badge.techforpalestine.org/default)](https://techforpalestine.org/learn-more)

A (somewhat opinionated) list of SQL tips and tricks that I've picked up over the years.

There's so much you can do with SQL but I've focused on what I find most useful in my day-to-day work as a data analyst and what 
I wish I had known when I first started writing SQL.

Please note that some of these tips might not be relevant for all RDBMSs.

## Table of contents

### Formatting/readability

- [Use a leading comma to separate fields](#use-a-leading-comma-to-separate-fields)
-  [Use a dummy value in the `WHERE` clause](#use-a-dummy-value-in-the-where-clause)
- [Indent your code](#indent-your-code)
- [Consider CTEs when writing complex queries](#consider-ctes-when-writing-complex-queries)
- [Comment your code](#comment-your-code)
- [Simplify joins with USING](#simplify-joins-with-using)

### Data wrangling
- [Anti-joins will return rows from one table that have no match in another table](#anti-joins-will-return-rows-from-one-table-that-have-no-match-in-another-table)
- [Use `QUALIFY` to filter window functions](#use-qualify-to-filter-window-functions)
- [You can (but shouldn't always) `GROUP BY` column position](#you-can-but-shouldnt-always-group-by-column-position)
- [Create a grand total with `GROUP BY ROLLUP`](#create-a-grand-total-with-group-by-rollup)
- [Use `EXCEPT` to find the difference between two tables](#use-except-to-find-the-difference-between-two-tables)

### Performance

- [`NOT EXISTS` is faster than `NOT IN` if your column allows `NULL`](#not-exists-is-faster-than-not-in-if-your-column-allows-null)
- [Implicit casting will slow down (or break) ](#implicit-casting-will-slow-down-or-break-your-query)

### Common mistakes

- [Be aware of how `NOT IN` behaves with `NULL` values](#be-aware-of-how-not-in-behaves-with-null-values)
- [Avoid ambiguity when naming calculated fields](#avoid-ambiguity-when-naming-calculated-fields)
- [Always specify which column belongs to which table](#always-specify-which-column-belongs-to-which-table)

### Miscellaneous

- [Understand the order of execution](#understand-the-order-of-execution)
- [Read the documentation (in full)](#read-the-documentation-in-full)
- [Use descriptive names for your saved queries](#use-descriptive-names-for-your-saved-queries)


## Formatting/readability
### Use a leading comma to separate fields

Use a leading comma to separate fields in the `SELECT` clause rather than a trailing comma.

- Clearly defines that this is a new column vs code that's wrapped to multiple lines.

- Visual cue to easily identify if the comma is missing or not. Varying line lengths makes it harder to determine.
 
```SQL
SELECT
employee_id
, employee_name
, job
, salary
FROM employees
;
```

- Also use a leading `AND` in the `WHERE` clause, for the same reasons (following tip demonstrates this).

-----

### **Use a dummy value in the WHERE clause**
Use a dummy value in the `WHERE` clause so you can easily comment out conditions when testing or tweaking a query.

```SQL
/*
If I want to comment out the job
condition the following query
will break:
*/
SELECT *
FROM employees
WHERE
--job IN ('Clerk', 'Manager')
AND dept_no != 5
;

/*
With a dummy value there's no issue.
I can comment out all the conditions
and 1=1 will ensure the query still runs:
*/
SELECT *
FROM employees
WHERE 1=1
-- AND job IN ('Clerk', 'Manager')
AND dept_no != 5
;
```

-----

### Indent your code
Indent your code to make it more readable to colleagues and your future self.

Opinions will vary on what this looks like, so be sure to follow your company/team's guidelines or, if that doesn't exist, go with whatever works for you.

You can also use an online formatter like [poorsql](https://poorsql.com/) or a linter like [sqlfluff](https://github.com/sqlfluff/sqlfluff).

``` SQL
SELECT
-- Bad:
vc.video_id
, CASE WHEN meta.GENRE IN ('Drama', 'Comedy') THEN 'Entertainment' ELSE meta.GENRE END as content_type
FROM video_content AS vc
INNER JOIN metadata ON vc.video_id = metadata.video_id
;

-- Good:
SELECT 
vc.video_id
, CASE 
	WHEN meta.GENRE IN ('Drama', 'Comedy') THEN 'Entertainment' 
	ELSE meta.GENRE 
END AS content_type
FROM video_content
INNER JOIN metadata 
	ON video_content.video_id = metadata.video_id
;
```
-----

### Consider CTEs when writing complex queries
For longer than I'd care to admit I would nest inline views, which would lead to
queries that were hard to understand, particularly if revisited after a few weeks.

If you find yourself nesting inline views more than 2 or 3 levels deep, 
consider using common table expressions, which keep your code more organised and readable and supports reusability and debugging.

```SQL
-- Using inline views:
SELECT 
vhs.movie
, vhs.vhs_revenue
, cs.cinema_revenue
FROM 
    (
    SELECT
    movie_id
    , SUM(ticket_sales) AS cinema_revenue
    FROM tickets
    GROUP BY movie_id
    ) AS cs
    INNER JOIN 
        (
        SELECT 
        movie
        , movie_id
        , SUM(revenue) AS vhs_revenue
        FROM blockbuster
        GROUP BY movie, movie_id
        ) AS vhs
        ON cs.movie_id = vhs.movie_id
;

-- Using CTEs:
WITH cinema_sales AS 
    (
        SELECT 
        movie_id
        , SUM(ticket_sales) AS cinema_revenue
        FROM tickets
        GROUP BY movie_id
    ),
    vhs_sales AS
    (
        SELECT 
        movie
        , movie_id
        , SUM(revenue) AS vhs_revenue
        FROM blockbuster
        GROUP BY movie, movie_id
    )
SELECT 
vhs.movie
, vhs.vhs_revenue
, cs.cinema_revenue
FROM cinema_sales AS cs
    INNER JOIN vhs_sales AS vhs
    ON cs.movie_id = vhs.movie_id
;
```

-----
### Comment your code
While in the moment you know why you did something, if you revisit
the code weeks, months or years later you might not remember.

In general you should strive to write comments that explain why you did something, not how.

Your colleagues and future self will thank you!

```SQL
SELECT 
video_content.*
FROM video_content
    LEFT JOIN archive
    ON video_content.video_id = archive.video_id
WHERE 1=1
-- Need to filter out as new CMS cannot process archive video formats:
AND archive.video_id IS NULL
;
```

-----
### Simplify joins with `USING`

If you're joining using a column with the same name in two tables you can use `USING` to
simplify your join:

```SQL
-- USING:
SELECT * 
FROM album 
	INNER JOIN artist 
	USING (artistid)

-- Traditional ON clause:
SELECT * 
FROM album 
	INNER JOIN artist 
	ON album.artistid = artist.ArtistId 
```

The other benefit of `USING` is that the column in common between the two tables is deduplicated, with only one column returned in the result set.

This means that there is no ambiguity, unlike the following query which would throw a `ambiguous column name` error as the database would not be sure
which column to which you are referring if you are using the `ON` clause:

```SQL
SELECT ArtistId -- Which table column?
FROM album
	INNER JOIN artist 
	ON album.artistid = artist.ArtistId
```

## Data wrangling

### Anti-joins will return rows from one table that have no match in another table

Use anti-joins when you want to return rows from one table that don't have a match in another table.

For example, you only want video IDs of content that hasn't been archived.

There are multiple ways to do an anti-join:

```SQL 
-- Using a LEFT JOIN:
SELECT 
vc.video_id
FROM video_content AS vc
    LEFT JOIN archive
    ON vc.video_id = archive.video_id
WHERE 1=1
AND archive.video_id IS NULL -- Any rows with no match will have a NULL value.
;

-- Using NOT IN/subquery:
SELECT 
video_id
FROM video_content
WHERE 1=1
AND video_id NOT IN (SELECT video_id FROM archive) -- Be mindful of NULL values.

-- Using NOT EXISTS/correlated subquery:
SELECT 
video_id
FROM video_content AS vc
WHERE 1=1
AND NOT EXISTS (
        SELECT 1
        FROM archive AS a
        WHERE a.video_id = vc.video_id
        )

```

Note that I advise against using `NOT IN` - see [this tip](#be-aware-of-how-not-in-behaves-with-null-values).

-----
### Use `QUALIFY` to filter window functions

`QUALIFY` lets you filter the results of a query based on a window function, meaning you don't need
to use an inline view to filter your result set and thus reducing the number of lines of code.

For example, if I want to return the top 10 markets per product I can use
`QUALIFY` rather than an inline view:

```SQL
-- Using QUALIFY:
SELECT 
product
, market
, SUM(revenue) AS market_revenue 
FROM sales
GROUP BY product, market
QUALIFY DENSE_RANK() OVER (PARTITION BY product ORDER BY SUM(revenue) DESC)  <= 10
ORDER BY product, market_revenue
;

-- Without QUALIFY:
SELECT 
product
, market
, market_revenue 
FROM
(
SELECT 
product
, market
, SUM(revenue) AS market_revenue
, DENSE_RANK() OVER (PARTITION BY product ORDER BY SUM(revenue) DESC) AS market_rank
FROM sales
GROUP BY product, market
)
WHERE market_rank  <= 10
ORDER BY product, market_revenue
;
```

Unfortunately it looks like `QUALIFY` is only available in the big data warehouses (Snowflake, Amazon Redshift, Google BigQuery) but I had to include this because it's so useful.

-----
### You can (but shouldn't always) `GROUP BY` column position

Instead of using the column name, you can `GROUP BY` or `ORDER BY` using the
column position.

- This can be useful for ad-hoc/one-off queries, but for production code
you should always refer to a column by its name.

```SQL
SELECT 
dept_no
, SUM(salary) AS dept_salary
FROM employees
GROUP BY 1 -- dept_no is the first column in the SELECT clause.
ORDER BY 2 DESC
;
```

-----
### Create a grand total with `GROUP BY ROLLUP`
Creating a grand total (and/or sub-total) row is possible thanks to `GROUP BY ROLLUP`.

For example, if you've aggregated a company's employees salary per department you 
can use `GROUP BY ROLLUP` to create a grand total that applies your aggregate functions as if 
the specified grouping hadn't been applied (thus creating a grand total row).

The [Transact-SQL documentation](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-group-by-transact-sql?view=sql-server-ver17) explains `GROUP BY ROLLUP` well:

_"Creates a group for each combination of column expressions. In addition, it 'rolls up' the results into subtotals and grand totals. To do this, it moves from right to left decreasing the number of column expressions over which it creates groups and the aggregation(s)."_

You may want to apply `COALESCE`, as below, to ensure the total row is labelled as such.

```SQL
SELECT 
COALESCE(dept_no, 'Total') AS department_number
, SUM(salary) AS dept_salary
FROM employees
GROUP BY ROLLUP(dept_no)
ORDER BY dept_salary -- Be sure to order by this column to ensure the Total appears last/at the bottom of the result set.
;
```

-----
### Use `EXCEPT` to find the difference between two tables

`EXCEPT` returns rows from the first query's result set that don't appear in the second query's result set.

```SQL
/*
Miles Davis will be returned from
this query
*/
SELECT artist_name
FROM artist
WHERE artist_name = 'Miles Davis'
EXCEPT 
SELECT artist_name
FROM artist
WHERE artist_name = 'Nirvana'
;

/*
Nothing will be returned from this
query as 'Miles Davis' appears in
both queries' result sets.
*/
SELECT artist_name
FROM artist
WHERE artist_name = 'Miles Davis'
EXCEPT 
SELECT artist_name
FROM artist
WHERE artist_name = 'Miles Davis'
;
```

You can also utilise `EXCEPT` with `UNION ALL` to verify whether two tables have the same data.

If no rows are returned the tables are identical - otherwise, what's returned are the rows causing the difference:

```SQL
/* 
The first query will return rows from
employees that aren't present in
department.

The second query will return rows from
department that aren't present in employees.

The UNION ALL will ensure that the
final result set returned combines
all of these rows so you know
which rows are causing the difference.
*/
(
SELECT 
id
, employee_name
FROM employees
EXCEPT 
SELECT 
id
, employee_name
FROM department
)
UNION ALL 
(
SELECT 
id
, employee_name
FROM department
EXCEPT
SELECT 
id
, employee_name
FROM employees
)
;

```

## Performance

### `NOT EXISTS` is faster than `NOT IN` if your column allows `NULL`

`NOT IN` is usually slower than using `NOT EXISTS`, if the values/column you're comparing against allows `NULL`.

I've experienced this when using Snowflake and the PostgreSQL Wiki explicitly [calls this out](https://wiki.postgresql.org/wiki/Don't_Do_This#Don.27t_use_NOT_IN):

*"...NOT IN (SELECT ...) does not optimize very well."*

Aside from being slow, using `NOT IN` will not work as intended if there is a `NULL` in the values being compared against - see [tip 11](#be-aware-of-how-not-in-behaves-with-null-values).

Why include this tip if `NOT IN` doesn't work with `NULL` values anyway?

Well just because a column allows `NULL` values does not mean there **are** any `NULL` values present and if you're working with a table that you cannot alter you'll want to use `NOT EXISTS` to speed up your query.

-----

### Implicit casting will slow down (or break) your query

If you specify a value with a different data type than a column's, your database may automatically (implicitly) convert the value.

For example, let's say the `video_id` column has a string data type and you specify an integer in the `WHERE` clause:

```SQL
SELECT video_name
FROM video_content 
 -- Behind the scenes the database will implicitly attempt to convert the video_id column to an integer:
WHERE video_id = 200050
```

There's a couple of problems with relying on implicit casting:

1) An error may be thrown if the implicit conversion isn't possible - for example, if one of the video IDs has a string value of _'abc2000'_

2) \*Your query may be slower, due to the additional work of converting each value to the specified data type.

Instead, use the same data type as the column you're operating on (`WHERE video_ID = '200050'`) or, to avoid errors, use a function like [`TRY_TO_NUMBER`](https://docs.snowflake.com/en/sql-reference/functions/try_to_decimal) that 
will attempt the conversion but handle any errors:

```SQL
SELECT video_name
FROM video_content 
 -- This won't result in an error:
WHERE TRY_TO_NUMBER(video_id) = 200050
```

\* Note that this depends on the size of the dataset being operated on. 

## Common mistakes

### Be aware of how `NOT IN` behaves with `NULL` values

`NOT IN` doesn't work if `NULL` is present in the values being checked against. As `NULL` represents Unknown the SQL engine can't verify that the value being checked is not present in the list.
- Instead use `NOT EXISTS`.

``` SQL
INSERT INTO departments (id)
VALUES (1), (2), (NULL);

-- Doesn't work due to NULL:
SELECT * 
FROM employees 
WHERE department_id NOT IN (SELECT DISTINCT id from departments)
;

-- Solution.
SELECT * 
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 
    FROM departments d 
    WHERE d.id = e.department_id
)
;
```

-----
### Avoid ambiguity when naming calculated fields

When creating a calculated field, naming it the same as an existing column can lead to unexpected behaviour.

Note [Snowflake's documentation](https://docs.snowflake.com/en/sql-reference/constructs/group-by) on the topic:

*"If a GROUP BY clause contains a name that matches both a column name and an alias, then the GROUP BY clause uses the column name."*

For example you might expect the following to return 2 rows but what's actually returned is 3 rows:

```SQL
CREATE TABLE products (
    product VARCHAR(50) NOT NULL,
    revenue INT NOT NULL
)
;

INSERT INTO products (product, revenue)
VALUES 
    ('Shark', 100),
    ('Robot', 150),
    ('Racecar', 90);

SELECT 
LEFT(product, 1) AS product -- Returns the first letter of the product value.
, MAX(revenue) as max_revenue
FROM products
GROUP BY product
;
```

|PRODUCT|MAX_REVENUE|
|-------|------------|
|S|100|
|R|150|
|R|90|

What's happened is that the `LEFT` function has been applied after the product column has been 
grouped and aggregation applied.

The solution is to use a unique alias or be more explicit in the `GROUP BY` clause: 

```SQL
-- Solution option 1:
SELECT 
LEFT(product, 1) AS product_letter
, MAX(revenue) AS max_revenue
FROM products
GROUP BY product_letter
;

-- Solution option 2:
SELECT 
LEFT(product, 1) AS product,
, MAX(revenue) AS max_revenue
FROM products
GROUP BY LEFT(product, 1)
;
```

Result:

|PRODUCT_LETTER|MAX_REVENUE|
|--------------|-----------|
|S|100|
|R|150|


Assigning an alias to a calculated field can also be problematic when it comes to window functions.

In this example the `CASE` statement is being applied AFTER the window function has executed:

```SQL
/*
The window function will rank the 'Robot' product as 1 when it should be 3.
*/
SELECT 
product
, CASE product WHEN 'Robot' THEN 0 ELSE revenue END AS revenue
, RANK() OVER (ORDER BY revenue DESC)
FROM products
;
```

Our earlier solutions apply:

```SQL
/*
Solution option 1 (note this might not work in all RDBMS, in which case use the other solution):
*/
SELECT 
product
, CASE product WHEN 'Robot' THEN 0 ELSE revenue END AS updated_revenue
, RANK() OVER (ORDER BY updated_revenue DESC)
FROM products
;

-- Solution option 2:
SELECT 
product
, CASE product WHEN 'Robot' THEN 0 ELSE revenue END AS revenue
, RANK() OVER (ORDER BY CASE product WHEN 'Robot' THEN 0 ELSE revenue END DESC)
FROM products
;
```

My advice - use a unique alias when possible to avoid confusion.

-----
### Always specify which column belongs to which table

When you have complex queries with multiple joins, it pays to be able to 
trace back an issue with a value to its source. 

Additionally, your RDBMS might raise an error if two tables share the same
column name and you don't specify which column you are using.

```SQL
SELECT 
vc.video_id
, vc.series_name
, metadata.season
, metadata.episode_number
FROM video_content AS vc 
    INNER JOIN video_metadata AS metadata
    ON vc.video_id = metadata.video_id
;
```

## Miscellaneous

### Understand the order of execution
If I had to give one piece of advice to someone learning SQL, it'd be to understand the order of 
execution (of clauses). It will completely change how you write queries. This [blog post](https://blog.jooq.org/a-beginners-guide-to-the-true-order-of-sql-operations/) is a fantastic resource for learning.

-----
### Read the documentation (in full)
Using Snowflake I once needed to return the latest date from a list of columns 
and so I decided to use `GREATEST()`.

What I didn't realise was that if one of the
arguments is `NULL` then the function returns `NULL`. 

If I'd read the documentation in full I'd have known! In many cases it can take just a minute or less to scan
the documentation and it will save you the headache of having to work
out why something isn't working the way you expected:

```SQL
/*
If I'd read the documentation
further I'd also have realised
that my solution to the NULL
problem with GREATEST()... 
*/

SELECT COALESCE(GREATEST(signup_date, consumption_date), signup_date, consumption_date);

/*
... could have been solved with the
following function:
*/
SELECT GREATEST_IGNORE_NULLS(signup_date, consumption_date);
```

-----
### Use descriptive names for your saved queries

There's almost nothing worse than not being able to find a query you need to re-run/refer back to.

Use a descriptive name when saving your queries so you can easily find what you're looking for.

I usually will write the subject of the query, the month the query was ran and the name of the requester (if they exist).
For example: `Lapsed users analysis - 2023-09-01 - Olivia Roberts`
