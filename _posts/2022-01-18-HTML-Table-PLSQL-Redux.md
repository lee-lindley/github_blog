---
layout: post
title: HTML Table Markup Redux
exerpt: "HTML and CSS suck once again, but we can overcome."
date: 2022-01-18 20:30:00 +0500
categories: [oracle, plsql, html]
tags: [oracle, plsql, html, table, css, xlst, dbms_xmlgen, xlstype]
---
# CSS Hell

Previously I wrote about a new package I published on github at [app_html_table_pkg](https://github.com/lee-lindley/app_html_table_pkg). My celebration was premature. The universe reared up and bit me in the hind quarters for daring
to mess around with HTML again. I knew better!

It turns out that Micrsoft Outlook and Google Gmail are partially braindead with respect to CSS. They support
some of it, but not all. In particular the fancy features I used for right justifying specified 
columns and alternating row colors were a bust.

I went back to work and retrofitted *app_html_table_pkg* with a *p_older_css_support* flag that you must use
if you are sending the HTML to Outlook clients (and perhaps others).

Gmail is a complete bust. It does not even respect the older method of specifying 
a style class in the HTML <tr><td> tags.
Bah! I'm not going to hard code everything in the HTML just to support Gmail.
Most businesses that would use this are on Outlook I suspect.

With these changes, the tables display correctly (or mostly correct) in both the web and PC versions
of Outlook 365 (PC version 2112 and also Version 2102). I do not know about older Outlook clients.

Google Chrome, Edge, Firefox, and Thunderbird email all support the original with *p_older_css_support* not
set to 'Y', but they also handle the less elegant way just fine. 

I'm not even going there on email clients on phones and tablets. Not interested. If you want to hack at it,
be my guest and send a pull request if you get it working. There may be others who want it too.


