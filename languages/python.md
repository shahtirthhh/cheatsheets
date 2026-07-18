# Python — Complete Language Mastery Cheatsheet
*Intern → Super Senior · Every data type, method, gotcha, and CPython internal*

---

## 1. Types and Type System

### Built-in Types

```
Immutable:
  int         42, 0xFF, 0b1010, 0o17
  float       3.14, 1e-3, float('inf'), float('nan')
  complex     3+4j
  bool        True, False (subclass of int!)
  str         "hello", 'hello', """multiline"""
  bytes       b"hello"
  tuple       (1, 2, 3)
  frozenset   frozenset({1, 2, 3})
  NoneType    None

Mutable:
  list        [1, 2, 3]
  dict        {"a": 1, "b": 2}
  set         {1, 2, 3}
  bytearray   bytearray(b"hello")
```

### Type Checking

```python
type(42)                  # <class 'int'>
type("hello")             # <class 'str'>
isinstance(42, int)       # True
isinstance(True, int)     # True!  (bool is a subclass of int)
isinstance(42, (int, float))  # True (check multiple types)
issubclass(bool, int)     # True

# id() — memory address (CPython: actual pointer)
a = [1, 2, 3]
b = a
id(a) == id(b)            # True (same object)
a is b                    # True (identity check — same object in memory)
a == b                    # True (equality check — same value)

c = [1, 2, 3]
a is c                    # False (different objects)
a == c                    # True  (same value)
```

### Everything Is an Object

```python
# Functions are objects
def greet(): pass
type(greet)              # <class 'function'>
greet.custom_attr = 42   # you can add attributes to functions

# Classes are objects
type(int)                # <class 'type'>
type(type)               # <class 'type'> — type is its own metaclass

# Modules are objects
import math
type(math)               # <class 'module'>
```

---

## 2. Numbers

### int

```python
# Arbitrary precision (no overflow!)
2 ** 1000                 # a 302-digit number — Python handles it natively

# Base conversions
0xFF                      # 255 (hex literal)
0b1010                    # 10  (binary literal)
0o17                      # 15  (octal literal)
bin(255)                  # '0b11111111'
hex(255)                  # '0xff'
oct(15)                   # '0o17'
int("FF", 16)             # 255 (string → int with base)
int("1010", 2)            # 10

# Underscores for readability
1_000_000                 # 1000000
0xFF_FF                   # 65535
```

### float

```python
# IEEE 754 double precision (same as JS)
0.1 + 0.2                # 0.30000000000000004
0.1 + 0.2 == 0.3         # False!

# Use decimal for precision
from decimal import Decimal
Decimal("0.1") + Decimal("0.2") == Decimal("0.3")  # True

# Special values
float('inf')              # positive infinity
float('-inf')             # negative infinity
float('nan')              # not a number
import math
math.isnan(float('nan'))  # True
math.isinf(float('inf')) # True
math.isfinite(42.0)      # True
```

### Operators

```python
10 / 3       # 3.3333...  (true division — always float)
10 // 3      # 3          (floor division — rounds down)
10 % 3       # 1          (modulo)
2 ** 10      # 1024       (exponentiation)
divmod(10, 3) # (3, 1)    (quotient and remainder)
abs(-5)      # 5
round(3.14159, 2)  # 3.14

# Floor division with negatives (Python rounds toward -∞)
-7 // 2      # -4  (not -3!)
# In JS/C: -7 / 2 truncates to -3
```

### bool Is a Subclass of int

```python
isinstance(True, int)     # True
True + True               # 2
True * 10                 # 10
False + 1                 # 1
sum([True, False, True, True])  # 3 (counts truthy values)
```

### Truthiness

```python
# Falsy values (everything else is truthy):
bool(0)         # False
bool(0.0)       # False
bool("")        # False
bool([])        # False  ← empty list is falsy (unlike JS!)
bool({})        # False  ← empty dict is falsy (unlike JS!)
bool(set())     # False
bool(())        # False
bool(None)      # False
bool(False)     # False

# Truthy:
bool(1)         # True
bool(-1)        # True
bool("0")       # True  (non-empty string)
bool([0])       # True  (non-empty list)
bool(" ")       # True  (non-empty string, even whitespace)
```

---

## 3. Strings

### Creation and Internals

Strings are immutable sequences of Unicode code points (UTF-8 internally in CPython 3.x). Every string method returns a new string.

```python
s = "Hello, World!"
s2 = 'single quotes'
s3 = """triple-quoted
multiline string"""
s4 = r"raw string: \n is literal backslash-n"
s5 = f"formatted: {2 + 2} = four"
s6 = b"bytes literal"       # bytes, not str
```

### String Methods (Complete)

```python
s = "  Hello, World!  "

# Case
s.upper()                  # "  HELLO, WORLD!  "
s.lower()                  # "  hello, world!  "
s.title()                  # "  Hello, World!  "
s.capitalize()             # "  hello, world!  " → capitalize first char of entire string
s.swapcase()               # "  hELLO, wORLD!  "
s.casefold()               # like lower() but more aggressive (ß → ss)

# Search
s.find("World")            # 9  (-1 if not found)
s.rfind("l")               # 14 (search from right)
s.index("World")           # 9  (raises ValueError if not found — unlike find)
s.count("l")               # 3
s.startswith("  Hello")    # True
s.endswith("!  ")          # True
"World" in s               # True (containment check)

# Strip (trim)
s.strip()                  # "Hello, World!"
s.lstrip()                 # "Hello, World!  "
s.rstrip()                 # "  Hello, World!"
"xxHelloxx".strip("x")    # "Hello" (strips any chars in the set)

# Transform
s.replace("World", "Python")   # "  Hello, Python!  "
s.strip().split(", ")          # ["Hello", "World!"]
"a-b-c".split("-")            # ["a", "b", "c"]
"a-b-c".split("-", 1)         # ["a", "b-c"]  (maxsplit=1)
"hello".split()                # ["hello"] (splits on whitespace by default)
"  a  b  ".split()             # ["a", "b"] (multiple whitespace collapsed)
", ".join(["a", "b", "c"])     # "a, b, c"
"hello".center(11, "*")        # "***hello***"
"42".zfill(5)                  # "00042"
"hello".ljust(10, ".")         # "hello....."
"hello".rjust(10, ".")         # ".....hello"

# Test
"hello123".isalnum()       # True  (all alphanumeric)
"hello".isalpha()          # True  (all alphabetic)
"123".isdigit()            # True  (all digits)
"123".isnumeric()          # True  (includes unicode numerals like ½)
"  ".isspace()             # True  (all whitespace)
"Hello".istitle()          # True  (title case)
"HELLO".isupper()          # True
"hello".islower()          # True
"hello".isascii()          # True  (Python 3.7+)
"x = 1".isidentifier()    # False (not a valid Python identifier)
```

### String Formatting

```python
name = "Alice"
age = 30

# f-strings (Python 3.6+) — fastest and most readable
f"Name: {name}, Age: {age}"
f"Pi: {3.14159:.2f}"                # "Pi: 3.14"
f"Binary: {255:08b}"                # "Binary: 11111111"
f"Hex: {255:#06x}"                  # "Hex: 0x00ff"
f"{'centered':^20}"                 # "      centered      "
f"{'left':<10}|{'right':>10}"       # "left      |     right"
f"{1000000:,}"                      # "1,000,000"
f"{0.5:.0%}"                        # "50%"

# f-string expressions (Python 3.8+ self-documenting)
f"{name=}"                          # "name='Alice'"
f"{2+2=}"                           # "2+2=4"

# .format() (older but still used)
"{} is {} years old".format(name, age)
"{name} is {age}".format(name=name, age=age)
"{0} {1} {0}".format("hello", "world")  # "hello world hello"

# % formatting (C-style, legacy)
"%s is %d years old" % (name, age)
```

### 🧩 TRICKY OUTPUT: Strings

```python
# String interning (CPython optimization)
a = "hello"
b = "hello"
print(a is b)       # True  (CPython interns small strings)

a = "hello world!"
b = "hello world!"
print(a is b)        # Could be True or False (depends on context, not guaranteed)

# String multiplication
"ha" * 3             # "hahaha"
"ha" * 0             # ""
"ha" * -1            # ""

# Immutability
s = "hello"
# s[0] = "H"         # TypeError: 'str' object does not support item assignment
s = "H" + s[1:]       # "Hello" (creates a new string)
```

---

## 4. Lists

### List Internals

Lists are dynamic arrays (like C++ `vector`). They're contiguous in memory (array of pointers). Appending is amortized O(1) because Python over-allocates.

### Methods

```python
lst = [1, 2, 3, 4, 5]

# Add
lst.append(6)              # [1,2,3,4,5,6]  — O(1) amortized
lst.insert(0, 0)           # [0,1,2,3,4,5,6] — O(n) shifts everything
lst.extend([7, 8])         # [0,1,2,3,4,5,6,7,8] — O(k) for k elements
lst += [9, 10]             # same as extend

# Remove
lst.pop()                  # 10 — removes and returns last element — O(1)
lst.pop(0)                 # 0  — removes index 0 — O(n)
lst.remove(5)              # removes first occurrence of 5 — O(n)
del lst[2]                 # removes index 2
lst.clear()                # removes all elements

# Search
lst = [3, 1, 4, 1, 5]
lst.index(1)               # 1 (index of first occurrence, raises ValueError if not found)
lst.index(1, 2)            # 3 (search starting from index 2)
lst.count(1)               # 2 (number of occurrences)
1 in lst                   # True — O(n) linear search

# Sort
lst.sort()                 # [1, 1, 3, 4, 5] — in-place, returns None!
lst.sort(reverse=True)     # [5, 4, 3, 1, 1]
lst.sort(key=len)          # sort by length (for strings)
lst.sort(key=lambda x: x["age"])  # sort by dict key

sorted(lst)                # returns a NEW sorted list (doesn't mutate)
sorted(lst, key=abs, reverse=True)

lst.reverse()              # in-place reverse
list(reversed(lst))        # returns reversed iterator → list

# Copy
shallow = lst.copy()       # same as lst[:]
shallow = list(lst)        # same
import copy
deep = copy.deepcopy(lst)  # recursive deep copy
```

### List Comprehensions

```python
# Basic
squares = [x**2 for x in range(10)]            # [0, 1, 4, 9, ..., 81]

# With condition
evens = [x for x in range(20) if x % 2 == 0]   # [0, 2, 4, ..., 18]

# Nested
matrix = [[1,2,3], [4,5,6], [7,8,9]]
flat = [x for row in matrix for x in row]       # [1,2,3,4,5,6,7,8,9]

# With transformation
words = ["Hello", "World"]
chars = [c.lower() for word in words for c in word]  # ['h','e','l','l','o','w','o','r','l','d']

# Dict comprehension
{k: v**2 for k, v in {"a": 1, "b": 2}.items()}  # {"a": 1, "b": 4}

# Set comprehension
{x % 3 for x in range(10)}                       # {0, 1, 2}

# Generator expression (lazy — doesn't create list in memory)
gen = (x**2 for x in range(1000000))              # no memory allocated yet
sum(gen)                                           # consumed lazily
```

### Slicing

```python
lst = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

lst[2:5]         # [2, 3, 4]       start:stop (exclusive)
lst[:3]          # [0, 1, 2]       first 3
lst[7:]          # [7, 8, 9]       from index 7
lst[-3:]         # [7, 8, 9]       last 3
lst[::2]         # [0, 2, 4, 6, 8] every 2nd element
lst[::-1]        # [9, 8, ..., 0]  reversed
lst[1:8:2]       # [1, 3, 5, 7]   start:stop:step

# Slice assignment (mutating)
lst[2:5] = [20, 30, 40]  # replace indices 2-4
lst[2:2] = [99]           # insert 99 at index 2
lst[2:5] = []             # delete indices 2-4
```

### 🧩 TRICKY OUTPUT: Lists

```python
# Mutable default argument
def add_item(item, lst=[]):
    lst.append(item)
    return lst

print(add_item(1))   # [1]
print(add_item(2))   # [1, 2]  ← same list! Default is shared across calls
print(add_item(3))   # [1, 2, 3]

# Fix: use None as default
def add_item(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst

# List multiplication creates shared references
matrix = [[0] * 3] * 3
matrix[0][0] = 1
print(matrix)    # [[1, 0, 0], [1, 0, 0], [1, 0, 0]]  ← all rows are the SAME object!

# Fix:
matrix = [[0] * 3 for _ in range(3)]  # each row is a new list

# Shallow copy gotcha
a = [[1, 2], [3, 4]]
b = a.copy()           # shallow copy
b[0][0] = 99
print(a[0][0])          # 99! (inner lists are shared)
```

---

## 5. Tuples

### Immutable Sequences

```python
t = (1, 2, 3)
t2 = 1, 2, 3           # parentheses are optional (it's the comma that makes a tuple)
t3 = (42,)              # single-element tuple (comma is required!)
t4 = ()                 # empty tuple

# Methods (only 2)
t.count(2)              # 1
t.index(2)              # 1

# Unpacking
a, b, c = (1, 2, 3)
first, *rest = (1, 2, 3, 4)  # first=1, rest=[2,3,4]
a, _, c = (1, 2, 3)          # _ convention for unused

# Swap
a, b = b, a                   # Python swap (uses tuple packing/unpacking)

# Named tuples
from collections import namedtuple
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
p.x              # 3
p._asdict()       # {'x': 3, 'y': 4}

# Tuples as dict keys (because they're immutable/hashable)
locations = {(40.7, -74.0): "NYC", (51.5, -0.1): "London"}
# Lists CANNOT be dict keys (unhashable)
```

### List vs Tuple

```
Feature          List             Tuple
───────          ────             ─────
Mutable          Yes              No
Syntax           [1, 2, 3]       (1, 2, 3)
Use case         Collection       Record / fixed structure
Hashable         No               Yes (if contents are hashable)
Dict key         No               Yes
Performance      Slightly slower  Slightly faster (less overhead)
Memory           More             Less (no over-allocation)
```

---

## 6. Dictionaries

### Dict Internals

Python dicts are hash tables. Since Python 3.7, insertion order is guaranteed. Key lookup is O(1) average.

```python
d = {"name": "Alice", "age": 30}
d2 = dict(name="Alice", age=30)       # keyword constructor
d3 = dict([("name", "Alice"), ("age", 30)])  # from pairs
d4 = {x: x**2 for x in range(5)}      # comprehension

# Access
d["name"]                # "Alice"  (KeyError if not found)
d.get("name")            # "Alice"  (None if not found)
d.get("job", "unknown")  # "unknown" (default if not found)

# Add / Update
d["job"] = "engineer"
d.update({"age": 31, "city": "NYC"})
d |= {"country": "US"}              # merge operator (Python 3.9+)
merged = d | {"extra": True}        # creates new dict (Python 3.9+)

# Remove
del d["job"]                         # KeyError if not found
d.pop("city")                        # removes and returns value
d.pop("missing", None)               # returns None instead of KeyError
d.popitem()                          # removes and returns last inserted (k, v)
d.clear()                            # removes all

# Keys, Values, Entries
d.keys()                             # dict_keys(['name', 'age'])
d.values()                           # dict_values(['Alice', 30])
d.items()                            # dict_items([('name', 'Alice'), ('age', 30)])
# These are views — they reflect changes to the dict

# Iteration
for key in d:                        # iterates over keys
for key, value in d.items():         # iterates over key-value pairs
for value in d.values():             # iterates over values

# Membership
"name" in d                          # True (checks keys, O(1))

# setdefault
d.setdefault("scores", []).append(95)  # creates key with default if missing, then appends
```

### 🧩 TRICKY OUTPUT: Dicts

```python
# Dict key types
d = {1: "int", 1.0: "float", True: "bool"}
print(d)            # {1: "bool"}
# Why? 1 == 1.0 == True and hash(1) == hash(1.0) == hash(True)
# So they're all the same key — last value wins

# Dict ordering
d = {}
d["b"] = 2
d["a"] = 1
d["c"] = 3
list(d.keys())      # ['b', 'a', 'c'] — insertion order preserved (Python 3.7+)
```

---

## 7. Sets

```python
s = {1, 2, 3}
s2 = set([1, 2, 2, 3])  # {1, 2, 3} — duplicates removed
empty = set()            # NOT {} (that's an empty dict!)

# Add/Remove
s.add(4)               # {1, 2, 3, 4}
s.discard(99)           # no error if missing
s.remove(99)            # KeyError if missing
s.pop()                 # removes arbitrary element
s.clear()

# Set operations
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

a | b                  # {1,2,3,4,5,6}  union
a & b                  # {3,4}           intersection
a - b                  # {1,2}           difference
a ^ b                  # {1,2,5,6}       symmetric difference

a.issubset(b)          # False
a.issuperset({1,2})    # True
a.isdisjoint({7,8})    # True (no common elements)

# Frozen set (immutable, hashable — can be a dict key or set element)
fs = frozenset([1, 2, 3])
```

---

## 8. Functions

### Definition and Arguments

```python
# Positional, keyword, default
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"

greet("Alice")                # "Hello, Alice!"
greet("Alice", greeting="Hi") # "Hi, Alice!"
greet(greeting="Hi", name="Alice")  # keyword order doesn't matter

# *args (variable positional) and **kwargs (variable keyword)
def func(*args, **kwargs):
    print(args)     # tuple of positional args
    print(kwargs)   # dict of keyword args

func(1, 2, 3, x=4, y=5)
# args = (1, 2, 3)
# kwargs = {"x": 4, "y": 5}

# Positional-only (/) and keyword-only (*) parameters (Python 3.8+)
def f(pos_only, /, normal, *, kw_only):
    pass

f(1, 2, kw_only=3)       # OK
f(1, normal=2, kw_only=3) # OK
f(pos_only=1, ...)        # TypeError! pos_only is positional-only

# Lambda (anonymous function)
square = lambda x: x ** 2
add = lambda x, y: x + y
sorted(items, key=lambda x: x["age"])
```

### First-Class Functions

```python
# Functions are objects — can be passed, returned, stored
def apply(func, value):
    return func(value)

apply(lambda x: x * 2, 5)   # 10

# Closures
def make_multiplier(n):
    def multiplier(x):
        return x * n       # n is captured from outer scope
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)
double(5)    # 10
triple(5)    # 15
```

### Decorators

```python
# A decorator is a function that wraps another function
def timer(func):
    import time
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time() - start:.4f}s")
        return result
    return wrapper

@timer                    # syntactic sugar for: slow_function = timer(slow_function)
def slow_function():
    import time
    time.sleep(1)

# Decorator with arguments
def repeat(n):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def say_hello():
    print("hello")
# Prints "hello" 3 times

# Preserve function metadata
from functools import wraps

def my_decorator(func):
    @wraps(func)           # preserves __name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

### 🧩 TRICKY OUTPUT: Functions

```python
# Late binding closures
functions = []
for i in range(3):
    functions.append(lambda: i)

print([f() for f in functions])   # [2, 2, 2]  (all capture the same `i`)
# Fix: default argument captures current value
functions = [lambda i=i: i for i in range(3)]
print([f() for f in functions])   # [0, 1, 2]

# Mutable default arguments (covered above)
# nonlocal keyword
def outer():
    x = 10
    def inner():
        nonlocal x       # without this, x would be a new local variable
        x = 20
    inner()
    return x
outer()                   # 20
```

---

## 9. Classes and OOP

```python
class Animal:
    species_count = 0          # class variable (shared by all instances)

    def __init__(self, name, sound):
        self.name = name       # instance variable
        self._sound = sound    # convention: "private" (not enforced)
        self.__secret = 42     # name-mangled to _Animal__secret
        Animal.species_count += 1

    def speak(self):           # instance method
        return f"{self.name} says {self._sound}"

    @classmethod
    def create(cls, name):     # class method (receives class, not instance)
        return cls(name, "...")

    @staticmethod
    def is_animal(obj):        # static method (no self or cls)
        return isinstance(obj, Animal)

    @property
    def sound(self):           # getter
        return self._sound

    @sound.setter
    def sound(self, value):    # setter
        if not value:
            raise ValueError("Sound cannot be empty")
        self._sound = value

    def __str__(self):         # called by str() and print()
        return f"Animal({self.name})"

    def __repr__(self):        # called by repr() and in interactive console
        return f"Animal(name={self.name!r}, sound={self._sound!r})"

    def __eq__(self, other):   # equality operator
        return isinstance(other, Animal) and self.name == other.name

    def __hash__(self):        # required if you define __eq__
        return hash(self.name)

    def __len__(self):         # len(animal)
        return len(self.name)


class Dog(Animal):
    def __init__(self, name):
        super().__init__(name, "Woof")

    def speak(self):           # method override
        return f"{super().speak()}! *tail wag*"
```

### Dunder (Magic) Methods

```python
# Numeric
__add__(self, other)     # self + other
__sub__                  # self - other
__mul__                  # self * other
__truediv__              # self / other
__floordiv__             # self // other
__mod__                  # self % other
__pow__                  # self ** other
__neg__                  # -self
__abs__                  # abs(self)

# Comparison
__lt__, __le__, __gt__, __ge__, __eq__, __ne__

# Container
__len__                  # len(self)
__getitem__(self, key)   # self[key]
__setitem__(self, k, v)  # self[key] = value
__delitem__(self, key)   # del self[key]
__contains__(self, item) # item in self
__iter__                 # for x in self
__next__                 # next(self)

# Callable
__call__(self, *args)    # self()  — makes instances callable

# Context manager
__enter__, __exit__      # with self: ...

# String
__str__                  # str(self), print(self)
__repr__                 # repr(self), interactive console
__format__               # f"{self:spec}"
```

### Multiple Inheritance and MRO

```python
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

D().method()           # "B"
print(D.__mro__)       # (D, B, C, A, object) — C3 linearization
# Python uses C3 linearization (MRO) to resolve method lookup order
```

### dataclasses

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int
    email: str = ""
    tags: list = field(default_factory=list)  # mutable default
    _id: int = field(init=False, repr=False)  # excluded from __init__ and __repr__

    def __post_init__(self):
        self._id = hash(self.name)

# Auto-generates: __init__, __repr__, __eq__
user = User("Alice", 30)
print(user)   # User(name='Alice', age=30, email='', tags=[])

@dataclass(frozen=True)    # immutable (like a named tuple but with types)
class Point:
    x: float
    y: float
```

---

## 10. Iterators and Generators

```python
# Iterator protocol: __iter__() + __next__()
class Countdown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

list(Countdown(5))   # [5, 4, 3, 2, 1]

# Generator function (simpler syntax for iterators)
def countdown(n):
    while n > 0:
        yield n        # pauses here, resumes on next()
        n -= 1

list(countdown(5))    # [5, 4, 3, 2, 1]

# Generator expression
squares = (x**2 for x in range(1000000))  # lazy — no memory for 1M items
next(squares)   # 0
next(squares)   # 1

# yield from (delegate to sub-generator)
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item

list(flatten([1, [2, [3, 4], 5], 6]))  # [1, 2, 3, 4, 5, 6]
```

---

## 11. Context Managers

```python
# Using with statement
with open("file.txt", "r") as f:
    content = f.read()
# f is automatically closed, even if an exception occurs

# Custom context manager (class-based)
class Timer:
    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.elapsed = time.time() - self.start
        print(f"Elapsed: {self.elapsed:.4f}s")
        return False     # don't suppress exceptions

with Timer() as t:
    time.sleep(1)
# Prints: "Elapsed: 1.0012s"

# Custom context manager (generator-based — simpler)
from contextlib import contextmanager

@contextmanager
def timer():
    start = time.time()
    yield                # code inside `with` block runs here
    print(f"Elapsed: {time.time() - start:.4f}s")
```

---

## 12. CPython Internals

### The GIL (Global Interpreter Lock)

```
The GIL is a mutex that allows only ONE thread to execute Python bytecode at a time.

Impact:
  • CPU-bound threads do NOT run in parallel (one at a time)
  • I/O-bound threads DO benefit from threading (GIL released during I/O)
  • Use multiprocessing for CPU-bound parallelism (separate processes, no shared GIL)

     Thread 1         Thread 2         Thread 3
     ────────         ────────         ────────
     [execute]        [waiting]        [waiting]
     [release GIL]
                      [acquire GIL]
                      [execute]        [waiting]
                      [release GIL]
                                       [acquire GIL]
                                       [execute]

GIL-free alternatives:
  • multiprocessing module (separate processes)
  • concurrent.futures.ProcessPoolExecutor
  • Python 3.13+ has experimental free-threading (no-GIL) build
```

### Object Interning

```python
# CPython caches small integers (-5 to 256) and short strings
a = 256
b = 256
a is b              # True (same object, interned)

a = 257
b = 257
a is b              # False (not interned — different objects)
# But in some contexts (same compilation unit), CPython may optimize

# String interning
a = "hello"
b = "hello"
a is b              # True (interned at compile time)

a = "hello world"
b = "hello world"
a is b              # May or may not be True (implementation-dependent)
```

### Reference Counting + Generational GC

```python
import sys
a = [1, 2, 3]
sys.getrefcount(a)    # 2 (one for `a`, one for the getrefcount argument)

# When refcount drops to 0, object is immediately freed
# Generational GC handles circular references
import gc
gc.collect()          # manually trigger garbage collection
gc.get_stats()        # generation 0, 1, 2 statistics
```

---

## 13. Error Handling

```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"Error: {e}")
except (TypeError, ValueError) as e:
    print(f"Type or Value error: {e}")
except Exception as e:             # catch-all (but not KeyboardInterrupt, SystemExit)
    print(f"Unexpected: {e}")
else:
    print("No error occurred")     # runs only if no exception
finally:
    print("Always runs")           # cleanup

# Raise
raise ValueError("invalid input")
raise ValueError("bad") from original_error    # chained exceptions

# Custom exception
class AppError(Exception):
    def __init__(self, message, code):
        super().__init__(message)
        self.code = code

# EAFP vs LBYL
# LBYL (Look Before You Leap) — JS/Java style
if key in dictionary:
    value = dictionary[key]

# EAFP (Easier to Ask Forgiveness than Permission) — Pythonic
try:
    value = dictionary[key]
except KeyError:
    value = default
```

---

## 14. Modules and Imports

```python
# Import styles
import math                    # math.sqrt(16)
from math import sqrt, pi     # sqrt(16), pi
from math import *             # import everything (avoid in production)
import math as m               # m.sqrt(16)
from collections import defaultdict as dd

# Relative imports (within a package)
from . import sibling_module
from .. import parent_module
from .utils import helper

# __name__ guard
if __name__ == "__main__":
    main()
# This code only runs when the file is executed directly, not when imported

# Important standard library modules
import os, sys, json, re, datetime, collections
import itertools, functools, operator
import pathlib, shutil, tempfile
import logging, argparse, unittest
import threading, multiprocessing, asyncio
import hashlib, secrets, hmac
import typing, dataclasses, abc, enum
```

### collections Module

```python
from collections import Counter, defaultdict, OrderedDict, deque, namedtuple, ChainMap

# Counter
Counter("abracadabra")     # Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
Counter([1,1,2,3,3,3]).most_common(2)  # [(3, 3), (1, 2)]

# defaultdict
dd = defaultdict(list)
dd["key"].append(1)        # no KeyError — auto-creates empty list
dd = defaultdict(int)
dd["count"] += 1           # auto-creates 0

# deque (double-ended queue — O(1) append/pop from both ends)
dq = deque([1, 2, 3])
dq.appendleft(0)           # [0, 1, 2, 3]
dq.popleft()               # 0
dq.rotate(1)               # [3, 1, 2] — rotate right
dq.rotate(-1)              # [1, 2, 3] — rotate left
```

### itertools Module

```python
from itertools import chain, product, permutations, combinations, groupby, count, cycle, repeat, islice, accumulate, zip_longest

chain([1,2], [3,4])              # 1,2,3,4  — flatten iterables
product([1,2], [3,4])            # (1,3),(1,4),(2,3),(2,4) — cartesian product
permutations([1,2,3], 2)         # (1,2),(1,3),(2,1),(2,3),(3,1),(3,2)
combinations([1,2,3], 2)         # (1,2),(1,3),(2,3)
accumulate([1,2,3,4])            # 1,3,6,10 — running sum
groupby(sorted(data), key=func)  # group by key function
islice(count(), 5)               # 0,1,2,3,4 — slice an infinite iterator
zip_longest([1,2], [3,4,5], fillvalue=0)  # (1,3),(2,4),(0,5)
```

---

## 15. Async/Await

```python
import asyncio

async def fetch_data(url):
    await asyncio.sleep(1)     # simulates I/O
    return f"Data from {url}"

async def main():
    # Sequential (slow — 3 seconds)
    r1 = await fetch_data("url1")
    r2 = await fetch_data("url2")
    r3 = await fetch_data("url3")

    # Concurrent (fast — 1 second)
    results = await asyncio.gather(
        fetch_data("url1"),
        fetch_data("url2"),
        fetch_data("url3"),
    )

asyncio.run(main())

# Async generator
async def stream():
    for i in range(5):
        await asyncio.sleep(0.1)
        yield i

async def consume():
    async for item in stream():
        print(item)

# Async context manager
async with aiohttp.ClientSession() as session:
    async with session.get(url) as response:
        data = await response.json()
```

---

## 16. Type Hints

```python
from typing import (
    List, Dict, Set, Tuple, Optional, Union,
    Any, Callable, Literal, TypeVar, Generic,
    TypedDict, Protocol, Final, ClassVar
)

# Basic
def greet(name: str) -> str:
    return f"Hello, {name}"

# Collections (Python 3.9+: use lowercase list, dict, set, tuple)
def process(items: list[int]) -> dict[str, int]:
    return {"sum": sum(items)}

# Optional (value or None)
def find(id: int) -> str | None:       # Python 3.10+
    pass
def find(id: int) -> Optional[str]:    # older syntax
    pass

# Union
def process(x: int | str) -> str:      # Python 3.10+
    pass

# Callable
def apply(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

# TypeVar (generics)
T = TypeVar("T")
def first(items: list[T]) -> T:
    return items[0]

# TypedDict
class UserDict(TypedDict):
    name: str
    age: int
    email: str

# Protocol (structural typing — like TS interfaces)
class Printable(Protocol):
    def __str__(self) -> str: ...
```

---

## 17. Walrus Operator and Modern Syntax

```python
# Walrus operator := (Python 3.8) — assign and use in one expression
if (n := len(data)) > 10:
    print(f"Too long: {n}")

# In list comprehension
results = [y for x in data if (y := expensive(x)) > threshold]

# In while loops
while chunk := file.read(8192):
    process(chunk)

# Match statement (Python 3.10) — structural pattern matching
match command:
    case "quit":
        sys.exit()
    case "hello" | "hi":
        print("Hello!")
    case {"action": "move", "direction": dir}:
        move(dir)
    case [x, y]:
        point(x, y)
    case _:
        print("Unknown")
```

## 18. Regular Expressions (re module)
 
```python
import re
 
# Basic usage
re.search(r"\d+", "abc 123 def")         # <Match: '123'> (first match)
re.match(r"\d+", "123 abc")              # <Match: '123'> (match at START only)
re.fullmatch(r"\d+", "123")             # <Match: '123'> (entire string must match)
re.findall(r"\d+", "a1 b22 c333")       # ['1', '22', '333'] (all matches as list)
re.finditer(r"\d+", "a1 b22 c333")      # iterator of Match objects
 
re.sub(r"\d+", "X", "a1 b22 c333")       # "aX bX cX" (replace all)
re.sub(r"(\w+) (\w+)", r"\2 \1", "Hello World")  # "World Hello"
re.split(r"[,;\s]+", "a, b; c d")        # ['a', 'b', 'c', 'd']
 
# Groups
m = re.search(r"(\d{4})-(\d{2})-(\d{2})", "Date: 2024-01-15")
m.group(0)    # "2024-01-15" (full match)
m.group(1)    # "2024"
m.group(2)    # "01"
m.groups()    # ('2024', '01', '15')
 
# Named groups
m = re.search(r"(?P<year>\d{4})-(?P<month>\d{2})", "2024-01-15")
m.group("year")    # "2024"
m.groupdict()      # {'year': '2024', 'month': '01'}
 
# Compiled patterns (faster when reused)
pattern = re.compile(r"\b\w{5}\b", re.IGNORECASE)
pattern.findall("Hello World Python")    # ['Hello', 'World']
 
# Flags
re.IGNORECASE   # re.I — case-insensitive
re.MULTILINE    # re.M — ^ and $ match line boundaries
re.DOTALL       # re.S — . matches newlines
re.VERBOSE      # re.X — allow comments and whitespace in pattern
 
# Verbose pattern (readable regex)
pattern = re.compile(r"""
    (?P<year>\d{4})    # year
    -                  # separator
    (?P<month>\d{2})   # month
    -                  # separator
    (?P<day>\d{2})     # day
""", re.VERBOSE)
 
# Common gotcha: use raw strings r"..." to avoid double-escaping
re.search("\\d+", "123")    # works but ugly
re.search(r"\d+", "123")    # cleaner — r-string treats \ as literal
```
 
---
 
## 19. File I/O
 
```python
# Reading
with open("file.txt", "r", encoding="utf-8") as f:
    content = f.read()          # entire file as string
    # or
    lines = f.readlines()       # list of lines (including \n)
    # or
    for line in f:               # iterate line by line (memory efficient)
        process(line.strip())
 
# Writing
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("Hello\n")
    f.writelines(["line1\n", "line2\n"])
 
# Appending
with open("log.txt", "a") as f:
    f.write("new entry\n")
 
# Binary mode
with open("image.png", "rb") as f:
    data = f.read()              # bytes object
 
with open("copy.png", "wb") as f:
    f.write(data)
 
# File modes
# "r"   read (default, file must exist)
# "w"   write (creates or truncates)
# "a"   append (creates if doesn't exist)
# "x"   exclusive create (fails if file exists)
# "b"   binary mode (rb, wb, ab)
# "t"   text mode (default, rt, wt)
# "+"   read and write (r+, w+, a+)
 
# pathlib (modern, object-oriented file paths)
from pathlib import Path
 
path = Path("data") / "file.txt"     # OS-appropriate path joining
path.exists()                         # True/False
path.is_file()                        # True/False
path.is_dir()                         # True/False
path.read_text(encoding="utf-8")      # read entire file
path.write_text("hello", encoding="utf-8")  # write entire file
path.read_bytes()                     # read as bytes
path.stem                             # "file" (name without extension)
path.suffix                           # ".txt"
path.parent                           # Path("data")
path.name                             # "file.txt"
list(Path(".").glob("**/*.py"))       # recursive glob
path.mkdir(parents=True, exist_ok=True)
 
# Temporary files
import tempfile
with tempfile.NamedTemporaryFile(mode="w", suffix=".txt", delete=False) as f:
    f.write("temp data")
    print(f.name)           # path to temp file
```
 
---
 
## 20. Built-in Functions (The Ones You Must Know)
 
```python
# Iteration
for i, val in enumerate(["a", "b", "c"]):       # i=0,1,2 val="a","b","c"
    print(i, val)
 
for a, b in zip([1,2,3], ["x","y","z"]):         # pairs: (1,"x"), (2,"y"), (3,"z")
    print(a, b)
 
# zip stops at shortest; zip_longest pads
from itertools import zip_longest
list(zip_longest([1,2], [3,4,5], fillvalue=0))   # [(1,3),(2,4),(0,5)]
 
# map, filter (prefer comprehensions, but know these)
list(map(str.upper, ["hello", "world"]))          # ["HELLO", "WORLD"]
list(map(lambda x: x**2, [1,2,3]))               # [1, 4, 9]
list(filter(lambda x: x > 2, [1,2,3,4]))         # [3, 4]
list(filter(None, [0, 1, "", "hi", None, True]))  # [1, "hi", True] — removes falsy
 
# Reduction
from functools import reduce
reduce(lambda acc, x: acc + x, [1,2,3,4], 0)     # 10
 
# any / all
any([False, False, True])       # True  (at least one truthy)
all([True, True, True])         # True  (all truthy)
any([])                          # False
all([])                          # True  (vacuous truth!)
 
# Practical: check if any items match a condition
any(x > 10 for x in numbers)
all(isinstance(x, int) for x in items)
 
# sorted (returns new list)
sorted([3, 1, 2])                               # [1, 2, 3]
sorted([3, 1, 2], reverse=True)                  # [3, 2, 1]
sorted(users, key=lambda u: u["age"])            # sort by key
sorted("hello")                                   # ['e', 'h', 'l', 'l', 'o']
 
# reversed (returns iterator)
list(reversed([1, 2, 3]))                        # [3, 2, 1]
 
# min, max
min(3, 1, 2)                    # 1
max(users, key=lambda u: u["age"])  # user with highest age
min([], default=0)              # 0 (without default: ValueError)
 
# sum
sum([1, 2, 3])                  # 6
sum([1, 2, 3], 10)              # 16 (start value)
 
# len, abs, round, pow
len([1, 2, 3])                  # 3
abs(-5)                         # 5
round(3.14159, 2)               # 3.14
pow(2, 10)                      # 1024
pow(2, 10, 1000)                # 24 (2^10 % 1000 — modular exponentiation)
 
# input/output
name = input("Enter name: ")    # reads string from stdin
print("Hello", name, sep=", ", end="!\n")  # sep between args, end after all
 
# repr vs str
repr("hello")                   # "'hello'" (with quotes — for debugging)
str("hello")                    # "hello" (human-readable)
 
# hash, id
hash("hello")                   # integer hash value
id(obj)                         # memory address (CPython)
 
# isinstance, issubclass (covered, but worth repeating)
isinstance(42, (int, float))    # True (check multiple types with tuple)
 
# callable
callable(print)                 # True
callable(42)                    # False
 
# dir (list attributes)
dir([])                         # all list methods
dir(str)                        # all string methods
 
# vars (instance __dict__)
class Foo:
    def __init__(self): self.x = 1
vars(Foo())                     # {'x': 1}
 
# getattr, setattr, hasattr, delattr
getattr(obj, "name", "default")    # obj.name or "default"
setattr(obj, "name", "Alice")      # obj.name = "Alice"
hasattr(obj, "name")               # True/False
delattr(obj, "name")               # del obj.name
```
 
---
 
## 21. Ternary Expression and Chained Comparisons
 
```python
# Ternary (conditional expression)
result = "even" if x % 2 == 0 else "odd"
 
# Nested ternary (avoid — hard to read)
grade = "A" if score >= 90 else "B" if score >= 80 else "C" if score >= 70 else "F"
 
# Chained comparisons (Python-specific — beautiful)
1 < x < 10           # True if x is between 1 and 10 (exclusive)
0 <= score <= 100     # True if score is in [0, 100]
a < b < c < d         # True if strictly ascending
a == b == c           # True if all equal
 
# This is NOT the same as JS/C:
# JS: 1 < 2 < 3 → (1 < 2) < 3 → true < 3 → 1 < 3 → true (accident!)
# Python: 1 < 2 < 3 → (1 < 2) and (2 < 3) → True (intentional!)
```
 
---
 
## 22. Copy: Shallow vs Deep
 
```python
import copy
 
# Assignment (no copy — same object)
a = [1, [2, 3]]
b = a
b[0] = 99
print(a[0])        # 99 (same object)
 
# Shallow copy (new outer container, shared inner objects)
a = [1, [2, 3]]
b = a.copy()        # or list(a), or a[:], or copy.copy(a)
b[0] = 99
print(a[0])         # 1 (outer is independent)
b[1][0] = 99
print(a[1][0])      # 99 (inner list is shared!)
 
# Deep copy (fully independent at all levels)
a = [1, [2, 3]]
b = copy.deepcopy(a)
b[1][0] = 99
print(a[1][0])      # 2 (fully independent)
 
# Works for all types
original = {"users": [{"name": "Alice"}], "count": 1}
clone = copy.deepcopy(original)
```
 
---
 
## 23. String Encoding: bytes vs str
 
```python
# str = text (Unicode), bytes = raw binary
text = "Hello, 世界"         # str
binary = b"Hello"            # bytes (ASCII only in literals)
 
# Encode (str → bytes)
text.encode("utf-8")          # b'Hello, \xe4\xb8\x96\xe7\x95\x8c'
text.encode("ascii")          # UnicodeEncodeError (can't encode 世界 in ASCII)
text.encode("ascii", errors="replace")  # b'Hello, ??'
text.encode("ascii", errors="ignore")   # b'Hello, '
 
# Decode (bytes → str)
b"Hello".decode("utf-8")     # "Hello"
b'\xe4\xb8\x96'.decode("utf-8")  # "世"
 
# Reading files in binary vs text
with open("file.txt", "r", encoding="utf-8") as f:
    text = f.read()            # str
 
with open("file.bin", "rb") as f:
    data = f.read()            # bytes
```
 
---
 
## 24. Enums
 
```python
from enum import Enum, auto, IntEnum, Flag
 
class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3
 
Color.RED                 # <Color.RED: 1>
Color.RED.name            # "RED"
Color.RED.value           # 1
Color(1)                  # Color.RED (lookup by value)
Color["RED"]              # Color.RED (lookup by name)
list(Color)               # [Color.RED, Color.GREEN, Color.BLUE]
 
# auto() assigns values automatically
class Direction(Enum):
    NORTH = auto()        # 1
    SOUTH = auto()        # 2
    EAST = auto()         # 3
    WEST = auto()         # 4
 
# String enum
class Status(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
 
Status.ACTIVE == "active"  # True (because it's also a str)
 
# IntEnum (can be compared to ints)
class Priority(IntEnum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
 
Priority.HIGH > 2          # True
 
# Flag (bitmask)
class Permission(Flag):
    READ = auto()
    WRITE = auto()
    EXECUTE = auto()
    ALL = READ | WRITE | EXECUTE
 
perms = Permission.READ | Permission.WRITE
Permission.READ in perms    # True
```
 
---
 
## 25. Abstract Base Classes (abc)
 
```python
from abc import ABC, abstractmethod
 
class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        """Must be implemented by subclasses."""
        pass
 
    @abstractmethod
    def perimeter(self) -> float:
        pass
 
    def description(self) -> str:      # concrete method (inherited as-is)
        return f"{self.__class__.__name__}: area={self.area():.2f}"
 
# Shape()  # TypeError: Can't instantiate abstract class
 
class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
 
    def area(self) -> float:
        return 3.14159 * self.radius ** 2
 
    def perimeter(self) -> float:
        return 2 * 3.14159 * self.radius
 
c = Circle(5)
c.area()            # 78.54
c.description()     # "Circle: area=78.54"
```
 
---
 
## 26. __slots__
 
```python
# By default, objects store attributes in a __dict__ (hash table)
# __slots__ replaces __dict__ with a fixed-size struct — saves memory
 
class Point:
    __slots__ = ("x", "y")    # only these attributes allowed
 
    def __init__(self, x, y):
        self.x = x
        self.y = y
 
p = Point(3, 4)
p.x              # 3
p.z = 5          # AttributeError! 'z' not in __slots__
p.__dict__       # AttributeError! No __dict__ exists
 
# Memory savings: ~40-50% for small objects with many instances
# Speed: slightly faster attribute access
 
# Gotcha: subclass without __slots__ gets __dict__ back
class Point3D(Point):
    pass               # has __dict__ again (can add arbitrary attrs)
 
class Point3D(Point):
    __slots__ = ("z",)  # extend slots properly
```
 
---
 
## 27. Descriptors
 
```python
# Descriptors control attribute access at the class level
# A descriptor is any object defining __get__, __set__, or __delete__
 
class Validated:
    def __init__(self, min_value=None, max_value=None):
        self.min_value = min_value
        self.max_value = max_value
 
    def __set_name__(self, owner, name):
        self.name = name              # auto-receives attribute name
 
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, f"_{self.name}", None)
 
    def __set__(self, obj, value):
        if self.min_value is not None and value < self.min_value:
            raise ValueError(f"{self.name} must be >= {self.min_value}")
        if self.max_value is not None and value > self.max_value:
            raise ValueError(f"{self.name} must be <= {self.max_value}")
        setattr(obj, f"_{self.name}", value)
 
class User:
    age = Validated(min_value=0, max_value=150)
    score = Validated(min_value=0, max_value=100)
 
    def __init__(self, name, age, score):
        self.name = name
        self.age = age       # goes through Validated.__set__
        self.score = score
 
user = User("Alice", 30, 95)
user.age = -1   # ValueError: age must be >= 0
 
# @property, @classmethod, @staticmethod are ALL implemented as descriptors internally
```
 
---
 
## 28. Metaclasses
 
```python
# A metaclass is a class whose instances are classes
# type is the default metaclass
 
# Class creation process:
# 1. Python sees `class Foo: ...`
# 2. Collects the class body into a namespace dict
# 3. Calls the metaclass: type("Foo", (bases,), namespace)
 
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        # Called when the CLASS is created (not the instance)
        if "required_method" not in namespace:
            raise TypeError(f"{name} must define required_method")
        return super().__new__(mcs, name, bases, namespace)
 
    def __init__(cls, name, bases, namespace):
        super().__init__(name, bases, namespace)
        cls.registry = []  # add class-level attribute
 
class Plugin(metaclass=Meta):
    def required_method(self):
        pass
 
# class BadPlugin(metaclass=Meta):
#     pass
# TypeError: BadPlugin must define required_method
 
# Simpler alternative: __init_subclass__ (Python 3.6+)
class Base:
    def __init_subclass__(cls, required=False, **kwargs):
        super().__init_subclass__(**kwargs)
        if required and not hasattr(cls, "validate"):
            raise TypeError(f"{cls.__name__} must define validate()")
 
class Child(Base, required=True):
    def validate(self):
        pass
```
 
---
 
## 29. __new__ vs __init__
 
```python
class Singleton:
    _instance = None
 
    def __new__(cls, *args, **kwargs):
        # __new__ creates the instance (before __init__)
        # Must return the instance
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
 
    def __init__(self, value):
        # __init__ initializes the instance (after __new__)
        # Does NOT return anything
        self.value = value
 
a = Singleton(1)
b = Singleton(2)
a is b           # True (same object)
a.value          # 2 (re-initialized by second call)
 
# __new__ is also used for immutable types:
class UpperStr(str):
    def __new__(cls, value):
        return super().__new__(cls, value.upper())
 
UpperStr("hello")   # "HELLO"
# Can't use __init__ because str is immutable — must set value in __new__
```
 
---
 
## 30. functools Module
 
```python
from functools import lru_cache, cache, partial, reduce, wraps, total_ordering
 
# lru_cache — memoization decorator
@lru_cache(maxsize=128)       # cache up to 128 unique argument sets
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
 
fibonacci(100)                  # instant (without cache: impossibly slow)
fibonacci.cache_info()          # CacheInfo(hits=98, misses=101, ...)
fibonacci.cache_clear()         # clear the cache
 
# cache — simpler unlimited cache (Python 3.9+)
@cache
def expensive(x):
    return x ** x
 
# partial — fix some arguments
from functools import partial
 
def power(base, exponent):
    return base ** exponent
 
square = partial(power, exponent=2)
cube = partial(power, exponent=3)
square(5)    # 25
cube(5)      # 125
 
# total_ordering — fill in comparison methods
@total_ordering
class Student:
    def __init__(self, name, grade):
        self.name = name
        self.grade = grade
 
    def __eq__(self, other):
        return self.grade == other.grade
 
    def __lt__(self, other):
        return self.grade < other.grade
 
    # __le__, __gt__, __ge__ are auto-generated from __eq__ and __lt__
 
# singledispatch — function overloading by type
from functools import singledispatch
 
@singledispatch
def process(value):
    raise TypeError(f"Unsupported type: {type(value)}")
 
@process.register(str)
def _(value):
    return value.upper()
 
@process.register(int)
def _(value):
    return value * 2
 
process("hello")    # "HELLO"
process(21)         # 42
process([1, 2])     # TypeError
```
 
---
 
## 31. global and nonlocal Keywords
 
```python
# global — access/modify module-level variable from inside a function
count = 0
 
def increment():
    global count        # without this, `count += 1` creates a LOCAL variable
    count += 1
 
increment()
print(count)            # 1
 
# nonlocal — access/modify enclosing function's variable
def outer():
    x = 10
    def inner():
        nonlocal x      # without this, `x = 20` creates a LOCAL variable
        x = 20
    inner()
    return x            # 20
 
# Without nonlocal:
def outer2():
    x = 10
    def inner():
        x = 20          # creates a NEW local x (shadows outer x)
    inner()
    return x            # 10 (unchanged)
```
 
### 🧩 TRICKY OUTPUT: Scope
 
```python
x = 10
 
def foo():
    print(x)         # UnboundLocalError!
    x = 20           # This makes x local to the ENTIRE function
                     # Python decides scope at compile time, not runtime
 
# Fix: use global, or don't assign to x
def foo_fixed():
    global x
    print(x)         # 10
    x = 20
```
 
---
 
## 32. Exception Groups (Python 3.11)
 
```python
# ExceptionGroup — bundle multiple exceptions together
try:
    raise ExceptionGroup("multiple errors", [
        ValueError("bad value"),
        TypeError("bad type"),
        KeyError("missing key"),
    ])
except* ValueError as eg:       # except* catches a subset
    print(f"Value errors: {eg.exceptions}")
except* TypeError as eg:
    print(f"Type errors: {eg.exceptions}")
# Remaining KeyError propagates up
 
# Use case: asyncio.TaskGroup (Python 3.11)
async with asyncio.TaskGroup() as tg:
    tg.create_task(task1())
    tg.create_task(task2())
    tg.create_task(task3())
# If multiple tasks fail, all exceptions are bundled in an ExceptionGroup
```
 
---
 
## 33. Comprehension Scope (Subtle!)
 
```python
# Comprehensions have their OWN scope (like a function)
x = 10
result = [x for x in range(5)]
print(x)        # 10 (Python 3 — x inside comprehension is scoped)
                # In Python 2, this would print 4 (leaked!)
 
# But they CAN read from enclosing scope
multiplier = 3
result = [x * multiplier for x in range(5)]   # [0, 3, 6, 9, 12]
 
# Walrus operator in comprehension assigns to enclosing scope
results = [y := x * 2 for x in range(5)]
print(y)        # 8 (last value assigned by :=, leaked to enclosing scope!)
```
 
---
 
## 34. is vs == (Deep Dive)
 
```python
# == compares VALUES (calls __eq__)
# is compares IDENTITY (same object in memory)
 
a = [1, 2, 3]
b = [1, 2, 3]
a == b           # True  (same content)
a is b           # False (different objects)
 
a = b            # now both point to same object
a is b           # True
 
# Singletons: always use `is` for None, True, False
x = None
x is None        # ✓ correct
x == None        # ✗ works but bad practice (could be overridden by __eq__)
 
# Integer caching: -5 to 256 are cached singletons
a = 256
b = 256
a is b           # True (same cached object)
 
a = 257
b = 257
a is b           # False (different objects, value > 256)
                 # (may be True in some contexts due to compiler optimization)
 
# String interning
a = "hello"
b = "hello"
a is b           # True (interned at compile time)
 
a = "hello world"
b = "hello world"
a is b           # Implementation-dependent (don't rely on this)
 
# RULE: Use == for value comparison, is for identity (None/True/False/singletons)
```
 
---
 
## 35. 🧩 TRICKY OUTPUT: Python Grand Finale
 
```python
# Tuple with one element
print(type((1)))       # <class 'int'>    (parentheses, not tuple!)
print(type((1,)))      # <class 'tuple'>  (comma makes it a tuple)
 
# Chained assignment
a = b = [1, 2, 3]
a.append(4)
print(b)               # [1, 2, 3, 4]  (same object!)
 
# Augmented assignment with immutables vs mutables
a = (1, 2, 3)
# a += (4, 5)          # creates NEW tuple (tuples are immutable)
# id changes
 
a = [1, 2, 3]
b = a
a += [4, 5]            # extends IN PLACE (lists are mutable)
print(b)               # [1, 2, 3, 4, 5]  (same object, b sees change)
 
# But:
a = [1, 2, 3]
b = a
a = a + [4, 5]         # creates NEW list!
print(b)               # [1, 2, 3]  (b still points to original)
# += mutates, + creates new
 
# Boolean is int
print(True + True + True)   # 3
print(True == 1)             # True
print(True is 1)             # False (different objects in Python 3.8+)
 
# Dictionary unpacking order
d1 = {"a": 1, "b": 2}
d2 = {"b": 3, "c": 4}
merged = {**d1, **d2}
print(merged)          # {"a": 1, "b": 3, "c": 4}  — d2's b wins (last write)
 
# else on loops (runs if loop completes WITHOUT break)
for i in range(5):
    if i == 10:
        break
else:
    print("completed")    # prints! (no break was hit)
 
for i in range(5):
    if i == 3:
        break
else:
    print("completed")    # does NOT print (break was hit)
 
# Mutable in tuple
t = ([1, 2], [3, 4])
t[0].append(99)           # works! The list inside is mutable
print(t)                   # ([1, 2, 99], [3, 4])
# t[0] = [5, 6]           # TypeError! Can't reassign tuple element
```
 
