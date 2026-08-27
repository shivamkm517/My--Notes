---
title: Pandas & Data Handling — Complete Notes
tags: [python, pandas, data-science, data-handling, cheatsheet]
related: "[[NumPy_Cheat_Sheet]], [[NumPy_Advanced_Notes]], [[Python_File_Handling_Notes]]"
---

	
# Pandas & Data Handling — Complete Notes

> [!info] Scope
> Everything from the ground up: core data structures, creating/loading data, inspecting, selecting, cleaning, transforming, combining, grouping, reshaping, time series, string operations, I/O for every major format, performance tips, and common gotchas.

```python
import pandas as pd
import numpy as np
```

---

## 1. Core Data Structures

### Series (1D labeled array)

```python
s = pd.Series([10, 20, 30], index=['a', 'b', 'c'], name='values')
```

```
a    10
b    20
c    30
Name: values, dtype: int64
```

- Has an **index** (labels) and **values** (a NumPy array under the hood).
- Can hold any data type: int, float, string, Python objects.

### DataFrame (2D labeled table)

```python
df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [25, 30, 35],
    'city': ['NY', 'LA', 'SF']
})
```

|     | name    | age | city |
| --- | ------- | --- | ---- |
| 0   | Alice   | 25  | NY   |
| 1   | Bob     | 30  | LA   |
| 2   | Charlie | 35  | SF   |

> [!note] Mental model
> A DataFrame is essentially a **dict of Series** sharing the same index. Each column is a Series.

---

## 2. Creating DataFrames — All the Ways

```python
# From dict of lists
pd.DataFrame({'a': [1,2], 'b': [3,4]})

# From list of dicts
pd.DataFrame([{'a':1,'b':2}, {'a':3,'b':4}])

# From list of lists (with explicit columns)
pd.DataFrame([[1,2],[3,4]], columns=['a','b'])

# From NumPy array
pd.DataFrame(np.random.rand(3,2), columns=['x','y'])

# From Series
pd.DataFrame(pd.Series([1,2,3]), columns=['val'])

# Empty DataFrame with defined columns
pd.DataFrame(columns=['a', 'b', 'c'])
```

---

## 3. Reading & Writing Data (I/O)

### CSV

```python
df = pd.read_csv("file.csv")
df = pd.read_csv("file.csv", sep=';', header=0, index_col=0,
                  usecols=['a','b'], dtype={'a': str}, na_values=['NA', '?'],
                  parse_dates=['date_col'], nrows=1000, encoding='utf-8')

df.to_csv("out.csv", index=False)
```

### Excel

```python
df = pd.read_excel("file.xlsx", sheet_name="Sheet1")
df = pd.read_excel("file.xlsx", sheet_name=None)   # dict of all sheets

df.to_excel("out.xlsx", sheet_name="Sheet1", index=False)

# Multiple sheets
with pd.ExcelWriter("out.xlsx") as writer:
    df1.to_excel(writer, sheet_name="Sheet1")
    df2.to_excel(writer, sheet_name="Sheet2")
```

### JSON

```python
df = pd.read_json("file.json")
df.to_json("out.json", orient="records", indent=2)
```

### SQL

```python
import sqlite3
conn = sqlite3.connect("db.sqlite")

df = pd.read_sql("SELECT * FROM table_name", conn)
df = pd.read_sql_query("SELECT * FROM table_name WHERE age > 25", conn)
df.to_sql("table_name", conn, if_exists="replace", index=False)
```

### Other formats

```python
pd.read_parquet("file.parquet")     # columnar, efficient for big data
pd.read_html("page.html")           # scrapes all <table> elements -> list of DataFrames
pd.read_clipboard()                 # reads copied tabular data
pd.read_pickle("file.pkl")
df.to_pickle("out.pkl")
df.to_parquet("out.parquet")
```

> [!tip]
> `parquet` is preferred over CSV for large datasets — faster read/write, smaller file size, and preserves dtypes.

---

## 4. Inspecting Data

```python
df.head(5)          # first 5 rows
df.tail(5)          # last 5 rows
df.sample(5)        # random 5 rows

df.shape             # (rows, columns)
df.info()            # dtypes, non-null counts, memory usage
df.describe()        # summary stats for numeric columns
df.describe(include='all')   # include categorical columns too

df.columns           # column labels
df.index             # row labels
df.dtypes            # data type of each column
df.values             # underlying NumPy array
df.memory_usage(deep=True)   # memory used per column

df.nunique()          # number of unique values per column
df['col'].unique()    # unique values in a column
df['col'].value_counts()  # frequency count of each value
```

---

## 5. Selecting & Indexing Data

### Column selection

```python
df['age']            # single column -> Series
df[['age', 'name']]  # multiple columns -> DataFrame
```

### Row selection — `loc` (label-based) vs `iloc` (position-based)

```python
df.loc[0]              # row with index label 0
df.loc[0:2]            # rows with labels 0 through 2 (INCLUSIVE)
df.loc[0:2, 'age']     # rows 0-2, column 'age'
df.loc[df['age'] > 25] # boolean filtering

df.iloc[0]              # row at position 0
df.iloc[0:2]            # rows at positions 0-1 (EXCLUSIVE of end, like Python slicing)
df.iloc[0:2, 1]         # rows 0-1, column at position 1
df.iloc[-1]             # last row
```

> [!warning] `loc` vs `iloc` slicing difference
> `df.loc[0:2]` includes index 2. `df.iloc[0:2]` excludes position 2. This is the #1 source of off-by-one confusion for beginners.

### Boolean / Conditional Filtering

```python
df[df['age'] > 25]
df[(df['age'] > 25) & (df['city'] == 'NY')]     # AND -> use &
df[(df['age'] > 25) | (df['city'] == 'NY')]     # OR  -> use |
df[~(df['age'] > 25)]                            # NOT -> use ~
df[df['city'].isin(['NY', 'LA'])]
df[df['name'].str.contains('Al')]
df.query('age > 25 and city == "NY"')            # query() syntax
```

> [!warning]
> Use `&`, `|`, `~` (not `and`, `or`, `not`) for element-wise boolean logic on Series, and always wrap each condition in parentheses due to operator precedence.

### `at` / `iat` — fast scalar access

```python
df.at[0, 'age']     # fast label-based single value access
df.iat[0, 1]        # fast position-based single value access
```

---

## 6. Adding, Modifying, Dropping Columns/Rows

```python
df['new_col'] = df['age'] * 2                 # add/overwrite column
df['category'] = np.where(df['age'] > 30, 'old', 'young')

df.insert(1, 'new_col', values)               # insert at specific position

df.drop('age', axis=1)                        # drop column (returns new df)
df.drop(columns=['age', 'city'])              # drop multiple columns
df.drop(0, axis=0)                            # drop row by index
df.drop([0,1], axis=0, inplace=True)          # drop rows in place

df.rename(columns={'age': 'years'})           # rename column(s)
df.rename(index={0: 'first_row'})             # rename index label(s)

df.assign(new_col = lambda d: d['age'] * 2)   # add column, chainable

df = df[['name', 'age', 'city']]              # reorder columns by selection
```

> [!tip]
> Most pandas operations return a **new object** by default; use `inplace=True` to modify the original DataFrame directly (though many recommend avoiding `inplace` for clarity/chaining).

---

## 7. Handling Missing Data

```python
df.isna()               # boolean mask of NaN/None
df.isnull()              # alias of isna()
df.notna()               # opposite of isna()

df.isna().sum()          # count of missing values per column
df.isna().sum().sum()    # total missing values in whole df

df.dropna()                        # drop rows with ANY NaN
df.dropna(axis=1)                  # drop columns with ANY NaN
df.dropna(how='all')               # drop rows only if ALL values are NaN
df.dropna(subset=['age'])          # drop rows where 'age' is NaN
df.dropna(thresh=2)                # keep rows with at least 2 non-NaN values

df.fillna(0)                       # replace NaN with 0
df.fillna({'age': 0, 'city': 'Unknown'})   # per-column fill values
df.fillna(method='ffill')          # forward fill (propagate last valid value)
df.fillna(method='bfill')          # backward fill
df['age'].fillna(df['age'].mean()) # fill with column mean

df.interpolate()                   # interpolate missing numeric values
```

> [!important]
> `NaN != NaN` in Python — you can't check for missing values with `== np.nan`. Always use `.isna()` / `.isnull()`.

---

## 8. Handling Duplicates

```python
df.duplicated()                    # boolean Series marking duplicate rows
df.duplicated(subset=['name'])     # check duplicates based on specific columns
df.drop_duplicates()               # remove duplicate rows
df.drop_duplicates(subset=['name'], keep='first')   # keep first occurrence
df.drop_duplicates(keep='last')    # keep last occurrence
df.drop_duplicates(keep=False)     # drop ALL duplicates entirely
```

---

## 9. Data Type Conversion

```python
df['age'] = df['age'].astype(int)
df['age'] = df['age'].astype(float)
df['date'] = pd.to_datetime(df['date'])
df['num'] = pd.to_numeric(df['num'], errors='coerce')   # invalid -> NaN

df.dtypes
df.convert_dtypes()      # auto-infer best dtypes (nullable Int64, string, etc.)

df['category_col'] = df['category_col'].astype('category')  # memory-efficient
```

> [!tip]
> Converting object columns with few unique repeated values to `'category'` dtype drastically reduces memory usage and speeds up groupby operations.

---

## 10. String Operations — `.str` Accessor

```python
df['name'].str.lower()
df['name'].str.upper()
df['name'].str.strip()
df['name'].str.len()
df['name'].str.replace('a', 'A')
df['name'].str.contains('Al', case=False)
df['name'].str.startswith('A')
df['name'].str.endswith('e')
df['name'].str.split(' ')
df['name'].str.split(' ', expand=True)    # split into separate columns
df['name'].str.cat(sep=', ')              # join all values into one string
df['name'].str.slice(0, 3)
df['name'].str.pad(10, side='left', fillchar='0')
df['name'].str.extract(r'(\d+)')          # regex extraction
```

---

## 11. Applying Functions

```python
df['age'].apply(lambda x: x * 2)                  # element-wise on a Series
df.apply(lambda row: row['age'] + 1, axis=1)      # row-wise across DataFrame
df.apply(lambda col: col.max(), axis=0)           # column-wise (default axis=0)

df.applymap(lambda x: x * 2)      # element-wise across entire DataFrame (deprecated in newer pandas -> use df.map())
df['age'].map({25: 'young', 30: 'mid'})           # value mapping via dict
df['age'].map(lambda x: x * 2)                    # same as apply for Series

df.pipe(custom_function, arg1, arg2)              # chain custom functions cleanly
```

> [!note] `apply` vs `map` vs `applymap`
> - `.map()` → Series only, element-wise, good for value substitution via dict.
> - `.apply()` → Series (element-wise) OR DataFrame (row/column-wise via axis).
> - `.applymap()` → DataFrame, element-wise on every cell (renamed to `.map()` on DataFrames in pandas 2.1+).

---

## 12. Sorting

```python
df.sort_values('age')                         # ascending
df.sort_values('age', ascending=False)
df.sort_values(['age', 'name'])               # sort by multiple columns
df.sort_values(['age','name'], ascending=[True, False])

df.sort_index()                               # sort by index
df.sort_index(ascending=False)

df.nlargest(3, 'age')                         # top 3 rows by column
df.nsmallest(3, 'age')                        # bottom 3 rows by column

df['rank'] = df['age'].rank()                 # rank values
```

---

## 13. GroupBy — Split, Apply, Combine

```python
df.groupby('city')['age'].mean()
df.groupby('city').agg({'age': 'mean', 'name': 'count'})
df.groupby('city').agg(
    avg_age=('age', 'mean'),
    total=('name', 'count')
)                                              # named aggregation

df.groupby(['city', 'category']).sum()        # multi-column groupby

df.groupby('city').apply(lambda g: g.nlargest(1, 'age'))  # custom per-group logic

df.groupby('city').size()                     # count of rows per group (incl. NaN in other cols)
df.groupby('city')['age'].count()             # count of non-null values per group

df.groupby('city').filter(lambda g: len(g) > 2)   # keep only groups meeting a condition

df.groupby('city')['age'].transform('mean')   # broadcast group stat back to original shape
```

> [!tip] `transform` vs `agg`
> `agg()` collapses each group into a single row. `transform()` returns a result the **same length as the original DataFrame**, useful for adding group-level stats as a new column without losing row-level detail.

**Common aggregation functions:** `'sum'`, `'mean'`, `'median'`, `'min'`, `'max'`, `'count'`, `'std'`, `'var'`, `'first'`, `'last'`, `'nunique'`

---

## 14. Pivot Tables & Cross-tabulation

```python
pd.pivot_table(df, values='age', index='city', columns='category', aggfunc='mean')
pd.pivot_table(df, values='age', index='city', aggfunc=['mean','sum'], margins=True)

df.pivot(index='date', columns='city', values='temp')   # reshape without aggregation

pd.crosstab(df['city'], df['category'])                  # frequency table
pd.crosstab(df['city'], df['category'], normalize='index')  # row percentages
```

---

## 15. Reshaping Data

### Melt (wide → long)

```python
pd.melt(df, id_vars=['name'], value_vars=['math', 'science'],
        var_name='subject', value_name='score')
```

### Stack / Unstack

```python
df.stack()          # pivot columns into inner index level (wide -> long)
df.unstack()        # pivot inner index level into columns (long -> wide)
```

### Transpose

```python
df.T
```

---

## 16. Merging, Joining, Concatenating

### Concat (stacking DataFrames)

```python
pd.concat([df1, df2])                    # stack rows (axis=0 default)
pd.concat([df1, df2], axis=1)            # stack columns side by side
pd.concat([df1, df2], ignore_index=True) # reset index after concatenation
```

### Merge (SQL-style joins)

```python
pd.merge(df1, df2, on='id', how='inner')    # inner join (default)
pd.merge(df1, df2, on='id', how='left')     # left join
pd.merge(df1, df2, on='id', how='right')    # right join
pd.merge(df1, df2, on='id', how='outer')    # full outer join

pd.merge(df1, df2, left_on='id1', right_on='id2')   # different key column names
pd.merge(df1, df2, on='id', suffixes=('_left', '_right'))  # handle overlapping cols
```

| `how` | Result |
|---|---|
| `'inner'` | Only matching keys in both |
| `'left'` | All rows from left, matched from right |
| `'right'` | All rows from right, matched from left |
| `'outer'` | All rows from both, NaN where no match |

### Join (index-based, shortcut for merge)

```python
df1.join(df2, how='left')          # joins on index by default
df1.set_index('id').join(df2.set_index('id'))
```

---

## 17. MultiIndex (Hierarchical Indexing)

```python
df.set_index(['city', 'category'])          # create MultiIndex
df.reset_index()                             # flatten back to columns

df.loc[('NY', 'A')]                          # select by tuple of index levels
df.xs('NY', level='city')                    # cross-section at one level

df.swaplevel()                               # swap index levels
df.sort_index(level=0)                       # sort by outer index level
```

---

## 18. Time Series Handling

```python
df['date'] = pd.to_datetime(df['date'], format='%Y-%m-%d')

df.set_index('date', inplace=True)

df.resample('M').mean()          # resample to monthly averages
df.resample('W').sum()           # weekly sums
df.resample('D').ffill()         # daily, forward-filled

df.index.year
df.index.month
df.index.day
df.index.dayofweek
df.index.weekday_name if hasattr(df.index, 'weekday_name') else df.index.day_name()

df['date'].dt.year               # accessor for datetime column (not index)
df['date'].dt.month
df['date'].dt.day_name()

pd.date_range(start='2024-01-01', end='2024-01-10', freq='D')
pd.date_range(start='2024-01-01', periods=5, freq='M')

df.shift(1)                      # shift values down by 1 (lag)
df.diff()                        # difference from previous row
df.rolling(window=3).mean()      # rolling/moving average
df.expanding().mean()            # expanding (cumulative) average
```

**Common `freq` codes:** `'D'` day, `'W'` week, `'M'` month end, `'MS'` month start, `'Q'` quarter, `'Y'` year, `'H'` hour, `'T'`/`'min'` minute.

---

## 19. Combining/Comparing DataFrames

```python
df1.equals(df2)                  # True if same shape & values
df1.compare(df2)                 # shows differing values side by side

df.combine_first(df2)            # fill NaNs in df with values from df2
```

---

## 20. Handling Outliers & Data Cleaning Patterns

```python
# Z-score method
z = (df['value'] - df['value'].mean()) / df['value'].std()
df_clean = df[(z.abs() < 3)]

# IQR method
Q1 = df['value'].quantile(0.25)
Q3 = df['value'].quantile(0.75)
IQR = Q3 - Q1
df_clean = df[(df['value'] >= Q1 - 1.5*IQR) & (df['value'] <= Q3 + 1.5*IQR)]

# Clipping
df['value'] = df['value'].clip(lower=0, upper=100)

# Removing whitespace / normalizing text columns
df['name'] = df['name'].str.strip().str.title()

# Replacing values
df['city'].replace({'NY': 'New York', 'LA': 'Los Angeles'})
df.replace(-999, np.nan)
```

---

## 21. Binning / Discretization

```python
pd.cut(df['age'], bins=[0,18,35,60,100], labels=['teen','young','adult','senior'])
pd.qcut(df['age'], q=4)          # bin into quartiles (equal-frequency bins)
```

---

## 22. Iterating Over Rows (avoid when possible)

```python
for index, row in df.iterrows():
    print(index, row['age'])

for row in df.itertuples():
    print(row.age)               # faster than iterrows()
```

> [!warning] Avoid looping over DataFrames
> `iterrows()`/`itertuples()` are dramatically slower than vectorized operations. Always prefer vectorized pandas/NumPy operations, `.apply()`, or `.map()` first.

---

## 23. Exporting / Displaying Options

```python
pd.set_option('display.max_columns', None)
pd.set_option('display.max_rows', 100)
pd.set_option('display.width', 1000)
pd.reset_option('all')

df.style.highlight_max(axis=0)     # conditional styling (Jupyter)
df.to_string()                     # full string representation, no truncation
```

---

## 24. Performance Tips

| Tip | Why |
|---|---|
| Use vectorized operations, not loops | Loops are 10-100x+ slower |
| Use `.category` dtype for repeated strings | Saves memory, speeds up groupby |
| Read only needed columns: `usecols=` | Reduces memory/load time |
| Use `dtype=` on `read_csv` | Avoids costly dtype inference |
| Use `chunksize=` for huge CSVs | Processes data in batches, avoids loading it all into RAM |
| Prefer `.loc`/`.iloc` over chained indexing | Avoids `SettingWithCopyWarning` and hidden copies |
| Use `.query()` for complex filters | More readable, sometimes faster on large data |
| Use `pyarrow` backend (pandas 2.0+) | Faster and more memory-efficient than default NumPy backend |

```python
# Reading huge CSVs in chunks
for chunk in pd.read_csv("huge.csv", chunksize=100000):
    process(chunk)
```

---

## 25. Common Gotchas Summary

| Gotcha | Explanation / Fix |
|---|---|
| `SettingWithCopyWarning` | Happens with chained indexing like `df[df.a>1]['b']=5`. Use `.loc[df.a>1, 'b'] = 5` instead |
| `loc` includes end label, `iloc` doesn't | Remember label-slicing is inclusive |
| `NaN != NaN` | Always use `.isna()`, never `== np.nan` |
| Using `and`/`or` on Series | Raises `ValueError`. Use `&`, `\|`, `~` with parentheses |
| Forgetting `ignore_index=True` in `concat` | Leads to duplicate index labels |
| Mixing dtypes silently upcasts to `object` | Check `.dtypes` after merges/concats |
| Modifying a DataFrame while iterating | Can cause undefined behavior — avoid |
| Default `read_csv` encoding assumptions | Explicitly set `encoding='utf-8'` for non-ASCII data |

---

## 26. Quick Reference Cheat Table

| Task | Code |
|---|---|
| Load CSV | `pd.read_csv('file.csv')` |
| First look | `df.head()`, `df.info()`, `df.describe()` |
| Filter rows | `df[df['col'] > x]` |
| Select columns | `df[['a','b']]` |
| Handle missing | `df.dropna()` / `df.fillna(val)` |
| Remove duplicates | `df.drop_duplicates()` |
| Group & aggregate | `df.groupby('col').agg(...)` |
| Merge tables | `pd.merge(df1, df2, on='key', how='left')` |
| Pivot | `pd.pivot_table(df, values=, index=, columns=)` |
| Sort | `df.sort_values('col')` |
| Apply function | `df['col'].apply(func)` |
| Convert dtype | `df['col'].astype(type)` |
| Save to CSV | `df.to_csv('out.csv', index=False)` |

---

> [!tip] See also
> [[NumPy_Cheat_Sheet]] · [[NumPy_Advanced_Notes]] · [[Python_File_Handling_Notes]]
