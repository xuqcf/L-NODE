---
tags: [python, functional-programming, programming]
links: ["[[Programming Index]]"]
---


---

## General Python Rules

- Python evaluates the arguments of a function **before** executing the function body.
    
    - This means values are already decided when the function starts running.
        
- To check if a dictionary is empty:
    
    ```python
    if not my_dict:
    ```
    
    - Empty dict → `False`
        
    - Non‑empty dict → `True`
        

---

## Strings

- Strings are **immutable**.
    
    - You cannot change a string in place.
        
    - Methods like `.replace()` return a **new string**.
        

```python
text.replace(old, new)
```

- Example:
    

```python
text = "hello"
new_text = text.replace("h", "H")
# text is still "hello"
# new_text is "Hello"
```

---

## Splitting Text

- `doc.split("\n")`
    
    - Splits a string by **lines**, not by spaces.
        

```python
doc = "a\nb\nc"
doc.split("\n")
# ['a', 'b', 'c']
```

---

## Membership / `in`

- `in` checks if something exists inside another thing.
    

```python
if "a" in "cat":
    pass
```

- Under the hood, this is the same as:
    

```python
"cat".__contains__("a")
```

- `in` is basically a cleaner way of writing a contains check.
    

---

## Escaping Characters (URLs)

### Escape parentheses

Used when URLs contain characters that break formatting.
```python
url = url.replace("(", "%28")
url = url.replace(")", "%29")
```

---

## Functional Programming Concepts

### Functions are values

- Functions can be:
    
    - passed as arguments
        
    - returned from other functions
        
    - stored in variables
        

```python
f = print
f("hello")
```

---

### Function transformations

- A function that takes data and returns transformed data.
    

```python
def to_upper(text):
    return text.upper()
```

- Transformations should:
    
    - take input
        
    - return output
        
    - not rely on hidden state
        

---

### Pure functions

- A pure function:
    
    - always returns the same output for the same input
        
    - does not change anything outside itself
        

```python
def add(a, b):
    return a + b
```

- Not pure:
    

```python
def add_and_print(a, b):
    print(a + b)
```

---

### Closures

- A closure is when an inner function **remembers variables** from an outer function.
    

```python
def outer(x):
    def inner(y):
        return x + y
    return inner
```

- `x` is remembered even after `outer` finishes.
    

---

### Currying

- Currying means taking arguments **one at a time** using nested functions.
    

```python
f(a)(b)(c)
```

- Each function call locks in one value.
    

---

### Important currying rule 

- Parameter names do **not** matter.
    
- What matters is **what value is passed in**.
    

```python
def create_pipeline(step_one):
    pass
```

- `step_one` is just a label.
    
- The test decides what function it actually is.
    
- You are not creating the function, you are receiving it.
    

---

### Control vs execution

- Options choose **what to run**.
    
- Data flows in **last**.
    

```python
pipeline(option)(text)
```

- Option controls behavior
    
- Text is the actual data
    

---

### Recursion

- A function that calls itself.
    

```python
def factorial(n):
    if n == 1:
        return 1
    return n * factorial(n - 1)
```

- Needs:
    
    - base case (when to stop)
        
    - recursive case (call itself)
        

---

### References vs values

- Mutable objects (lists, dicts) are passed by reference.
    
- Immutable objects (ints, strings) are passed by value.
    

```python
def add_item(lst):
    lst.append(1)
```

- Modifies the original list.
    

---

## Resizing Variables (min / max clamp)

- `min()` and `max()` **do not modify variables**
    
- They **return a value**
    
- **Always assign** the result back to a variable
    

```python
# enforce minimum (raise if too small)
width = max(width, min_width)

# enforce maximum (cap if too large)
width = min(width, max_width)

# clamp both (keep within range)
width = max(min_width, min(width, max_width))
```

- `max()` → fixes **too small** values (pushes up)
    
- `min()` → fixes **too large** values (caps down)
    
- Argument **order does not matter** (`min(a, b)` == `min(b, a)`)
    
- This pattern is called **clamping**
    
- Equivalent to:
    

```python
if width > max_width:
    width = max_width
if width < min_width:
    width = min_width
```

---


---

## Python Decorators

- A **decorator** is just a **higher-order function**: it takes a function and returns a new function that adds extra behavior.
    
- `@decorator` is **syntactic sugar** (just cleaner syntax) for passing a function into another function manually.
    

---

### What `@decorator` really means

```python
@vowel_counter
def process_doc(doc):
    ...
```

is the same as:

```python
def process_doc(doc):
    ...

process_doc = vowel_counter(process_doc)
```

---

### Key idea

- The decorator function (`vowel_counter`) is called **once**, when the function is defined.
    
- The function it returns (usually called `wrapper`) is called **every time** the decorated function is used.
    
- Any variables inside the decorator stay alive using **closures**.
    

---

### Stacking decorators

You can put **more than one decorator** on a function.

```python
@decorator_a
@decorator_b
def func():
    ...
```

This means:

```python
func = decorator_a(decorator_b(func))
```

Important rules:

- **Decorators are applied (wrapped) from bottom to top**
    
- **Functions are executed from top to bottom**
    

So:

- `decorator_b` wraps the function first
    
- `decorator_a` wraps the result of that
    

But when `func()` is called:

- `decorator_a` runs first
    
- then `decorator_b`
    
- then the original function
    

---

### Decorators with arguments (currying)

If a decorator needs arguments, you add **one more function layer**.

```python
def get_truncate(length):
    def truncate(func):
        def wrapper(document):
            return func(document[:length])
        return wrapper
    return truncate
```

Using it:

```python
@get_truncate(9)
def print_input(text):
    print(text)
```

This means:

1. `get_truncate(9)` runs immediately
    
2. It returns `truncate`
    
3. `truncate` wraps `print_input`
    

So this is equivalent to:

```python
print_input = get_truncate(9)(print_input)
```

---

### Stacking + currying together

```python
@to_uppercase
@get_truncate(9)
def print_input(text):
    print(text)
```

This becomes:

```python
print_input = to_uppercase(get_truncate(9)(print_input))
```

What happens when you call it:

```python
print_input("Keep Calm and Carry On")
```

Order of execution:

1. `to_uppercase` runs first
    
2. `get_truncate(9)`’s wrapper runs next
    
3. The original `print_input` runs last
    

Data flow:

```text
"Keep Calm and Carry On"
→ "KEEP CALM AND CARRY ON"
→ "KEEP CALM"
→ printed
```

---

### Simple mental model

- Each decorator adds a **layer**
    
- Bottom layers are built first
    
- Top layers run first
    
- Data flows through each layer before reaching the original function
    




---

## `*args` and `**kwargs` (Python)

- `*args` lets a function accept **any number of positional arguments**
    
- `**kwargs` lets a function accept **any number of keyword (named) arguments**
    

---

## What they collect

### `*args`

- Collects **extra positional arguments**
    
- Stored as a **tuple**
    
- Order matters
    
- Values have **no names**, only position
    

```python
def f(*args):
    print(args)

f(1, 2, 3)
# (1, 2, 3)
```

---

### `**kwargs`

- Collects **extra keyword arguments**
    
- Stored as a **dictionary**
    
- Order does not matter
    
- Each value has a **name (key)**
    

```python
def f(**kwargs):
    print(kwargs)

f(a=1, b=2)
# {'a': 1, 'b': 2}
```

---

## How arguments are assigned

```python
def f(a, b, *args, **kwargs):
    pass
```

- First values go to `a` and `b`
    
- Remaining positional values go into `args`
    
- Named values go into `kwargs`
    

Example:

```python
f(1, 2, 3, 4, x=5)
# a=1, b=2
# args = (3, 4)
# kwargs = {'x': 5}
```

---

## Positional vs keyword arguments

### Positional arguments

- Passed by position
    
- Order matters
    

```python
sub(3, 2)  # a=3, b=2
```

---

### Keyword arguments

- Passed by name
    
- Order does not matter
    

```python
sub(b=3, a=2)
```

---

## Ordering rule

- **Positional arguments must come before keyword arguments**
    
- This is invalid:
    

```python
sub(b=3, 2)
```

---
## `enumerate()` (Python)

- `enumerate(iterable)` returns **pairs** of `(index, value)`
    
- Each pair is a **tuple**
    
- Used to get an index and value at the same time
    

### Example

```python
items = ["hi", "there"]

for item in enumerate(items):
    print(item)
```

Output:

```text
(0, 'hi')
(1, 'there')
```

---

### Common usage (unpacking)

```python
for i, item in enumerate(items):
    print(i, item)
```

---

### Start counting from 1

```python
for i, item in enumerate(items, start=1):
    print(f"{i}. {item}")
```

---

## Tuple Unpacking

- Assigns elements of a tuple to multiple variables in one step
    
- Number of variables must match number of elements
    

### Example

```python
pair = ("age", 17)
key, value = pair
```

---

### Tuple unpacking in loops

```python
for key, value in kwargs.items():
    print(key, value)
```

Equivalent to:

```python
for item in kwargs.items():
    key = item[0]
    value = item[1]
```

---

## Rule to remember

- Each item must have the **same number of elements** as variables
    

```python
a, b = (1, 2)      # works
a, b = (1, 2, 3)   # error
```

---
**Unpacking a dictionary** means:

> Take each `key: value` pair in the dictionary and pass it as a **named argument** to a function.


---

## `map()` (Python)

- `map(function, iterable)` applies a function to **each item** in an iterable
    
- Returns a **map object** (lazy iterator)
    
- Often converted to a list or dict
    

### Basic example

```python
nums = [1, 2, 3]
result = list(map(lambda x: x * 2, nums))
# [2, 4, 6]
```

### How `map` works

- Takes one item at a time
    
- Calls the function on it
    
- Stores the returned value
    
- Repeats until the iterable ends
    

---

## `map()` with tuples

```python
pairs = [("a", 1), ("b", 2)]
new = list(map(lambda item: (item[0], item[1] * 2), pairs))
# [("a", 2), ("b", 4)]
```

---

## `map()` with dictionaries

- Use `.items()` to get `(key, value)` tuples
    
- The function must return `(key, value)` tuples
    
- Wrap with `dict()` to rebuild the dictionary
    

```python
data = {"a": 1, "b": 2}

new_data = dict(
    map(lambda item: (item[0], item[1] * 2), data.items())
)
# {"a": 2, "b": 4}
```

---

## `map`, `str`, and `list`

### `str()`

- Built-in function that **converts a value into a string**
    
- Works on almost anything
    

```python
str(5)      # "5"
str(True)   # "True"
str(None)   # "None"
```

---

### `map(function, iterable)`

- Applies a function to **every item** in an iterable
    
- **Does not return a list**
    
- Returns a **map object (iterator)**
    

```python
map(str, [1, 2, 3])   # <map object>
```

---

### `list()`

- Converts an iterable (like `map`) into a **real list**
    
- Forces evaluation
    

```python
list(map(str, [1, 2, 3]))   # ["1", "2", "3"]
```

---

### Nested `map`

- One `map` for rows
    
- One `map` for items inside each row
    
- Each `map` must be wrapped in `list()`
    

```python
list(
    map(
        lambda row: list(map(str, row)),
        data
    )
)
```

---

### Key rule

- **Every `map` you want to use like a list must be wrapped in `list()`**
    

---

If you want, I can also compress this into a **one-screen cheat sheet** or rewrite it in the exact style of your other FP notes.
---

## Lambda Functions

- A **lambda** is a small, one-line function
    
- Has no name
    
- Used when the function is simple and used once
    

### Syntax

```python
lambda arguments: expression
```

### Example

```python
lambda x: x * 2
```

Same as:

```python
def double(x):
    return x * 2
```

---

## Lambda with `map`

```python
map(lambda x: x + 1, [1, 2, 3])
```

- `lambda` defines what to do
    
- `map` applies it to every item
    

---

## Why lambda is useful

- Avoids defining small helper functions
    
- Keeps logic close to where it’s used
    
- Common in decorators and data transformations
    

---

## Key rules

- `map` always applies the function **one item at a time**
    
- Lambda must return the **new value**
    
- For dicts, return `(key, value)` pairs
    

---

## `lru_cache` (Python)

- `lru_cache` is a decorator from the `functools` module
    
- It **remembers function inputs and outputs**
    
- If the function is called again with the same input, Python **returns the stored result instead of recomputing**
    

---

## Memoization

- Memoization means **saving results of expensive function calls**
    
- Same input → same output → no need to redo the work
    
- Uses a cache (basically a dictionary under the hood)
    

Example idea:

```
input → output
5 → 120
6 → 720
```

If `5` is requested again, Python just returns `120`.

---

## Why `lru_cache` is useful

- Speeds up slow functions
    
- Especially useful when:
    
    - The function is **recursive**
        
    - The function is called many times with the **same inputs**
        
    - The function does heavy computation, disk access, or network requests
        

---

## `@lru_cache()` as a decorator

```python
@lru_cache()
def factorial_r(x):
    ...
```

- This tells Python:
    
    > “Before running this function, check if the result is already cached.”
    

Equivalent to:

```python
factorial_r = lru_cache()(factorial_r)
```

---

## Recursive example (factorial)

```python
from functools import lru_cache

@lru_cache()
def factorial_r(x):
    if x == 0:
        return 1
    return x * factorial_r(x - 1)
```

---

## How recursion works here

Calling `factorial_r(5)` causes:

```
factorial_r(5)
factorial_r(4)
factorial_r(3)
factorial_r(2)
factorial_r(1)
factorial_r(0)
```

Each call depends on the previous one.

---

## What happens WITHOUT `lru_cache`

- Every call recomputes everything from scratch
    
- Repeated inputs cause repeated work
    
- Very inefficient for recursive functions
    

Calling:

```python
factorial_r(10)
factorial_r(5)
factorial_r(12)
```

Would recompute many of the same values again.

---

## What happens WITH `lru_cache`

### First call

```python
factorial_r(10)
```

- Computes all values from `0` to `10`
    
- Stores them in the cache
    

Cache contains:

```
0 → 1
1 → 1
2 → 2
...
10 → 3628800
```

---

### Second call

```python
factorial_r(5)
```

- Python checks the cache
    
- Value already exists
    
- **No recursion happens**
    
- Result is returned instantly
    

---

### Third call

```python
factorial_r(12)
```

- Values up to `10` already cached
    
- Only computes:
    
    - `factorial_r(11)`
        
    - `factorial_r(12)`
        

---

## Why it’s called LRU (Least Recently Used)

- The cache has a limited size
    
- When it fills up:
    
    - The **least recently used** entries are removed
        
- Prevents unlimited memory usage
    

Example:

```python
@lru_cache(maxsize=128)
```

---

## Mental model (important)

Think of `lru_cache` as doing this automatically:

```python
cache = {}

def function(x):
    if x in cache:
        return cache[x]

    result = compute(x)
    cache[x] = result
    return result
```

You don’t write this — the decorator handles it.

---

## Rule of thumb

Use `lru_cache` when:

- Function is **pure** (same input → same output)
    
- Function is **called repeatedly**
    
- Function is **slow or recursive**
    

Do NOT use it when:

- Function depends on changing external state
    
- Function returns different results for the same input
    

---

## Key takeaway

- `lru_cache` = automatic memoization
    
- Decorator that saves time by trading a bit of memory
    
- Extremely powerful for recursion and repeated computation
    

---

## Sum Types (Plain English)

- A **sum type** means something can be **one of a few fixed options**, and **only one at a time**
    
- It is the **opposite of a product type**
    

---

### Product type vs Sum type

**Product type**

- Uses **multiple fields at once**
    
- Every new field **multiplies** the number of possible cases
    
- You have to check **combinations** of values
    

Example idea:

- studies_finance = True / False
    
- has_trust_fund = True / False
    
- has_blue_eyes = True / False
    

Each new boolean doubles the cases

---

**Sum type**

- Uses **one category**
    
- The number of cases is **fixed**
    
- You check **which case it is**, not combinations
    

Example idea:

- Dateable
    
- MaybeDateable
    
- Undateable
    

Only one can be true

---

### Why sum types are useful

- You don’t check every possible combination
    
- You don’t need complex condition chains
    
- You handle a **small, known set of cases**
    
- Impossible states are avoided by design
    

---

### How this works in Python

- Python doesn’t have real sum types
    
- You fake them using:
    
    - subclasses
        
    - `isinstance()` checks
        
- Each object is created as **one specific type**
    
- Your code switches behavior based on that type
    

---

## Sum Types – Runtime Behavior (Plain English)

- Even if multiple classes are defined, **only one is ever created at runtime**
    
- The classes represent **possible outcomes**, not things that all run together
    

---

### How this works

- A base class (like `MaybeParsed`) represents **“one of these outcomes”**
    
- Subclasses represent **each specific outcome**
    
    - Success → `Parsed`
        
    - Failure → `ParseError`
        
- When the function runs, it **returns exactly one instance**
    
- The other classes exist only as definitions, not active objects
    

---

### Important distinction

- **Classes** = blueprints (definitions)
    
- **Objects** = things that actually exist at runtime
    

Defining 3 classes does **not** mean all 3 are used at once.

---

### What happens in practice

- A function runs
    
- It chooses **one path**
    
- It creates **one object**
    
- That object has **one type**
    
- Code checks the type to decide what to do next
    

---

### Why the base class exists

- It groups related outcomes under one name
    
- It lets you say:
    
    - “This function always returns a `MaybeParsed`”
        
- The specific subclass tells you **which outcome happened**
    

---

### Key idea

- Sum types mean **one result, many possible shapes**
    
- You handle behavior by checking **which shape you got**
    
---

### Key idea

- **Product types** ask:  
    “What combination of traits does this have?”
    1
- **Sum types** ask:  
    “Which single case is this?”
    

---

## Enums (Plain English)

- An **enum** represents a **fixed set of allowed values**
    
- A value can be **only one** of those options
    
- Anything outside that set is **invalid**
    

---

### The problem enums solve

- Using strings allows:
    
    - typos
        
    - invalid values
        
    - repeated defensive checks
        
- There is no single place that defines “what’s allowed”
    

---

### What an enum does

- Defines all valid options **in one place**
    
- Prevents invalid values automatically
    
- Forces mistakes to fail early
    

Example idea:

- Color can be:
    
    - RED
        
    - GREEN
        
    - BLUE
        
- Color **cannot** be anything else
    

---

### Enums as sum types

- An enum is a sum type with:
    
    - a **fixed number of cases**
        
    - no extra data per case
        
- A value is:
    
    - this case **or**
        
    - that case
        
- Never a combination
    

---

### Name vs value

- Each enum item has:
    
    - a **name** (e.g. `Color.RED`)
        
    - a **value** (usually an integer)
        
- You use the **name** in code
    
- Python uses the **value** internally
    

---

### Why enums are useful

- Invalid states are blocked
    
- All cases are visible and explicit
    
- Code is easier to read and reason about
    
- Slight memory and performance benefits over strings
    

---

### When to use enums

- You need a **small, fixed set of options**
    
- You **don’t need extra data** per option
    
- You want safer code than plain strings
    

---

### When not to use enums

- Each case needs its own data
    
- Each case needs different behavior  
    → use classes instead
    

---
### Why exceptions are avoided in FP

In Python, errors are usually handled like this:

`raise ParseError("invalid syntax")`

This causes a **side effect**:

- The function suddenly stops
    
- Control jumps somewhere else
    
- You can’t see that behavior just by looking at the return type
    

From a functional programming mindset, that’s bad because:

- The function is no longer predictable
    
- The function doesn’t always return a value
    
- You have to know hidden behavior (exceptions) to use it safely

---

## `match / case` (Python)

### What it is

- `match` is Python’s **pattern matching** statement
    
- Used to choose **one code path** based on a value
    
- Similar to `switch` in other languages
    

---

### Basic structure

```python
match value:
    case pattern1:
        do_something()
    case pattern2:
        do_something_else()
    case _:
        fallback()
```

- `value` is matched **top to bottom**
    
- First matching `case` runs
    
- `_` means **anything else**
    

---

### Matching Enums

```python
match status:
    case CSVExportStatus.PENDING:
        ...
    case CSVExportStatus.SUCCESS:
        ...
```

- Enum cases must use **EnumName.VALUE**
    
- Strings and Enums are **not the same**
    

Wrong:

```python
case "PENDING":
```

Correct:

```python
case CSVExportStatus.PENDING:
```

---

### Important rules

- No `break` needed
    
- Only **one case** runs
    
- `_` should usually be **last**
    

---

## Handling Errors Properly

### When to raise an error

- Use errors for **invalid states**
    
- Not for normal program flow
    

Example:

```python
case _:
    raise Exception("unknown export status")
```

---

### `raise`

- Stops execution immediately
    
- Sends an error up the call stack
    

```python
raise Exception("something went wrong")
```

---

### Why not return errors as strings?

- Errors should be **loud**
    
- Forces the caller to handle the problem
    
- Prevents silent bugs
    

Bad:

```python
return "error"
```

Good:

```python
raise Exception("error")
```

---

## Pattern used in real code

```python
match state:
    case VALID:
        handle_valid()
    case INVALID:
        handle_invalid()
    case _:
        raise Exception("invalid state")
```

---

## Key mental model

> `match` selects behavior  
> `raise` rejects invalid situations

---
### Summary mindset


- Names don’t matter, bindings do
    
- Functions are data
    
- Pure functions are predictable
    
- Closures remember
    
- Currying delays arguments
    
- Options decide behavior
    
- Data comes last