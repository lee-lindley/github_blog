---
layout: post
title: Context Swtich can Dwarf Runtime
date: 2022-04-08 21:30:00 +0500
categories: [oracle, sql, plsql]
tags: [oracle, sql, plsql, profiler]
---
# Introduction

In a prior post, [Cost of UDT Object Methods in SQL](https://lee-lindley.github.io/oracle/sql/plsql/2022/04/02/Object-Methods-in-SQL.html), I hypothesized a performance issue was due to Context Switching between SQL and PL/SQL engine. Moving
the suspect PL/SQL function call into a PIPELINED Table function solved the issue.

I had three outstanding doubts about what is really going on.

1. The magnitude of the performance penalty caught me off gaurd. I've remediated context switch issues with 
function calls in a SQL SELECT list before. I haven't seen one where the impact was this dramatic except
where the PL/SQL function was also calling SQL.
2. I am unsure whether Object type method calls in a SELECT list incur the context switch penalty. I know they do if they call PL/SQL methods. It is less clear when they do not. I've demonstrated that simple Object method calls do NOT incur the context switch penalty. That included calling a simple default constructor passing in a collection as well as an access *get* method that returned an element of the collection.
3. Unrelated, documentation for chained PIPELINED Table functions suggests bulk collect logic is not needed. I wanted to verify that.

# PL/SQL Hierarchical Profiler

If you haven't used the profiler before, this [Jeff Smith post](https://www.thatjeffsmith.com/archive/2019/02/sql-developer-the-pl-sql-hierarchical-profiler/) is a nice shortcut for getting started with SqlDeveloper taking care of some of the details.
Remember to recompile with debug any of your packages and types that are called by the outer block if you want details
for them.

This is the first time I used it where Object methods were significant (or where I expected them to be signifiant). Of note
is that the default constructor may be shown as anonymous. Likewise, a WITH function I tried showed up as anonymous (though
it definitely involved context switching as it was no faster than calling the function in the package).

# Chained PIEPLINED Table Functions and Bulk Fetching

My working solution to the context switch problem used chained PIPELINED Table functions. Per what I perceived the
documetation to be implying (without ever coming out and saying it), I implemented the function that reads the cursor
from the chain (not the first entry in the chain) without any bulk collection/array processing. The profiler
showed that function taking 3% of the execution time.

Adding BULK COLLECT LIMIT 100 to the cursor fetch and processing the resulting array caused it to take 3.1% of the execution
time (which could be noise). In my mind this confirms that there is no advantage to buffering that cursor from one
pipe row call to another. It is extra, unneeded processing.

# Conext Switching Really Can be That Expensive

First the pertinent profiler output from the chained PIPELINED Table function solution:

| *PLSQL Profiler Output Chained PIPELINED Table Functions* |
|:--:|
| ![](/images/Screenshot 2022-04-08 212727.gif) |
{:.img-table-centered}

I sliced and diced the function *split_csv* and the *perlish_util_udt* constructor multiple ways. I had *split_csv*
called from inside the *perlish_util_udt* constructor. I had it called separately as a parameter to *perlish_util_udt*
and I had it as a WITH function instead of calling it from the package. In all cases the profiler output looked
similar to the following:

| *PLSQL Profiler Output PL/SQL call in Select List* |
|:--:|
| ![](/images/Screenshot 2022-04-08 212907.gif) |
{:.img-table-centered}

The call to *split_csv*, regardless of how it is called or how it is constructed, takes five times longer than
when it is all taking place inside the PL/SQL engine.

I'm not sure I can say I've proven that the time is because of context switching, but I'm out of ideas as to what else
it could be.

Of interest is how little time the calls to *perlish_util_udt.get* method take. That cannot be doing a context switch.
Neither can the call to the object constructor. I am still not clear on how much code you can put into an
object method before it moves into the PL/SQL engine with a context switch.



