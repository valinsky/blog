---
title: "Python CSV quirks"
date: 2025-04-01
draft: false
tags: ["python"]
showToc: false
TocOpen: false
hidemeta: false
comments: false
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: false
ShowPostNavLinks: false
ShowWordCount: false
ShowRssButtonInSectionTermList: true
UseHugoToc: false
---

Python can't distinguish between a null value and an empty string in a CSV file.

Let's say we have a CSV file with 3 columns `a,b,c`. The first column contains a non-empty string, the second column contains a null, and the third column contains an empty string:

```
a,b,c
some string,,""
```

Let's write a simple Python script that reads the CSV file:

```python
from csv import DictReader
row_list = list(DictReader(open('input_csv', 'r')))
print(row_list)
```

The output for `row_list` is:

`[{'a': 'some string', 'b': '', 'c': ''}]`

Notice that both `b` and `c` have similar values, the **empty string**. As far as Python is concerned, these 2 columns are no different, even though their values were different in the original CSV.

Now let's write `row_list` to another CSV file:

```python
from csv import DictReader, DictWriter
row_list = list(DictReader(open('input_csv', 'r')))
print(row_list)

writer = DictWriter(open('output_csv', 'w'), fieldnames=['a', 'b', 'c', ])
writer.writeheader()
writer.writerows(row_list)
```

The result CSV file looks like this:

```
a,b,c
some string,,
```

Both `b` and `c` columns now contain a **null value**.

So `b` and `c` started out as null and empty string, were both read as empty strings, and are now written as nulls.

<img src="/img/confused.png" alt="Description of the image" width="300" height="300">

## My personal experience with this issue

I worked on a service that processed incoming CSVs, wrote the processed data to other CSVs, and then [sql copied](https://www.postgresql.org/docs/17/sql-copy.html) the files into a PostgreSQL table. The table was defined to accept null values for column `b`, and to not accept null values for column `c`, although it could accept empty strings.

```python
b = models.CharField(null=True, blank=True)
c = models.CharField(null=False, blank=True)
```

If we try to copy our initial file into this table, there would be no issues since the data is valid relative to the table structure. On the other hand, if we try to copy the processed file, PostgreSQL would throw an exception:

```
Copy exception for table: null value in column "c" violates not-null constraint
```

This issue has been reported [here](https://bugs.python.org/msg396621).

## Solution

Python's CSV parsing functionality is written in [CPython](https://github.com/python/cpython/blob/f4c03484da59049eb62a9bf7777b963e2267d187/Modules/_csv.c), so my hopes of simply overriding a method to parse the CSV according to my needs were quickly shattered.

Updating the PostgreSQL table definition to allow null values for column `c` also wasn't an option for me at the time.

The solution I ultimately opted for was to update the copy SQL with the `FORCE NOT NULL` option for the problematic columns.

```sql
COPY table (a, b, c) FROM stdin WITH CSV HEADER DELIMITER as ',' FREEZE FORCE NOT NULL c;
```

This forcefully inserts an empty string instead of a null for the specified columns.

### Closing thoughts

After experiencing this firsthand in a production environment, I find it intriguing that Python's CSV module behaves the way it does. Nulls and empty strings are fundamentally different types, and the inability to properly distinguish between them forces developers to seek workarounds for what seems like a basic issue.

Yomguithereal's CSV love letter references [CSV's dinamically typed](https://github.com/medialab/xan/blob/master/docs/LOVE_LETTER.md#6-csv-is-dynamically-typed) nature, and how it *could* be beneficial if handled properly.

> *Consider JavaScript, for instance, that is unable to represent 64 bits integers. Or what languages, frameworks and libraries consider as null values (don't get me started on pandas and null values). CSV lets you parse values as you see fit and is in fact dynamically typed. But this is as much of a strength as it can become a potential footgun if you are not careful.*

I can't help but feel that Python is shooting itself in the tail here, with a relatively small-caliber bullet, true, but the wound could get infected and cause other more serious problems. Then again, I'm not a CPython developer, and I may not be aware of all the arguments behind its current behavior. One can only hope for improvements in the future.
