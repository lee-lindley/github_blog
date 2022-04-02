---
layout: post
title: The Ubiquitous CSV File
date: 2021-09-10 20:30:00 +0500
categories: [oracle, sql, plsql]
tags: [oracle, sql, plsql, csv]
---
# The Oh So Common CSV Files 

One of the most common methods for transferring data between applications and businesses 
is the comma separated value format file. It has many advantages:

- It is relatively simple.
- It is entirely text, so it can be transported between computers running pretty much any operating system.
- It can be opened by common desktop applications like Excel.
- It isn't that difficult to generate manually.
- Many applications can generate and consume it automatically.

There is a semi-official standard for the format 
at [RFC4180](https://www.loc.gov/preservation/digital/formats/fdd/fdd000323.shtml),
but if you read it you will find it is fairly permissive about how you can quote
fields that contain separators and even newlines.

Parsing a CSV file seems simple
until you read all of the rules and sit down to try to do it. I have what I think
is a complete solution for parsing CSV in the Oracle PL/SQL 
function [split](https://github.com/lee-lindley/plsql_utilities#split). The regular
expression it employs is so ugly the comments outweigh the code ten to one. Yet
it seems to work just fine, passing all of the test cases I can conjour to throw at it.

# Loading CSV File

*sqlldr* (and by extension external tables) support a text import option that should
work for most honest CSV input files:

    FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'

I did not go out of my way to torture it the way I did my *split* function, yet the only
issues I've ever encountered with it had to do with funky non-ascii characters, which is
only tangentially related to the CSV format in that it depends on an understood common
or convertible character set. When the input file does not comply, as for instance when
a non-quoted string contains the delimiter, all bets are off.

If the input file follows [RFC4180](https://www.loc.gov/preservation/digital/formats/fdd/fdd000323.shtml),
I suspect Oracle *sqlldr* will parse it just fine. I could be wrong. 

# Generating CSV

From an Oracle database there are tools we can use to generate a CSV output file. 

## GUI Clients

*Toad* and *Sqldeveloper* both have CSV format file output options. You chan choose whether
or not to quote and whether or not to include column headers. These work well, but they
require a person to execute them. That is not a viable solution to something you want to automate.

## SQLcl (aka sql command line)

*SQLcl* (SQL Developer Command Line) is a Java-based tool. It supports most of the cruft of *sqlplus*
while also adding new stuff and many of the features of *SqlDeveloper*. Although it ships with the Oracle
client, it may or may not be installed on your ETL server. It has a robust set of
output formats and is quite handy with CSV, both for exporting and for importing! 

I have not seen
it used as a replacement for sqlplus and am not completely sure why. It may be that the people who
install and maintain database and ETL servers are just dinosaurs who don't trust new stuff. I mean
it is only up to version 20.4. *sqlplus* on the other hand has been around as long as Oracle has been
and it is everywhere.

## sqlplus

*SQL\*Plus 12.2* has added a [set markup csv](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/sqpug/SET-system-variable-summary.html#GUID-0AA910C4-C22A-4A9E-BE13-AAA059CC7919)
option, and when you turn on text quoting

    set markup csv on quote on

it will "escape" double quotes embedded witin the text. 
This can also give you the column headers in CSV format when you have "set heading on", 
so it may be a decent option. We will come back to that.

If you are stuck with an older version, you can concatenate data elements converted to text
with comma's between them in your SQL statement and spool it to a text file. For example:

```plsql
set linesize 200
set pagesize 0
set heading off
set trimspool on
set echo off
set termout off
spool test1.csv
SELECT 
    TO_CHAR(department_id)
    ||','|| department_name
    ||','|| TO_CHAR(manager_id)
    ||','|| TO_CHAR(location_id)
FROM hr.departments 
ORDER BY department_name
;
spool off
```

I have seen countless scripts like this in my career. I cringe every time I see one because I know it is
one careless data entry event away from waking up some poor support person in the middle of the night
when the file won't load because there is a comma in one of the field values. A little better pattern
can protect against that:

```plsql
set linesize 200
set pagesize 0
set heading off
set trimspool on
set echo off
set termout off
spool test2.csv
SELECT 
                  TO_CHAR(department_id)
    ||','|| '"'|| department_name           ||'"'
    ||','||       TO_CHAR(manager_id)
    ||','||       TO_CHAR(location_id)
FROM hr.departments 
ORDER BY department_name
;
spool off
```

Aside from how ugly it is and the chance that you make a mistake while typing it in, there is another
potential issue. What if the comma we fear is not what gets entered into the field value?
What if it is a double quote mark? What if it is a newline character?

I chose to not enclose the numeric fields with double quotes, but what if some joker changed the NLS
settings on us and the new default number format uses a comma for the decimal? DOH!

Another issue with this is that you don't get the column headers. If you need them there are
several clever ways to do it. You can google search it. I'm not happy with any of them.

The markup csv option available starting in sqlplus that comes with Oracle 12c looks promising. 
Let's try it out. We'll add some tricks with the NLS
date and numeric default formats to see how smart it is. I've made both default conversion
formats contain a comma in the result.

First we will try with *quote off*.

```plsql
ALTER SESSION SET nls_date_format = 'YYYYMMDD, HH24:MI';
ALTER SESSION SET nls_numeric_characters = ',.';
set markup csv on quote off
set linesize 200
set pagesize 0
--set heading off
set trimspool on
set echo off
set termout off
spool test_markup_noquote.csv
SELECT 
    'a string with comma(,)' AS "s1"
    ,'a string with dquote(")' AS "s2"
    ,'a string with newline(
)' AS "s3"
    ,SYSDATE AS "date1"
    ,12345 / 100 AS "num1"
FROM DUAL
;
spool off
```
And now the spool file:

    $ cat test_markup_noquote.csv
    
    s1,s2,s3,date1,num1
    a string with comma(,),a string with dquote("),a string with newline(
    ),20210910, 04:52,123,45
    

As I suspected if you set quote off in the markup directive, you get what you paid for. That does
not comply with the CSV format specification.

It also seems to have both a leading and trailing blank line in it. Not sure what is up with that or whether
you can get rid of them short of doing something in the shell afterward. If I decide to use it
for a job, I'll have to figure that out.

Let's try the same but with *quote on*

```plsql
ALTER SESSION SET nls_date_format = 'YYYYMMDD, HH24:MI';
ALTER SESSION SET nls_numeric_characters = ',.';
set markup csv on quote on
set linesize 200
set pagesize 0
--set heading off
set trimspool on
set echo off
set termout off
spool test_markup_wquote.csv
SELECT 
    'a string with comma(,)' AS "s1"
    ,'a string with dquote(")' AS "s2"
    ,'a string with newline(
)' AS "s3"
    ,SYSDATE AS "date1"
    ,12345 / 100 AS "num1"
FROM DUAL
;
spool off
```
And now the spool file:

    
    "s1","s2","s3","date1","num1"
    "a string with comma(,)","a string with dquote("")","a string with newline(
    )","20210910, 05:06",123,45
    
Well that is interesting. It handles the character types that contain double quotes just
like the RFC specifies by doubling up the internal doublequote marks. A competent CSV parser
should have no trouble with that. As a bonus, it was smart enough to put double quotes around the 
date value converted by the default NLS format. Sweet! But it did NOT quote the number "num1" value
even though it contained a separator comma,
and that is a potential issue.  It is not a very likely issue to happen to you in the real world though.
And if you think it could be, you can always do the TO_CHAR conversion of numbers to strings in your
query.

Overall using sqlplus with the *markup csv*
option looks to be a fine way to generate CSV files.

# Custom Tools

Maybe the ability to run a batch operation on an ETL server using sqlplus is not part of your
operational toolset. Maybe you need to generate the file from inside a program that is not able
to launch sqlplus, say for example you have a job that is running from DBMS_SCHEDULER_JOBS directly 
in the database, or the task is launched from a trigger or a web application interacting with the database.

For those scenarios where you cannot use a client program to generate the file, we can generate the
file using PL/SQL.

## The Hard Way

You can open a file on the database server and manually construct and write the CSV file.
This is a fairly common technique I have seen deployed by clients (though they rarely bothered
with the bulk collect).

```plsql
DECLARE
    CURSOR c1 IS
        SELECT e.employee_id, e.last_name, e.first_name, d.department_name, salary
        FROM hr.employees e
        INNER JOIN hr.departments d
            ON d.department_id = e.department_id
        ORDER BY d.department_name, e.last_name, e.first_name
    ;
    SUBTYPE t_c1_rec IS c1%ROWTYPE;
    TYPE t_arr_c1_rec IS TABLE OF t_c1_rec;
    v_arr_c1_rec    t_arr_c1_rec;
    v_file          UTL_FILE.file_type;
    v_dir           CONSTANT VARCHAR2(30) := 'TMP_DIR';
    v_file_name     CONSTANT VARCHAR2(30) := 'test_csv_dbfile.csv';
BEGIN
    v_file := UTL_FILE.fopen(
        filename        => v_file_name
        ,location       => v_dir
        ,open_mode      => 'w'
        ,max_linesize   => 32767
    );
    UTL_FILE.put_line(v_file, 'employee_id,last_name,first_name,department_name,salary');
    OPEN c1;
    LOOP
        FETCH c1 BULK COLLECT INTO v_arr_c1_rec LIMIT 100;
        EXIT WHEN v_arr_c1_rec.COUNT = 0;
        FOR i IN 1..v_arr_c1_rec.COUNT
        LOOP
            UTL_FILE.put_line(v_file,
                TO_CHAR(v_arr_c1_rec(i).employee_id)
                ||',"'||v_arr_c1_rec(i).last_name           ||'"'
                ||',"'||v_arr_c1_rec(i).first_name          ||'"'
                ||',"'||v_arr_c1_rec(i).department_name     ||'"'
                ||',"'||LTRIM(TO_CHAR(v_arr_c1_rec(i).salary,'$999,999,999.99'))
                                                            ||'"'
            );
        END LOOP;
    END LOOP;
    CLOSE c1;
    UTL_FILE.fclose(v_file);
    RETURN;
EXCEPTION WHEN OTHERS THEN
    IF UTL_FILE.is_open(v_file)
        THEN UTL_FILE.fclose(v_file);
    END IF;
    RAISE;
END;
/
```
    lee@linux2 [/tmp]
    $ head -10 test_csv_dbfile.csv
    employee_id,last_name,first_name,department_name,salary
    206,"Gietz","William","Accounting","$8,300.00"
    205,"Higgins","Shelley","Accounting","$12,008.00"
    200,"Whalen","Jennifer","Administration","$4,400.00"
    102,"De Haan","Lex","Executive","$17,000.00"
    100,"King","Steven","Executive","$24,000.00"
    101,"Kochhar","Neena","Executive","$17,000.00"
    110,"Chen","John","Finance","$8,200.00"
    109,"Faviet","Daniel","Finance","$9,000.00"
    108,"Greenberg","Nancy","Finance","$12,008.00"

That works well enough, but man oh man is it painful.

## refcursor-to-csv

William Robertson published a nice package that can generate CSV data that complies with the RFC
from any ref cursor. [refcursor-to-csv](https://www.williamrobertson.net/documents/refcursor-to-csv.shtml)
is an approach that is completely within the database rather than in a client. This means
you can select the resulting CSV rows in any client or write them to a file on the database server. It is
a fine tool that may well meet your needs. He adds some bells and whistles for producing
optional header and trailer records that may be exactly what you are looking for.

Here are two examples from Mr. Robertson's documentation at the above link:

    select column_value
    from   table(csv.report(cursor(
            select * from dept
        ), p_separator => '|', p_label => 'DEPT', p_heading => 'Y', p_rowcount => 'Y'));
    
    COLUMN_VALUE
    ----------------------------------------------
    HEADING|DEPT|DEPTNO|DNAME|LOC
    DEPT|10|ACCOUNTING|NEW YORK
    DEPT|20|RESEARCH|DALLAS
    DEPT|30|SALES|CHICAGO
    DEPT|40|OPERATIONS|BOSTON
    ROW_COUNT|DEPT|4

    6 rows selected.

And the example of writing to a file on the database server:

    declare
        l_dataset sys_refcursor;
    begin
        open l_dataset for select * from dept;
    
        csv.write_file
        ( p_dataset => l_dataset
        , p_separator => '|', p_label => 'DEPT', p_heading => 'Y', p_rowcount => 'Y'
        , p_directory => 'DATA_FILE_DIR'
        , p_filename => 'dept.csv' );
    end;

## app_csv

I was inspired by Mr. Robertson's work to extend it in several ways where I wanted more flexibility.
I also had another use for the underlying mechanism of using DBMS_SQL to gather
the cursor results for any unknown query. I created base Object types I 
call [app_dbms_sql](https://github.com/lee-lindley/plsql_utilities#app_dbms_sql)
and then built [app_csv](https://github.com/lee-lindley/app_csv) on top of those.

The [app_csv documentation on use cases](https://github.com/lee-lindley/app_csv#use-cases)
show selecting from a TABLE function, writing to a database server file, and retrieving
a CLOB. The latter is particularly useful if you want to email the CSV data as an attachment.
(See [html_email](https://github.com/lee-lindley/html_email) for a tool that can send email with
attachments from the database server.)


Both of these tools process completely within the database engine.

## ExcelGen

If the only reason you are producing CSV files is because you want to give them to a user to open
in Excel, why stop there? [ExcelGen](https://github.com/mbleron/ExcelGen) is a package by Marc Bleron that
can convert a refcursor (just like *app_csv* and *refcursor-to-csv* do) and generate an XLSX
file. Maybe your users are happy enough with opening a CSV file in excel and mucking with the column formats
for a few minutes every day, but you can go the extra step without a whole lot more trouble. Plus as a bonus
if you have multiple queries and files you are producing for them, you can combine them as separate
tabs/sheets in the same workbook. The format is compressed too so files are smaller.

It is just a little bit more work than producing CSV files, yet the end product is sure to be appreciated
by your users.

# Conclusion

These are some techniques you may be able to use to fit a need. Hope it was helpful.
