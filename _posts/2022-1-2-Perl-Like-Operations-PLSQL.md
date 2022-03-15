---
layout: post
title: Perl Like Operations in PL/SQL
date: 2022-01-02 20:30:00 +0500
categories: [oracle, plsql, perl]
tags: [oracle, plsql, perl, regexp]
---

# The Problem Needs String Hacking

I was writing a PL/SQL function to generate a MERGE sql statement from input data, the name of a table, 
a list of columns in the input data,
and either a list of join columns or if not provided, getting them from the primary key constraint.
This is something I've done in Perl before relatively easily, and I wanted to turn my hand to
doing it in PL/SQL.

My first effort let me down the road of creating a Polymorphic Table Function to parse the CSV data.
It was a nice exercise and I produced a workable PTF implementation to parse a clob and generate
Oracle column data, but it is too complex for this relatively simple use case.

Next I turned towards doing it all in a set of inline PL/SQL WITH functions where I intended to have
it generate the UNION ALL set of rows for the data for the MERGE USING clause. I don't need to deploy
any code in the database that my team may or may not be able to support. It winds up being
a script I use to generate deployment code.

Man, what a grind.
PL/SQL is very good for database things. It sucks badly for text processing like this.
I found myself pining away for Perl operations like *split*, *map* and *join*. I'm not even going to complain
about lack of variable interpolation in PL/SQL strings and how you must concatentate everything with hideous syntax
because it is what it is. Yet some things we can do something about.

Consider needing to take a nested table of column names and generate the list that goes
with the INSERT(...) clause of the MERGE.

```sql
    v_s := 'WHEN NOT MATCHED THEN INSERT('||v_arr_cols(1);
    FOR i IN 2..v_arr_cols.COUNT
    LOOP
        v_s := v_s||', '||v_arr_cols(i);
    END LOOP;
    v_s := v_s||')';
```
Phew, that is ugly. Or how about the MERGE UPDATE clause? We need to write:

    WHEN MATCHED THEN UPDATE SET
        t.col1 = q.col1,
        t.col2 = q.col2

I started off writing a custom *join* function that also took a template string as an argument.
It was sort of a meld between the Perl *join* and *map* methods.

```sql
    FUNCTION join(
        p_arr           sys.ku$_vcnt
        ,p_separator    VARCHAR2 DEFAULT ', '
        ,p_template     VARCHAR2 DEFAULT NULL
    ) RETURN VARCHAR2
    IS
        l_s     VARCHAR2(4000);
        FUNCTION apply_template(
            p_val   VARCHAR2
        )
        RETURN VARCHAR2
        IS
            l_t VARCHAR2(4000);
        BEGIN
            IF p_template IS NULL THEN
                l_t := p_val;
            ELSE 
                l_t := REPLACE(p_template, '__PLACEHOLDER__', p_val);
            END IF;
            RETURN l_t;
        END;
    BEGIN
        IF p_arr.COUNT = 0 THEN
            RETURN NULL;
        END IF;
        l_s := apply_template(p_arr(1));
        FOR i IN 2..p_arr.COUNT
        LOOP
            l_s := l_s||p_separator||apply_template(p_arr(i));
        END LOOP;
        RETURN l_s;
    END -- join
    ;
```

Now to write the UPDATE portion I have
```sql
    v_s := 'WHEN MATCHED THEN UPDATE SET'
        ||join(v_arr_non_pk_cols, p_separator => ',
    '
                ,p_template => 't.__PLACEHOLDER__ = q.__PLACEHOLDER__')
    ;
```
It did not make me happy though. It was a little too customized.

I also had some utility methods as standalone functions (*split_csv* and *transform_perl_regexp*) that
really needed a package or user defined type home. I wound up creating a new User Defined Type
to hold my Perlish methods. I called it *japh_util_udt* originally, which comes from the phrase "I'm just another perl hacker."
Most people don't get that, so I renamed it to *perlish_util_udt*.

# Perlish Utility User Defined Type

You can find it in 
my [plsql_utilities github repository](https://github.com/lee-lindley/plsql_utilities). 

From the REAMDE.md in the repository:

## perlish_util_udt

It isn't Perl, but it makes some Perlish things a bit easier in PL/SQL. We also get
handy methods for splitting Comma Separated Value (CSV) text into lines and fields,
which you can use independent of the Perlish methods, and even one that turns a CSV
clob into a private temporary table.

> There is valid argument
that when you are programming in a language you should use the facilities of that language, 
and that attempting to layer the techniques of another language upon it is a bad idea. I see the logic
and partially agree. I expect those who later must support my work that uses this utility will curse me. Yet
PL/SQL really sucks at some string and list related things. This uses valid PL/SQL object techniques
to manipulate strings and lists in a way that is familiar to Perl hackers. 

A *perlish_util_udt* object instance holds an *arr_varchar2_udt* collection attribute which you will use when employing the following member methods;

- map
- join
- sort
- get
- combine

All member methods except *get* have static alternatives using *arr_varchar2_udt* parameters and return types, so you
are not forced to use the Object Oriented syntax.

It has static method *split_csv* (returns *arr_varchar2_udt*) that 
formerly lived as a standalone function in the plsql_utilities library as *split*.
We have a static method *split_clob_to_lines* that returns an *arr_varchar2_udt* collection
of "records" from what is assumed to be a CSV file. It parses for CSV syntax when splitting the lines
which means there can be embedded newlines in text fields in a "record".

There is a static procedure *create_ptt_csv* that consumes a CLOB containing lines of CSV data
and turns it into a private temporary table for your session. The PTT has column names from the
first line in the CLOB.

It also has a static method named *transform_perl_regexp* that has nothing to do with arrays/lists, but is Perlish.

Most of the member methods are chainable which is handy when you are doing a series of operations.


# Conclusion

I will work on a full demonstration to generate that deployable MERGE I described
in the introduction. Seeing it in real action rather than the contrived cases
of the documentation examples will show why I'm excited about it. Maybe PL/SQL
can suck a little less for hacking strings and lists.
