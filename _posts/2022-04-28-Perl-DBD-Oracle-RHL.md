---
layout: post
title: Intalling Perl DBD::Oracle on RHL
exerpt: "Installing Perl DBD::Oracle has over the years been a heavy lift. It's still not a button push, but it has gotten better."
date: 2022-04-28 21:30:00 +0500
categories: [oracle, perl, linux]
tags: [oracle, perl, installation, linux, DBI, DBD, RHL]
---
# Introduction

Over the years I've had many opportunities to struggle with compiling *Perl* and *DBD::Oracle* on many different
flavors of Unix. It was never easy.

When package management (yum, etc..) became a thing and I eagerly started looking at it, reality slapped me back down.
Nobody was packaging *DBD::Oracle* and the reason seemed to be the Oracle client library license requirements.

When I went looking to do it all again on my home Linux instances, I was pleasantly surprised to find
*DBD::Oracle* was already included on my **Oracle Linux Server release 7.9** server. Oracle apparently doesn't
have an issue getting a distribution license for their own dogfood and were able to include *DBD::Oracle*. Nice!

I also have a **Red Hat Enterprise Linux release 8.5 (Ootpa)** server running under Microsoft *Hyper-V*.
In spite of the close business relationship Oracle has with Red Hat, RHL does not have *DBD::Oracle* installed,
nor was it found in a *dnf* search. Note that I've already installed
Oracle Database on this puppy, so I've added Oracle repositories. If there is a yum repository out there that
contains DBD::Oracle for Linux, I haven't found it.

So here we go again.

I know before I start that I need my shell environment to be set so that I can connect with sqlplus.
My database SID is **rhl1db** which is a container database.
This works for me given how I installed Oracle:

```sh
export ORACLE_HOSTNAME=$(hostname)
export ORAENV_ASK=NO
export ORACLE_SID=rhl1db
. oraenv -s
```
From the README for DBD::Oracle I know I need to set a variable **ORACLE_USERID** with the connection string.
I could include the @pdbname part there I think, but I also know I can use **TWO_TASK** to specify
the PDB. Remember my SID is for the Container DB. I'm going to connect to a PDB named **rhl1pdb**. 

```sh
export TWO_TASK=rhl1pdb
export ORACLE_USERID=lee/MYPASSWORD
sqlplus $ORACLE_USERID
# BINGO! I'm in
```

At this point I've taken care of prerequisites in my environment for doing the make. We could download
it and run the steps outlined in the README, but I thought I would swing for the fence.

# Trying CPAN

First, I tried to run a cpan shell.

```sh
perl -MCPAN -e shell
```

Yeah, we could be so lucky. It wasn't there, but ahoy! There is a yum package for it.

I'm running as root. I realize we are supposed to use *sudo*. You do you, and leave the old dinosaur alone.

```sh
dnf search perl-CPAN
# bingo
dnf install perl-CPAN
perl -MCPAN -e shell
```

```
Terminal does not support AddHistory.

cpan shell -- CPAN exploration and modules installation (v2.18)
Enter 'h' for help.

cpan[1]> install DBD::Oracle
Reading '/root/.local/share/.cpan/Metadata'
    Database was generated on Thu, 28 Apr 2022 07:29:03 GMT
Running install for module 'DBD::Oracle'
...
```

My first attempt the make failed for missing library *aio*. A little google search and we find we need to
install *libaio-devel*

```sh
dnf install libaio-devel
perl -MCPAN -e shell
cpan[1]> install DBD::Oracle
...
```

This time it compiles, but I run into trouble in the tests. It wants *Test::More* and *Devel::Peek*. OK, I'll try to get those.

```sh
perl -MCPAN -e shell
install Test::More
# that worked
install Devel::Peek
# no dice
```

*Devel::Peek* is tied up somehow. I didn't try to figure it out. I know that not all tests will pass anyway.
How would you know that? You wouldn't. It is the source of endless frustration for people new to this.
But if you google install issues for DBD::Oracle you will see the blase response to just ignore the
failed tests (when there are only a few non-critical failures!) and run make install anyway.

This time when I ran the install from CPAN shell it completed the majority of the tests and I honestly
don't care about the failures.

```
Scanning cache /root/.local/share/.cpan/build for sizes
...
Files=41, Tests=1867, 17 wallclock secs ( 0.27 usr  0.05 sys +  3.86 cusr  0.68 csys =  4.86 CPU)
Result: FAIL
Failed 5/41 test programs. 2/1867 subtests failed.
make: *** [Makefile:1105: test_dynamic] Error 2
  ZARQUON/DBD-Oracle-1.83.tar.gz
  /usr/bin/make test -- NOT OK
//hint// to see the cpan-testers results for installing this module, try:
  reports ZARQUON/DBD-Oracle-1.83.tar.gz
Failed during this command:
 ZARQUON/DBD-Oracle-1.83.tar.gz               : make_test NO

cpan[2]>
```

We are left with it not installed though. Bummer. Maybe there is a way to tell the CPAN shell to install even
after a failed test, but I didn't bother. At the top of the block above note that it tells me it
is doing this make in a cache directory "/root/.local/share/.cpan/build". I go down that path
and find "DBD-Oracle-1.83-0". In that directory is the Makefile that was created for this build. I do

```sh
make install
```

All done.

# Testing DBD::Oracle

My test script (sans my database password and notice I'm not root anymore):

```perl
#!/usr/bin/env perl
use DBI;
use DBD::Oracle;
use strict;
use warnings;

my $dbh = DBI->connect('dbi:Oracle:', 'lee@rhl1pdb', 'my secret password'
                        , {RaiseError => 1, AutoCommit => 0, RowCacheSize => -102400, ora_module_name => 'Perl' }
                      ) or die "Database connection not made: DBI::errstr";
$dbh->do("alter session set nls_date_format = 'mm/dd/yyyy'");
print $dbh->selectrow_array("SELECT sysdate from dual"), "\n";
$dbh->disconnect;
```

And the output:

```
lee@rhl1 [/home/lee/git/perl_oracle]
$ ./test_dbd_oracle.pl 
04/28/2022
```

That may have been the easiest install of *DBD::Oracle* I've ever done.

Granted, I had already installed the Oracle database on this server, configured it, and configured
connections (listener, TNS, etc...). I had all that working before I started. If you are on a new
machine you must first install the Oracle client and get that all working. That's also easier
than it used to be, but no picnic.

Hope this helped.
