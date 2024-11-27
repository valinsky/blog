---
title: "The Pitfalls of NULLs and Empty Strings in Python's CSV Handling"
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

Both `b` and `c` columns now have a **NULL value**.

So `b` and `c` started out as NULL and empty string, were both read as empty strings, and are now written as NULLs. This is a mess! This issue is discussed [here](https://bugs.python.org/issue23041).

## My personal experience with this issue

I worked on a service that processed incoming CSVs, wrote the processed data to other CSVs, and then [copied](https://www.postgresql.org/docs/9.2/sql-copy.html) the files into a PostgreSQL table. The table was defined to accept NULL values for column `b`, and to not accept NULL values for column `c`, although it could accept empty strings.

```python
b = models.CharField(null=True, blank=True)
c = models.CharField(null=False, blank=True)
```

If we try to copy our initial file into this table there would be no issues since the data is valid relative to the table structure. On the other hand, if we try to copy the processed file, PostgreSQL would throw an exception:

```
Copy exception for table: null value in column "c" violates not-null constraint
```

This issue is mentioned [here](https://bugs.python.org/msg396621).

## Solution

Python's CSV parsing functionality is written in [CPython](https://github.com/python/cpython/blob/f4c03484da59049eb62a9bf7777b963e2267d187/Modules/_csv.c). My hopes of simply overriding a method were quickly shattered, so I had to find a workaround.

I opted for updating the copy SQL with the `FORCE NOT NULL` option for the problematic columns.

```sql
COPY table (a, b, c) FROM stdin WITH CSV HEADER DELIMITER as ',' FREEZE FORCE NOT NULL c;
```

This forcefully inserts an empty string instead of a NULL for the specified columns.

Another potential solution was to update the PostgreSQL table definition to allow NULL values for column `c`. This solution didn't work for my use case because there were too many dependencies on the table definition, and updating all the relevant queries and transformations wasn't worth the hassle.

### Conclusion

After experiencing this first hand I find it intriguing that the Python CSV module behaves the way it does, and forces developers into having to find workarounds. I think the Python CSV module should be able to properly distinguish between these two distinct values, and to maintain data integrity throughout the transformation phases. In my opinion this would be the optimal solution. I'm not a CPython developer so I am more than happy to hear (what I am missing) why I could be wrong.
