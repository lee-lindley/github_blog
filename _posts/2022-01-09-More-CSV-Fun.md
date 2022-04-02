---
layout: post
title: More CSV Fun - Turn CSV into SQL Resultset
date: 2022-01-9 20:30:00 +0500
categories: [oracle, sql, plsql, perl]
tags: [oracle, sql, plsql, perl, regexp]
---
# Ad-Hoc CSV Parsing

I've been going down some dark holes trying to be able to use CSV data as a basis for starting an ad-hoc query.
I finally arrived at something workable. While I was at it I built myself some tools for manipulating
lists and strings that we can use to make PL/SQL a little less verbose for some tasks.

Consider the classic way you can hard code data in a SQL script:

```plsql
WITH a AS (
    SELECT 'abc' AS col1, 123 AS col2, 'xyz' AS col3
    UNION ALL
    SELECT 'def' AS col1, 456 AS col2, 'ghi' AS col3
    UNION ALL
    SELECT 'lmn' AS col1, 789 AS col2, 'opq' AS col3
) 
SELECT * FROM a
;
```

Uggh, that is painful. If you have bind variables and maybe some TO_DATE functions to throw into
the mix it can get even worse.

# Perlish Utility Methods

There are two basic tool types provided by this User Defined Type (UDT) Object you can find 
at [perlish_util_udt](https://github.com/lee-lindley/plsql_utilities#perlish_util_udt). One type are
methods that work like some Perl builtin methods for manipulating lists. The other type facilitates
parsing CSV data.

One of the constructor methods for *perlish_util_udt* takes a string as a parameter that it expects
to contain Comma Separated Values (CSV) per RFC4180 (which means anything Excel puts out as CSV and anything else you are 
likely to find that calls itself CSV). The constructor calls *perlish_util_udt.split_csv* on that puppy
and populates the member attribute named *arr* with the resulting collection of VARCHAR2 elements.
Together with the *perlish_util_udt.get* method that takes an index parameter, this makes
it possible to build the object and access those array elements in a SELECT.

Now that is all well and good for a single line of CSV, but what if we have a bunch of lines of CSV? The answer
is another method in that toolbox named *perlish_util_udt.split_clob_to_lines*.

Together these two static methods plus that instance constructor and *get* method give us the tools we need
to deal with that set of CSV records. 

An example tells the story better than all those words:

```plsql
WITH a AS (
    SELECT perlish_util_udt(t.column_value) AS p
    FROM TABLE( perlish_util_udt.split_clob_to_lines(
q'["abc",123,xyz
def,456,"ghi"
lmn,789,opq]'
                                                    )
    ) t
) -- remember you need a table alias to access object methods and attributes
-- thus making the table alias x for a here.
SELECT x.p.get(1) AS col1, x.p.get(2) AS col2, x.p.get(3) AS col3
FROM a x
;
```

which results in the following (where the quotes were put around the pieces by SqlDeveloper upon Text output:

    "COL1"	"COL2"	"COL3"
    "abc"	"123"	"xyz"
    "def"	"456"	"ghi"
    "lmn"	"789"	"opq"

Looking at the query in pieces

```plsql
                perlish_util_udt.split_clob_to_lines(
q'["abc",123,xyz
def,456,"ghi"
lmn,789,opq]'
                                                    )
```

*split_clob_to_line* returns a collection of VARCHAR2(4000) elements. We wrap that function call in a TABLE cast (perhaps
unnecessarily depending on your Oracle version), and now we can select those lines as rows from the input string.

```plsql
    SELECT t.column_value AS r
    FROM TABLE( perlish_util_udt.split_clob_to_lines(
q'["abc",123,xyz
def,456,"ghi"
lmn,789,opq]'
                                                    )
    ) t
;
```
Showing the result in JSON may or may not help. All of the outputs from SqlDevloper want to mess with the quotes.
This shows those quotes are still in the data

    {
      "results" : [
        {
          "columns" : [
            {
              "name" : "R",
              "type" : "VARCHAR2"
            }
          ],
          "items" : [
            {
              "r" : "\"abc\",123,xyz"
            },
            {
              "r" : "def,456,\"ghi\""
            },
            {
              "r" : "lmn,789,opq"
            }
          ]
        }
      ]
    }

Now that we have our CSV input broken into lines, we need to break the individual lines into fields. This
is where we usually run into trouble with the SQL toolbox. If you start thinking about reaching for PIVOT/UNPIVOT
remember that our data has natural order (we want the columns to come out in an order or with names), but there
is nothing in the data itself that let's us specify that to the SQL engine. You can use a bunch 
of REGEXP_SUBSTR calls to parse the pieces into fields, but c'mon man, that is worse than the problem we are trying
to solve. Well, usually anyway. But if we can hide that complexity in an object that lets us split the
data into fields and then access the fields by index number, we preserve our natural order without
getting too gross.

That leads us to using *split_csv* to parse that CSV row/line into a collection:

```plsql
    SELECT perlish_util_udt(t.column_value) AS p
    FROM TABLE( perlish_util_udt.split_clob_to_lines(
q'["abc",123,xyz
def,456,"ghi"
lmn,789,opq]'
                                                    )
    ) t
```
Showing that result as JSON we can see each row *p* is an object with a member attribute that is a collection:

	{
	  "results" : [
	    {
	      "columns" : [
	        {
	          "name" : "P",
	          "type" : "LEE.PERLISH_UTIL_UDT"
	        }
	      ],
	      "items" : [
	        {
	          "p" : "LEE.PERLISH_UTIL_UDT(LEE.ARR_VARCHAR2_UDT(abc, 123, xyz))"
	        },
	        {
	          "p" : "LEE.PERLISH_UTIL_UDT(LEE.ARR_VARCHAR2_UDT(def, 456, ghi))"
	        },
	        {
	          "p" : "LEE.PERLISH_UTIL_UDT(LEE.ARR_VARCHAR2_UDT(lmn, 789, opq))"
	        }
	      ]
	    }
	  ]
	}

Then the final piece is to access individual elements of a collection. In PL/SQL you could just use the syntax

    p.arr(3) 

but that does not work in SQL. Fortunately, the object *perlish_util_udt* gives us a *get* function we can call
with an index parameter. The only trick is that in SQL in order to access an object method or attributes,
you must have a table alias for the "table" that provided the object instance in the query. In our example
our "table" is named "a" and our alias for "a" is "x".

```plsql
WITH a AS (
    SELECT perlish_util_udt(t.column_value) AS p
    FROM TABLE( perlish_util_udt.split_clob_to_lines(
q'["abc",123,xyz
def,456,"ghi"
lmn,789,opq]'
                                                    )
    ) t
) -- remember you need a table alias to access object methods and attributes
-- thus making the table alias x for a here.
SELECT x.p.get(1) AS col1, x.p.get(2) AS col2, x.p.get(3) AS col3
FROM a x
;
```

Now, the final result this time in json:

	{
	  "results" : [
	    {
	      "columns" : [
	        {
	          "name" : "COL1",
	          "type" : "VARCHAR2"
	        },
	        {
	          "name" : "COL2",
	          "type" : "VARCHAR2"
	        },
	        {
	          "name" : "COL3",
	          "type" : "VARCHAR2"
	        }
	      ],
	      "items" : [
	        {
	          "col1" : "abc",
	          "col2" : "123",
	          "col3" : "xyz"
	        },
	        {
	          "col1" : "def",
	          "col2" : "456",
	          "col3" : "ghi"
	        },
	        {
	          "col1" : "lmn",
	          "col2" : "789",
	          "col3" : "opq"
	        }
	      ]
	    }
	  ]
	}

# Conclusion

Pretty handy, eh? I'll save the other half of the functionality provided by *perlish_util_udt* for another day.
