---
tags: [python, oop, programming]
links: ["[[Programming Index]]"]
---
1Here's a list of how the operators translate into method names.

| Operation           | Operator | Method         |
| ------------------- | -------- | -------------- |
| Addition            | +        | `__add__`      |
| Subtraction         | -        | `__sub__`      |
| Multiplication      | *        | `__mul__`      |
| Power               | **       | `__pow__`      |
| Division            | /        | `__truediv__`  |
| Floor Division      | //       | `__floordiv__` |
| Remainder (modulo)  | %        | `__mod__`      |
| Bitwise Left Shift  | <<       | `__lshift__`   |
| Bitwise Right Shift | >>       | `__rshift__`   |
| Bitwise AND         | &        | `__and__`      |
| Bitwise OR          | \|       | `__or__`       |
| Bitwise XOR         | ^        | `__xor__`      |
| Bitwise NOT         | ~        | `__invert__`   |

#  In Python, every list has a method called `.index(value)` that returns **the position of that value in the list**. 


```python

SUITS = ["Clubs", "Diamonds", "Hearts", "Spades"]

RANKS = ["2", "3", "4", "5", "6", "7", "8", "9", "10", "Jack", "Queen", "King", "Ace"]


RANKS.index("2")      # → 0
RANKS.index("10")     # → 8
RANKS.index("King")   # → 11
RANKS.index("Ace")    # → 12
```

So when you write:

```python
self.rank_index = RANKS.index(rank)
```

and you create:

```python
card = Card("King", "Hearts")
```

it becomes:

```python
self.rank_index = RANKS.index("King")  # → 11
self.suit_index = SUITS.index("Hearts")  # → 2
```

Now the card internally looks like:

- rank = `"King"`
    
- suit = `"Hearts"`
    
- rank_index = `11`
    
- suit_index = `2`
    

Those numbers are what let you do real comparisons:

```python
Card("Ace", "Hearts") > Card("King", "Hearts")
# becomes
12 > 11   → True
```



