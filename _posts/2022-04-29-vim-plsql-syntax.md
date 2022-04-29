---
layout: post
title: Syntax Highlighting for PL/SQL in vim
exerpt: "Becoming the 'maintainer' of the plsql syntax file for vim wasn't what I set out to do, but wimping out was not in the cards. Now I get the colors I like in vim."
date: 2022-04-29-21:30:00 +0500
categories: [oracle, plsql, sql, vim]
tags: [oracle, sql, plsql, vim, syntax-highlighting, syntax]
---
# Introduction

In the blog post [A Ruby/Rouge Lexer Class for Oracle PL/SQL](https://lee-lindley.github.io/plsql/sql/2022/03/20/Ruby-Rouge-Lexer-PLSQL.html)
I describe how I wound up creating a PL/SQL lexer in Ruby-Rouge so that I could make the PL/SQL code
blocks in my posts look good.

I tried to apply my color scheme to *vim* but quickly ran into trouble. The PL/SQL syntax file for *vim*
had not been updated since Oracle 9i and it had flaws such as not supporting q-quote operator. 
It also did not have folding. 

Previously having done work to get all of the 19c keywords straight, and familiar with what needed to
be done to parse PL/SQL and SQL, I applied what I knew to the existing *plsql.vim* file. 

> It does a good job with Oracle SQL too, not just PL/SQL. *sqloracle.vim* works differently (not better or worse), and folding support is simpler. You may be satisifed with *sql.vim/sqloracle.vim* for SQL, especially if you write SQL for multiple databases.

# From Supplicant to Maintainer

I tried to
find the maintainer of the file to offer my work. Sadly, that individual's email account is no longer
recognized by the provider, and he has not updated anything online in many years.

I contacted the *vim-dev* mailing list asking what to do. Their suggestion was to assume ownership
of the maintainer role. Balking at first, I quickly ran out of excuses I was willing to live with and agreed to do it.
Once committed, I felt obligated to go at it full-bore. 

I spent way more of my evenings trying to understand
the arcane rules of both *vim* regular expressions and the API rules for *syntax*, especially *region*'s, than
I care to quantify. There are limitations that made it exceedingly difficult to do folding the way I initially
envisioned it.

# Vim PL/SQL Syntax File

[vim_plsql_syntax](https://github.com/lee-lindley/vim_plsql_syntax) repository now lives on *github*. It contains both the 
syntax file *plsql.vim* and the colors file *lee.vim*. The colors file is completely gratuitous. You don't need
to use it to take advantage of the updates in *plsql.vim*.

The repository *README.md* is extensive. It includes screenshots and an installation section. 

At this time the *main*
branch contains the version submitted to the *vim* project for inclusion in version 9 (which should be released
soon for some definition of soon). You should select a more current branch, which although they are works in process, 
are passing my eyeball tests. Right now the branch 
[procedure_folding](https://github.com/lee-lindley/vim_plsql_syntax/tree/procedure_folding) 
is stable with some nice new folding features.  It is likely to be merged into *main* soon.

# Conclusion

I think I've done good work here, but if this is something that interests you, you will have to decide for yourself.
I'm pretty happy with it. I will of course attempt to accomodate suggestions. Submitting a Pull Request is the best
way to get what you want.

The secondary conclusion is to be careful when you report a problem. It isn't that uncommon to be told "so fix it."
I'm apparently a really slow learner on that lesson.
