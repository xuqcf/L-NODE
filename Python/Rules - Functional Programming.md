---
tags: [python, functional-programming, programming]
links: ["[[Programming Index]]", "[[Functional Programming Notes]]"]
---

# Tuples vs. Lists

Tuples and lists are both ordered collections of values, but tuples are _immutable_ and lists are _mutable_.

You can append to a list, but you can _not_ append to a tuple. You can create a _new copy_ of a tuple using values from an existing tuple, but you can't change the existing tuple.
[[Arch Commands]]
### Lists Are Mutable

```python
ages = [16, 21, 30]
# 'ages' is being changed in place
ages.append(80)
# [16, 21, 30, 80]
```

### Tuples Are Immutable

```python
ages = (16, 21, 30)
more_ages = (80,) # note the comma! It's required for a single-element tuple
# 'all_ages' is a brand new tuple
all_ages = ages + more_ages
# (16, 21, 30, 80)

# or we can even reassign the same variable to point to a new tuple:
ages = ages + more_ages
# (16, 21, 30, 80)
```