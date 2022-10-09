---
layout: post
title: Tuning Query with OR Conditions in a NOT EXISTS
exerpt: "When tuning a query you sometimes run into uncommon problems that do not respond to the bag of tricks you brought to the game like hints and small rewrites. We know that 'OR' conditions can't be hash joined and have seen the optimizer join twice with a 'CONCATENATION'. It doesn't do that for 'anti-joins', so you may have to do something counter intuitive."
date: 2022-10-09 11:00:00 +0500
categories: [oracle, sql]
tags: [oracle, sql, antijoin]
---
# Introduction

While tuning an Oracle query provided by a business user I ran into a roadblock that confounded me. The query contained
a *NOT EXISTS* clause against a large row set, one that was a Common Table Expression (CTE or WITH clause)
that was relatively complex. The CTE plan was fine, but it resulted in about a million rows. An ideal plan
to satisfy the *NOT EXISTS* from my perspective
would use a *HASH ANTIJOIN* against that row set. That was not happening. It was doing a *FILTER* operation.
I'm not 100% sure what Oracle is doing under the covers on the *FILTER*.

# Problem

The problem SQL is too complex and intertwined with my client's business to reproduce here, but pseudo code is
good enough for the discussion.

```plsql
WITH ne AS (
    SELECT 
        lookup_value 
    FROM large_result_set L
    WHERE some_fancy_condition = 'TRUE'
)
SELECT --+ cardinality(a 1000000)
    my_key_value, my_lookup_name, column_list
FROM another_large_result_set a
WHERE 
    -- this is done using an index one by one which is ok
    NOT EXISTS (
        SELECT 1
        FROM some_other_table s
        WHERE s.key_value = a.my_key_value
    )
    -- below is the problem antijoin
    AND NOT EXISTS (
        SELECT 1
        FROM ne
        WHERE ne.lookup_name = a.my_lookup_name
            -- this DECODE represents an OR condition
            AND DECODE(ne.lookup_value, '*', 1, a.my_key_value, 1, 0) = 1
            /* --I would prefer this syntax
                AND (ne.lookup_value = '*' OR ne.lookup_value = a.my_key_value)
            */
    )
```

The picture below (Toad tree view) from the actual plan shows a *FILTER* operation with 3 components - *another_large_result_set*,
*some_other_table*, and *ne*. We can tell from the fact that it shows the index lookup, that it is doing a 
one by one index probe of *some_other_table*. Less obvious what it is doing with *ne*, but in the absence of other
evidence, we must assume it is walking through the heap in memory looking for a match. Maybe it has sorted it and is 
doing something smarter than that, but it is not a hash.

| *Figure 1 - Explain Plan of Filter Operation for Antijoin with OR* |
|:--:|
| ![](/images/hash_aj_or_1.png) |
{:.img-table-centered}

Why could it not hash the *ne* row set and probe it with a *HASH ANTIJOIN*? Because you cannot hash an *OR* condition.

If it is a regular join instead of an anti-join that has this *OR* condition, the optimizer can
break the problem into multiple *HASH JOIN*'s and do
a *CONCATENATION* operation of the results for each of the *OR* conditions. 

I do not recall seeing a plan where the optimizer chooses two separate *ANTIJOIN* operations in series. 
It would not be a *CONCATENATION*
operation between them, but the opposite of that because they are anti-joins. 
I don't think the Oracle optimizer has an operation for splitting an *ANTIJOIN* into two *ANTIJOIN*'s in sequence.

# Solution

I first thought maybe the *DECODE* function on the value, being a function, was confusing the optimizer.
Rewriting that in my preferred syntax with an *OR* condition rather than the implied *OR* of the *DECODE*
did not solve the issue. It was a long shot given my understanding of the optimizer, but I don't know everything,
the optimizer is constantly evolving, and I wanted to give it the best chance to solve the issue.

I've seen this before on regular joins with a similar construct of a lookup table having wild cards. In that
case it was multiple conditions in the join with wild cards optional on all of them. The optimizer
was overwhelmed and just did a giant filter as we saw with this anti-join. One would think it might
be able to break the problem into multiple joins with concatenation, but the multiple join conditions
seems to have exceeded the optimizer's breadth of options.

I recall having to break the problem down into two
separate joins manually, one for the wild card and one for the direct match of the key, at least for
the first of multiple wild-carded join conditions. That at least was able to reduce the join set
to one that could be filtered for the remaining conditions.

I tried a similar trick here rewriting the query as two separate *NOT EXISTS* with an *AND* 
condition between them.

```plsql
WITH ne AS (
    SELECT 
        lookup_value 
    FROM large_result_set L
    WHERE some_fancy_condition = 'TRUE'
)
SELECT --+ cardinality(a 1000000)
    my_key_value, my_lookup_name, column_list
FROM another_large_result_set a
WHERE 
    -- this is done using an index one by one which is ok
    NOT EXISTS ( -- not exists 0
        SELECT 1
        FROM some_other_table s
        WHERE s.key_value = a.my_key_value
    )
    AND NOT EXISTS ( -- not exists 1
        SELECT 1
        FROM ne
        WHERE ne.lookup_name = a.my_lookup_name
            AND ne.lookup_value = '*' 
    ) AND NOT EXISTS ( -- not exists 2
        SELECT 1
        FROM ne
        WHERE ne.lookup_name = a.my_lookup_name
            AND ne.lookup_value <> '*' 
            AND ne.lookup_value = a.my_key_value
    )
```

| *Figure 2 - Explain Plan of Filter Operation for Antijoin no OR* |
|:--:|
| ![](/images/hash_aj_or_2.png) |
{:.img-table-centered}

That worked swimmingly. The optimizer hashed all of the rows from *ne* with a lookup value of '\*',
also hashed all of the rows from *ne* with at lookup value that was not '\*', and did a *HASH ANTIJOIN*
against each in pipelined series. This was a much more efficient plan that will also scale well as
the number of rows increases over time. The actual run time was about a third what it was before the re-write.
Honestly I expected better than that, so maybe Oracle is doing some form of optimization on that filter
operation under the covers.

It is still showing the antijoin of *some_other_table* as a *FILTER*. I do not know why this presents as
a filter rather than a *NESTED LOOP ANTIJOIN*. I've seen nested loop antijoin be shown by the optimizer 
rather than filter in other situations and am unsure what the difference is. I went trolling through
Jonathan Lewis's fine book *Cost-Based Oracle Fundamentals*, but did not find an example of a plan
using *FILTER*. It may be something that Oracle added after 10g which is the release the book covered.

# Conclusion

When dealing with *OR* join conditions (*IN* lists are *OR* conditions too), 
there is only so much the optimizer can do. When you are
getting an unacceptable plan for a query with *OR*s in the join conditions, or especially anti-join conditions,
consider how you might be able to rewrite the query with two joins, one for each of the *OR*s. 
