
---

# 1. What is a Lambda Function?

- Definition
    
- Why it exists
    
- Anonymous functions
    
- Difference between `def` and `lambda`
    
- When Python creates the function object
    

Example

```python
square = lambda x: x * x

print(square(5))
```

---

# 2. Lambda Syntax

General syntax

```python
lambda parameters: expression
```

Understand every part:

- `lambda` keyword
    
- Parameters
    
- Expression
    
- Return value (implicit)
    

Equivalent with `def`

```python
def add(a, b):
    return a + b

add = lambda a, b: a + b
```

---

# 3. Lambda Always Returns

Unlike `def`

```python
lambda x: x + 10
```

is equivalent to

```python
def func(x):
    return x + 10
```

Things to know

- No explicit `return`
    
- Last expression automatically returned
    

---

# 4. Single Expression Rule

A lambda can contain **only one expression**.

Allowed

```python
lambda x: x * 2
```

Not allowed

```python
lambda x:
    print(x)
    return x
```

Understand why Python restricts this.

---

# 5. Parameters in Lambda

You should know all kinds.

### Single parameter

```python
lambda x: x * 2
```

---

### Multiple parameters

```python
lambda a, b: a + b
```

---

### Default arguments

```python
lambda x, y=10: x + y
```

---

### Variable positional arguments

```python
lambda *args: sum(args)
```

---

### Variable keyword arguments

```python
lambda **kwargs: kwargs["name"]
```

---

### Combination

```python
lambda x, *args, **kwargs: x
```

---

# 6. Calling Lambda Immediately (IIFE)

```python
print((lambda x: x + 5)(10))
```

Understand

- Function creation
    
- Immediate invocation
    

---

# 7. Assigning Lambda

```python
cube = lambda x: x ** 3
```

Understand

- Lambda is just another function object.
    

---

# 8. Passing Lambda as an Argument

One of the biggest uses.

```python
def apply(func, value):
    return func(value)

print(apply(lambda x: x * 5, 4))
```

---

# 9. Returning Lambda

Functions can return lambdas.

```python
def multiplier(n):
    return lambda x: x * n
```

---

# 10. Lambda Closures

Very important.

```python
def power(n):
    return lambda x: x ** n

square = power(2)
cube = power(3)
```

Understand

- Free variables
    
- Closure
    
- Remembering outer variables
    

---

# 11. Lambda and LEGB Rule

Understand how lambda accesses variables.

```python
x = 100

f = lambda y: x + y
```

What happens if

```python
x = 500
```

before calling?

Understand late binding.

---

# 12. Late Binding Problem

Classic interview question.

```python
funcs = []

for i in range(5):
    funcs.append(lambda: i)

for f in funcs:
    print(f())
```

Output

```
4
4
4
4
4
```

Why?

How to fix

```python
funcs.append(lambda i=i: i)
```

---

# 13. Lambda with `map()`

Very common.

```python
numbers = [1,2,3]

result = list(map(lambda x: x*x, numbers))
```

Know

- How `map()` works
    
- Lazy evaluation
    
- Iterator
    

---

# 14. Lambda with `filter()`

```python
numbers = [1,2,3,4,5]

even = list(filter(lambda x: x % 2 == 0, numbers))
```

Understand

- Predicate functions
    
- Boolean return values
    

---

# 15. Lambda with `reduce()`

```python
from functools import reduce

total = reduce(lambda a, b: a + b, [1,2,3,4])
```

Understand

- Accumulator
    
- Initial value
    
- Reduction process
    

---

# 16. Lambda with `sorted()`

Most important real-world usage.

Sort by

### Length

```python
words = ["apple", "kiwi", "banana"]

sorted(words, key=lambda x: len(x))
```

---

### Last character

```python
sorted(words, key=lambda x: x[-1])
```

---

### Multiple keys

```python
students = [
    ("Alice", 90),
    ("Bob", 80),
    ("Charlie", 90)
]

sorted(students, key=lambda x: (-x[1], x[0]))
```

---

# 17. Lambda with `list.sort()`

```python
numbers.sort(key=lambda x: abs(x))
```

Difference between

- `sort()`
    
- `sorted()`
    

---

# 18. Lambda with Dictionaries

Sorting dictionary items

```python
data = {
    "a": 5,
    "b": 2,
    "c": 9
}

sorted(data.items(), key=lambda item: item[1])
```

---

# 19. Lambda with Tuples

```python
pairs = [(2,3), (1,9), (5,0)]

sorted(pairs, key=lambda x: x[1])
```

---

# 20. Lambda with Lists

Nested lists

```python
matrix = [
    [2,9],
    [1,5],
    [3,4]
]

sorted(matrix, key=lambda row: row[1])
```

---

# 21. Lambda with Sets

Possible after conversion

```python
s = {5,1,9,2}

sorted(s, key=lambda x: -x)
```

---

# 22. Lambda with Objects

```python
class Student:
    def __init__(self, name, age):
        self.name = name
        self.age = age

students.sort(key=lambda s: s.age)
```

Very common in OOP.

---

# 23. Lambda with Strings

Sort by lowercase

```python
words.sort(key=lambda x: x.lower())
```

---

# 24. Lambda inside List Comprehension

```python
funcs = [lambda x: x+i for i in range(5)]
```

Understand why this has the late binding issue.

---

# 25. Nested Lambda

```python
add = lambda x: lambda y: x + y
```

Equivalent to

```python
def add(x):
    def inner(y):
        return x + y
    return inner
```

---

# 26. Higher-Order Functions

Lambda is heavily used with higher-order functions.

Know

- Functions as objects
    
- Passing functions
    
- Returning functions
    

---

# 27. Functional Programming

Understand concepts

- Pure functions
    
- Immutability
    
- Composition
    
- map
    
- filter
    
- reduce
    

---

# 28. Lambda vs `def`

Know the comparison:

|Feature|Lambda|def|
|---|---|---|
|Name|Anonymous (unless assigned)|Named|
|Multiple statements|❌|✅|
|`return`|Implicit|Explicit|
|Docstring|❌|✅|
|Type hints|Limited|Fully supported|
|Readability|Good for simple cases|Better for complex logic|

---

# 29. Performance

Understand that

```python
lambda
```

is **not faster** than

```python
def
```

Both create function objects.

---

# 30. Best Practices

Use lambda when

- Short
    
- One expression
    
- Used once
    
- Passed as an argument
    
- Sorting
    
- Mapping
    
- Filtering
    

Avoid lambda when

- Complex logic
    
- Multiple conditions
    
- Loops
    
- Exception handling
    
- Multiple return paths
    
- You need documentation or type annotations
    

---

# 31. Interview Questions

Be prepared to answer questions like:

- Why is lambda called an anonymous function?
    
- Why can't lambda have multiple statements?
    
- What is the difference between `def` and `lambda`?
    
- Can lambda access global variables?
    
- Can lambda modify outer variables?
    
- What is the late binding problem?
    
- Why does `lambda i=i: i` fix the late binding issue?
    
- Can lambda be recursive?
    
- When should you avoid lambda?
    
- Why is `key=` often used with lambda in sorting?
    
- Is lambda faster than `def`?
    
- Can lambda have `*args` and `**kwargs`?
    
- Can lambda capture variables from an enclosing scope (closures)?
    

---

## Mastery Checklist

If you understand all of the following, you can consider yourself proficient with lambda functions:

- ✅ Syntax and implicit return
    
- ✅ Single-expression limitation
    
- ✅ Parameter types (`*args`, `**kwargs`, defaults)
    
- ✅ Function objects and anonymous functions
    
- ✅ Passing and returning lambdas
    
- ✅ Closures and variable capture
    
- ✅ LEGB rule and late binding
    
- ✅ `map()`, `filter()`, and `reduce()`
    
- ✅ Sorting with `key=` and lambda
    
- ✅ Using lambda with lists, tuples, dictionaries, sets, strings, and custom objects
    
- ✅ Nested lambdas and higher-order functions
    
- ✅ Functional programming concepts
    
- ✅ Performance characteristics
    
- ✅ Best practices and readability trade-offs
    
- ✅ Common interview questions and edge cases
    

Mastering these topics will cover nearly all practical uses of lambda functions you'll encounter in Python development and technical interviews.