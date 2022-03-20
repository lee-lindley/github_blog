---
layout: post
title: CSV Clob and Private Temporary Table
date: 2022-01-23 20:30:00 +0500
categories: [oracle, sql, plsql, perl]
tags: [oracle, sql, plsql, csv, perl]
---
# Deploying Table Data Using CSV 

I have been obsessing about parsing CSV data with a focus on the use case of 
code promotion through Continuous Improvement/Devops process. The current method of
deploying a bunch of single INSERT statements rubs me the wrong way. It is ridiculous.

I get the feeling I'm the only one who cares, but hey, I'm the one who matters!

In [my last post on the topic](https://lee-lindley.github.io/oracle/sql/plsql/perl/2022/01/09/More-CSV-Fun.html)
I described how one could use some new tools I created 
in [perlish_util_udt](https://github.com/lee-lindley/plsql_utilities#perlish_util_udt) 
to parse a CSV clob into records and fields, then use that parsed data in a SQL statement,
perhaps including DML.

That blog post left it still a bit rough though. The user would need to remember or relearn the
syntax for two related functions to do the parsing, and the syntax for using
an object method in a SQL query is easy to forget about or mess up.

I had also been experimenting with Polymorphic Table Functions for this purpose because you
can determine the resultset structure at run time, which is important for this task. Yet PTFs
are complicated. Maybe too complicated.

I was rereading the "What's New" Oracle documentation for the last several releases and
Private Temporary Tables (added in 18c) caught my eye. Much of the issue we face with this use case
is that it needs to be dynamic and DDL is not dynamic. It is resource expensive and
regulated in most production environments. Private Temporary Tables are DDL without the downside.
It does not alter the data dictionary (well, maybe there is something going on as it
take advantage of the TEMPORARY tablespace), it does not leave anything behind after a session
completes, and it is not regulated by our corporate rules.

Private Temporary Tables let us define our column list on the fly!

# perlish_util_udt.create_ptt_csv

This static procedure recently added to 
[perlish_util_udt](https://github.com/lee-lindley/plsql_utilities#perlish_util_udt) 
combines the operations demonstrated in my prior blog post about CSV data to build
and populate a Private Temporary Table to contain your data from the CSV clob. How cool is that?

## Example

```plsql
BEGIN
    perlish_util_udt.create_ptt_csv('Employee ID, Last Name, First Name, nickname
999, "Baggins", "Bilbo", "badboy, ringbearer"
998, "Baggins", "Frodo",
997, "Orc", "Ogg", "i kill you"');
END;
/
select "Employee ID", "Last Name", "First Name", "nickname"
FROM ora$ptt_csv
;
```

    "Employee ID"                 "Last Name"                   "First Name"                  "nickname"                    
    "999"                         "Baggins"                     "Bilbo"                       "badboy, ringbearer"          
    "998"                         "Baggins"                     "Frodo"                       ""                            
    "997"                         "Orc"                         "Ogg"                         "i kill you"                  

The value of "nickname" in the Fodo record is NULL.

That is a lot easier than doing the parsing into lines and the parsing into fields and selecting by index number
we were using before. Under the covers that is what it does, but now we have something that is easy to use.

## The Code

Given the prior work with *perlish_util_udt.split_clobs_to_lines* and *perlish_util_udt.split_csv*, this was not
difficult to implement.

```plsql
	STATIC PROCEDURE create_ptt_csv (
         -- creates private temporary table ora$ptt_csv with columns named in first row of data case preserved.
         -- All fields are varchar2(4000)
	     p_clob         CLOB
	    ,p_separator    VARCHAR2    DEFAULT ','
	    ,p_strip_dquote VARCHAR2    DEFAULT 'Y' -- also unquotes \" and "" pairs within the field to just "
	) IS
        v_rows      arr_varchar2_udt := perlish_util_udt.split_clob_to_lines(p_clob);
        v_cols      perlish_util_udt;
```

The first step, which is in the variable declarations, splits the clob up into an array of rows. We are
going to parse the first row into *v_cols* which we make into an object type so we can take advantage
of our *map* and *join* methods to build the SQL we need to execute.

```plsql
        v_sql       CLOB;
    BEGIN
        v_cols := perlish_util_udt(perlish_util_udt.split_csv(v_rows(1), p_separator => p_separator, p_strip_dquote => 'Y'));
        -- remove the header row so can bind the array to read the data
        v_rows.DELETE(1);
```

We parse the first row differently than the rest. We are going to enclose the data in double quotes so we must strip
them if they exist. After we get our data from the first row, we delete it from the collection. We will later
bind the collection array to a SQL statement and do not want the first row to be present.

```plsql
        v_sql := 'DROP TABLE ora$ptt_csv';
        BEGIN
            EXECUTE IMMEDIATE v_sql;
        EXCEPTION WHEN OTHERS THEN NULL;
        END;
```

Drop the PTT if it already exists.

```plsql
        v_sql := 'CREATE PRIVATE TEMPORARY TABLE ora$ptt_csv(
'
            ||v_cols.map('"$_"    VARCHAR2(4000)').join('
,')
            ||'
)'
            ;
        DBMS_OUTPUT.put_line(v_sql);
        EXECUTE IMMEDIATE v_sql;
```
We use our collection of column names from the first row to create the column defintions for the PTT.
*map* and *join* are handy for this. As you can see we do not try to figure out what kind of data
each column is. We just stuff each column value into a VARCHAR2(4000) field.

The serveroutput of this code from running the example above was:

```plsql
CREATE PRIVATE TEMPORARY TABLE ora$ptt_csv(
"Employee ID"    VARCHAR2(4000)
,"Last Name"    VARCHAR2(4000)
,"First Name"    VARCHAR2(4000)
,"nickname"    VARCHAR2(4000)
)
```
Now for the INSERT.

```plsql
        v_sql := q'[INSERT INTO ora$ptt_csv 
WITH a AS (
    SELECT perlish_util_udt(
            perlish_util_udt.split_csv(t.column_value, p_separator => :p_separator, p_strip_dquote => :p_strip_dquote, p_keep_nulls => 'Y')
        ) AS p
    FROM TABLE(:bind_array) t
) SELECT ]'
        ||v_cols.map('x.p.get($##index_val##) AS "$_"').join('
,')
        ||'
FROM a x';
        -- must use table alias and fully qualify object name with it to be able to call function or get attribute of object
        -- Thus alias x for a.
        DBMS_OUTPUT.put_line(v_sql);
        EXECUTE IMMEDIATE v_sql USING p_separator, p_strip_dquote, v_rows;

    END -- crate_ptt_csv
    ;
```

Building the INSERT statement was a little tricky because we need to use the object function *get* with the array
index value for each column. Once again *map* and *join* are handing for building this. I added the '$##index_val##'
substitution to *map* for this release.

The serveroutput of this section from the example:
```plsql
INSERT INTO ora$ptt_csv
WITH a AS (
    SELECT perlish_util_udt(
            perlish_util_udt.split_csv(t.column_value, p_separator => :p_separator, p_strip_dquote => :p_strip_dquote, p_keep_nulls => 'Y')
        ) AS p
    FROM TABLE(:bind_array) t
) SELECT x.p.get(1) AS "Employee ID"
,x.p.get(2) AS "Last Name"
,x.p.get(3) AS "First Name"
,x.p.get(4) AS "nickname"
FROM a x
```

And of course you have seen the results. I'm starting to like this. Sure, I've been seduced by the dark side
and am writing PL/SQL code that looks like Perl, but I am what I am.
