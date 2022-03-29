---
layout: post
title: Devops Data Deployment for Oracle
date: 2022-03-28 19:00:00 +0500
categories: [plsql, sql, oracle]
tags: [oracle, sql, plsql, devops, deploy, CLOB]
---
# Introduction

Most large corporations have adopted some form of **Continuous Improvement** (CI)
strategy with a **Devops** build and deploy component. 
The people who manage and direct these operations tend to
be Web guys and the tools tend to focus on deploying Web applications (this includes
middleware component on Java and .NET servers). If I am
guilty of stereotyping incorrectly, please tell me about organizations with 
a broader focus. I could be wrong, but that is what I have observed so far.

Other application types are supported, just not as first class citizens.
Specifically, database application deployment is a red-headed step-child
of Devops. I'm not blaming the Devops guys either. We database practioners
are likely guilty of obstruction. Any tool other than what we know works
will be looked upon with suspicion. 

# How Can We Deploy?

We need to deploy database application code (packages, table definitions, etc..)
and we need to deploy data (configuration table entries for example).

We have a suite of client tools we can use as Oracle clients for deploying
code and data manually, all of which require providing or storing a password:

- SQL\*PLUS
- imp
- sqlldr
- external tables (assuming you can deliver files to a database directory)
- sqlCL
- sqlDeveloper
- Toad

We can rule out the gui tools *SqlDeveloper* and *Toad*. These are not suitable for automated
deployment.

*SQL\*PLUS* is perfectly adequate for deploying our code to the database. Oracle uses it to 
deploy database code and patches. We have
all been using *SQL\*PLUS* for as long as we have used Oracle. It is clunky, limiting and antiquated,
but it works and we know how to make it do the needful.

Strangely, *sqlCL* does not seem well supported. I know some reasons for that including
security concerns with having Java client software on servers, but the
primary reason is likely inertia. This is a shame because *sqlCl* supports CSV file 
loading, which would address one of our two deployment needs.

I have not seen a Devops deployment configuration that uses anything other than *SQL\*PLUS*
to deploy code. If there are some, my guess is the database application guys are less
than thrilled. I could be wrong.

# Deploying Data

For batch operations we have mechanisms for transfering files between production
systems and/or with outside vendors. We have custom processes using
*sqlldr*, External tables and perhaps an ETL tool to import the data from those files. 
Yet the mechanisms
we have for slinging files around in a production environment are probably not supported
through Devops. 

We have files in our build that are on the deployment server and we have a *SQL\*PLUS* client.

Traditionally when faced with this we resort to what I call "stupid data load." We
create a series of Insert or Merge statements with hard coded data and run those
in *SQL\*PLUS*. Maybe we get a little more ambitious and create a single INSERT
or MERGE statement with a series of SELECT FROM DUAL UNION ALL ... statements
providing the input. It is still "stupid data load." Works fine for a small number of
records. Not so fine when you need to create several thousand records.

If Devops configured a capability to use *sqlldr*, we could do what we must. The use case
for it is limited though and the firm may not choose to invest the resources necessary
to configure and manage that capability.

# An Alternative for Loading Data with SQL\*PLUS

Using tools provided in [plsql_utilities](https://github.com/lee-lindley/plsql_utilities) we can
build deployment scripts that run in *SQL\*PLUS* handling relatively large amounts of
data as Comma Separated Value (CSV) records efficiently. 
Not as efficiently as using *sqlldr*, but good enough. 

From the documentation at [plsql_utilities/app_csv_pkg](https://github.com/lee-lindley/plsql_utilities/tree/main/app_csv_pkg)...

Read the data from a table (with an optional WHERE clause) and convert it
into a CSV CLOB with a header row. Break the CLOB into a set of quoted
string literals. Generate a script that

1. Creates a Private Temporary Table from the CLOB (input as contactenated string literals).
2. Inserts or Merges records from the PTT into the target Table

We call the deployment generator:

```plsql
SELECT APP_CSV_PKG.gen_deploy_merge(
    p_table_name    => 'MY_TABLE_NAME'
    ,p_key_cols     => 'ID'
    ,p_where_clause => 'id <= 10'
) FROM DUAL;
```

That provides the following script as a CLOB in your query result. Note that
if the size of the CSV CLOB holding the records was greater than 32767, 
it would be represented by a concatenated set of quoted literals instead
of just one as shown here.

```plsql
BEGIN
    APP_CSV_PKG.create_ptt_csv(q'{"ID","MSG","DT"
1,"testing...","03/26/2022"
2,"testing...","03/27/2022"
3,"testing...","03/28/2022"
4,"testing...","03/29/2022"
5,"testing...","03/30/2022"
6,"testing...","03/31/2022"
7,"testing...","04/01/2022"
8,"testing...","04/02/2022"
9,"testing...","04/03/2022"
10,"testing...","04/04/2022"
}'
);
END;
/
MERGE INTO MY_TABLE_NAME t
USING (
    SELECT *
    FROM ora$ptt_csv
) q
ON (
    t."ID" = q."ID"
)
WHEN MATCHED THEN UPDATE SET
    t."MSG" = q."MSG"
    ,t."DT" = q."DT"
WHEN NOT MATCHED THEN INSERT(
    "ID", "MSG", "DT"
) VALUES (
    q."ID", q."MSG", q."DT"
);
COMMIT;
```

# Conclusion

I explored multiple mechanisms to avoid "stupid data load." The obvious best choice would be if the right
tools for the job like *sqlldr*, *imp*, or *sqlCL* were available, but we play the hand we are dealt. If you
are stuck with *SQL\*PLUS*, this package can be a solution.

