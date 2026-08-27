---
tags: [oop, python, programming, computer-science, design]
created: 2026-07-08
aliases: [OOP, Four Pillars of OOP, Object-Oriented Programming]
---

# Four Pillars of OOP — Deep Dive

## 0. What is OOP?

**Object-Oriented Programming (OOP)** is a programming paradigm that organizes code around **objects** — bundles of **data (attributes)** and **behavior (methods)** — rather than around functions and logic alone.

> [!note] Core Idea
> Instead of writing procedures that operate on data, you model real-world (or abstract) entities as **classes** (blueprints) and **objects** (instances), each responsible for managing its own state and behavior.

### The Four Pillars
1. **Encapsulation** — bundling data + behavior, restricting direct access
2. **Abstraction** — hiding complexity, exposing only essentials
3. **Inheritance** — reusing and extending behavior from existing classes
4. **Polymorphism** — same interface, different underlying implementations

> [!tip] Quick Mnemonic
> **E.A.I.P** — Encapsulate the mess, Abstract away the detail, Inherit what's shared, Polymorphism lets one interface wear many faces.

---

## 1. Encapsulation

### Definition
**Encapsulation** is the bundling of data (attributes) and the methods that operate on that data into a single unit (a class), while **restricting direct access** to some of an object's internal state to protect it from unintended interference.

> [!note] Analogy
> Think of a capsule of medicine — the ingredients are sealed inside; you interact with the capsule as a whole, not by touching the raw chemicals directly.

### Why It Matters
- Protects internal state from invalid/inconsistent modification
- Hides implementation details — the internal representation can change without breaking external code
- Enforces controlled access via **getters/setters** or **properties**
- Reduces coupling between components

### Access Control in Python

Python doesn't have true `private`/`protected`/`public` keywords like Java or C++ — it relies on **naming conventions**:

| Convention | Syntax | Meaning |
|---|---|---|
| Public | `self.name` | Freely accessible from anywhere |
| Protected (convention) | `self._name` | "Internal use" — a hint to other devs, not enforced |
| Private (name-mangled) | `self.__name` | Python renames it internally to `_ClassName__name` to discourage (not prevent) external access |

```python
class BankAccount:
    def __init__(self, owner, balance):
        self.owner = owner          # public
        self._account_type = "savings"  # protected (convention only)
        self.__balance = balance    # private (name-mangled)

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self.__balance += amount

    def withdraw(self, amount):
        if amount > self.__balance:
            raise ValueError("Insufficient funds")
        self.__balance -= amount

    def get_balance(self):
        return self.__balance


acc = BankAccount("Alice", 1000)
acc.deposit(500)
print(acc.get_balance())     # 1500
print(acc._BankAccount__balance)  # 1500 -- name mangling doesn't fully hide it
```

> [!warning] Python's "Private" is Not Truly Private
> `__balance` becomes `_BankAccount__balance` internally — accessible if you know the mangled name. Python follows the philosophy **"we're all consenting adults here"** — encapsulation is convention-based, relying on discipline rather than compiler enforcement.

### The Pythonic Way — `@property`

Rather than explicit `get_x()`/`set_x()` methods (common in Java), Python uses **properties** to expose controlled access while keeping attribute-style syntax.

```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("Temperature below absolute zero!")
        self._celsius = value

    @property
    def fahrenheit(self):
        return self._celsius * 9/5 + 32   # computed / read-only property


t = Temperature(25)
print(t.celsius)       # 25 (looks like attribute access, but calls a method)
t.celsius = 30          # calls the setter, runs validation
print(t.fahrenheit)     # 86.0 (no setter defined -> read-only)
```

> [!success] Why `@property` is Better Than Java-Style Getters/Setters
> - Keeps the clean `obj.attr` syntax while still validating on assignment
> - You can start with a plain public attribute and **add validation later** without breaking calling code
> - Read-only computed properties (like `fahrenheit`) are trivial to add

### Encapsulation Summary Table

| Aspect | Detail |
|---|---|
| Purpose | Protect state, control access, hide internal details |
| Python mechanism | Naming convention (`_`, `__`) + `@property` |
| Enforcement | Convention-based, not compiler-enforced |
| Key benefit | Internal implementation can change freely without breaking external callers |

---

## 2. Abstraction

### Definition
**Abstraction** means hiding complex implementation details and exposing only the essential features/interface needed to use an object — the "what" without the "how."

> [!note] Analogy
> When you drive a car, you use the steering wheel, pedals, and gear stick — you don't need to know how the engine's combustion cycle works internally. The car "abstracts away" that complexity.

### Abstraction vs Encapsulation (Commonly Confused)

| Aspect | Encapsulation | Abstraction |
|---|---|---|
| Focus | **How** data is protected/bundled | **What** is exposed to the user |
| Mechanism | Access modifiers, `@property` | Abstract classes, interfaces |
| Goal | Data hiding & integrity | Complexity hiding & design simplification |
| Level | Implementation-level | Design-level |

> [!tip] Simple Distinction
> Encapsulation = wrapping data with methods and restricting access.
> Abstraction = showing only relevant details and hiding the rest of the design/logic.

### Achieving Abstraction in Python — `abc` module

Python provides the `abc` (Abstract Base Class) module to define abstract classes and methods that **must** be implemented by subclasses.

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

    @abstractmethod
    def perimeter(self):
        pass

    def describe(self):   # concrete (non-abstract) method — shared by all subclasses
        return f"This shape has area {self.area()} and perimeter {self.perimeter()}"


class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

    def perimeter(self):
        return 2 * (self.width + self.height)


# shape = Shape()          # TypeError: Can't instantiate abstract class Shape
rect = Rectangle(4, 5)
print(rect.describe())      # This shape has area 20 and perimeter 18
```

> [!warning] Enforcement
> Attempting to instantiate `Shape` directly raises `TypeError` because it has unimplemented abstract methods. Any subclass **must** override all `@abstractmethod`s or it also becomes abstract (can't be instantiated).

### "Duck Typing" — Python's Informal Abstraction

Python often relies on **duck typing** instead of strict interfaces: *"If it walks like a duck and quacks like a duck, it's a duck."* If an object has the right methods, Python doesn't care about its exact type.

```python
class Duck:
    def sound(self):
        return "Quack!"

class Dog:
    def sound(self):
        return "Woof!"

def make_it_speak(animal):
    print(animal.sound())   # works for ANY object with a .sound() method

make_it_speak(Duck())
make_it_speak(Dog())
```

> [!info]
> This is a more relaxed, informal form of abstraction unique to dynamically typed languages — you rely on **behavior**, not explicit type declarations. `abc` gives you a formal contract when you need stricter design guarantees.

### Abstraction Summary Table

| Aspect | Detail |
|---|---|
| Purpose | Hide complexity, expose only necessary interface |
| Python mechanism | `abc.ABC`, `@abstractmethod`, duck typing |
| Key benefit | Simplifies usage, enforces a contract for subclasses |
| Real-world example | `list.append()` — you don't know/care about internal resizing logic |

---

## 3. Inheritance

### Definition
**Inheritance** allows a class (child/subclass) to acquire attributes and methods from another class (parent/superclass), promoting **code reuse** and establishing an "is-a" relationship.

> [!note] Analogy
> A `Dog` **is an** `Animal`. It inherits general animal behaviors (eating, sleeping) while adding/overriding its own specific behavior (barking).

### Basic Syntax

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def eat(self):
        return f"{self.name} is eating."

    def make_sound(self):
        return "Some generic sound"


class Dog(Animal):          # Dog inherits from Animal
    def make_sound(self):    # method overriding
        return "Woof!"


d = Dog("Rex")
print(d.eat())         # inherited method -> "Rex is eating."
print(d.make_sound())  # overridden method -> "Woof!"
```

### `super()` — Calling the Parent Class

```python
class Animal:
    def __init__(self, name, species):
        self.name = name
        self.species = species

class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name, species="Dog")  # reuse parent's init logic
        self.breed = breed

d = Dog("Rex", "Labrador")
print(d.name, d.species, d.breed)   # Rex Dog Labrador
```

> [!tip] Why use `super()` instead of `Animal.__init__(self, ...)`?
> `super()` correctly follows the **Method Resolution Order (MRO)**, especially important in multiple inheritance — hardcoding the parent class name bypasses MRO and can cause bugs in complex hierarchies.

### Types of Inheritance

```python
# 1. Single Inheritance
class A: pass
class B(A): pass

# 2. Multilevel Inheritance
class A: pass
class B(A): pass
class C(B): pass          # C -> B -> A

# 3. Multiple Inheritance
class A: pass
class B: pass
class C(A, B): pass       # C inherits from both A and B

# 4. Hierarchical Inheritance
class A: pass
class B(A): pass
class C(A): pass          # B and C both inherit from A

# 5. Hybrid Inheritance -- a combination of the above patterns
```

### Method Resolution Order (MRO) — Multiple Inheritance

Python uses the **C3 Linearization algorithm** to determine the order in which base classes are searched when calling a method.

```python
class A:
    def greet(self):
        return "Hello from A"

class B(A):
    def greet(self):
        return "Hello from B"

class C(A):
    def greet(self):
        return "Hello from C"

class D(B, C):
    pass

d = D()
print(d.greet())          # "Hello from B" (B comes before C in MRO)
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

> [!warning] The Diamond Problem
> When `D` inherits from both `B` and `C`, which both inherit from `A`, there's ambiguity about which `greet()` to use. Python's MRO (via `D.__mro__` or `D.mro()`) resolves this deterministically — left-to-right, depth-first, but avoiding revisiting classes already covered (C3 linearization).

### Overriding vs Overloading

| Concept | Supported in Python? | Explanation |
|---|---|---|
| Method Overriding | Yes | Subclass redefines a method inherited from parent |
| Method Overloading (classic, multiple same-name methods with different signatures) | **No** (not natively) | Python only keeps the **last** defined method with a given name; achieved instead via default arguments, `*args`/`**kwargs`, or `functools.singledispatch` |

```python
# Simulating "overloading" using default args / *args
class Calculator:
    def add(self, a, b=0, c=0):
        return a + b + c

calc = Calculator()
print(calc.add(5))         # 5
print(calc.add(5, 10))      # 15
print(calc.add(5, 10, 15))   # 30
```

### Composition vs Inheritance

> [!tip] "Favor Composition Over Inheritance"
> A well-known OOP design principle. Inheritance creates a tight ("is-a") coupling that can become fragile in deep hierarchies. **Composition** ("has-a" relationship — building objects using other objects) is often more flexible.

```python
# Inheritance ("is-a")
class Engine:
    def start(self):
        return "Engine starting..."

class Car(Engine):    # awkward: a Car "is-an" Engine? Not really.
    pass

# Composition ("has-a") -- generally preferred here
class Car:
    def __init__(self):
        self.engine = Engine()   # Car HAS an Engine

    def start(self):
        return self.engine.start()
```

### Inheritance Summary Table

| Aspect | Detail |
|---|---|
| Purpose | Code reuse, establishing "is-a" relationships |
| Python mechanism | `class Child(Parent):`, `super()` |
| Key concepts | MRO, method overriding, multiple inheritance |
| Design tip | Prefer composition when relationship isn't a true "is-a" |

---

## 4. Polymorphism

### Definition
**Polymorphism** ("many forms") means the same interface (method/function name) can behave differently depending on the object it's called on.

> [!note] Analogy
> Pressing the "play" button behaves differently for a music app, a video app, or a game console — same action, different underlying behavior depending on context.

### 4.1 Method Overriding (Runtime Polymorphism)

Already seen in inheritance — subclasses provide their own version of a parent's method.

```python
class Shape:
    def area(self):
        raise NotImplementedError

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    def area(self):
        return 3.14159 * self.radius ** 2

class Square(Shape):
    def __init__(self, side):
        self.side = side
    def area(self):
        return self.side ** 2


shapes = [Circle(5), Square(4)]
for shape in shapes:
    print(shape.area())   # calls the CORRECT area() for each object automatically
# 78.53975
# 16
```

> [!success] The Power of Polymorphism
> The loop above doesn't need to know or check *what kind* of shape it's dealing with — it just calls `.area()` and trusts each object to "do the right thing." This is the essence of polymorphism.

### 4.2 Duck Typing (Python's Natural Polymorphism)

Since Python doesn't enforce strict types, **any object with the right method** can be used interchangeably — no shared base class required.

```python
class Bird:
    def move(self):
        return "Flies through the sky"

class Fish:
    def move(self):
        return "Swims in water"

class Robot:
    def move(self):
        return "Rolls on wheels"

for entity in [Bird(), Fish(), Robot()]:
    print(entity.move())   # works despite zero shared inheritance
```

### 4.3 Operator Overloading (via Dunder/Magic Methods)

Python lets you redefine how built-in operators behave for custom objects using **dunder methods** (double-underscore / "magic" methods).

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, other):          # overloads the + operator
        return Vector(self.x + other.x, self.y + other.y)

    def __eq__(self, other):            # overloads ==
        return self.x == other.x and self.y == other.y

    def __repr__(self):                  # controls how print() displays it
        return f"Vector({self.x}, {self.y})"


v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)        # Vector(4, 6) -- calls __add__ automatically
print(v1 == Vector(1, 2))   # True -- calls __eq__
```

### Common Dunder Methods for Polymorphism

| Method | Operator/Behavior |
|---|---|
| `__add__` | `+` |
| `__sub__` | `-` |
| `__mul__` | `*` |
| `__eq__` | `==` |
| `__lt__` / `__gt__` | `<` / `>` |
| `__len__` | `len(obj)` |
| `__str__` | `str(obj)` / `print(obj)` |
| `__repr__` | `repr(obj)` — official/debug string |
| `__getitem__` | `obj[index]` |
| `__iter__` | makes an object iterable (`for x in obj`) |
| `__call__` | lets an object be called like a function: `obj()` |

### 4.4 Polymorphism with Built-in Functions

```python
print(len("hello"))     # 5   -- __len__ on str
print(len([1,2,3]))       # 3   -- __len__ on list
print(len({"a":1,"b":2}))  # 2   -- __len__ on dict

# The SAME function `len()` works differently depending on the object's type
```

> [!info]
> This is a great real-world example: `len()` is polymorphic because each type implements its own `__len__`, and `len()` simply calls whatever version matches the object passed in.

### Polymorphism Summary Table

| Aspect | Detail |
|---|---|
| Purpose | Same interface, different behavior per object |
| Python mechanisms | Method overriding, duck typing, operator overloading (dunder methods) |
| Key benefit | Write generic code that works across many types without type-checking |
| Native Python example | `len()`, `+`, `str()` all behave polymorphically |

---

## 5. How the Four Pillars Work Together

```python
from abc import ABC, abstractmethod

# ABSTRACTION: defines a contract every employee type must follow
class Employee(ABC):
    def __init__(self, name, salary):
        self._name = name            # ENCAPSULATION: protected attribute
        self.__salary = salary        # ENCAPSULATION: private attribute

    @property
    def salary(self):                 # controlled access via property
        return self.__salary

    @salary.setter
    def salary(self, value):
        if value < 0:
            raise ValueError("Salary can't be negative")
        self.__salary = value

    @abstractmethod
    def calculate_bonus(self):         # ABSTRACTION: must be implemented by subclasses
        pass

    def __str__(self):                  # POLYMORPHISM: dunder method
        return f"{self._name}: ${self.salary} + bonus ${self.calculate_bonus()}"


class Manager(Employee):                # INHERITANCE
    def calculate_bonus(self):           # POLYMORPHISM: overridden method
        return self.salary * 0.20


class Developer(Employee):               # INHERITANCE
    def calculate_bonus(self):            # POLYMORPHISM: overridden method
        return self.salary * 0.10


employees = [Manager("Alice", 90000), Developer("Bob", 75000)]
for emp in employees:
    print(emp)   # POLYMORPHISM: same loop, different calculate_bonus() per object

# Alice: $90000 + bonus $18000.0
# Bob: $75000 + bonus $7500.0
```

> [!success] Annotated Takeaway
> - **Encapsulation** — `__salary` is private, accessed safely via `@property`
> - **Abstraction** — `Employee` defines *what* every employee must do (`calculate_bonus`) without saying *how*
> - **Inheritance** — `Manager` and `Developer` reuse `Employee`'s shared logic
> - **Polymorphism** — the `for` loop calls `calculate_bonus()` and `__str__()` without caring which subclass it's dealing with

---

## 6. Best Practices ✅

> [!success] Do's
> - Use `@property` instead of manual getter/setter methods for Pythonic encapsulation
> - Use `abc.ABC` when you want to **enforce** a contract across subclasses
> - Prefer **composition over inheritance** when the relationship isn't a genuine "is-a"
> - Use `super()` (not hardcoded parent class names) to respect MRO
> - Implement `__repr__` on custom classes for better debugging output
> - Keep class hierarchies shallow — deep inheritance chains become hard to reason about

> [!failure] Don'ts
> - Don't rely on double-underscore name mangling as real security — it's a convention, not protection
> - Don't overuse multiple inheritance — it increases MRO complexity and diamond-problem risk
> - Don't create deep inheritance hierarchies "just because" — often composition is simpler and more flexible
> - Don't forget `super().__init__()` when overriding `__init__` in a subclass (parent state won't initialize otherwise)
> - Don't abuse operator overloading for unintuitive behavior (e.g., `__add__` that doesn't actually "add" anything conceptually)

---

## 7. Quick Revision Summary

| Pillar | One-Line Definition | Python Mechanism |
|---|---|---|
| **Encapsulation** | Bundle data + behavior, restrict direct access | `_protected`, `__private`, `@property` |
| **Abstraction** | Hide complexity, expose only essentials | `abc.ABC`, `@abstractmethod`, duck typing |
| **Inheritance** | Reuse/extend behavior from another class | `class Child(Parent):`, `super()` |
| **Polymorphism** | Same interface, different behavior per type | Method overriding, dunder methods, duck typing |

**Golden Rules:**
1. Encapsulation ≠ true privacy in Python — it's convention + `@property`
2. Abstraction defines *what*; Inheritance/Polymorphism handle *how* and *which version*
3. `super()` respects MRO — always prefer it over hardcoded parent calls
4. Python favors duck typing — polymorphism doesn't require shared inheritance
5. Favor composition over inheritance when in doubt

---

## Related Notes
- [[Exception Handling (Python)]]
- [[Python Comprehensions]]
- [[SOLID Principles]]
- [[Design Patterns in Python]]
- [[Dunder Methods Reference]]
