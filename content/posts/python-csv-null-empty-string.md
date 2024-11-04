---
title: Python can't distinguish between NULL and empty strings in CSVs
date: 2024-11-03
draft: true
tags: ["python"]
showToc: false
TocOpen: false
hidemeta: false
comments: false
# description: "Desc Text."
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: false
ShowPostNavLinks: false
ShowWordCount: false
ShowRssButtonInSectionTermList: false
UseHugoToc: false
---

Python can't distinguish between a NULL value and an empty string in a CSV file.

Let's say we have a CSV file with 3 columns `a,b,c`. The first column contains a non empty string, the second column contains a NULL and the third column contains an empty string:

```
a,b,c
some string,,""
```

Let's write a simple Python script that reads the CSV file:

```python
from csv import DictReader
row_list = list(DictReader(open('csv_file', 'r')))
print(row_list)
```

The output for `row_list` is:

`[{'a': 'some string', 'b': '', 'c': ''}]`

Notice that both `b` and `c` have similar values, the empty string. As far as Python is concerned, these 2 columns are no different, even though their values were different in the original CSV.

Let's write `row_list` to another CSV file:

```python
from csv import DictReader, DictWriter
row_list = list(DictReader(open('csv_file', 'r')))
print(row_list)
writer = DictWriter(open('csv_file_processed', 'w'), fieldnames=['a', 'b', 'c', ])
writer.writeheader()
writer.writerows(row_list)
```

The result CSV file looks like this:

```
a,b,c
some string,,
```

Both `b` and `c` columns now have a NULL value.

So `b` and `c` started out as NULL and empty string, were both read as empty strings, and are now written as NULLs. What a mess! This issue is discussed [here](https://bugs.python.org/issue23041).

## My personal experience with this issue

I worked on a service that reads CSV files, processes the data, writes the processed data to other CSV files, and [copies](https://www.postgresql.org/docs/9.2/sql-copy.html) the processed files into a PostgreSQL table. The intake PostgreSQL table was defined to accept NULL values for columns `b`, and to not accept NULL values for column `c`, although it could accept empty strings.

If we try to copy our initial file into this table there would be no copy issues since the data is valid relative to the table structure. On the other hand, if we try to copy the processed file, PostgreSQL would throw an exception:

```
Copy exception for table: null value in column "c" violates not-null constraint
```

This issue is mentioned [here](https://bugs.python.org/msg396621).

## Solutions

Python's CSV parsing functionality is written in [CPython](https://github.com/python/cpython/blob/f4c03484da59049eb62a9bf7777b963e2267d187/Modules/_csv.c). My hopes of simply overriding a method were quickly shattered.

### Update the Postgres model

Update the table definition to allow NULL values for column `c`.

This solution works for smaller apps that don't have multiple dependencies on table definitions. For my use case there would've been too many tables to change, queries to update, and issues to debug, and I opted for a different solution.

### Use FORCE NOT NULL in the copy query

A faster and less error prone solution is to update the copy SQL with the `FORCE NOT NULL` option for the problematic columns. This will forcefully insert an empty string instead of a NULL.

```sql
COPY table (a, b, c) FROM stdin WITH CSV HEADER DELIMITER as ',' FREEZE FORCE NOT NULL c;
```

A couple things to consider:
- You need to manually specify which columns to `FORCE NOT NULL` on, it's not dynamic.
- This is not a solution to the parsing problem. You're still changing the original input data, and then trying to work around it.

### Switch from Python to PostgreSQL transformations

Instead of parsing the data with Python, you can copy the input file in an intermediary Postgres table, transform the data using SQL, then export the table as a CSV. As opposed to Python, Postgres does distinguish between NULL values and empty strings when exporting to CSV.

### Update the Python CSV module

I think the Python CSV module should be able to distinguish between NULL values and empty strings. In my opinion this would be the optimal solution.
