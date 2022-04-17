---
layout: post
title: Sql Tuning for Multiple Use Cases
exerpt: "Using parameterized hints in Dynamic SQL to satisfy multiple use cases with one set of code."
date: 2021-10-17 20:30:00 +0500
categories: [oracle, sql, plsql]
tags: [oracle, sql, plsql, tuning]
---
# Introduction

The primary goal of tuning an Oracle SQL statement is to obtain the best performance 
(shortest response time and minimal resourse utilization)
for the intended task.  The best answer usually achieves both,
but occasionally you sacrifice some of one for the other.

The Oracle optimizer is good at achieving an optimal execution plan (*access path, join order, join mechanism*),
defined as minimal *cost*, if it has sufficient statistics about the data.
There are reams of books and articles about it. I bashed my
head against the rocks of Jonathan Lewis's book *Cost Based Oracle Fundamentals* for several years. The subject is
so massive and fluid, I do not even pretend to get it all. This article is not about that.

**The problem to be solved is that we have a single SQL statement with multiple use cases, each of which require a
different execution plan.**

I will present two use cases for this article. 

- A nightly batch that touches a full set of data
- An incremental, near real-time batch that touches a very small subset of the data

| *Multiple Use Case SQL Statements* |
|:--:|
| ![use_case_sql_tuning.gif](/images/use_case_sql_tuning.gif) |
{:.img-table-centered}

(In practice I have third and forth use cases of nightly "correction" batches that touch 
what can be either middling sized sets of the data or really huge, almost full sized subsets.)

How can this be addressed?

# Adaptive Plan

Modern Oracle databases have *dynamic* execution plans where the database attempts
to re-plan after finding that the statistics used to calculate the original plan do not match reality.
That is a good idea, and I am sure it is helpful in situations where the data changes slowly over time. For example
at the start of the day one of the source tables may be tiny and a Full Table Scan (FTS) access path is chosen.
As the table fills during the day, the *adaptive plan* logic may determine an Index Access Path is lower
cost and switch to it. Adaptive plans may be a great idea and it may solve many issues in many environments,but I have
not personally experienced that. Your mileage may vary.

The adaptive plan mechanism does not help sufficiently with this use case. 
Imagine that we start with the nightly batch case where the
driver table statistics would tip off the optimizer to use FTS and Hash join plans. Works great for the nightly batch
which runs for thirty minutes (or hours).
Now we begin the daily mini-batches. We re-gather statistics on our driver table, but Oracle does not bother
to re-evaluate our execution plan. Nope, the idiot savant has a perfectly good plan already, and it is going to use it. It does
full table scans and hash joins to obtain what we need for our tiny subset of accounts,
wasting resources and taking forever on the first *near real-time* run.
Probably on the second one too. By some later run (perhaps hours later) the adaptive plan technology may figure
out something isn't right and substitute a better plan. By now though we are fielding calls from the business that
stuff isn't working right. Booooooooo!

Even worse would be the behavior that night. Oracle has a good plan now for the small driver table footprint but when
we load it up for the nightly batch, Oracle continues to use an unscalable execution plan with index access that runs
all night. Uggghhhh!

# Invalidate the Plan

We could do something to invalidate the plan forcing Oracle to evaluate the SQL again with the changed statistics.

## DBMS_SHARED_POOL

Although there is a package named DBMS_SHARED_POOL that provides fine grained control over invalidating
individual sql plans, I have yet to meet a DBA willing to grant execute on the package and I do not blame
them one bit. You would also have to do so on every instance of a RAC.

## DBMS_STATS

There is also a parameter you can set during statistics gathering that is supposed to invalidate any SQL
plan using the table (no_invalidate => FALSE), but it comes with caveats and is generally ignored for at least a while.
(DMS_STATS.AUTO_INVALIDATE default value is to let Oracle decide and it does not invalidate the plans immediately.)

## DDL Invalidates Plans

At least for Oracle versions 10 through 19, updating a comment on a table counted as DDL that marks all SQL that
used the table as needing to be parsed again. (You do have to make the comment different than the existing one though
or it is a noop.) Observe:

```plsql
select * from app_log_app
;
SELECT sql_id,  parse_calls, sql_text
FROM v$sql
WHERE sql_text LIKE '%app_log_app'
AND sql_text NOT LIKE '%v$sql%'
;
```

SQL_ID | PARSE_CALLS | SQL_TEXT
---|---:|---
4t32sfzvq63zf | 1 | select * from app_log_app

Notcie the number of parse calls is *one*.

```plsql
BEGIN
EXECUTE IMMEDIATE q'{COMMENT ON TABLE app_log_app IS 'invalidate plan comment }'
    ||TO_CHAR(SYSDATE,'MM/DD/YYYY HH24:MI:SS')
    ||q'{'}';
END;
/
-- repeat SQL listed above to query the table then v$sql.
```

SQL_ID | PARSE_CALLS | SQL_TEXT
---|---:|---
4t32sfzvq63zf | 2 | select * from app_log_app

DDL, even a simple COMMENT, invalidates all plans that reference the object, forcing a re-parse. So we could do this
once during the nightly batch after gathering statistics on our driver table, and again on the first run of the
mini-batch in the morning. It is a bit fragile though as a solution, and there is a downside that there could
be large numbers of SQLs that need to be re-parsed if the table you chose was a common one. The scenario I 
describe it would be a driver table specific to our process, but one should be aware of the possible impact. Nevertheless,
this is a viable answer for the multiple Use Case issue assuming we keep gathering statistics 
on the driver table when needed.

# Multiple Sql Statements

So what if we just use two different SQL statements in our code so that we get the right plan for each case?

My response, having gone down that road, is "what if they come up with different answers?". This is particularly
egregious if you wind up only testing one path. Worse yet when you go to make changes, you have to make it in
two places.

It may not be possible to completely avoid this though. If you decide to use a pattern that sometimes uses Update/Delete
of records in place, and sometimes does a Create Table AS (CTAS) followed by partition exchange, then of course
you will be executing different SQL for the different use cases.

Yet whenever possible we should strive to have a single set of code that implements the business logic. This implies
we will likely be using Dynamic SQL for the task. Even so, when building the dynamic SQL it would be best if we could avoid
having more than one set of code that implements the same *logic*.

"Sure" you say, you get that. But there will come a time when you do it anyway for expediency. I've done that, and have
the scars to prove it. If you are like me, you will likely have to learn the hard way once or twice before it sticks.
Maybe this article will help you avoid as many scars as this slow learner has accumulated.

It turns out that using two different SQL statements is the answer I'm proposing as far as the database is
concerned, just using a common base.

# Use Case Specific Hints

Many purists turn up their noses at hints, and lecture you to just make sure your statistics are correct
so that the optimizer can determine the best answer. 

I can happily engage in a religious war dialogue on the subject if you are one who wishes to be pendantic on either side.
I can argue both sides fairly well. I agree that if you can solve the issue by controlling the statistics on the tables
and indexes, then by all means do so. In particular this means making rational decisions about histograms
and partition level statistics. Mastering that can address many issues with both current and future SQL. 

People who argue against hints often have their opinion informed by too often seeing inappropriate hints used
by well meaning developers or business users. I get that too. I'm even guilty of it and I'm sure you are too at
least at times over your career.

I can also make the case that it is not possible to solve all problems without hints. The case of a piplelined TABLE(f()) function
in your query means the optimizer has no clue how many records will come from it. You can give it a *cardinality* hint,
but what if the number of records is variable with the use case?

We also just demonstrated that gathering statistics again on a table does not invalidate existing plans.

There are other scenarios where you as the data expert simply have more information than the optimizer will
determine from statistics, regardless of how carefully you gather or even customize the statistics it uses. That
rabbit hole is deep. 

Let's embrace the suck and use a minimal set of hints that will guide the optimizer into understanding our
use case. The term "minimal" leaves some wiggle room. For the scenario I described we might get by with
as little as a single *CARDINALITY* hint on the driver table plus a degree of parallel hint for the entire query. 
Example:

```plsql
    v_sql := q'!INSERT INTO ...
        SELECT /*+ CARDINALITY(d __DRIVER_TABLE_CARDINALITY__) __PARALLEL__ */
            ...
        FROM my_driver_table d
        INNER JOIN ...
    !';
    ...
    EXECUTE IMMEDIATE REPLACE(
        REPLACE(v_sql, '__DRIVER_TABLE_CARDINALITY__'
            ,CASE WHEN p_is_nightly_batch THEN '1000000' ELSE '10' END
            ), '__PARALLEL__', 
                ,CASE WHEN p_is_nightly_batch THEN 'PARALLEL(16)' ELSE 'NO_PARALLEL' END
    );
```

When we call our procedure the parameter p_is_nightly_batch will be TRUE or FALSE. Now the numbers 10 and a million
are not exactly what we have in our driver table, and in fact we could get a mini-batch where the number
of records is enough that we would be better off with a different plan than that we get from telling
the optimizer to assume 10 records. That is something you might have to experiment with.

Note that this winds up being two different SQL statements in v$sql, each with a different plan.

Putting the parallel degree into the hint is somewhat controversial. One can say it should be determined
from the DOP on the table. There are other considerations depending on system load. That is a topic for another
day.

In practice I would likely put those values, 10 and 100000 plus the DOP, into a parameter table that I could adjust at run time.
(See *app_parameter* in [plsql_utilities](https://github.com/lee-lindley/plsql_utilities).)  That way if something
happened either over time or during a particular event, I could adjust the values fed to the optimizer on the fly,
and since the text of the SQL is different from that already parsed, it of course parses again with the new hint value.

This may not be enough though. There are cases where the optimizer simply refuses to come up with 
a correct plan without hints, and you must bludgeon it into submission.
I offer [this article on hash joins](https://jonathanlewis.wordpress.com/2020/12/08/hash-joins-2/)
as an example of how you might want to compel the optimizer to do things a particular way (though I am almost certain 
Mr. Lewis would never say the words "bludgeon optimizer into submission").

You may likely find yourself needing to use more than a single cardinality hint plus degree of parallelism. Example:

```plsql
    v_sql := q'!INSERT INTO ...
        SELECT 
            /*+ 
                __USE_CASE_HINT__
            */
            ...
        FROM my_driver_table d
        INNER JOIN big_table1 t1
            ON ...
        INNER JOIN midsize_table2 t2
            ON ...
        INNER JOIN big_table t3
            ON ...
    !';
    ...
    EXECUTE IMMEDIATE REPLACE(v_sql, '__USE_CASE_HINT__'
        ,CASE WHEN p_is_nightly_batch 
            THEN q'+
                leading(d t1 t2 t3)
                full(d)
                use_hash(t1) full(t1) swap_join_inputs(t1)
                use_hash(t2) full(t2) no_swap_join_inputs(t2)
                use_hash(t3) full(t3) swap_join_inputs(t3)
                +'
            ELSE q'+
                leading(d t1 t2 t3)
                full(d)
                use_nl(t1) index(t1)
                use_nl(t2) index(t2)
                use_nl(t3) index_ss(t3)
                +'
            END
    );
```

As you can imagine it can get fairly complex especially if you have a very large query perhaps
broken down using WITH query subfactoring (as I heartily recommend). It would be good to establish
a naming convention for the hint parameters. You may also want to add an option to the procedure
call that prints the SQL with DBMS_OUTPUT rather than executing it. It will help with 
debugging and establishing the best plan.

# Conclusion

When you have one business problem with multiple SQL optimization use cases, make every attempt
to keep the business logic in a single set of code. Decorate the dynamic sql with replaceable hints
that are appropriate for the use case. Use the minimal set of hints that achieve the objective, 
but also give yourself some room to adapt when the datbase changes out from under you.
Parameterizing the hint values so that they may be changed at run time is a good option
that does NOT mean you are changing the code at run time. The actual business logic
is that which was tested and promoted to production. Only the optimization is changing
and that in response to a business need. This is a solution that comports with best practices
while providing some flexibility.
