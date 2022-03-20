---
layout: post
title: Extracting BLOB from Oracle with Sqlplus
date: 2021-12-18 20:30:00 +0500
categories: [oracle, sql, plsql]
tags: [oracle, sql, plsql, blob, sqlplus]
---

# How do we get a BLOB from the database to a client file?

I wrote about using a WITH clause function for various purposes
[Why do I need Inline PL/SQL Methods?](https://lee-lindley.github.io/oracle/sql/plsql/2021/09/25/Inline-PLSQL-Methods.html), 
one example of which was
extracting a BLOB that contained an XLSX file.
In that article I blithely waved my hands and said use Toad or SqlDeveloper to then extract the output into a file.

What do we do if we want to script this? There are options. 
You can send email from inside
the database and attach the blob (see [html_email](https://github.com/lee-lindley/html_email)), but if your goal is to
automatically deliver it to a shared drive or other file destination, email is not a good option.

You can use pretty much any language that supports
a robust Oracle database interface to do it from a client ETL server. Pro-C, Java, Perl DBI::DBD::Oracle, 
and I presume Python all
have the capability to extract a BLOB from the database and write to the local filesystem. I also assume
the big commercial ETL packages like Ab-initio and Informatica will do it, but I haven't looked.

Maybe you do not have access to or expertise in these tools.  The Oracle client library has SQLcl, which can be
scripted to download a BLOB as mentioned in this article 
[Using SQLcl to write out BLOBs to files in 20 lines of js](https://www.thatjeffsmith.com/archive/2020/07/using-sqlcl-to-write-out-blobs-to-files-in-20-lines-of-js/). But it turns out that you don't always have sqlcl or even Java
installed on an ETL server. I know that seems odd, but I've seen it enough I'm no longer surprised.

I wanted to do it with *sqlplus* because it is almost universally available at every client and on every ETL
server. Unfortunately *sqlplus* does not support BLOB data type as a local variable for binding. Drat. When 
I searched I hit the article [Download BLOBs with SQLPlus](https://ogobrecht.com/posts/2020-01-01-download-blobs-with-sqlplus/)
by Ottmar Gobrect. He spreads plenty of credit around for the idea. The crux of the solution is to convert
from BLOB, which *sqlplus* does not support, to CLOB which it does. How, you ask? The same way we do it when we attach
a binary file to an email. We base64 encode it. Clever!

# Extracting a BLOB with Sqlplus

Reusing my spreadsheet example from the article mentioned at the top, and a base64encode function
published by Tim Hall we have a sqlplus script that generates and extracts a BLOB
to a file as base64 encoded text.

```plsql
-- 16mb should be plenty for most spreadsheets.
set long 16777216
set longchunksize 32767
set heading off
set verify off 
set feedback off 
set trimout on 
set trimspool on 
set pagesize 0 
set linesize 1000 
whenever sqlerror exit failure
-- sqlplus supports clob variables but not blob
variable vo_clob clob
--
DECLARE
    c SYS_REFCURSOR;

    FUNCTION base64encode(p_blob IN BLOB)
    RETURN CLOB
    -- -----------------------------------------------------------------------------------
    -- File Name    : https://oracle-base.com/dba/miscellaneous/base64encode.sql
    -- Author       : Tim Hall
    -- Description  : Encodes a BLOB into a Base64 CLOB.
    -- Last Modified: 09/11/2011
    -- -----------------------------------------------------------------------------------
    IS
        l_clob CLOB;
        l_step PLS_INTEGER := 12000; -- make sure you set a multiple of 3 not higher than 24573
    BEGIN
        FOR i IN 0 .. TRUNC((DBMS_LOB.getlength(p_blob) - 1 )/l_step) LOOP
            l_clob := l_clob || UTL_RAW.cast_to_varchar2(UTL_ENCODE.base64_encode(DBMS_LOB.substr(p_blob, l_step, i * l_step + 1)));
        END LOOP;
        RETURN l_clob;
    END;

    FUNCTION get_xlsx(p_src SYS_REFCURSOR) 
    RETURN CLOB AS
        v_blob          BLOB;
        v_ctxId         ExcelGen.ctxHandle;
        v_sheetHandle   BINARY_INTEGER;
    BEGIN
        v_ctxId := ExcelGen.createContext();
        v_sheetHandle := ExcelGen.addSheetFromCursor(v_ctxId, 'Employee Salaries', p_src, p_sheetIndex => 1);
        -- freeze the top row with the column headers
        ExcelGen.setHeader(v_ctxId, v_sheetHandle, p_frozen => TRUE);
        -- style with alternating colors on each row. 
        ExcelGen.setTableFormat(v_ctxId, v_sheetHandle, 'TableStyleLight11');
        -- single column format on the salary column. The ID column keeps default format
        ExcelGen.setColumnFormat(
            p_ctxId     => v_ctxId
            ,p_sheetId  => v_sheetHandle
            ,p_columnId => 5        -- the salary column
            ,p_format   => '$#,##0.00'
        );
        v_blob := ExcelGen.getFileContent(v_ctxId);
        ExcelGen.closeContext(v_ctxId);
        RETURN base64encode(v_blob);
    END;
BEGIN

    OPEN c FOR WITH add_bilbo AS (
            SELECT e.employee_id AS employee_id, e.last_name, e.first_name, d.department_name, e.salary
            FROM hr.employees e
            INNER JOIN hr.departments d
                ON d.department_id = e.department_id
            UNION ALL
            SELECT 999 AS employee_id, 'Baggins' As last_name, 'Bilbo' as first_name, 'Sales' AS department_name
                ,123.45 AS salary
            FROM dual
        ) SELECT * FROM add_bilbo ORDER BY last_name, first_name
    ;
    -- assign the uuencoded clob data to the sqlplus bind variable
    :vo_clob := get_xlsx(c);
END;
/
-- spool out the encoded clob
set termout off
spool "x.xlsx.base64"
print vo_clob
spool off
set termout on
--  base64 -d -i x.xlsx.base64 >x.xlsx && rm x.xlsx.base64
-- OR on windows
-- certutil -decode x.xlsx.base64 x.xlsx
-- del x.xlsx.base64
```

On my windows machine I can run the following command in cygwin. I assume it works fine in Linux.

    base64 -d -i spool_file_name > spreadsheet_name.xlsx

If you fail to included the -i option, it may complain about invalid input. Turns out my cygwin
version was not forgiving of the carriage return characters (\015) sqlplus put into the file. It
was fine with the linefeeds (\012). The -i option makes it ignore them. It may not be necessary
on a true Unix machine or if there is an option to turn off carriage returns in sqlplus on Windows.
I didn't try to find out.

On windows the following command works just fine.

    certutil -decode spool_file_name spreadsheet_name.xlsx

In my small example the encoding added around 25% to the file size. I'm not claiming that is the
amount you will get, but it should be in that ballpark. It isn't enough to break the bank.

# Conclusion

If you are looking to get a BLOB out of Oracle and you don't have access to a better scripting
language or tool, knowing you can at least do it with sqlplus is a nice option to have in your
back pocket.
