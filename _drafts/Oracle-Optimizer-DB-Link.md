---
layout: post
title: Limitations of Oracle Optimizer through DB Link
exerpt: ""
date: 2022-05-07 11:00:00 +0500
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
> Access to the views is granted as needed, including to our *DB_LINK* user. 

In that scenario our local optimizer may choose to do a nested loop with the remote table when a full
table scan would be better, or vice versa. When it submits the request to the remote system, the optimizer
on the remote system can see the table structure and optimizer statistics. It does the best it can to fulfill
the request, but if the wrong request was sent, there isn't much it can do. 

In some ways it may turn out better
when there are multiple remote tables/views involved. Our local optimizer may decide to send 
an entire sub-query to the remote system, hash joining the returning resultset rather than attempting a nested loop
on each remote table/view. The remote system can then decide the best plan to achieve
the resultset it will return. 

# Demonstration

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

On the remote system (**rhl1pdb**) as demo user **sh** we create a view and grant *SELECT* privilege
to **dbl_read_user**. 


```plsql
CREATE OR REPLACE VIEW sales_v
AS
SELECT *
FROM sales
;
GRANT SELECT ON sales_v TO dbl_read_user;
```

A quick check logging in as user **dbl_read_user** shows we can access the view, but cannot see the table **sales**.

```
SQL> select count(*) from sh.sales_v;

  COUNT(*)
----------
    918843

SQL> select count(*) from sh.sales;
select count(*) from sh.sales
                        *
ERROR at line 1:
ORA-00942: table or view does not exist
```




# Inconsistent Behavior

In the described scenario our local optimizer makes a guess on the cardinality of the remote data source
and assumes no index access.

a) cardinality guessed?
b) no index guessed?
c) what cardinality flips it from nested loop to hash?
