---
layout: post
title: Oracle REGEXP_INSTR and Beginning of Line Anchor
date: 2022-04-10 11:45:00 +0500
categories: [oracle, sql, plsql]
tags: [oracle, sql, plsql, regexp]
---
# The Problem

*REGEXP_INSTR*, *REGEXP_COUNT*, *REGEXP_REPLACE*, and *REGEXP_SUBSTR* all have a *position* parameter
defined as

> *position* is a positive integer indicating the character of *source_char* where Oracle should begin the search. The default is 1, meaning that Oracle begins the search at the first character of *source_char*.

This is handy when you want to walk through a string applying the regular expression starting at different points, such as after the last match. 

Problem: **When *position* is not 1, the beginning of line anchor '^' does not match the beginning of the substring**.

> There is also an *occurence* parameter that can be used to similar effect. I presume that internally the regular expression
> engine keeps track of the last match rather than parsing the entire string again, but Oracle does not say. Without details
> about it I'm leary of trusting it for high performance, and in fact have some tangential evidence that using *occurence*
> for this purpose is not as performant as using *position*. I could be wrong but am not going to try to prove it today.

# REGEXP_INSTR

This matches the beginning of the string which is two space characters. Since we specify *return_opt*=1,
we are returned the character position AFTER the matched string. This meets expectations.

```plsql
SELECT REGEXP_INSTR('  <<= string starts with space', '^\s*'
                    , /*position*/ 1, /*occurence */ 1, /*return_opt*/ 1
                   ) AS end_position
FROM dual;
```

    3

Same thing but without any leading spaces in *source_char*. 
Since we have the zero or more modifer for spaces, we still expect a match
on the zero width start of line anchor. 

```plsql
SELECT REGEXP_INSTR('String starts with S', '^\s*'
                    , /*position*/ 1, /*occurence */ 1, /*return_opt*/ 1
                   ) AS end_position
FROM dual;
```

    1

We do get a match (non-zero return value). As expected the character position AFTER the matched string (which is 0 length) 
is still 1. 

What happens when we advance
the position so that it is no longer at the start of *source_char*? Here I set *position* to 2 so that
we start looking for a match at character position 2.

```plsql
SELECT REGEXP_INSTR('String starts with S'
                    , '^\s*', /*position*/ 2, /*occurence */ 1, /*return_opt*/ 1
                   ) AS end_position
FROM dual;
```

    0

That was a huge surprise to me. 
The '^' anchor no longer matches the beginning of what we think of as the string we are matching 
(sub-string starting at position 2). I don't know that I can say it is a bug because Oracle does not 
explain how *position* is applied, and Oracle is careful in the wording that '^' matches the start of the
**entire source string** (or after a newline when the 'm' *match_param* is specified). Nowhere does it
say that '^' matches at starting *position*.

Needless to say, it does NOT work the way I expect which is as demonstrated by using *SUBSTR* to achieve
the goal of starting at position 2 rather than using the *position* argument to *REGEXP_INSTR*.

```plsql
SELECT REGEXP_INSTR(SUBSTR('String starts with S',2)
                    , '^\s*', /*position*/ 1, /*occurence */ 1, /*return_opt*/ 1
                   ) AS end_position
FROM dual;
```

    1

We can work with this if we must, but we are making a copy of the rest of the string to do it rather than 
walking through it in place. This does not make me happy if I'm writing a tight subroutine that is called millions
of times.

If we can write the *pattern* such that it works correctly without '^', we can be efficient using *position*. 

# REGEXP_COUNT

The behavior is the same for *REGEXP_COUNT* as *REGEXP_SUBSTR. Starting with the *position*=1 we get what we expect.

```plsql
SELECT REGEXP_COUNT('XXX
Xab
Xcd'
                    ,'^X'
                    , /*position*/ 1, /*match_param*/ 'm'
                   ) AS cnt
FROM dual;
```

    3

Moving to *position*=2 my expectation would be to match the 'X' in the second character position 
along with the ones after newlines to give an answer of 3.

```plsql
SELECT REGEXP_COUNT('XXX
Xab
Xcd'
                    ,'^X'
                    , /*position*/ 2, /*match_param*/ 'm'
                   ) AS cnt
FROM dual;
```

    2

Of course the behavior is same as for REGEXP_INSTR and we do not match until after a newline. 

Interestingly if we advance to *position*=4 which means it is sitting on a newline character as
the starting position for the string, we still get an answer of 2.

```plsql
SELECT REGEXP_COUNT('XXX
Xab
Xcd'
                    ,'^X'
                    , /*position*/ 4, /*match_param*/ 'm'
                   ) AS cnt
FROM dual;
```

    2

When we advance to character position 5 which is after the newline, the '^' no longer matches at our
start of string position and the first X on line 2 is not matched. We only match on the 3rd line.

```plsql
SELECT REGEXP_COUNT('XXX
Xab
Xcd'
                    ,'^X'
                    , /*position*/ 5, /*match_param*/ 'm'
                   ) AS cnt
FROM dual;
```

    1

# REGEXP_REPLACE

Starting position of 1 works as expected as we anchor at the beginning of *source_char*.
The starting 'X' on all three lines is replaced by 'Y'.

```plsql
SELECT REGEXP_REPLACE('XXX
Xab
Xcd'
                    ,'^X', 'Y'
                    , /*position*/ 1, /*occurence*/ 0, /*match_param*/ 'm'
                   ) AS x_to_y
FROM dual
```

    YXX
    Yab
    Ycd

Starting position of 2 my expectation is that the 'X' in character position 2 of line 1 be changed to 'Y'. That
does not happen.

```plsql
SELECT REGEXP_REPLACE('XXX
Xab
Xcd'
                    ,'^X', 'Y'
                    , /*position*/ 2, /*occurence*/ 0, /*match_param*/ 'm'
                   ) AS x_to_y
FROM dual;
```

    XXX
    Yab
    Ycd

# REGEXP_SUBSTR

Starting position of 1 works as expected.

```plsql
SELECT REGEXP_SUBSTR('XXX
Xab
Xcd'
                    ,'^X.*'
                    , /*position*/ 1, /*occurence*/ 1, /*match_param*/ 'm'
                   ) AS x_to_y
FROM dual;
```

    XXX

Setting *occurence* to 2 also works as expected.

```plsql
SELECT REGEXP_SUBSTR('XXX
Xab
Xcd'
                    ,'^X.*'
                    , /*position*/ 1, /*occurence*/ 2, /*match_param*/ 'm'
                   ) AS x_to_y
FROM dual;
```

    Xab

Setting *position* to 2 we do not get a match until after a newline which is same behavior as the other
regular expression functions.

```plsql
SELECT REGEXP_SUBSTR('XXX
Xab
Xcd'
                    ,'^X.*'
                    , /*position*/ 2, /*occurence*/ 1, /*match_param*/ 'm'
                   ) AS x_to_y
FROM dual;
```

    Xab

# Conclusion

Maybe Oracle can claim this is working as designed. As far as I'm concerned it is a bug, but not one I
expect them to fix. It would break too much code already dependent on this behavior.  Oracle should amend 
the documentation to explain this behavior as it is different from the way we work with regular
expressions to walk through a string in other languages like Perl.
