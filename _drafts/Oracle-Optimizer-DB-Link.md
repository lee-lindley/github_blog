---
layout: post
title: Limitations of Oracle Optimizer through DB Link
exerpt: ""
date: 2022-05-08 11:00:00 +0500
categories: [oracle, sql]
tags: [oracle, sql, optimzer, tuning, db_link]
---
# Introduction

While tuning existing queries for a client I ran into befuddling behavior by the Oracle Optimizer
with respect to *VIEW*'s the query accessed through a *DB_LINK*. We are accustomed to an optimizer
that is omniscient when it comes to *TABLE* structure and optimizer statistics on the local database. It does
not matter how restricted the schema executing the query may be (as long as it has privilege
to read the *TABLE* or *VIEW*) - the optimizer knows all. It can pierce the veil through a *VIEW* the
calling schemas can see to the underlying *TABLE*(s), associated structure, and optimizer statistics.

It seems obvious now, but the local optimizer only knows as much about the remote system as is available
to the account used by the *DB_LINK*.

# Surprising Scenario

When we are accessing a *TABLE* on which that account has *SELECT* privilege,
we are in pretty good shape. That account can see the structure of the *TABLE* (partitions, indexes, etc..)
as well as optimizer statistics and so can our local optimizer. 
Our local optimizer can make reasonable decisions about how to structure
our queries to the remote system. 

When we access a *VIEW* on the remote system, and our *DB_LINK* account
does not have visibility on the underlying *TABLE*(s) the view references, we are in a dark hole. Our local optimizer
cannot see the structure or optimizer statistics on that *TABLE* or *TABLE*s. 

> The subject tables in my client's system are in schemas populated by a commercial vendor that we
> touch as little as possible.
> A separate schema is granted *SELECT* on those tables. In that schema the views are created.
> Access to the views is granted as needed, including to our *DB_LINK* user. This 
> isolation of the vendor objects seems appropriate and works fine on that single database. 

In that scenario our local optimizer may choose to do a nested loop with the remote table when a full
table scan would be better, or vice versa. When it submits the request to the remote system, the optimizer
on the remote system can see the table structure and optimizer statistics. It does the best it can to fulfill
the request, but if the wrong request was sent, there isn't much it can do. 

In some ways it may turn out better
when there are multiple remote tables/views involved. Our local optimizer may decide to send 
an entire sub-query to the remote system, hash joining the returning resultset rather than attempting a nested loop
on each remote table/view. The remote system can then decide the best plan to achieve
the resultset it will return. 

# Demonstration Setup

We have two databases. The remote database is named **rhl1pdb** and our local database is **leepdb**. (Don't judge me on 
the names. At the time I created them I was only thinking about the install.) The Oracle sample
schemas are installed on both systems.

On the remote system (**rhl1pdb**) we create a limited privilege user for access from a *DB_LINK*.

```plsql
CREATE USER DBL_READ_USER IDENTIFIED BY "my secret password"  
DEFAULT TABLESPACE "USERS"
TEMPORARY TABLESPACE "TEMP";
-- SYSTEM PRIVILEGES
GRANT ALTER SESSION TO dbl_read_user ;
GRANT CREATE SESSION TO dbl_read_user ;
```

Also on the remote system (**rhl1pdb**) as demo user **sh** we grant *SELECT* privilege
to schema **lee** on table *SALES* with grant option. This is significant. If we keep
the grant option, our remote user can see the underlying table enough to get statistics
on it.

```plasl
GRANT SELECT ON sales TO lee WITH GRANT OPTION;
```

As user **lee** we create a view and grant select to our *DB_LINK* user.

```plsql
CREATE OR REPLACE VIEW sales_v
AS
SELECT *
FROM SH.sales
;
GRANT SELECT ON sales_v TO dbl_read_user;
```

A quick check logging in as user **dbl_read_user** shows we can access the view, but cannot see the table **sales**.

```
SQL> select count(*) from lee.sales_v;

  COUNT(*)
----------
    918843

SQL> select count(*) from lee.sales;
select count(*) from lee.sales
                         *
ERROR at line 1:
ORA-00942: table or view does not exist
```

On our primary system (**leepdb**) as our test user we create the *DB_LINK* named **rhl1_dbl_read**
and successfully test our access to the new view.

```plsql
CREATE DATABASE LINK RHL1_DBL_READ 
CONNECT TO dbl_read_user IDENTIFIED BY "my secret password"
USING 'rhl1pdb';
--
SELECT COUNT(*) from lee.sales_v@rhl1_dbl_read;
-- returned 918843
```

If I try to get an explain plan for that query, no joy:

```
SQL> explain plan for SELECT COUNT(*) from sh.sales_v@rhl1_dbl_read;
explain plan for SELECT COUNT(*) from sh.sales_v@rhl1_dbl_read
*
ERROR at line 1:
ORA-01039: insufficient privileges on underlying objects of the view
ORA-02063: preceding line from RHL1_DBL_READ
```

!!!!!!!!!!!!!!!! ARRRGGGHHH somehow it is getting the cardinality for that table on my database !!!!!!!!!!!!!!!!!!!!!!!!!1

Next we create a local table with some key values from **SH.sales** that we can use in a query
that joins to the remote view.

```plsql
CREATE TABLE loc_sales_keys
AS SELECT prod_id from sh.products@rhl1_sh
WHERE rownum <= 5;
```

# Inconsistent Behavior

In the described scenario our local optimizer makes a guess on the cardinality of the remote data source
and assumes no index access.

a) cardinality guessed?
b) no index guessed?
c) what cardinality flips it from nested loop to hash?
