---
layout: post
title: Inline External Tables for CSV Load
exerpt: "If you can deliver CSV data to an Oracle Directory, you can parse it without deploying any code by using an <i>Inline External Table</i>"
date: 2022-01-24 20:30:00 +0500
categories: [oracle, sql, plsql, perl]
tags: [oracle, sql, plsql, csv, perl]
---
# Yet Another CSV Load Option - *Inline External Tables*

While reviewing the Oracle What's New documentation for 18c I found *Private Temporary Tables* which
I wrote about in [my last post](https://lee-lindley.github.io/oracle/sql/plsql/perl/2022/01/23/CSV-Clob-PTT.html)
on the neverending saga of loading CSV data. Also found in What's New for 18c
is *Inline External Tables*.

Just like with PTTs, *Inline External Tables* are DDL without the downside. It is ad-hoc, and can be dynamically
generated. In order to use it you must have an Oracle DIRECTORY object granted to you with both READ and WRITE,
plus EXECUTE privilege on *DBMS_LOB* (and *UTL_FILE* if you want to clean up after yourself).

# Example

From our last post we had an example of CSV data with a header row that looked as such:

    Employee ID, Last Name, First Name, nickname
    999, "Baggins", "Bilbo", "badboy, ringbearer"
    998, "Baggins", "Frodo",
    997, "Orc", "Ogg", "i kill you"

We could get fancy and try to parse the header row like we did last time, but for this effort I'm going the cheap route
and assume you, the developer who wants to load the CSV data, will hand craft the code.

# Create File on Oracle Server

```plsql
BEGIN
    DBMS_LOB.clob2file(q'[999, "Baggins", "Bilbo", "badboy, ringbearer"
998, "Baggins", "Frodo",
997, "Orc", "Ogg", "i kill you"]'
                        ,'TMP_DIR'
                        ,'temp_csv_load.csv'
    );
END;
/
```
# Read from the Inline External Table

Now we read from the file:

```plsql
    SELECT *
    FROM EXTERNAL(
        (
            "Employee ID"   VARCHAR2(4000)
            ,"Last Name"    VARCHAR2(4000)
            ,"First Name"   VARCHAR2(4000)
            ,"nickname"     VARCHAR2(4000)
        )
        TYPE ORACLE_LOADER
        DEFAULT DIRECTORY TMP_DIR
        ACCESS PARAMETERS (
            RECORDS DELIMITED BY NEWLINE
            FIELDS TERMINATED BY ',' 
            OPTIONALLY ENCLOSED BY '"'
            MISSING FIELD VALUES ARE NULL
        )
        LOCATION ('temp_csv_load.csv')
        REJECT LIMIT UNLIMITED
    ) temp_csv_load_ext
    ;
```
The export from SQL Developer is adding the double quotes here. These are not in the data.

    "Employee ID"                 "Last Name"                   "First Name"                  "nickname"                    
    "999"                         "Baggins"                     "Bilbo"                       "badboy, ringbearer"          
    "998"                         "Baggins"                     "Frodo"                       ""                            
    "997"                         "Orc"                         "Ogg"                         "i kill you"                  

You could of course have the external table sqlldr driver convert to dates and numbers as needed. As far as I can tell
you have everything at your disposal that is there for a normal external table.

The line

    OPTIONALLY ENCLOSED BY '"'

makes sqlldr parse CSV data in a way that I believe mostly comports with the RFC on CSV data. It will handle
most of the test cases I threw at it and you are unlikely to give it the oddball stuff.

The line

    MISSING FIELD VALUES ARE NULL

is because we do not have a trailing comma after the last field (which would be delimited data rather than separated)
and because the sqlldr syntax of 'TRAILING NULLCOLS' I would pick is not available in the external table driver. Maddening.
Without it our second record would fail to load.

# Cleanup

Being a good citizen, we remove our trash:

```plsql
BEGIN
    UTL_FILE.fremove('TMP_DIR','temp_csv_load.csv');
END;
/
```
# Drawback

The big drawback to this technique (aside from how hard it is to get the external table definition right) is
that it requires you have READ/WRITE privs on a directory on the database server. In many organizations this is forbidden.
I retrofitted forty something load jobs
from external table to sqlldr because of that restriction imposed upon us by architecture/DBA teams
a few years back.
We can debate whether that is reasonable or not, but it is what it is. 

# Conclusion

There are plenty of client tools that will generate load data for you from CSV files or even directly from Excel.
As nice as those are, you probably cannot use them for Continuous Improvement deployments. I'm trying to come up
with a way to make that better. This is one more technique we might be able to use.
