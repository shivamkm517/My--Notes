---
title: NumPy Advanced Notes
tags: [python, numpy, data-science, advanced]
related: "[[NumPy_Cheat_Sheet]]"
---

# NumPy Advanced Notes

> [!info] Why this note exists
> The basic cheat sheet covers syntax, but skips a few concepts that matter a lot in real usage. This note fills those gaps.

---

## 1. Broadcasting

> [!important] Broadcasting is the most important NumPy concept
> It's how NumPy performs arithmetic on arrays of **different shapes** without explicit loops.

**Rules** (compare shapes from the right):
1. If dimensions differ in size, the size-1 dimension is stretched to match.
2. If a dimension is missing, it's treated as size 1.
3. If shapes don't match and neither is 1, it fails with a `ValueError`.

```python
a = np.array([1, 2, 3])          # shape (3,)
b = np.array([[1], [2], [3]])    # shape (3,1)

a + b
# shape (3,) broadcasts with (3,1) -> result shape (3,3)

M = np.ones((3, 4))
v = np.array([1, 2, 3, 4])       # shape (4,)
M + v                            # v is broadcast across each row
```

> [!example] Common failure
> ```python
> np.ones((3,4)) + np.ones((3,))   # ValueError: shapes not aligned
> np.ones((3,4)) + np.ones((4,))   # Works: (4,) broadcasts against last dim
> ```

---

## 2. The `axis` Parameter

Think of `axis` as **"the dimension that gets collapsed."**

```python
arr = np.array([[1, 2, 3],
                 [4, 5, 6]])

arr.sum(axis=0)   # -> [5, 7, 9]   (collapse rows, sum down each column)
arr.sum(axis=1)   # -> [6, 15]     (collapse columns, sum across each row)
```

| axis | Meaning in 2D | Result shape |
|---|---|---|
| `axis=0` | Operate down columns | shape shrinks in dim 0 |
| `axis=1` | Operate across rows | shape shrinks in dim 1 |
| `axis=None` (default) | Flatten and operate on everything | scalar |

For 3D+ arrays, the same logic extends — the specified axis disappears from the output shape.

---

## 3. Random Number Generation

```python
np.random.seed(42)              # reproducibility (legacy API)

np.random.rand(3, 2)            # uniform [0,1), shape (3,2)
np.random.randn(3, 2)           # standard normal distribution
np.random.randint(0, 10, size=5)  # random ints in [0,10)
np.random.choice([1,2,3,4], size=3, replace=False)  # random sample
```

> [!tip] Modern API (recommended)
> ```python
> rng = np.random.default_rng(seed=42)
> rng.random((3, 2))
> rng.integers(0, 10, size=5)
> rng.choice([1, 2, 3, 4], size=3, replace=False)
> ```
> The `Generator` API (`default_rng`) is preferred over `np.random.seed()` / `np.random.rand()` in modern NumPy — better statistical properties and clearer reproducibility.

---

## 4. Handling NaN / Missing Data

```python
a = np.array([1, 2, np.nan, 4])

np.isnan(a)          # array([False, False, True, False])
np.nanmean(a)        # mean ignoring NaN
np.nansum(a)         # sum ignoring NaN
np.nanmax(a)         # max ignoring NaN
a[~np.isnan(a)]      # filter out NaNs
```

> [!warning]
> Regular `a.mean()` or `a.sum()` will propagate `NaN` into the result. Use the `nan*` variants when your data has missing values.

---

## 5. Conditional Selection

```python
a = np.array([1, 2, 3, 4, 5])

np.where(a > 2, a, 0)        # keep values >2, else 0 -> [0,0,3,4,5]
np.where(a > 2)              # returns indices where condition is True

np.select(
    [a < 2, a < 4],
    ['small', 'medium'],
    default='large'
)

np.clip(a, 2, 4)             # clip values to range [2,4] -> [2,2,3,4,4]
```

`np.where()` is one of the most commonly used NumPy functions — worth memorizing.

---

## 6. Linear Algebra (`np.linalg`)

```python
A = np.array([[1, 2], [3, 4]])

A @ B                  # matrix multiplication (preferred over .dot() for 2D+)
np.linalg.inv(A)       # inverse
np.linalg.det(A)       # determinant
np.linalg.solve(A, b)  # solve Ax = b
np.linalg.eig(A)       # eigenvalues and eigenvectors
np.linalg.norm(A)      # matrix/vector norm
```

> [!tip] `@` vs `.dot()`
> For 2D arrays they behave the same. `@` is the modern, more readable operator introduced in Python 3.5 / NumPy — prefer it in new code.

---

## 7. Set Operations

```python
a = np.array([1, 2, 2, 3, 4])

np.unique(a)                          # [1, 2, 3, 4]
np.unique(a, return_counts=True)      # values + counts

np.intersect1d([1,2,3], [2,3,4])      # [2, 3]
np.union1d([1,2,3], [2,3,4])          # [1,2,3,4]
np.setdiff1d([1,2,3], [2,3])          # [1]
np.isin([1,2,3,4], [2,4])             # [False, True, False, True]
```

---

## 8. Views vs Copies (the gotcha)

> [!warning] This causes real bugs
> Basic slicing returns a **view** (shares memory with the original array). Fancy indexing and boolean indexing return a **copy**.

```python
a = np.array([1, 2, 3, 4, 5])

b = a[1:3]        # VIEW — modifying b modifies a
b[0] = 99
print(a)          # [1, 99, 3, 4, 5]  <- changed!

c = a[[0, 1]]     # COPY — modifying c does NOT affect a
c[0] = -1
print(a)          # unchanged
```

Use `.copy()` explicitly whenever you need an independent array:

```python
b = a[1:3].copy()
```

---

## 9. Performance: Vectorization

> [!important] Avoid Python-level loops over NumPy arrays
> Loops defeat the purpose of NumPy and are dramatically slower.

```python
# Slow
result = []
for x in a:
    result.append(x ** 2 + 1)

# Fast (vectorized)
result = a ** 2 + 1
```

If you truly need element-wise custom logic that can't be vectorized directly:

```python
f = np.vectorize(lambda x: x**2 if x > 0 else 0)
f(a)
```

Note: `np.vectorize` is a convenience wrapper, not a real performance optimization — it still loops internally in Python. True speed comes from using built-in vectorized ufuncs.

---

## 10. Dtype Casting Gotchas

```python
a = np.array([1, 2, 3], dtype=np.int8)
a + 200          # overflow! int8 max is 127, wraps around silently

b = np.array([1, 2, 3])         # int
c = np.array([1.0, 2.0, 3.0])   # float
(b + c).dtype                   # -> float64 (implicit upcasting)

np.array([1, 2, 3], dtype=np.int32).astype(np.float64)  # explicit cast
```

> [!warning]
> Mixing integer and float arrays upcasts silently to float — usually fine, but integer overflow (e.g. small dtypes like `int8`/`uint8`) can silently produce wrong results with **no error or warning**.

---

> [!tip] See also
> [[NumPy_Cheat_Sheet]] for basic syntax and array creation/manipulation reference.
