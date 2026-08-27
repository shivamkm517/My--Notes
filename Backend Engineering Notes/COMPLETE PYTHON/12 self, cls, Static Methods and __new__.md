---
tags: [python, oop, interview-prep, programming, computer-science]
created: 2026-07-08
aliases: [self vs cls, Static Methods, __new__ vs __init__, Python Class Internals]
---

# self, cls, Static Methods, `__new__` & Related Internals — Deep Dive

> [!info] Why This Note Matters
> This is one of the **most commonly asked Python interview topics**. Interviewers use it to check whether you understand what's actually happening under the hood when you write a class — not just that you can use one.

---

## 1. `self` — The Instance Reference

### Definition
`self` is the **conventional name** (not a keyword!) for the first parameter of an **instance method** — it refers to the specific object the method was called on.

> [!warning] `self` is NOT a keyword
> Unlike `this` in Java/C++ (which is a language keyword), `self` in Python is just a **strong naming convention**. You could technically name it anything (`this`, `obj`, `me`) and it would still work — but doing so breaks readability and violates PEP 8. Every experienced Python dev expects `self`.

### How It Actually Works — Method Binding

```python
class Dog:
    def __init__(self, name):
        self.name = name

    def bark(self):
        return f"{self.name} says Woof!"


d = Dog("Rex")
d.bark()
```

> [!note] What Really Happens
> `d.bark()` is syntactic sugar for `Dog.bark(d)`. Python automatically passes the instance (`d`) as the first argument. This is called **instance method binding**.

```python
# These two calls are IDENTICAL:
d.bark()
Dog.bark(d)   # explicitly passing the instance yourself
```

### `self` Gives Access to Instance State
```python
class Counter:
    def __init__(self):
        self.count = 0          # instance attribute — belongs to THIS object only

    def increment(self):
        self.count += 1
        return self.count


c1 = Counter()
c2 = Counter()
c1.increment()
c1.increment()
c2.increment()
print(c1.count, c2.count)   # 2 1  -- each instance has its OWN state
```

---

## 2. `cls` — The Class Reference

### Definition
`cls` is the **conventional name** for the first parameter of a **class method** — it refers to the **class itself**, not any particular instance.

```python
class Dog:
    species = "Canis familiaris"   # class variable — shared across ALL instances

    def __init__(self, name):
        self.name = name            # instance variable

    @classmethod
    def get_species(cls):
        return cls.species           # accesses the class, not an instance

    @classmethod
    def create_puppy(cls, name):
        return cls(name)              # cls() == Dog() -- but works correctly even in subclasses!


print(Dog.get_species())          # Canis familiaris
puppy = Dog.create_puppy("Max")
print(puppy.name)                  # Max
```

### Why `cls` Instead of Hardcoding the Class Name?

```python
class Animal:
    @classmethod
    def create(cls, name):
        return cls(name)             # uses cls, NOT "Animal(name)"

class Dog(Animal):
    def __init__(self, name):
        self.name = name
        self.sound = "Woof"


d = Dog.create("Rex")
print(type(d))    # <class '__main__.Dog'> -- correctly creates a Dog, not an Animal!
```

> [!success] Key Insight
> If `create()` had hardcoded `Animal(name)` instead of `cls(name)`, calling `Dog.create("Rex")` would incorrectly return an `Animal` instance instead of a `Dog`. Using `cls` makes the method **inheritance-aware** — this is the classic **"alternative constructor" pattern**.

### Common Real-World Use: Alternative Constructors

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    @classmethod
    def from_birth_year(cls, name, birth_year):
        import datetime
        age = datetime.date.today().year - birth_year
        return cls(name, age)          # calls Person(name, age) via cls


p = Person.from_birth_year("Alice", 1995)
print(p.age)   # 31 (as of 2026)
```

> [!tip] Why This is Useful
> Python only allows **one `__init__`** (no method overloading). Classmethods let you offer **multiple, clearly-named ways to construct an object** (`from_birth_year`, `from_json`, `from_dict`, etc.) without cramming everything into `__init__` with complex optional arguments.

---

## 3. Static Methods — `@staticmethod`

### Definition
A **static method** belongs to the class's namespace but does **not** receive `self` or `cls` automatically. It behaves like a plain function that just happens to live inside a class — for organizational purposes.

```python
class MathUtils:
    @staticmethod
    def add(a, b):
        return a + b

    @staticmethod
    def is_even(n):
        return n % 2 == 0


print(MathUtils.add(3, 4))       # 7 -- called without an instance
print(MathUtils.is_even(10))      # True

m = MathUtils()
print(m.add(3, 4))                 # 7 -- also callable on an instance, still no self passed
```

### When to Use a Static Method
- The method's logic is **related to the class conceptually**, but doesn't need `self` (instance data) or `cls` (class data)
- Utility/helper functions that make more sense grouped inside the class than floating as standalone module functions

```python
class TemperatureConverter:
    @staticmethod
    def celsius_to_fahrenheit(c):
        return c * 9/5 + 32

    @staticmethod
    def fahrenheit_to_celsius(f):
        return (f - 32) * 5/9
```

---

## 4. `self` vs `cls` vs `@staticmethod` — Full Comparison

| Aspect | Instance Method (`self`) | Class Method (`cls`) | Static Method |
|---|---|---|---|
| Decorator | None (default) | `@classmethod` | `@staticmethod` |
| First parameter | `self` (the instance) | `cls` (the class) | None |
| Access instance data? | Yes | No | No |
| Access class data? | Yes (via `self.__class__` or `type(self)`) | Yes (directly via `cls`) | No (must reference class name explicitly) |
| Called on instance | `obj.method()` | `obj.method()` (still works) | `obj.method()` (still works) |
| Called on class | `Class.method(obj)` (must pass instance) | `Class.method()` | `Class.method()` |
| Typical use case | Operating on/modifying instance state | Alternative constructors, modifying class-level state | Grouped utility/helper logic |
| Inheritance-aware? | Yes (via `self`) | Yes (via `cls` — respects subclassing) | No (hardcoded logic only) |

```python
class Demo:
    class_var = "I'm shared"

    def instance_method(self):
        return f"self: {self}, class_var: {self.class_var}"

    @classmethod
    def class_method(cls):
        return f"cls: {cls}, class_var: {cls.class_var}"

    @staticmethod
    def static_method():
        return "I don't know about self or cls"


d = Demo()
d.instance_method()   # needs an instance
Demo.class_method()    # works directly on the class
Demo.static_method()    # works directly on the class, like a plain function
```

---

## 5. Class Variables vs Instance Variables

```python
class Employee:
    company = "TechCorp"        # CLASS variable -- shared by ALL instances

    def __init__(self, name):
        self.name = name          # INSTANCE variable -- unique per object


e1 = Employee("Alice")
e2 = Employee("Bob")

print(e1.company, e2.company)   # TechCorp TechCorp (shared)

Employee.company = "NewCorp"     # changes it for ALL instances
print(e1.company, e2.company)    # NewCorp NewCorp

e1.company = "StartupX"            # creates a NEW instance attribute, shadows the class one!
print(e1.company, e2.company)      # StartupX NewCorp  -- e1 now has its OWN 'company'
```

> [!warning] Classic Interview Gotcha — Mutable Class Variables
> ```python
> class Team:
>     members = []   # DANGER: shared mutable default across ALL instances
>
>     def add_member(self, name):
>         self.members.append(name)
>
> t1 = Team()
> t2 = Team()
> t1.add_member("Alice")
> print(t2.members)   # ['Alice'] -- BUG! t2 sees t1's data because members is shared
> ```
> **Fix:** Always initialize mutable attributes inside `__init__`, not as class-level defaults:
> ```python
> class Team:
>     def __init__(self):
>         self.members = []   # NOW each instance gets its own list
> ```

---

## 6. `__new__` vs `__init__` — Object Creation Deep Dive

### The Two-Step Object Creation Process

When you write `obj = MyClass(args)`, Python actually performs **two separate steps**:

1. **`__new__(cls, ...)`** — creates and **returns a new (empty) instance** of the class (this is the actual "constructor")
2. **`__init__(self, ...)`** — **initializes** the already-created instance (sets up attributes) — returns `None`

```python
class Demo:
    def __new__(cls, *args, **kwargs):
        print("1. __new__ called -- creating the instance")
        instance = super().__new__(cls)
        return instance

    def __init__(self, value):
        print("2. __init__ called -- initializing the instance")
        self.value = value


d = Demo(42)
# Output:
# 1. __new__ called -- creating the instance
# 2. __init__ called -- initializing the instance
```

> [!note] Analogy
> `__new__` is like a construction company **building the house** (the physical structure exists now). `__init__` is like **moving in the furniture** and setting things up once the house already exists.

### Key Differences

| Aspect | `__new__` | `__init__` |
|---|---|---|
| Purpose | Creates the instance (allocates memory) | Initializes the instance (sets attributes) |
| First parameter | `cls` (the class) | `self` (the instance) |
| Return value | Must return the new instance (or another object) | Must return `None` |
| When called | Before `__init__` | After `__new__` (only if `__new__` returns an instance of `cls`) |
| Overridden how often | Rarely — mostly for singletons, immutable types, metaclasses | Very commonly — standard way to set up instance state |
| Type it belongs to | Static method (implicitly) | Instance method |

> [!warning] `__init__` Only Runs If `__new__` Returns an Instance of `cls`
> ```python
> class Weird:
>     def __new__(cls):
>         return "not even an instance of Weird!"   # returns a plain string
>
>     def __init__(self):
>         print("This will NEVER print")
>
> w = Weird()
> print(type(w))   # <class 'str'> -- __init__ was skipped entirely!
> ```

### Why Override `__new__`? — Real Use Cases

#### 1. Singleton Pattern
```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance


a = Singleton()
b = Singleton()
print(a is b)   # True -- both variables point to the SAME object
```

#### 2. Subclassing Immutable Types (str, int, tuple)
```python
class UpperStr(str):
    def __new__(cls, value):
        return super().__new__(cls, value.upper())


s = UpperStr("hello")
print(s)   # HELLO
```
> [!info] Why `__new__` and not `__init__` here?
> Immutable types (`str`, `int`, `tuple`) are already fully constructed by the time `__init__` runs — their value **can't be changed afterward**. You must intervene in `__new__`, during creation, to control the actual value.

---

## 7. Bonus: Related Concepts Interviewers Often Chain Into

### 7.1 `__del__` — Destructor (rarely used directly)
```python
class Resource:
    def __del__(self):
        print("Cleaning up resource")
```
> [!warning] Not Reliable for Cleanup
> `__del__` timing is **not guaranteed** (depends on garbage collection). Prefer context managers (`with` + `__enter__`/`__exit__`) for deterministic cleanup — see [[Exception Handling (Python)]].

### 7.2 `__init_subclass__` — Hook for When a Class is Subclassed
```python
class Plugin:
    registry = []

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry.append(cls)    # auto-registers every subclass

class AudioPlugin(Plugin):
    pass

class VideoPlugin(Plugin):
    pass

print(Plugin.registry)   # [<class 'AudioPlugin'>, <class 'VideoPlugin'>]
```

### 7.3 Metaclasses — "The Class of a Class"
```python
print(type(42))          # <class 'int'>
print(type(int))          # <class 'type'>  -- 'type' is the metaclass of all classes

class Meta(type):
    def __new__(mcs, name, bases, namespace):
        print(f"Creating class: {name}")
        return super().__new__(mcs, name, bases, namespace)

class MyClass(metaclass=Meta):
    pass
# Output: Creating class: MyClass
```
> [!info] Interview Note
> Every class in Python is itself an **instance of `type`** (the default metaclass). Custom metaclasses let you hook into **class creation itself** — advanced, but a common "how deep do you understand Python" interview question.

### 7.4 Descriptors — What `@property` Actually Is
```python
class Descriptor:
    def __get__(self, obj, objtype=None):
        return 42

    def __set__(self, obj, value):
        print(f"Setting to {value}")


class MyClass:
    attr = Descriptor()

m = MyClass()
print(m.attr)       # 42 -- calls __get__
m.attr = 100          # calls __set__ -> "Setting to 100"
```
> [!info]
> `@property` is actually just Python's built-in **descriptor** implementation. Understanding descriptors explains *why* `@property` works, and is a strong signal of deep Python knowledge in interviews.

---

## 8. Classic Interview Questions & Answers

> [!question] Q1: What's the difference between `self` and `cls`?
> `self` refers to the **specific instance** a method is called on (used in instance methods). `cls` refers to the **class itself** (used in classmethods) — useful for alternative constructors and accessing/modifying class-level state, and it's inheritance-aware.

> [!question] Q2: Can you call an instance method without creating an object?
> Yes, if you explicitly pass an instance: `Dog.bark(some_dog_instance)`. But you cannot call it as `Dog.bark()` with zero arguments — Python needs something to bind as `self`.

> [!question] Q3: Why would you use `@staticmethod` instead of just a regular function outside the class?
> Purely organizational — it groups logically-related utility functions under the class's namespace, improving discoverability and namespacing (e.g. `MathUtils.add()` instead of a loose `add()` function polluting the module).

> [!question] Q4: What happens if you don't return anything from `__new__`?
> If `__new__` doesn't return an instance of `cls` (or a subclass), `__init__` is **never called**. This is a classic trick question.

> [!question] Q5: Why use `cls()` instead of `ClassName()` inside a classmethod?
> `cls()` respects **inheritance** — if a subclass calls the classmethod, `cls` refers to the subclass, so the correct subclass instance is created (rather than always creating the base class).

> [!question] Q6: Is `self` a reserved keyword in Python?
> No — it's purely a strong naming convention (unlike Java's `this`). You could technically use another name, but it would violate PEP 8 and confuse every other Python developer.

> [!question] Q7: What's the danger of using a mutable default as a class attribute?
> It's **shared across all instances** since it lives on the class, not the instance — mutating it from one instance affects all others. Always initialize mutable attributes inside `__init__`.

> [!question] Q8: What is the Method Resolution Order (MRO) and how does it relate to `cls`?
> MRO determines which class's method is used when multiple classes in a hierarchy define the same method (relevant especially in multiple inheritance). `cls` in classmethods dynamically resolves to whichever class the method was actually called through, respecting the MRO. See [[Four Pillars of OOP]] for the full MRO breakdown.

---

## 9. Quick Revision Summary

| Concept | Belongs To | Key Point |
|---|---|---|
| `self` | Instance methods | Refers to the calling instance; not a keyword, just convention |
| `cls` | Class methods (`@classmethod`) | Refers to the class; inheritance-aware; used for alt constructors |
| `@staticmethod` | Neither instance nor class | No automatic first argument; just a namespaced function |
| `__new__` | Object creation | Actual constructor; creates & returns the instance; rarely overridden |
| `__init__` | Object initialization | Sets up instance state; runs only if `__new__` returns an instance of `cls` |
| `__init_subclass__` | Subclass hook | Runs automatically whenever a class is subclassed |
| Metaclass (`type`) | Class creation | The "class of a class"; controls how classes themselves are built |
| Descriptor (`__get__`/`__set__`) | Attribute access control | The mechanism underlying `@property` |

**Golden Rules:**
1. `self` = this specific object; `cls` = the class itself
2. Use `@classmethod` for alternative constructors — always via `cls()`, never hardcode the class name
3. Use `@staticmethod` for logic that doesn't need instance or class data
4. `__new__` creates, `__init__` initializes — `__init__` is skipped if `__new__` doesn't return a `cls` instance
5. Never use mutable objects as class-level default attributes

---

## Related Notes
- [[Four Pillars of OOP]]
- [[Exception Handling (Python)]]
- [[Python Comprehensions]]
- [[Dunder Methods Reference]]
- [[Metaclasses and Descriptors Deep Dive]]
