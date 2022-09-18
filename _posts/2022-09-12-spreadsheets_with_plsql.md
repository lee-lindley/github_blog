---
layout: post
title: Manipulating XLSX Spreadsheets in PL/SQL
exerpt: "Business users often embed their work processes in spreadsheets. One use case I see often is data is created and maintained by the business users in a spreadsheet but they want to supplement it from the database. A full blown application for their process is not high on the priority list. There are several ways to accomodate them. We discuss a method of inputting their spreadsheet to Oracle and outputting it again with the supplemental columns from queries."
date: 2022-09-12 19:30:00 +0500
categories: [oracle, perl]
tags: [oracle, plsql]
---
# Introduction

We have a need to get large volumes of data and/or *BLOB/CLOB* content out of the database
and onto disk on a remote client machine. 

We can do it from *sqlplus*. Even *BLOB* data can be output as I demonstrated 
in [Extracting BLOB from Oracle with Sqlplus](https://lee-lindley.github.io/oracle/sql/plsql/2021/12/18/sqlplus-blob.html).
I happen to think it is a clunky way to do things when there are nice scripting languages around, 
but the one constant you can count on having available on a client is *sqlplus* and some form of *shell* script capability,
even if it is *Powershell* on Windows.

If you have access to *Pro-C* compilation, there is a nice tool Tom Kyte published long ago named *flat*
you can find [here](https://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:459020243348).
It is a super-fast and efficient solution.
It doesn't do *CLOB* or *BLOB*, but with a little elbow grease I'm sure one could manage it. *Pro-C*
has everything you need to handle a *LOB Locator* and retrieve the content.

I know it is doable in Java, and I suspect most other languages that have database connection libraries.

*sqlCL* can do it as described in [Using SQLcl to write out BLOBs to files in 20 lines of js](https://www.thatjeffsmith.com/archive/2020/07/using-sqlcl-to-write-out-blobs-to-files-in-20-lines-of-js/); however, as I've mentioned
in other posts, *sqlCL* is strangely lacking on every ETL server I've had the pleasure of visiting. For that
matter the security folk aren't crazy about having java client programs on servers. I think they overreacted
to a security issue from many years ago and have never gotten over it, but it is what it is.

This article is about extracting *CSV* flat files, *CLOB*'s and *BLOB*'s from Oracle using *Perl*.

# Do You Have Perl DBD::Oracle?

You may not. It does not ship with any OS I know of other than Oracle's own version of RHL. Neither
is it available in any *YUM* repository I could find, so your Unix Admin is not going to be able to
install it easily. They may balk at doing it at all depending on how strict the rules are under which
they work.

That said, it has gotten easier as I demonstrated 
in [Installing Perl DBD::Oracle on RHL](https://lee-lindley.github.io/oracle/perl/linux/2022/04/28/Perl-DBD-Oracle-RHL.html).
There is a good chance your Unix Admin isn't allowed to alter the vendor *Perl*. If that is the case you may
need to install your own *Perl* and add *DBD::Oracle* to that. Perhaps the Unix admin can do that for you.

> I don't mean to make light of this, but if you are going to have a useful scripting environment, whether it be
> *Perl*, *Python*, or something else, at some point the security mavens and corporate rule makers need to give
> you one and it needs to have the database connect libraries linked in. "Of course that's true" you are thinking.
> Surprise! You may find the security mavens and IT infrastructure folk are not sympathetic to your need
> for a decent scripting environment with a built-in Oracle connection library on the ETL server. At one company I
> encountered a mindset that shell and *sqlplus*, along with a vendor ETL tool, were all you need. It was a
> large, sophisticated organization too. They were responding to pressures from senior management that resulted in
> what I would call unintended consequences.

# Running Data Through Perl

*Perl* is fantastic as a scripting language. But like all scripting languages it has some overhead costs.
It isn't the bytecode, which is pretty efficient. The overhead is in the data structures which are fat pigs.
If you look at the underlying structure of a *Perl* *scalar* variable, you will find multiple pointer and length
members. For example it has a place to store an integer, a floating point number, a string, an array, and something
called "magic" among other things.  There is a lot of wasted space for any given scalar object.

Under normal
circumstances that hardly matters. If you are running a *fetchrow_array* operation over a large number of rows
on a big *SQL* query, you are creating and tearing down a scalar for every column on every row. It can really
add up.

Yeah, don't do that. You would be better off spooling it out from *sqlplus*. On the other hand, if you are reading
and writing large chunks of data through Perl, the overhead is negligible. The underlying data movement is
all handled in tight C code and is efficient.

# Using DBD::Oracle to Read CLOB/BLOB Data

In the article [The Ubiquitous CSV File](https://lee-lindley.github.io/oracle/sql/plsql/2021/09/10/Ubiquitous-CSV_file.html)
I described multiple ways for generating CSV data and getting it out of the database.
One of those ways was for the use case that you were doing it for a business user who
just turns around and loads it to a spreadsheet. You can up your game by producing an *XLSX* 
file directly from the database using [ExcelGen](https://github.com/mbleron/ExcelGen). The output
of that is a *BLOB*.

I also have a CSV file generator in package named *app_csv_pkg* available in [plsql_utilities](https://github.com/lee-lindley/plsql_utilities#app_csv_pkg). I described how it it works in
[Polymorphic Table Function (PTF) for CSV (take 3)](https://lee-lindley.github.io/oracle/sql/plsql/2021/12/31/Polymorphic-Table-Functions-3.html) The output of that is a *CLOB*. Well, you can take the output as single column rows in a fetch, but
you still have to stream the data out somehow.

Given that you have a procedure that can produce a *BLOB* or *CLOB* with everything you need to put in the file
on the client, how can we get it from the database? Let's walk through an example.

This first part is boilerplate you will have in all of your scripts unless you have 
refactored it into a connection object, perhaps one that handles the password management.

We need the Oracle Type definitions, thus the extra argument to use DBD::Oracle.

```perl
#!/usr/bin/env perl
use DBI;
use DBD::Oracle qw(:ora_types);
use strict;
use warnings;

my $dbh = DBI->connect('dbi:Oracle:', 'lee@rhl1pdb', 'my secret password'
                        , {RaiseError => 1, AutoCommit => 0, RowCacheSize => -102400, ora_module_name => 'Perl' }
                      ) or die "Database connection not made: DBI::errstr";
$dbh->do("alter session set nls_date_format = 'mm/dd/yyyy'");
```
Next we prepare our *PL/SQL* anonymous block. This could have been a call to ExcelGen or your own procedure
that produces a *BLOB* or *CLOB*. In this case I'm demonstrating with *app_csv_pkg* where we pass it a query
to run as a string. It executes the query, fetches the data, converts it into *CSV* rows, and concatenates
them into a *CLOB*. It then returns the *CLOB* as an OUT parameter.

The local hash with *ora_auto_lob* setting to false is so that we get back a *LOB* locator rather than
letting the driver convert the *CLOB* into one giant string. We might not want to put that much data into 
memory, plus we have to know in advance how big it could be and set some variables to allow for it.
Search the DBD::Oracle perldoc for CLOB and read all about it. Unfortunately, it is a pretty big topic.

```perl
my $sth = $dbh->prepare(q!
    BEGIN
        app_csv_pkg.get_clob(
            p_sql           => :sql
            ,p_clob         => :clob
            ,p_rec_count    => :rec_count
        ); 
    END;
!
    , { ora_auto_lob => 0 } # this says pass LOB locator, not entire lob
);
```
Next we create a variable with our query and 2 variables we will bind as OUT parameters.
The bind options are important. The *SQLT_CHR* type will convert
a perl string to the proper type for the *CLOB* input parameter up to about 2MB in size.
If your query is bigger than that, you are doing something wrong. If you really need to
input a CLOB, read the DBD::Oracle perldoc.

The output parameter type *ORA_CLOB* in this case means we will get back a *LOB* locator.
If in the *prepare* statement above we had allowed *ora_auto_lob* to be the default TRUE,
we would get back a string (as long as it wasn't too big).

*rec_count* is just a number output parameter. After it runs we can find out how
many records the query returned and were placed in our *CLOB*. Useful for logging.

```perl
my $sql = 'SELECT * FROM v$reserved_words';
my ($rec_count, $clob);

$sth->bind_param( ':sql', $sql, {ora_type => SQLT_CHR } ); # This type converts to CLOB on input up to 2MB which is plenty
$sth->bind_param_inout( ':clob', \$clob, 0, {ora_type => ORA_CLOB } ); # will be a CLOB locator
$sth->bind_param_inout( ':rec_count', \$rec_count, 0);
$sth->execute;
```
After the *execute*, *$rec_count* has the number of rows which we can report and *$clob* has a *LOB*
locator (which is only good to use during this transaction; a *commit* or *rollback* destroys the underlying *LOB*.)

Now we write out the data. We are writing to STDOUT here, but you could have opened
a named file and be writing to that.

Rather than bring back the entire CLOB into one huge chunk of memory, we will read and 
write it in pieces. Each of these is a round trip to the database so do not make them
too small, but neither do we want to use so much memory on both sides of the connection
that it is an issue. Just like where Oracle has a rule of thumb that 100 rows is a good
bulk fetch size, the right answer is probably below 1MB and more than 10K. I don't really know
where the sweet spot is without some experimentation. Here I picked 256K bytes.

```perl
print STDERR $rec_count, " rows returned in clob\n";

# 256K chunks are not that much memory, but still big enough perl scalar creation/destruction not an issue
my $chunk_size=256*1024; 
my $offset = 1; # starts at 1, not 0
while(1) {
    my $data = $dbh->ora_lob_read( $clob, $offset, $chunk_size );
    last unless length $data;
    print $data;
    $offset += $chunk_size;
}

$dbh->rollback; # to end the transaction concerning temp lob locator and freeing it so perl destructor doesn't complain
```
I'm not going to print out the results, but this works just fine. My sample query output is not bigger than
the chunk size, so it didn't loop at first. I had to make it smaller and put in some debug
prints to prove it, but it works.

# Conclusion

I've waved my hands around a lot in prior articles saying
you can get *CLOB* and *BLOB* data out on the client. I showed it
with *sqlplus* and *sqlCL*. I have used Tom Kyte's *flat* utility for years (though the security mavens
don't want me near a C compiler and Devops won't give me a deploy method for it). It was time to put up
some proof that we can do it all with a scripting language efficiently too. Here ya' go.

Hope this helps.
