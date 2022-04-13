---
layout: post
title: Cost of UDT Object Methods in SQL
exerpt: "A journey to optimize some unexpectedly slow code with Chained Pipeline Row Functions."
date: 2022-04-02 12:30:00 +0500
categories: [oracle, sql, plsql]
tags: [oracle, sql, plsql, objects, user-defined-type-objects]
---
# Introduction

In a prior post, [CSV Clob and Private Temporary Table](https://lee-lindley.github.io/oracle/sql/plsql/perl/2022/01/23/CSV-Clob-PTT.html),
I described how one could use some new tools I created 
to parse a CSV clob into records and fields, then use that parsed data in a SQL statement,
perhaps including DML.

The article was about a work in progress and the code has changed, but it will give you an idea of how
we got here if you are interested.

I ran into performance issues for large datasets while testing it. This article is about
the techniques I tried for tuning it, and a fact about Oracle Object Type methods that
I had not thought about much before.

>Note: you will see what may be unfamiliar methods of *perlish_util_udt* Object type in this code. They work like Perl operators on lists.

# SQL Engine and PL/SQL Method Calls

Here is an example query that is built as dynamic SQL in a PL/SQL procedure. It
returns a resultset built from reading CSV data provided to the program as a CLOB.

You can find the packages at [https://github.com/lee-lindley/plsql_utilities](https://github.com/lee-lindley/plsql_utilities).
The procedure I'm showing examples from is *app_csv_pkg.create_ptt_csv*.

```plsql
WITH a AS (
    SELECT perlish_util_udt(
            app_csv_pkg.split_csv(t.s, p_separator => :p_separator, p_strip_dquote => :p_strip_dquote, p_keep_nulls => 'Y')
        ) AS p
    FROM TABLE( app_csv_pkg.split_clob_to_lines(:p_clob, p_skip_lines => 1) ) t
) SELECT 
    x.p.get(1) AS "Employee ID"
    ,x.p.get(2) AS "Last Name"
    ,x.p.get(3) AS "First Name"
    ,x.p.get(4) AS "nickname"
FROM a x
```

The pipelined table function *split_clob_to_lines* is efficient. 
What turned out to be not efficient is the call to the *perlish_util_udt* object conststructor and/or the 
call to the function *split_csv* in the SQL SELECT list. Those are both efficient when called inside a PL/SQL program.

This should not have surprised me. I know that the SQL engine must do a context switch for each call to a PL/SQL
program unless it is cached. In this case the cost was much, much larger than I expected.

For a 10K row CLOB of CSV data (20 fields each row) the above construct ran for 20 minutes to process the query on an AIX
based database, and 6 minutes on my Intel NUC running in a Hyper-V Linux partition.
It is difficult
to tell what it is doing from the WAIT events, but I suspected it was in-memory operations and context switching.

# Removing the Object

I also suspected the object method *get* calls might be doing context switches. My first refinement attempt
was to remove the object from the picture and use a WITH FUNCTION to snag the field values from the
array returned by *split_csv*.

```plsql
WITH FUNCTION wget(
    p_arr   ARR_VARCHAR2_UDT
    ,p_i    NUMBER
) RETURN VARCHAR2
AS
BEGIN
    RETURN p_arr(p_i);
END;
a AS (
    SELECT app_csv_pkg.split_csv(t.s
                    , p_separator => :p_separator, p_strip_dquote => :p_strip_dquote, p_keep_nulls => 'Y'
            ) AS arr
    FROM TABLE( app_csv_pkg.split_clob_to_lines(:p_clob, p_skip_lines => 1) ) t
) SELECT 
     get(a.arr, 1) AS "Employee ID"
    ,get(a.arr, 2) AS "Last Name"
    ,get(a.arr, 3) AS "First Name"
    ,get(a.arr, 4) AS "nickname"
FROM a x
```
This also ran for 6 minutes on my Intel NUC database. We haven't ruled out that the object could play a
part, but we know that the function call *split_csv* is a problem. 

If you are thinking about using
a *UDF* pragma on that puppy (*split_csv*), I could not 
get any of the variations I tried to give an improvement. There are many examples shown of the limitations
of that pragma (or using a WITH function) that does anything very complicated. 

# Make the Function Pipelined in a Chain

The next refactoring I tried was moving the call to *split_csv* into another Pipelined Table Function
to be called in a chain with *split_clob_to_lines*. I called it *split_lines_to_fields* (catcalls
from the peanut gallery may be deserved). Maybe I'll come up with a better name before I merge this code.

From the package header:

```plsql
    TYPE t_csv_fields_rec IS RECORD(
        arr ARR_VARCHAR2_UDT
        ,rn NUMBER
    );
    TYPE t_arr_csv_fields_rec IS TABLE OF t_csv_fields_rec;

    FUNCTION split_lines_to_fields(
        p_curs          t_curs_csv_row_rec
        ,p_separator    VARCHAR2    DEFAULT ','
        ,p_strip_dquote VARCHAR2    DEFAULT 'Y' -- also unquotes \" and "" pairs within the field to just "
        ,p_keep_nulls   VARCHAR2    DEFAULT 'Y'
    ) 
    RETURN t_arr_csv_fields_rec
    PIPELINED
    ;
```

and from the package body:

```plsql
    FUNCTION split_lines_to_fields(
        p_curs          t_curs_csv_row_rec
        ,p_separator    VARCHAR2    DEFAULT ','
        ,p_strip_dquote VARCHAR2    DEFAULT 'Y' -- also unquotes \" and "" pairs within the field to just "
        ,p_keep_nulls   VARCHAR2    DEFAULT 'Y'
    ) 
    RETURN t_arr_csv_fields_rec
    PIPELINED
    IS
        v_row       t_csv_fields_rec;
        v_in_row    t_csv_row_rec;
    BEGIN
        LOOP
            FETCH p_curs INTO v_in_row;
            EXIT WHEN p_curs%NOTFOUND;
            v_row.rn := v_in_row.rn;
            v_row.arr := split_csv(v_in_row.s
                                    , p_separator       => p_separator
                                    , p_strip_dquote    => p_strip_dquote
                                    , p_keep_nulls      => p_keep_nulls
            );
            PIPE ROW(v_row);
        END LOOP;
        RETURN;
    END split_lines_to_fields
    ;
```
The documentation never shows a Bulk Collect in the chained pipeline examples, and the descriptions
it uses and other hints make me believe it is not needed. I wish it was
more explicit.

As recommended in the documentation, it is called in a chain of pipelined table functions 
directly in SQL with CURSOR casts as 

```plsql
WITH FUNCTION wget(
    p_arr   ARR_VARCHAR2_UDT
    ,p_i    NUMBER
) RETURN VARCHAR2
AS
BEGIN
    RETURN p_arr(p_i);
END;
a AS (
    SELECT t.arr 
    FROM TABLE(
                app_csv_pkg.split_lines_to_fields(
                    CURSOR(SELECT * 
                           FROM TABLE( app_csv_pkg.split_clob_to_lines(:p_clob, p_skip_lines => 1) )
                    )
                    , p_separator => :p_separator, p_strip_dquote => :p_strip_dquote, p_keep_nulls => 'Y'
                )
    ) t
) SELECT 
     get(a.arr, 1) AS "Employee ID"
    ,get(a.arr, 2) AS "Last Name"
    ,get(a.arr, 3) AS "First Name"
    ,get(a.arr, 4) AS "nickname"
FROM a

```

This ran in 23 seconds. Moving the function call into a pipelined table chain solves the problem, but I
still want to know whether or how much the Object plays a part.

# Call the Object Constructor in SQL

I could have moved the object constructor into the pipeline chain (and did try that), but it turns out
to not be necessary.

```plsql
WITH a AS (
    SELECT perlish_util_udt(t.arr) AS p
    FROM TABLE(
                app_csv_pkg.split_lines_to_fields(
                    CURSOR(SELECT * 
                           FROM TABLE( app_csv_pkg.split_clob_to_lines(:p_clob, p_skip_lines => 1) )
                    )
                    , p_separator => :p_separator, p_strip_dquote => :p_strip_dquote, p_keep_nulls => 'Y'
                )
    ) t
) SELECT 
     X.p.get(1) AS "Employee ID"
    ,X.p.get(2) AS "Last Name"
    ,X.p.get(3) AS "First Name"
    ,X.p.get(4) AS "nickname"
FROM a X
```

This also ran in 23 seconds. I prefer the object way better than the WITH FUNCTION way, but if you don't
already have an object wrapped around your array with a handy method, the WITH FUNCTION is just dandy.

# Conclusion

Calls to a PL/SQL function in a SELECT list can lead to significant cost via context switching. This is
well known, and there are several caching strategies (DETERMINISTIC, RESULT_CACHE, Scalar Subquery) to mitigate
it. Using a Pipelined Table Function chain as I did here is also a viable mitigation strategy, and I believe
the appropriate one for this use case.

My concern that object construction and simple method function calls in the SQL engine could context
switch seems to be unfounded. I re-read the Object Relational Developer Guide and searched Google several
ways looking for more information about how object methods are implemented, but it is not clearly stated.
They seem to be neither fish nor fowl, SQL engine nor PL/SQL engine.

I am wondering why using an object type is not touted as another mitigation strategy for PL/SQL Function context switching
(at least for when the method does not need to call anything other than builtin functions)? 
I realize we have WITH FUNCTIONs, UDF pragma, and with Oracle 21c, SQL Macros,
but it would be nice to know if Object methods are an alternative.

I may explore this as a context switching mitigation strategy another day.


# Appendix - The Full Monty

If you have a use case that calls for the ultimate in performance (this one does not, but
I happened to try this early in my analysis), you can get down and dirty with DBMS_SQL
and I believe what is called "Method 4" dynamic SQL. That classification is because we have a variable
number of bind elements.

Unlike the other examples, this one shows the entire procedure that creates a
Private Temporary Table (PTT) and populates it from the CSV Clob.

The best I could achieve with the code shown in the article was 23 seconds. This version
operates in 19 seconds. It isn't worth the complexity for this package, but it is nice
to know how to do it for the rare occassions when you might need to squeeze out
that last little bit of efficiency.

The first part is mostly the same as the code I didn't show you for the other variations. 
It parses the first row of the CLOB to get the column names and creates the PTT.

```plsql
    PROCEDURE create_ptt_csv (
         --
         -- creates private temporary table "ora$ptt_csv" with columns named in first row of data (case preserved).
         -- from a CLOB containing CSV lines.
         -- All fields are varchar2(4000)
         --
	     p_clob         CLOB
	    ,p_separator    VARCHAR2    DEFAULT ','
	    ,p_strip_dquote VARCHAR2    DEFAULT 'Y' -- also unquotes \" and "" pairs within the field to just "
    ) IS
        v_cols          perlish_util_udt; -- for manipulating column names into SQL statement
        v_sql           CLOB;
        v_first_row     VARCHAR2(32767);
        v_ins_curs      INT;
        v_num_rows      INT;
        v_last_row_cnt  BINARY_INTEGER := 0;
        v_col_cnt       BINARY_INTEGER;
        v_vals_1_row    ARR_VARCHAR2_UDT;  -- from split_csv on 1 line
        v_rows          DBMS_SQL.varchar2a;     -- from split_clob_to_lines fetch
        --
        -- variable number of columns, each of which has a bind array.
        --
        TYPE varchar2a_tab  IS TABLE OF DBMS_SQL.varchar2a INDEX BY BINARY_INTEGER;
        v_vals          varchar2a_tab;          -- array of columns each of which holds array of values
        --
        -- We get all but the header row when we read the clob in a loop.
        --
        CURSOR c_read_rows IS
            SELECT t.s
            FROM TABLE(app_csv_pkg.split_clob_to_lines(p_clob, p_skip_lines => 1))  t
            ;
    BEGIN
        BEGIN
            -- read the first row only
            SELECT s INTO v_first_row 
            FROM TABLE( app_csv_pkg.split_clob_to_lines(p_clob, p_max_lines => 1) )
            ;
            IF v_first_row IS NULL THEN
                raise_application_error(-20222,'app_csv_pkg.create_ptt_csv did not find csv rows in input clob.');
            END IF;
        EXCEPTION WHEN NO_DATA_FOUND THEN
            raise_application_error(-20222,'app_csv_pkg.create_ptt_csv did not find csv rows in input clob.');
        END;
        -- split the column header values into collection
        v_cols := perlish_util_udt(split_csv(v_first_row, p_separator => p_separator, p_strip_dquote => 'Y'));
        v_col_cnt := v_cols.arr.COUNT;

        -- create the private global temporary table with "known" name and columns matching names found
        -- in csv first record
        --
        v_sql := 'DROP TABLE ora$ptt_csv';
        BEGIN
            EXECUTE IMMEDIATE v_sql;
        EXCEPTION WHEN OTHERS THEN NULL;
        END;

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

The next part populates the PTT. This variation uses bulk collect to read the data as lines,
splits the lines into columns, stuffs the column data into individual arrays, then
executes a DBMS_SQL INSERT cursor bound to those arrays. Pretty slick, if a little hard core.


```plsql
        -- 
        -- Dynamic sql for dbms_sql. will be used with bind arrays.
        -- Of note is that it reports conventional load even if specify append.
        -- I don't understand that as I've seen other reports that direct path load works.
        -- Does not seem to matter though.
        --
        v_sql := 'INSERT INTO ora$ptt_csv(
'
        ||v_cols.map('"$_"').join(', ')             -- the column names in dquotes
        ||'
) VALUES (
'
        ||v_cols.map(':$##index_val##').join(', ')  -- :1, :2, :3, etc... bind placeholders
        ||'
)';
        DBMS_OUTPUT.put_line(v_sql);
        v_ins_curs := DBMS_SQL.open_cursor;
        DBMS_SQL.parse(v_ins_curs, v_sql, DBMS_SQL.native);

        OPEN c_read_rows;
        LOOP
            FETCH c_read_rows BULK COLLECT INTO v_rows LIMIT 100;
            EXIT WHEN v_rows.COUNT = 0;
            FOR i IN 1..v_rows.COUNT
            LOOP
                v_vals_1_row := app_csv_pkg.split_csv(v_rows(i), p_separator => p_separator, p_strip_dquote => p_strip_dquote, p_keep_nulls => 'Y');
                -- j is column number
                FOR j IN 1..v_col_cnt
                LOOP
                    v_vals(j)(i) := v_vals_1_row(j);
                END LOOP;
            END LOOP;

            IF v_last_row_cnt != v_rows.COUNT THEN -- will be true on first loop iteration and maybe last
                v_last_row_cnt := v_rows.COUNT;
                -- bind each column array. v_vals has an array for every column
                FOR j IN 1..v_col_cnt
                LOOP
                    DBMS_SQL.bind_array(v_ins_curs, ':'||TO_CHAR(j), v_vals(j), 1, v_last_row_cnt);
                END LOOP;
            END IF;

            v_num_rows := DBMS_SQL.execute(v_ins_curs);

        END LOOP;
        DBMS_SQL.close_cursor(v_ins_curs);
        CLOSE c_read_rows;
    END create_ptt_csv
    ;
```
