---
date:
    created: 2024-11-11
    updated: 2024-11-11
categories:
    - Programming
tags:
    - Python
---
# Python One Liner
This post is a collection of Python one-liners, which will make your code neat without breaking the readability.
<!-- more -->

## Basic Syntax
We will start with the basic syntax of Python.

### Chain Comparison
In most programming languages, you can only compare two values at a time. However, Python allows you to compare multiple values at once, which is called "chain comparison".

```python
x, y = 1, 2
assert 0 < x < y < 100
```

But you'd better include one directional comparison in the chain to avoid ambiguity.

```python
assert 0 < x > y < 100 # Don't do this!
```

### Comprehension
Comprehension is a powerful feature of Python that allows you to create lists, sets, and dictionaries in a concise and readable way.

```python
# List Comprehension
squares = [x**2 for x in range(10)]

# Set Comprehension
squares_set = {x**2 for x in range(10)}

# Dictionary Comprehension
squares_dict = {x: x**2 for x in range(10)}
```

### Control Flow
Python has a few control flow statements that are commonly used in one-liners.

```python
# If-Else
x = 1 if 1 < 2 else 2

# For-Else
for i in range(10):
    if i == 5:
        break
else:
    print("No break")
```

## Advanced 
There are also other **Best Practices** in Python that can make your code more readable and efficient.

### Sequence Reverse
```python
s = "hello"
assert s[::-1] == "olleh"
```

### Getter
use `get` method to provide a default value when the key is not found in a dictionary. This may be useful than a try-except block.

```python
d = {"a": 1, "b": 2}
assert d.get("c", 3) == 3
```

In general, get values from Getters is more recommended.

### Partial
Pattial function is a function that takes some arguments and returns a new function that takes the remaining arguments. This can be useful when you want to partially apply a function to some arguments and then call it later with the remaining arguments.

```python
from functools import partial

def func(a, b, c):
    ...

kwargs = {"a": 1, "c": 3}

fn = partial(func, **kwargs)
```

Assume that you are training a machine learning model, and you want to tune the hyperparameters. 
However, there are many hyperparameters, and you don't want to write a lot of code to try different combinations.
You can use partial function to partially apply some arguments to the function, and then call it later with the remaining arguments.

### Zip
Zip is a built-in function in Python that takes two or more iterables and returns an iterator of tuples, where the i-th tuple contains the i-th element from each of the input iterables.

```python
names = ["Alice", "Bob", "Charlie"]
ages = [25, 30, 35]

for name, age in zip(names, ages):
    print(f"{name} is {age} years old")
```

### Enumerate
Enumerate is a built-in function in Python that takes an iterable and returns an iterator of tuples, where the i-th tuple contains the i-th element from the iterable and its index.

```python
names = ["Alice", "Bob", "Charlie"]

for i, name in enumerate(names):
    print(f"{i}: {name}")
```

It is much better than mainly using `range(len(names))` and `names[i]`.

### Remove Duplicates
You can use `set` to remove duplicates from a list.

```python
names = ["Alice", "Bob", "Charlie", "Alice"]
names = list(set(names))
```

However, this will not preserve the order of the elements. If you want to preserve the order, you can use `dict.fromkeys()`.

```python
names = ["Alice", "Bob", "Charlie", "Alice"]
names = list(dict.fromkeys(names))
```

You may worry about the performance of `dict.fromkeys()`. Indeed, if this part of the code is a bottleneck, you can rewrite the logic to avoid using `dict.fromkeys()`. For most cases, the performance difference is negligible.

