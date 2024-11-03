---
date: 2024-10-31
title: Minimizing the usage of else statements
draft: false
comments: true
# weight: 1
# aliases: ["/first"]
tags: ["clean code", "python"]
showToc: false
TocOpen: false
hidemeta: false
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
One way to keep production code clean and simple is by minimizing the usage of `else` statements. Conditional branching can negatively impact your code, by making it overcomplicated and hard to test. I actively look for ways to avoid using `else` whenever I write code. Below I go through a few simple, abstract Python examples to showcase how the same logic can be written with and without `else`, the latter improving code quality. These are simple patterns that can get you a long way in complex code bases.

### Example 1
A variable needs to be initialized with different values dependent on a condition.

```python
if condition:
    var = "value"
else:
    var = "other_value"
```

`else` can be removed if a default value is initially assigned to the variable, while the other value is assigned to the variable only if the condition is met.
```python
var = "other_value"
if condition:
    var = "value"
```

### Example 2
A function needs to be called (or an object needs to be instantiated) with different arguments depending on a condition.

```python
if arg2:
    value = func(arg1=arg1, arg2=arg2)
else:
    value = func(arg1=arg1)
```

Using a similar approach as in `Example 1` above, `else` can be removed if a dictionary is instantiated with a default set of arguments. If the condition is met, the dictionary will be updated with the new set of arguments. Finally, `func` is called with the unpacked dictionary.
```python
args = {"arg1": arg1, }
if arg2:
    args[arg2] = arg2
value = func(**args)
```

### Example 3
Distinct, independent functions need to be called depending on a condition.

The `if / else` would conditionally segregate the functions.
```python
if condition:
    func1()
else:
    func2()
```

In this situation we can't use the same pattern used in `Examples 1`, because that would default to calling `func2()` before checking the condition, and we don't want that. Instead, we can remove `else` and ensure a default by using a dictionary and its associated `.get()` method.

```python
func = {
    condition: func1
}.get(condition, func2)

func()
```

### Example 4
Different values need to be returned depending on a condition.

```python
if condition:
    # some logic
    return "value"
else:
    # some other logic
    return "other_value"
```

In this case `else` is redundant and can simply be removed.
```python
if condition:
    # some logic
    return "value"

# some other logic
return "other_value"
```

### `else` has its place

I'm not opposed to using `else`. It exists for a reason. Ternary operators are a great example.
```python
var = "value" if condition else "other_value"
```

In other cases, depending on the complexity of the code base and time constraints, it could be the best solution for a particular situation.

That said, if using `else` seems to be the only option, more than likely there's a refactored solution that could avoid it. Refactoring takes time and effort and it's understandable sometimes to just say fuck it, throw an `else` in the mix, and get the job done quickly. We've all been there, and we'll probably do so again, so more power to us.

### Conclusion

One of the primary reasons code bases become complicated and hard to maintain is the excessive use of `else` statements, particularly when they are nested. For your peace of mind and that of your teammates, and for enhancing your developer skills, it's highly recommended to minimize its usage. And when you do use, use it wisely.

As a side note, `else` can freely be used with `except` statements and `for` loops, since that's not conditional branching.
