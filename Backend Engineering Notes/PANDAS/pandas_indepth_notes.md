# Pandas In-Depth Notes

## 1. Introduction to Pandas

Pandas is an open-source Python library built on top of NumPy, used for data manipulation and analysis. The name comes from "panel data," a term from econometrics for multi-dimensional structured datasets. Pandas was originally created by Wes McKinney in 2008 and open-sourced in 2010.

**Why Pandas is popular:**
- Handles large, messy, real-world datasets efficiently
- Two core data structures: `Series` (1D) and `DataFrame` (2D)
- Reads/writes many file formats: CSV, Excel, SQL, JSON, Parquet, HTML
- Built-in tools for cleaning, filtering, grouping, merging, and reshaping data
- Integrates well with NumPy, Matplotlib, scikit-learn, and other data-science libraries

**Installation:**
```bash
pip install pandas
```

**Standard import convention:**
```python
import pandas as pd
import numpy as np
```

---

## 2. Core Data Structures

### 2.1 Series — 1D labeled array

A `Series` is a one-dimensional array-like object that can hold any data type (int, float, string, Python objects). Every value has an associated label called the **index**.

```python
import pandas as pd

data = [10, 20, 30, 40]
s = pd.Series(data)
print(s)
```
Output has a default index `0, 1, 2, 3` unless you specify one:
```python
s = pd.Series(data, index=['a', 'b', 'c', 'd'])
```

A Series can also be built from a dictionary (keys become the index) or a NumPy array.

### 2.2 DataFrame — 2D labeled table

A `DataFrame` is a two-dimensional, size-mutable, tabular structure with labeled rows (index) and columns — conceptually similar to an Excel sheet or SQL table.

```python
data = {
    'Name': ['Alice', 'Bob', 'Charlie'],
    'Age': [25, 30, 35],
    'City': ['Delhi', 'Mumbai', 'Pune']
}
df = pd.DataFrame(data)
print(df)
```

DataFrames can also be created from:
- A list of dictionaries
- A list of lists / 2D NumPy array (with `columns=` specified)
- Another DataFrame or Series
- External files (CSV, Excel, SQL, JSON)

```python
# From list of lists
df = pd.DataFrame([[1, 'A'], [2, 'B']], columns=['ID', 'Label'])

# From list of dicts
df = pd.DataFrame([{'a': 1, 'b': 2}, {'a': 3, 'b': 4}])
```

---

## 3. Reading and Writing Data

| Format | Read | Write |
|---|---|---|
| CSV | `pd.read_csv('file.csv')` | `df.to_csv('file.csv', index=False)` |
| Excel | `pd.read_excel('file.xlsx')` | `df.to_excel('file.xlsx', index=False)` |
| JSON | `pd.read_json('file.json')` | `df.to_json('file.json')` |
| SQL | `pd.read_sql(query, connection)` | `df.to_sql('table', connection)` |
| HTML | `pd.read_html('url')` | `df.to_html('file.html')` |
| Parquet | `pd.read_parquet('file.parquet')` | `df.to_parquet('file.parquet')` |
| Clipboard | `pd.read_clipboard()` | `df.to_clipboard()` |

Useful `read_csv` parameters: `sep`, `header`, `names`, `index_col`, `usecols`, `dtype`, `na_values`, `parse_dates`, `nrows`, `skiprows`, `encoding`.

---

## 4. Inspecting Data

```python
df.head(n)        # first n rows (default 5)
df.tail(n)        # last n rows
df.shape           # (rows, columns)
df.info()          # dtypes, non-null counts, memory usage
df.describe()      # summary statistics (numeric columns by default)
df.columns         # list of column names
df.index           # index object
df.dtypes          # data type of each column
df.values           # underlying NumPy array
df.memory_usage()   # memory usage per column
df.nunique()        # number of unique values per column
```

---

## 5. Selection and Indexing

Pandas offers several ways to select data. Understanding the difference between label-based and position-based indexing is essential.

| Method | Basis | Example |
|---|---|---|
| `df['col']` | Column selection | Returns a Series |
| `df[['col1','col2']]` | Multiple columns | Returns a DataFrame |
| `df.loc[]` | **Label**-based row/column selection | `df.loc[2, 'Name']` |
| `df.iloc[]` | **Position**-based (integer) selection | `df.iloc[2, 0]` |
| `df.at[]` | Fast scalar access by label | `df.at[2, 'Name']` |
| `df.iat[]` | Fast scalar access by position | `df.iat[2, 0]` |

```python
df.loc[1]              # row with label/index 1
df.loc[1:3]             # rows 1 through 3, inclusive of 3 (label-based!)
df.loc[:, 'Age']         # all rows, column 'Age'
df.loc[df['Age'] > 25]    # conditional (Boolean) selection

df.iloc[1]              # row at position 1
df.iloc[0:3]             # rows 0,1,2 (position-based, exclusive of end)
df.iloc[:, 0:2]           # all rows, first two columns
```

**Key distinction:** `.loc` slicing is *inclusive* of the end label; `.iloc` slicing follows Python's normal *exclusive* end convention.

### 5.1 Boolean (Conditional) Indexing / Filtering

```python
df[df['Age'] > 25]
df[(df['Age'] > 25) & (df['City'] == 'Delhi')]   # use & | ~ with parentheses
df[df['Name'].isin(['Alice', 'Bob'])]
df.query('Age > 25 and City == "Delhi"')          # alternative syntax
```

### 5.2 Setting values with `.loc` / `.iloc`

```python
df.loc[df['Age'] > 30, 'Category'] = 'Senior'
df.iloc[0, 1] = 99
```

---

## 6. Handling Missing Data

Missing values appear as `NaN` (Not a Number). Pandas provides dedicated tools to detect, remove, or fill them.

```python
df.isnull()          # Boolean mask, True where value is missing
df.isnull().sum()     # count of missing values per column
df.notnull()          # opposite of isnull()

df.dropna()                        # drop rows with ANY missing value
df.dropna(axis=1)                   # drop columns with any missing value
df.dropna(subset=['Age'])            # drop rows missing values in specific columns
df.dropna(how='all')                 # drop rows only if ALL values are missing

df.fillna(0)                        # replace all NaN with 0
df.fillna(method='ffill')            # forward-fill (propagate last valid value)
df.fillna(method='bfill')            # backward-fill
df['Age'].fillna(df['Age'].mean())    # fill with column mean
df.interpolate()                     # interpolate missing numeric values
```

---

## 7. Adding, Removing, and Modifying Columns/Rows

```python
# Add a column
df['Total'] = df['Price'] * df['Quantity']

# Add a column conditionally
df['Status'] = df['Age'].apply(lambda x: 'Adult' if x >= 18 else 'Minor')

# Remove a column
df.drop('City', axis=1, inplace=True)
df.drop(columns=['City', 'Age'])

# Remove a row
df.drop(index=2)
df.drop(0, axis=0)

# Rename columns
df.rename(columns={'Name': 'FullName'}, inplace=True)

# Reorder columns
df = df[['Age', 'Name', 'City']]

# Insert a column at a specific position
df.insert(1, 'NewCol', value=0)
```

`inplace=True` modifies the DataFrame directly and returns `None`; the general best practice is to avoid `inplace` on large or shared DataFrames and instead reassign to a new/same variable, since it makes chains harder to debug and doesn't always save memory.

---

## 8. Data Types and Conversion

```python
df.dtypes
df['Age'] = df['Age'].astype(int)
df['Date'] = pd.to_datetime(df['Date'])
df['Category'] = df['Category'].astype('category')   # memory-efficient for repeated strings
pd.to_numeric(df['col'], errors='coerce')              # invalid parsing -> NaN
```

---

## 9. Sorting

```python
df.sort_values('Age')                         # ascending by default
df.sort_values('Age', ascending=False)
df.sort_values(['City', 'Age'])                 # multi-column sort
df.sort_index()                                # sort by index labels
df.rank()                                       # rank values
```

---

## 10. GroupBy and Aggregation

`groupby()` splits data into groups based on column values, applies a function to each group, and combines the results — the classic "split-apply-combine" pattern.

```python
df.groupby('Category')['Value'].sum()
df.groupby('Category').mean(numeric_only=True)
df.groupby('Category').agg({'Value': ['sum', 'mean'], 'Quantity': 'count'})
df.groupby(['Category', 'City']).size()          # count per group combination

# Custom aggregation
df.groupby('Category')['Value'].agg(lambda x: x.max() - x.min())

# Iterating over groups
for name, group in df.groupby('Category'):
    print(name)
    print(group)
```

Common aggregate functions: `sum()`, `mean()`, `median()`, `min()`, `max()`, `count()`, `std()`, `var()`, `nunique()`, `first()`, `last()`.

---

## 11. Merging, Joining, and Concatenating

### 11.1 `merge()` — SQL-style joins
```python
pd.merge(df1, df2, on='ID', how='inner')    # inner, left, right, outer
pd.merge(df1, df2, left_on='id1', right_on='id2')
```
| `how` | Behavior |
|---|---|
| `inner` | Keep only matching keys in both |
| `left` | Keep all rows from left, matched from right |
| `right` | Keep all rows from right, matched from left |
| `outer` | Keep all rows from both, filling unmatched with NaN |

### 11.2 `concat()` — Stack DataFrames
```python
pd.concat([df1, df2])              # stack rows (axis=0 default)
pd.concat([df1, df2], axis=1)       # stack columns side by side
pd.concat([df1, df2], ignore_index=True)  # reset index after stacking
```

### 11.3 `join()` — Combine on index
```python
df1.join(df2, how='left')
```

---

## 12. Reshaping Data

```python
# Pivot: long -> wide
df.pivot(index='Date', columns='City', values='Temp')

# Pivot table (allows aggregation, handles duplicates)
df.pivot_table(index='City', columns='Month', values='Sales', aggfunc='sum')

# Melt: wide -> long
pd.melt(df, id_vars=['Name'], value_vars=['Math', 'Science'], var_name='Subject', value_name='Score')

# Stack/unstack (MultiIndex reshaping)
df.stack()
df.unstack()

# Transpose
df.T
```

---

## 13. String Operations (`.str` accessor)

Pandas exposes vectorized string methods through the `.str` accessor on Series of string data.

```python
df['Name'].str.upper()
df['Name'].str.lower()
df['Name'].str.strip()
df['Name'].str.contains('an')
df['Name'].str.replace('a', 'A')
df['Name'].str.split(' ')
df['Name'].str.len()
df['Name'].str.startswith('A')
```

---

## 14. Date and Time Handling

```python
df['Date'] = pd.to_datetime(df['Date'])
df['Year'] = df['Date'].dt.year
df['Month'] = df['Date'].dt.month
df['Day'] = df['Date'].dt.day
df['Weekday'] = df['Date'].dt.day_name()

pd.date_range(start='2026-01-01', periods=10, freq='D')   # generate a date sequence

df.set_index('Date').resample('M').sum()   # resample time series data (e.g., monthly totals)
```

---

## 15. Applying Functions

```python
df['Age'].apply(lambda x: x + 1)                 # element-wise on a Series
df.apply(lambda row: row['Price'] * row['Qty'], axis=1)   # row-wise on a DataFrame
df.applymap(lambda x: x * 2)                       # element-wise on entire DataFrame (deprecated in favor of df.map in newer versions)
df['Category'].map({'A': 'Alpha', 'B': 'Beta'})     # value substitution via a dict/Series
```

---

## 16. Duplicates

```python
df.duplicated()                 # Boolean mask of duplicate rows
df.drop_duplicates()             # remove duplicate rows
df.drop_duplicates(subset=['Name'], keep='first')   # keep first occurrence only
```

---

## 17. Combining Statistics

```python
df['Age'].mean()
df['Age'].median()
df['Age'].std()
df['Age'].var()
df['Age'].sum()
df['Age'].min(), df['Age'].max()
df['Age'].value_counts()          # frequency count of unique values
df.corr(numeric_only=True)         # correlation matrix
df.cov(numeric_only=True)          # covariance matrix
```

---

## 18. MultiIndex (Hierarchical Indexing)

```python
arrays = [['A', 'A', 'B', 'B'], [1, 2, 1, 2]]
index = pd.MultiIndex.from_arrays(arrays, names=('Letter', 'Number'))
df = pd.DataFrame({'Value': [10, 20, 30, 40]}, index=index)

df.loc['A']                 # select outer level
df.xs(1, level='Number')     # cross-section on inner level
df.reset_index()             # convert index levels back to columns
```

---

## 19. Window and Rolling Functions

```python
df['RollingMean'] = df['Value'].rolling(window=3).mean()
df['CumSum'] = df['Value'].cumsum()
df['Expanding'] = df['Value'].expanding().mean()
df['Shifted'] = df['Value'].shift(1)     # lag by 1 period
df['PctChange'] = df['Value'].pct_change()
```

---

## 20. Useful Settings

```python
pd.set_option('display.max_columns', None)   # show all columns
pd.set_option('display.max_rows', 100)
pd.set_option('display.width', 1000)
pd.reset_option('all')
```

---

## 21. Performance Tips

- Prefer **vectorized operations** (e.g., `df['a'] + df['b']`) over `.apply()` or Python loops — vectorized code runs in optimized C internally and is significantly faster.
- Use `astype('category')` for columns with a small number of repeated string values to reduce memory.
- Avoid chained indexing like `df[df['a']>1]['b'] = 2` (triggers `SettingWithCopyWarning`); use `.loc` instead.
- Read only needed columns with `usecols=` in `read_csv` for large files.
- Use `df.copy()` when you need an independent DataFrame instead of a view.

---

## 22. Quick Reference Cheat Sheet

| Task | Code |
|---|---|
| Create DataFrame | `pd.DataFrame(data)` |
| Read CSV | `pd.read_csv('file.csv')` |
| First rows | `df.head()` |
| Shape | `df.shape` |
| Column info | `df.info()` |
| Filter rows | `df[df['col'] > x]` |
| Select columns | `df[['a','b']]` |
| Handle missing | `df.fillna()` / `df.dropna()` |
| Sort | `df.sort_values('col')` |
| Group & aggregate | `df.groupby('col').sum()` |
| Merge | `pd.merge(df1, df2, on='key')` |
| Pivot | `df.pivot_table(...)` |
| Apply function | `df['col'].apply(func)` |
| Save to CSV | `df.to_csv('out.csv', index=False)` |

---

### Reference
Structured with topic coverage informed by GeeksforGeeks' Pandas tutorial series (geeksforgeeks.org/pandas), rewritten and reorganized in original wording for study notes.
