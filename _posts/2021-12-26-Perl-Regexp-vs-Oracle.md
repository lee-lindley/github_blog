---
layout: post
title: Perl REgexp vs Oracle
date: 2021-12-26 20:30:00 +0500
categories: [oracle, plsql]
tags: [oracle, plsql]
---
# Writing Regular Expressions for Oracle As If You Were in a Perl Script

I am accustomed to writing regular expressions for Perl. It has gotten to the point
where I'm super annoyed when I write regular expressions for Oracle because of both how strings
are handled and lack of comment support in the RE itself. I prefer building the
regular expression in small pieces with assembler style comments on the right side to remind
myself what it does. I also favor using '\n' and '\t' to represent newline and tab rather than
having to hardcode these values in the regexp string. Having a newline in the middle
of an Oracle regular expression is particularly disconcerting. 

Likewise building the RE by concatentating quoted string pieces and having regular SQL comments
out on the right is less than satisfying. It gets ugly with all of the "||' whatever'", especially
with embedded newlines or even concatenating a constant that contains a newline.
These may seem trivial concerns, but I'm not changing.

The following Function transforms a regular expression in a manner similar to the way
Perl would treat the 'x' modifier stripping comments and whitespace, plus allowing
us to use '\n', '\r' and '\t' as character representations that are not stripped as whitespace. 

```sql
    FUNCTION transform_perl_regexp(p_re VARCHAR2)
    RETURN VARCHAR2
    IS
        /*
            strip comment blocks that start with at least one blank, then
            '--' or '#', then everything to end of line or string
        */
        c_strip_comments_regexp CONSTANT VARCHAR2(32767) := '[[:blank:]](--|#).*($|
)';
    BEGIN
    -- note that \n, \r and \t will be replaced if not preceded by a \
    -- \\n and \\t will not be replaced. Unfortunately, neither will \\\n or \\\t.
    -- If you need \\\n, use \\ \n since the space will be removed.
    -- We are not parsing into tokens, so this is as close as we can get cheaply
      RETURN 
        REGEXP_REPLACE(
          REGEXP_REPLACE(
            REGEXP_REPLACE(
              REGEXP_REPLACE(
                REGEXP_REPLACE(p_re, c_strip_comments_regexp, NULL, 1, 0, 'm') -- strip comments
                , '\s+', NULL, 1, 0                 -- strip spaces and newlines too like 'x' modifier
              ) 
              , '(^|[^\\])\\t', '\1'||CHR(9), 1, 0    -- replace \t with tab character value so it works like in perl
            ) 
            , '(^|[^\\])\\n', '\1'||CHR(10), 1, 0       -- replace \n with newline character value so it works like in perl
          )
          , '(^|[^\\])\\r', '\1'||CHR(13), 1, 0         -- replace \r with CR character value so it works like in perl
        ) 

      ;

    END;
```
The function *transform_perl_regexp* is distributed in 
a [utility libray I support](https://github.com/lee-lindley/plsql_utilities#transform_perl_regexp). 
You can use it without attribution.

Below is an example of writing a somewhat convoluted regular expression to parse a multi-line CLOB
that contains CSV "records" that may contain embedded newlines with quoting as described 
in [RFC4180](https://www.loc.gov/preservation/digital/formats/fdd/fdd000323.shtml). 

I have more about that in the
*plsql_utilities* library if it interests you, but the main point is to show the regular expression
presentation in the Perl style and how it is transformed.

```sql
DECLARE
    v_lines     CLOB := 
------------------------
'23, "this contains a comma (,)", 06/30/2021
47, "this contains a newline (
)", 01/01/2022

73, and we can have backwacked comma (\,), 12/25/2021
92, what about backwacked dquote >\"<?, 12/28/2021'
;
------------------------

    -- beware this is using Perl-like regexp so we have to apply a transform
    v_regexp    VARCHAR2(32767) := '
(                               # capture in \1
  (                             # going to group 0 or more of these things
    [^"\n\\]+                   # any number of chars that are not dquote, backwack or newline
    |
    (                           # just grouping for repeat
        \\ \n                   # or a backwacked \n but put space between them so gets transformed correctly
    )+                          # one or more protected newlines (as if they were in dquoted string)
    |
    (                           # just grouping for repeat
        \\"                     # or a backwacked "
    )+                          # one or more protected "
    |
    "                           # double quoted string start
        (                       # just grouping. Order of the next set of things matters. Longest first
            ""                  # literal "" which is a quoted dquoute within dquote string
            |
            \\"                 # a backwacked dquote 
            |
            [^"]                # any single character not the above two multi-char constructs, or a dquote
                                #     Important! This can be embedded newlines too!
        )*                      # zero or more of those chars or constructs 
    "                           # closing dquote
    |                           
    \\                          # or a backwack, but do this last as it is the smallest and we do not want
                                #   to consume the backwack before a newline or a dquote
  )*                            # zero or more strings on a single "line" that could include newline in dquotes
                                # or even a backwacked newline
)                               # end capture \1
(                               # just grouping 
    $|\n                        # require match newline or string end 
)                               # close grouping
';
    v_line      VARCHAR2(32767);
    v_i         BINARY_INTEGER := 1;
    v_cnt       BINARY_INTEGER;
BEGIN
    v_regexp := transform_perl_regexp(v_regexp);
    DBMS_OUTPUT.put_line('preprocessed regexp: >'||v_regexp||'<');
    --
    v_cnt := REGEXP_COUNT(v_lines, v_regexp) - 1;   -- get number of times regexp will match less match on $
    FOR v_i IN 1 ..v_cnt                            -- for each time we know it will match including NULL match values
    LOOP
        v_line := REGEXP_SUBSTR(v_lines, v_regexp, 1, v_i, NULL, 1);
        dbms_output.put_line('line '||TO_CHAR(v_i)||' is >'||v_line||'<');
    END LOOP;
END;
/
```
Output:

    preprocessed regexp: >(([^"
    \\]+|(\\\n)+|(\\")+|"(""|\\"|[^"])*"|\\)*)($|
    )<
    line 1 is >23, "this contains a comma (,)", 06/30/2021<
    line 2 is >47, "this contains a newline (
    )", 01/01/2022<
    line 3 is ><
    line 4 is >73, and we can have backwacked comma (\,), 12/25/2021<
    line 5 is >92, what about backwacked dquote >\"<?, 12/28/2021<

*transform_perl_regexp* is NOT a sophisticated parser. 
If you want to have the strings ' --' (space dash dash)
or ' #' (space hash) appear in your regular expression, you will have to get creative or not use it.
(HINT: you can put one of the characters in character class brackets).
Likewise when looking for the '\n' and friends, we are not actually
parsing anything, so it gets messy. I've protected '\\\n', but not dealt with '\\\\\n'. These are
easier though because you can separate '\\' from '\n' with a space that is going to be stripped.
It works the way I expect most of the time.

# Conclusion

Regular expressions, even those you wrote last week, take signficant effort for your brain to parse.
It is worthwhile to have them well documented in the code. A way to do that in Perl has been established
and is used by many serious practitioners. I much prefer it and can now use it in Oracle SQL and PL/SQL
programs.
