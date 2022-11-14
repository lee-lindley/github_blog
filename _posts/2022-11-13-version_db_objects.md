---
layout: post
title: Versioning Oracle Code from Open Source
exerpt: "When Open Source code is already deployed and in use by many production jobs, but you want to implement the latest release, consider adding version identifiers to the object names."
date: 2022-11-13 20:00:00 +0500
categories: [oracle, perl]
tags: [oracle, perl]
---
# Introduction

My team have a moderately large installed base of Oracle packages that 
use [Marc Bleron's ExcelGen](https://github.com/mbleron/ExcelGen) package.
Version 3 of the package was released recently and we have use cases that can benefit from the new features.
Regression testing the installed base of code as is required under corporate
SDLC policy is not an attractive proposition.

The strategy we chose is to name the newly changed Oracle objects differently than the existing ones so
that we can have multiple versions installed in production. For example *ExcelGen* package will be named *ExcelGen_v3*.
This isn't ideal, but all future uses,
including any time a job is changed and must be regression tested anyway, will use the new version. Eventually,
the existing version will be retired.

This article describes a technique and a Perl script to facilitate versioning of the changed objects. Full disclosure -
initially I did much of this manually. For this article I cleaned up the Perl script to address oddities I found
during the exercise and to keep my Perl skills from fading away.

# Scenario

*ExcelGen* is open-source software hosted on *GitHub*. Instructions for cloning the repository point out you
should use *--recurse-submodules* or else you must copy the two submodules down separately. The use of submodules
is good technique, but complicates our task.

The repository follows a model with a separate file (or files) for every Oracle object.
The files are also named the same as the Oracle objects except for the file suffix. 
This is best practice for source code control of Oracle artifacts in my opinion, and fortunate for the pattern I followed here.

We need to identify all files that changed since our production deployment, the objects defined
in those files, and any dependent objects that would be impacted by the compile.

# Identifying Changed Files in Git

We could simply run diff commands to identify files that changed, but we have a nice tool in *git* that
keeps track of all this. In order to use it
we must know the commit point of the files we already deployed. In my case I know that tag "v2.5.1" represents
the files I deployed from the main *ExcelGen* repository. The *MSUtilities* repository has no changes in the
last two years, and *ExcelCommons* change I last deployed was commit *383ff141bffd3bd31ed3054106346a948575469a*.
In git a tag, a branch and a commit are all simply pointers that are interchangeable for identifying 
the state of code at a particular point in the history.

To identify changed files from the top level directory we run the command:
```sh
#git diff --name-only prior_version_tag_or_commit
git diff --name-only v2.5.1
```
Discounting the test_cases, samples and resources files, it turns out the only ones impacted in the main repository are

- plsql/ExcelGen.pkb
- plsql/ExcelGen.pks

Changing directory to *ExcelCommons* we run the command for this submodule:
```sh
git diff --name-only 383ff141bffd3bd31ed3054106346a948575469a
```

This yields:

- plsql/ExcelTableCell.tps
- plsql/ExcelTypes.pkb
- plsql/ExcelTypes.pks
- plsql/xutl_xlsb.pkb
- plsql/xutl_xlsb.pks

From this we have a list of changed Oracle objects:

- ExcelGen
- ExcelTableCell
- ExcelTypes
- xutl_xlsb

We could simply look at each one with Toad or SqlDeveloper for dependencies, but I felt like playing, so
we'll let Perl find the dependencies for us while we prepare to change all occurrences of these
four identifiers by adding "\_v3" to the end. Note it is not as simple as a global search and replace
with your text editor because these could be sub-parts of larger names that don't change. For example
the string *ExcelTableCell* appears in *ExcelTableCellList*.

# Perl Script Overkill

At a whopping 125 lines this monster outgrew my immediate need, but it does some things you may find
interesting. The start of the script gathers the arguments. The first argument is the name of the install
script from which it will read the names of the files we would install if this were a fresh install.
After that we provide a version string to add to the object names. In this case I provided "\_v3".
Finally follow the names of the objects that we know need to be versioned as a seed. The script may add more.

```perl
#!/usr/bin/env perl
use strict;
use warnings;
my $usage = "USAGE: $0 install_script version_string identifier1 [identifier2..]\nwhere 'identifier' is oracle object that you know has changed in the release.\n";
#
# In each git module and submodule, 
# "git diff --name-only prior_version_tag_or_commit"
# That gives us list of files from which we should be able to name objects 
# we need to version and install that are different than
# what we already deployed. 
# This script assumes the files are named after the objects they build.
# object_name_version.pl will also check recursively for objects in the
# install script that use changed objects as those need to be versioned too.
#
die "$0: must provide at least 3 args.\n$usage\n" unless @ARGV >= 3;
my $install_script = shift;
my $version_string = shift;
# now only idendifiters are left in @ARGV

my (@on_re_list, %on_list);
for my $on (@ARGV) { # we want unique list of lower case object names
    @on_list { lc $on } = 1;
}
```

Next we create subroutines that we can call multiple times as we identify dependencies. The first creates
regular expressions for each versioned object that we can use both for searching and also
for modifying the code. The parsing of an identifier is something I've done before for
the *vim* plsql syntax file and also for the *Ruby Rouge* plsql lexer.

```perl
sub build_on_re_list {
  # we know the objects that changed. 
  # Find those and any files that contain them.
  # Let us construct an array of regular expressions to match on. Then
  # we can use those same regexp's to substiture the new names.
  print STDERR "calling build_on_re_list\n";
  @on_re_list = ();
  for my $identifier (keys %on_list) {
        print STDERR "adding re for $identifier\n";
        push @on_re_list, qr/
    (                                       # Capture before identifier string if any
            ^                               # could be matching at start of line
        |
            [^a-z0-9_#\$]                   # any character that is not an identifier
    )
    (                                       # capture our object word
        $identifier
    )
    (                                       # capture character after our object word
            [^a-z0-9_#\$]                   # any character that is not an identifier
        |
            $                               # could be end of line
    )
/imx;
    # ignore case, ignore whitespace in RE, match ^$ at newline
  }

  # the replacement string will be "${1}${2}${version_string}$3"
}
```

This subroutine can be called two ways. The first to identify impacted files
and the second option creates new files named with the version string that
we can deploy.

```perl
# sub find_files uses these so set before defining
my ($install_script_out);
($install_script_out = $install_script) =~ s/\.sql/${version_string}.sql/;
open(OFL, ">", "$install_script_out") or die "failed open $install_script_out for write: $!";

sub find_files {
    # look through files for references to the objects that changed
    my ($look_only) = @_ > 0 ? $_[0] : 0;
    print STDERR "calling find_files with look_only=$look_only\n";
    my @file_names;
    open(FL, "<", $install_script) or die "failed to open install script $!";
    while (<FL>) {
        next unless /^@/;
        chomp;
        my ($fn) = $_ =~ /@@?(.+)/;
        print STDERR "checking $fn\n";
        my $data;
        {
            open my $f, '<', $fn or die;
            local $/ = undef; # slurp mode
            $data = <$f>;
            close $f;
        }
        my $match_file;
        for my $re (@on_re_list) {
            if ($data =~ $re) {
                $match_file = $fn;
                last;
            }
        }

        next unless $match_file; # we did not find any interesting identifiers

        # reduce file name to oracle identifier. no slash or dot, followed
        # by last dot and non-dots to end of string.
        my ($id) = $match_file =~ m!([^./]+)\.[^.]+$!i; 
        $on_list{ lc $id } = 1;
        
        next if $look_only;     
        # if we are done recursively looking for all objects we need to change
        # continue on to making a new version of the files needed.

        # need to change all occurences of versioned objects inside this file
        for my $re (@on_re_list) {
            $data =~ s/$re/${1}${2}${version_string}$3/g;
        }
        # write the versioned file out with new name
        my $ofname;
        # new output file name has version string before dot
        ($ofname = $match_file) =~ s/\.(.*)/${version_string}.$1/; 
        print STDERR "found changed object names in ${fn}. Creating $ofname\n";
        open OF, '>', $ofname;
        print OF $data;
        close OF;
        # add it to the new install script
        print OFL '@@', $ofname, "\n";
    }
} # find_files
```
We call the subroutines repeatedly to add to the object lists as dependencies are found. When
we find no new ones we exit the loop and call the version that creates our new files.
We also write out a new install script with only the objects that we change.

```perl
# keep looking recursively for more objects we may need to version because
# of dependencies until we don't find any new ones
do {
    build_on_re_list();
    find_files(1); # could add some to on_list
} while (@on_re_list < keys %on_list) 
;

# Now call it so that it creates the versioned files
find_files(); 

print STDERR "writing $install_script_out\n";
close OFL;
```
# Execute

```sh
[C:/Users/Leeli/Documents/ExcelGen] (master)
$ perl -w object_name_version.pl install.sql _v3 excelgen exceltablecell exceltypes xutl_xlsb
```

Output:
```log
calling build_on_re_list
adding re for exceltypes
adding re for exceltablecell
adding re for excelgen
adding re for xutl_xlsb
calling find_files with look_only=1
checking MSUtilities/CDFManager/xutl_cdf.pks
checking MSUtilities/CDFManager/xutl_cdf.pkb
checking MSUtilities/OfficeCrypto/xutl_offcrypto.pks
checking MSUtilities/OfficeCrypto/xutl_offcrypto.pkb
checking ExcelCommons/plsql/ExcelTableCell.tps
checking ExcelCommons/plsql/ExcelTableCellList.tps
checking ExcelCommons/plsql/ExcelTypes.pks
checking ExcelCommons/plsql/ExcelTypes.pkb
checking ExcelCommons/plsql/xutl_xlsb.pks
checking ExcelCommons/plsql/xutl_xlsb.pkb
checking plsql/ExcelGen.pks
checking plsql/ExcelGen.pkb
calling build_on_re_list
adding re for exceltablecelllist
adding re for excelgen
adding re for xutl_xlsb
adding re for exceltypes
adding re for exceltablecell
calling find_files with look_only=1
checking MSUtilities/CDFManager/xutl_cdf.pks
checking MSUtilities/CDFManager/xutl_cdf.pkb
checking MSUtilities/OfficeCrypto/xutl_offcrypto.pks
checking MSUtilities/OfficeCrypto/xutl_offcrypto.pkb
checking ExcelCommons/plsql/ExcelTableCell.tps
checking ExcelCommons/plsql/ExcelTableCellList.tps
checking ExcelCommons/plsql/ExcelTypes.pks
checking ExcelCommons/plsql/ExcelTypes.pkb
checking ExcelCommons/plsql/xutl_xlsb.pks
checking ExcelCommons/plsql/xutl_xlsb.pkb
checking plsql/ExcelGen.pks
checking plsql/ExcelGen.pkb
calling find_files with look_only=0
checking MSUtilities/CDFManager/xutl_cdf.pks
checking MSUtilities/CDFManager/xutl_cdf.pkb
checking MSUtilities/OfficeCrypto/xutl_offcrypto.pks
checking MSUtilities/OfficeCrypto/xutl_offcrypto.pkb
checking ExcelCommons/plsql/ExcelTableCell.tps
found changed object names in ExcelCommons/plsql/ExcelTableCell.tps. Creating ExcelCommons/plsql/ExcelTableCell_v3.tps
checking ExcelCommons/plsql/ExcelTableCellList.tps
found changed object names in ExcelCommons/plsql/ExcelTableCellList.tps. Creating ExcelCommons/plsql/ExcelTableCellList_v3.tps
checking ExcelCommons/plsql/ExcelTypes.pks
found changed object names in ExcelCommons/plsql/ExcelTypes.pks. Creating ExcelCommons/plsql/ExcelTypes_v3.pks
checking ExcelCommons/plsql/ExcelTypes.pkb
found changed object names in ExcelCommons/plsql/ExcelTypes.pkb. Creating ExcelCommons/plsql/ExcelTypes_v3.pkb
checking ExcelCommons/plsql/xutl_xlsb.pks
found changed object names in ExcelCommons/plsql/xutl_xlsb.pks. Creating ExcelCommons/plsql/xutl_xlsb_v3.pks
checking ExcelCommons/plsql/xutl_xlsb.pkb
found changed object names in ExcelCommons/plsql/xutl_xlsb.pkb. Creating ExcelCommons/plsql/xutl_xlsb_v3.pkb
checking plsql/ExcelGen.pks
found changed object names in plsql/ExcelGen.pks. Creating plsql/ExcelGen_v3.pks
checking plsql/ExcelGen.pkb
found changed object names in plsql/ExcelGen.pkb. Creating plsql/ExcelGen_v3.pkb
writing install_v3.sql
```
The resulting install script "install\_v3.sql" is
```sql
@@ExcelCommons/plsql/ExcelTableCell_v3.tps
@@ExcelCommons/plsql/ExcelTableCellList_v3.tps
@@ExcelCommons/plsql/ExcelTypes_v3.pks
@@ExcelCommons/plsql/ExcelTypes_v3.pkb
@@ExcelCommons/plsql/xutl_xlsb_v3.pks
@@ExcelCommons/plsql/xutl_xlsb_v3.pkb
@@plsql/ExcelGen_v3.pks
@@plsql/ExcelGen_v3.pkb
```
This install assumes other base objects were already installed and are neither changed nor impacted.

# Conclusion

This is overkill for the task. The regular expression for replacing object names is super useful, but
much of the rest of this task could have been (and was) completed manually this time. I can see a day when
it would be a much bigger effort though, and the exercise was useful to maintain my skills. Maybe you will
see something useful to you.

