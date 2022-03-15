---
layout: post
title: Polymorphic Table Function for CSV (take 2)
date: 2021-12-28 20:30:00 +0500
categories: [oracle, sql, plsql]
tags: [oracle, sql, plsql]
---

# Introduction

In a prior blog post [Polymorphic Table Function to the Rescue?](https://lee-lindley.github.io/2021/12/27/PTF-Hard-Coding-Data)
I used a hammer to make a Polymorphic Table Function do what I wanted. I wanted a single clob as input
and multiple rows as output. I abused a PTF replication feature to get my way, but
it wasn't the right thing to do.

The general design pattern for a PTF is that it transforms rows from one result set into another, but
for the most part there is a one to one relationship on the number of rows (replication feature nothwithstanding).

There is a capability to specify how many output rows there are for any given input row using
*DBMS_TF.row_replication* procedure. The argument you provide is a table you populate with a
value for every row returned by *DBMS_TF.get_row_set* in that fetch call. The value can be 0 meaning
you do not return a value for that row. In the documentation
for *DBMS_TF.get_row_set* is an example section titled *Stack Polymorphic Table Function Example*
that shows a PTF named *stack* that does just that.

Although it is possible to do what I originally intended, it still seems klunky.

In order to conform to the more common PTF design pattern I broke my problem into two parts:

- split a clob into lines
- split each line into fields and output as Typed column values

It requires the user to do two steps, but it is a cleaner design that fits the pattern of other PTF functions.

The first part is achieved with an ordinary Pipelined table function that takes a CLOB as input
and splits it into lines. It respects the CSV format quoting mechanism for protecting newlines in the data,
so it is a little more complex than you might think.

The second part is achieved much as I did in the above mentioned blog post, but using the CSV row data as
TABLE input rather than reading the CLOB directly.

From the README.md on github:

## csv_to_table

Given a set of rows containing CSV strings, or a CLOB containing multiple lines of CSV strings,
split the records into component column values and return a resultset
that appears as if it was read from 
a table in your schema (or a table to which you have SELECT priv).
We provide a Polymorphic Table Function for your use to achieve this.

You can either start with a set of CSV strings as rows, or with a CLOB
that contains multiple lines, each of which are a CSV record. Note that 
this is a full blown CSV parser that should handle any records that comply
with RFC4180 (See https://www.loc.gov/preservation/digital/formats/fdd/fdd000323.shtml).

```sql
    FUNCTION ptf(
        p_tab           TABLE
        ,p_table_name   VARCHAR2
        ,p_columns      VARCHAR2 -- csv list -- could be COLUMNS() construct instead
        ,p_date_fmt     VARCHAR2 DEFAULT NULL -- uses nls_date_format if null
        ,p_separator    VARCHAR2 DEFAULT ','
    ) RETURN TABLE
    PIPELINED ROW POLYMORPHIC USING csv_to_table_pkg
    ;

    -- public type to be returned by split_clob_to_lines PIPE ROW function
    TYPE t_csv_row_rec IS RECORD(
        s   VARCHAR2(4000)  -- the csv row
        ,rn NUMBER          -- line number in the input
    );
    TYPE t_arr_csv_row_rec IS TABLE OF t_csv_row_rec;

    --
    -- split a clob into a row for each line.
    -- Handle case where a "line" can have embedded LF chars per RFC for CSV format
    -- Throw out completely blank lines (but keep track of line number)
    --
    FUNCTION split_clob_to_lines(p_clob CLOB)
    RETURN t_arr_csv_row_rec
    PIPELINED
    ;
```
## Example

To continue with my example from before I have a demo table from which to grab column types:

    CREATE TABLE my_table_name(id number, msg VARCHAR2(1024), dt DATE);

Here is the query using the revised package:

```sql
    WITH R AS (
        SELECT *
        FROM csv_to_table_pkg.split_clob_to_lines(
q'!23, "this contains a comma (,)", 06/30/2021
47, "this contains a newline (
)", 01/01/2022

73, and we can have backwacked comma (\,),
92, what about backwacked dquote >\"<?, 12/28/2021
!'
        )
    ) SELECT *
    FROM csv_to_table_pkg.ptf(R, 'my_table_name', 'id, msg, dt', 'MM/DD/YYYY')
    ;
```
The blank line is ignored; however, the line numbers are maintained through the process so that
errors/problems can be reported.

The resultset is as expected. Note the NULL date value on line 4. Here is a JSON representation:

	{
	  "results" : [
	    {
	      "columns" : [
	        {
	          "name" : "ID",
	          "type" : "NUMBER"
	        },
	        {
	          "name" : "MSG",
	          "type" : "VARCHAR2"
	        },
	        {
	          "name" : "DT",
	          "type" : "DATE"
	        }
	      ],
	      "items" : [
	        {
	          "id" : 23,
	          "msg" : "this contains a comma (,)",
	          "dt" : "06/30/2021"
	        },
	        {
	          "id" : 47,
	          "msg" : "this contains a newline (\n)",
	          "dt" : "01/01/2022"
	        },
	        {
	          "id" : 73,
	          "msg" : "and we can have backwacked comma (,)",
	          "dt" : ""
	        },
	        {
	          "id" : 92,
	          "msg" : "what about backwacked dquote >\\\"<?",
	          "dt" : "12/28/2021"
	        }
	      ]
	    }
	  ]
	}

As I mentioned this tracks the input line numbers including the blank lines that it discards.
Here is an example of an error in the date conversion on the 4th row:

```sql
    WITH R AS (
        SELECT *
        FROM csv_to_table_pkg.split_clob_to_lines(
q'!23, "this contains a comma (,)", 06/30/2021
47, "this contains a newline (
)", 01/01/2022

73, and we can have backwacked comma (\,),12/24/
92, what about backwacked dquote >\"<?, 12/28/2021
!'
        )
    ) SELECT *
    FROM csv_to_table_pkg.ptf(R, 'my_table_name', 'id, msg, dt', 'MM/DD/YYYY')
    ;
```

The error text that it raises is:

    ORA-20202: line number:4 col:3
    Line: 73, and we can have backwacked comma (\,),12/24
    has Oracle error: ORA-01840: input value not long enough for date format
    ORA-06512: at "LEE.CSV_TO_TABLE_PKG", line 283
    ORA-06512: at line 1

## Getting the Code

You can find the package on my github site under repository [plsql_utilities](https://github.com/lee-lindley/plsql_utilities).
For the moment it is in the branch named [parse_csv](https://github.com/lee-lindley/plsql_utilities/tree/parse_csv),
but I expect to merge it to main in the not too distant future.

Hope it was helpful.
