---
layout: post
title: HTML Table Markup in Oracle
date: 2022-01-16 20:30:00 +0500
categories: [oracle, plsql, html]
tags: [oracle, plsql, html, table, css, xlst, dbms_xmlgen, xlstype]
---
# HTML Table Markup for Oracle Query Resultset

## Use Case - Generate HTML Table Markup Inside Oracle Database

Whenever a question is posted on this subject, the immediate knee-jerk reaction is to tell the hapless
suppicant to use sql\*plus HTML markup option. All well and good, but we do have use cases where
going out to a client program from the database is not practical. One of the most common is
to generate an HTML Table that is included in an email sent directly from the database.

There is a well established pattern to do this with *DBMS_XMLGEN* and *XLSTYPE* stylesheet processing.
A web search will return plenty of examples. 

There are a few issues with the technique, the most
vexing of which is that all of the table data is left justified in the cells. If you want numbers
to be right justified, the standard mechanism we reach for with TO_CHAR or LPAD(TO_CHAR to left pad the
result with spaces does not work because HTML rendering ignores whitespace. 
We need to add a modifier to the HTML Table Data (\<td\>) tag to right-align the cells, but only for particular columns.

## CSS Style Sheet Solution

The implemented solution can be found on my github [app_html_table_pkg](https://github.com/lee-lindley/app_html_table_pkg)
repository page. If you are an HTML guru, you should skip straight to the package and skip my explanation.

I have of course dabbled in HTML but most of that was in pre-historic times before CSS stylesheets gained
traction. I found the CSS subject overwhelming, and it was compounded by being a moving target. Much of the information
you find on the web is no longer best practice.

We do not have control of the overall document in this case, or at least not always. We may not be able to
put CSS style information in the HTML Header. Luckily for us, somewhere along the way the committee to make
web page development obtuse decided we could use locally scoped styles, meaning it is legal and not
just a hack to put styles directly in the HTML Body. Coolio!

I complain about how the rules seem obscure and a moving target, but once I finally got a handle on enough
of the pieces, it turns out to be a relatively understandable and isolated bit of HTML code that will allow
us to control the formatting of our table without impacting anything else in the document.

## HTML Components

### div

We wrap our entire construct in \<div id="plsql-table"\>...\</div\> to isolate it from the rest of the document.

### style

Next we produce a \<style type="text/css" scoped\> section encapsulating our style rules for the table.

We have directives that adjust the table border and spacing, the optional caption font and spacing,
and directions for \<th\> table header and \<td\> element formatting. We can set background and foreground
colors as well.

This is fairly standard HTML attribute processing that is familiar enough for most of us who are not CSS gurus.

### specific column/row styles

Where the solution becomes more complex is when we direct style elements to particular
columns or rows. The one to solve our specific issue of right justifying particular columns is encoded
as:

    tr > td:nth-of-type(4) { text-align:right; }

Huh. That was my first thought. I'll read that as for any table row do this for the fourth table data element (aka column).
I'm not going to try to explain the rules because I'll butcher it, but that is what it does.

A related construct that lets us alternate the background color on rows is

    tr:nth-child(odd) { background-color: AliceBlue }

That says for table row elements, on the odd rows use this background color. You could leave the even rows
with the default background or make another directive for nth-child(even).

## Example

Putting it all together the package can output an HTML table configured specifically
for our query with any style adjustments you desire. You don't have to understand it all. Most of the time
you can copy/paste the default style and tweak it a bit. This one is on the fancy side while the 
default from the package is an austere black and white.

	<div id="plsql-table">
	<style type="text/css" scoped>
	
	table {
	    border: 1px solid black; 
	    border-spacing: 0; 
	    border-collapse: collapse;
	}
	caption {
	    font-style: italic;
	    font-weight: bold;
	    font-size: larger;
	    margin-bottom: 0.5em;
	}
	th {
	    text-align:left;
	    background-color: LightGrey
	}
	th, td {
	    border: 1px solid black; 
	    padding:4px 6px;
	}
	tr:nth-child(odd) { background-color: AliceBlue }
	tr > td:nth-of-type(1) {
	    text-align:right;
	}
	tr > td:nth-of-type(4) {
	    text-align:right;
	}
	</style>
	<table>
	<caption>Poorly Paid People</caption>
	<tr>
	  <th>Emp ID</th>
	  <th>Full Name</th>
	  <th>Date_x002C_Hire</th>
	  <th>Salary</th>
	</tr>
	<tr>
	  <td>999</td>
	  <td>  Baggins, Bilbo &quot;badboy&quot; </td>
	  <td>12/31/1999</td>
	  <td>     $123.45</td>
	</tr>
	<tr>
	  <td>206</td>
	  <td>Gietz, William</td>
	  <td>06/07/2002</td>
	  <td>   $8,300.00</td>
	</tr>
	<tr>
	</table></div>

And here is how it is rendered in my browser:

| *HTML Table Markup from PL/SQL* |
|:--:|
| ![html_table.gif](/images/html_table.gif) |
{:.img-table-centered}
