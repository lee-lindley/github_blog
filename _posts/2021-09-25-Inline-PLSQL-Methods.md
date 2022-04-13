---
layout: post
title: Why do I need Inline PL/SQL Methods?
exerpt: "<i>WITH</i> clause Functions and uses you may not have considered."
date: 2021-09-25 20:30:00 +0500
categories: [oracle, sql, plsql]
tags: [oracle, sql, plsql]
---
# Declaring PL/SQL in an Oracle SQL *WITH* Clause

The capability to define PL/SQL functions and procedures inside an Oracle SQL query (and even 
the query portion of DML statements) was added in Oracle version 12.1. The Oracle documentation 
for it that I have found so far is sparse. Tim Hall of *Oracle Base* fame has a good primer
on it [WITH Clause Enhancements in Oracle Database 12c Release 1 (12.1)](https://oracle-base.com/articles/12c/with-clause-enhancements-12cr1).

The primary reason (allegedly) that Oracle provided for defining the PL/SQL code inline is
to improve performance by avoiding context switching. Yet in the same release we were given
*PRAGMA UDF* which allows optimizing schema level PL/SQL functions as inline. That seems a wash.

The best case I have for deploying inline methods in production code is when the logic is specific to that single query, and is not easily and cleanly done directly in SQL. There are not that many tasks that are so difficult in SQL that this condition exists, but there are some where procedural code is just a better answer. I would also add a caveat that if the procedure is big and gnarly, I would rather compile it in the schema than trying to debug it in the middle of a giant query.

# Another Reason for Needing Inline PL/SQL

I have another use for inline PL/SQL. Consider the case where you cannot deploy schema level PL/SQL. The best example I have is doing adhoc queries on a production system. You can deploy PL/SQL to that system after going through the formal process, but not today.

Maybe you have a real need for PL/SQL, like calling procedures with OUT parameters then selecting the value in a query, but you have query-only access on a system. Or perhaps your access to the database elements you need is through roles, not direct grants, so you cannot deploy PL/SQL that uses those elements.

## Role Access for Inline PL/SQL

For this test my schema has not been granted *SELECT* on *HR.job_history*. I have the role privilege *HR_SELECT*,
and that role has been granted *SELECT* on *HR.job_history*. If I attempted to create a function
that read from the table *HR.job_history*, the create would fail. The inline PL/SQL function works though.

```plsql
WITH FUNCTION get_hist_job_id(p_emp number) RETURN VARCHAR2
AS
    l_job_id VARCHAR2(30);
BEGIN
    SELECT MAX(job_id) KEEP (DENSE_RANK FIRST ORDER BY end_date DESC) INTO l_job_id
    FROM hr.job_history
    WHERE employee_id = p_emp
    ;
    RETURN l_job_id;
END;
SELECT get_hist_job_id(176) FROM dual;
/
```

    GET_HIST_JOB_ID(176)
    --------------------------------------------------------------------------------
    SA_MAN


Granted that the function doesn't do anything that we could not have done directly in the query, but the test was merely to prove that we can avoid the issue with grants through roles versus direct grants to the schema owner. If you need PL/SQL and face a limitation where you cannot get the direct grants, this might be a way around it. Note that this is not likely to please certain control freaks, so keep it to yourself.

You might also be able to get around this limitation by using dynamic sql (EXECUTE IMMEDIATE) and AUTHID CURRENT_USER. I admit to not having a complete grasp of all of the intricacies of the privilege stack where dynamic sql is involved, but I think it can be done. I like the inline PL/SQL method for it though.

## Selecting a Value from a Procedure OUT Parameter

Consider a scenario where the only methods available to you for retrieving some value
is through a procedure with an OUT parameter. You cannot call that directly from a
SQL select. In this system we cannot deploy schema level PL/SQL today, so we will use
an inline function to provide the result.

The *DBMS_LOB* package has a procedure *LOADBLOBFROMFILE* that puts the result in an OUT parameter.
I want to SELECT that BLOB value and save it to my local drive (which I can do easily from Toad
or SqlDeveloper).

As of Oracle 12.2 you can do this in straight SQL (see [TO_CLOB and TO_BLOB enhancements](https://odieweblog.wordpress.com/2017/04/17/oracle-12-2-to_clob-and-to_blob-enhancements/)), 
but I'm going to use the example anyway.

```plsql
WITH FUNCTION get_blob(p_directory VARCHAR2, p_filename VARCHAR2) RETURN BLOB
AS
    l_bfile         BFILE;
    l_blob          BLOB;
    l_blob_out      BLOB;
    l_dest_offset   INTEGER := 1;
    l_src_offset    INTEGER := 1;
BEGIN
    l_bfile := BFILENAME(p_directory, p_filename);
    DBMS_LOB.fileopen(l_bfile, DBMS_LOB.file_readonly);
    IF DBMS_LOB.getlength(l_bfile) > 0 THEN
        DBMS_LOB.createtemporary(l_blob,TRUE);
        DBMS_LOB.loadblobfromfile(
            dest_lob        => l_blob
            ,src_bfile      => l_bfile
            ,amount         => DBMS_LOB.getlength(l_bfile)
            ,dest_offset    => l_dest_offset
            ,src_offset     => l_src_offset
        );
        l_blob_out := l_blob;
        DBMS_LOB.freetemporary(l_blob);
    END IF;
    DBMS_LOB.fileclose(l_bfile);
    RETURN l_blob_out;
END;
SELECT get_blob('TMP_DIR','x.xlsx') FROM dual
;
/
```
That gives me a "(BLOB)" in the query result in SqlDeveloper that I can download to my local disk.

## Procedure Required to Produce Result

In a similar vein you may require execution of multiple PL/SQL procedural steps to produce
the output you ultimately want to select in your adhoc query.

Marc Bleron has an excellent tool for generating MS Excel files directly in the database [ExcelGen - An Oracle PL/SQL Generator for MS Excel Files](https://github.com/mbleron/ExcelGen). Most of the time you will use this
tool within PL/SQL procedures or packages that you deploy. Yet there are times when it can
be useful for an adhoc task. Sure, you can have Toad or Sqldeveloper produce CSV or even Excel
files from a query, but they do not match this for capability. And if you happen to have
already written a procedure that does much of what you need for your adhoc query, you can
quickly copy/paste the code into a WITH clause and be ready to rock and roll.

```plsql
WITH 
FUNCTION get_xlsx(p_src SYS_REFCURSOR) RETURN BLOB AS
    v_blob          BLOB;
    v_ctxId         ExcelGen.ctxHandle;
    v_sheetHandle   BINARY_INTEGER;
BEGIN
        v_ctxId := ExcelGen.createContext();
        v_sheetHandle := ExcelGen.addSheetFromCursor(v_ctxId, 'Employee Salaries', p_src, p_sheetIndex => 1);
        -- freeze the top row with the column headers
        ExcelGen.setHeader(v_ctxId, v_sheetHandle, p_frozen => TRUE);
        -- style with alternating colors on each row. 
        ExcelGen.setTableFormat(v_ctxId, v_sheetHandle, 'TableStyleLight2');
        -- single column format on the salary column. The ID column keeps default format
        ExcelGen.setColumnFormat(
            p_ctxId     => v_ctxId
            ,p_sheetId  => v_sheetHandle
            ,p_columnId => 5        -- the salary column
            ,p_format   => '$#,##0.00'
        );
        v_blob := ExcelGen.getFileContent(v_ctxId);
        ExcelGen.closeContext(v_ctxId);
        RETURN v_blob;
END;
-- begin sql portion 
add_bilbo AS (
    SELECT e.employee_id AS employee_id, e.last_name, e.first_name, d.department_name, e.salary
    FROM hr.employees e
    INNER JOIN hr.departments d
        ON d.department_id = e.department_id
    UNION ALL
    SELECT 999 AS employee_id, 'Baggins' As last_name, 'Bilbo' as first_name, 'Sales' AS department_name
        ,123.45 AS salary
    FROM dual
), emp_curs AS (
    SELECT * FROM add_bilbo ORDER BY last_name, first_name
) SELECT get_xlsx(CURSOR(SELECT * FROM emp_curs)) FROM DUAL
;
/
```
The trick of passing a CURSOR variable using the last named subquery of the WITH clause (last line of above sample)
is awesome. Many times I've written huge queries that were stuffed into a CURSOR cast to pass into a Function
before I discovered this handy syntax.

Since the ExcelGen package uses session level package global variables to store everything, 
you could probably execute an anonymous PL/SQL
block using a client declared context variable that does the first part of this through *setColumnFormat*, then use a simple 
```plsql
SELECT ExcelGen.getFileContent(:v_ctxId) FROM dual;
```
to get the result. I like the *WITH PL/SQL* version much better.

![ExcelPage](/images/excelpage.gif){:.centered}

Here is another example using [PdfGen](https://github.com/lee-lindley/PdfGen), a package I wrote that wraps
the core *as_pdf3* written by Anton Scheffer.

```plsql
WITH FUNCTION get_blob(p_src SYS_REFCURSOR) RETURN BLOB
IS
    v_blob      BLOB;
    v_widths    PdfGen.t_col_widths;
    v_headers   PdfGen.t_col_headers;
    v_formats   PdfGen.t_col_headers;
BEGIN
                                                -- Similar to the sqlplus COLUMN HEADING commands
    v_headers(1) := 'Employee ID';
    v_widths(1)  := 11;
    v_headers(2) := 'Last Name';
    v_widths(2)  := 25;
    v_headers(3) := 'First Name';
    v_widths(3)  := 20;
                                                -- will not print this column, 
                                                -- just capture it for column page break
    v_headers(4) := NULL;                       --'Department Name'
    v_widths(4)  := 0;                          -- sqlplus COLUMN NOPRINT 
    v_headers(5) := 'Salary';
    v_widths(5)  := 16;
    -- override default number format for this column
    v_formats(5) := '$999,999,999.99';
    --
    PdfGen.init;
    PdfGen.set_page_format(
        p_format            => 'LETTER' 
        ,p_orientation      => 'PORTRAIT'
        ,p_top_margin       => 1
        ,p_bottom_margin    => 1
        ,p_left_margin      => 0.75
        ,p_right_margin     => 0.75
    );
    PdfGen.set_footer;                          -- 'Page #PAGE_NR# of "PAGE_COUNT#' is the default
                                                -- sqlplus TITLE command
    PdfGen.set_page_header(
        p_txt_center        => 'Employee Salary Report'
        ,p_font_family      => 'helvetica'
        ,p_style            => 'b'
        ,p_fontsize_pt      => 16
        ,p_txt_left_2       => 'Department: !PAGE_VAL#'
        ,p_font_family_2    => 'helvetica'
        ,p_fontsize_pt_2    => 12
        ,p_style_2          => 'n'
    );
    --
    as_pdf3.set_font('courier', 'n', 10);
    PdfGen.refcursor2table(
        p_src                       => p_src
        ,p_widths                   => v_widths
        ,p_headers                  => v_headers
        ,p_formats                  => v_formats
        ,p_bold_headers             => TRUE     -- also light gray background on headers
        ,p_char_widths_conversion   => TRUE
        ,p_break_col                => 4        -- sqlplus BREAK ON column
        ,p_grid_lines               => FALSE
    );
    v_blob := PdfGen.get_pdf;
    RETURN v_blob;                              
END;
-- begin sql
a AS (
    SELECT e.employee_id, e.last_name, e.first_name, d.department_name
        ,SUM(salary) AS salary          -- emulate sqplus COMPUTE SUM
    FROM hr.employees e
    INNER JOIN hr.departments d
        ON d.department_id = e.department_id
    GROUP BY GROUPING SETS (
                                        -- seemingly useless SUM on single record, 
                                        -- but required to get detail records
                                        -- in same query as the subtotal and total aggregates
        (e.employee_id, e.last_name, e.first_name, d.department_name)
        ,(d.department_name)            -- sqlplus COMPUTE SUM of salary ON department_name
        ,()                             -- sqlplus COMPUTE SUM of salary ON report - the grand total
    )
), b AS (
    SELECT employee_id
        -- NULL last_name indicates an aggregate result.
        -- NULL department_name indicates it was the grand total
        -- Similar to the LABEL on COMPUTE SUM
        ,NVL(last_name, CASE WHEN department_name IS NULL
                            THEN LPAD('GRAND TOTAL:',25)
                            ELSE LPAD('DEPT TOTAL:',25)
                        END
        ) AS last_name
        ,first_name
        ,department_name
        ,salary
    FROM a
    ORDER BY department_name NULLS LAST     -- to get the aggregates after detail
        ,a.last_name NULLS LAST             -- notice based on FROM column value, not the one we munged in resultset
        ,first_name
)
SELECT get_blob(CURSOR(SELECT * FROM b)) FROM dual
;
/
```
An example page from that pdf report follows. That is all a bit much for something you are doing adhoc, but
given the scrutiny placed over moving code into production in modern corporate environments, you may
find yourself reaching at times. This is a capability that will allow you to reach pretty far without
deploying any code to production. That is nice to have in your back pocket when everyone is looking to
you for a miracle.

![PDFGen_Page](/images/test0_pg1.png){:.centered}

# Writes

Since this facility is part of a SELECT query (or the SELECT portion of DML), it is limited to read only operations. If your inline procedure or function will perform DML or DDL, you need to declare it with PRAGMA AUTONOMOUS_TRANSACTION 
as mentioned in this [asktom question](https://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:9537276300346582444).

I won't disagree with the assertion that it is generally a bad idea, but when constrained by circumstances, one does as needs must.

# Conclusion

Although avoidance of context switching may have been the driving force behind Oracle delivering
the inline PL/SQL capability for the SQL engine, we can take advantage of it for many reasons.
I hope these examples were enlightening.
