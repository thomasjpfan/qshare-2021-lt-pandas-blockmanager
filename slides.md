title: A Shallow Dive into Pandas' BlockManager
use_katex: False
class: title-slide

# A Shallow Dive into Pandas' BlockManager 🐼

.larger[Thomas J. Fan]<br>
@thomasjpfan<br>
<a href="https://www.github.com/thomasjpfan" target="_blank"><span class="icon icon-github icon-left"></span></a>
<a href="https://www.twitter.com/thomasjpfan" target="_blank"><span class="icon icon-twitter"></span></a>
<a class="this-talk-link", href="https://github.com/thomasjpfan/qshare-2021-lt-pandas-blockmanager" target="_blank">
This talk on Github: thomasjpfan/qshare-2021-lt-pandas-blockmanager</a>

---

class: middle, center

# Pandas users?

.grid[
.grid-6[.success.bold.center[
    Press Y for Yes
]]
.grid-6[.alert.bold.center[
    Press N for No
]]
]

---

# Lets create a DataFrame

```py
df = pd.DataFrame({
    'int_1': np.arange(1000000, dtype=int),
    'float_1': np.arange(1000000, dtype=float),
    'int_2': np.arange(10, 1000010, dtype=int),
    'bool_1': np.ones(1000000, dtype=bool),
    'int_3': np.arange(20, 1000020, dtype=int),
    'float_2': np.arange(10, 1000010, dtype=float),
})

df.head(2)
```

|    |   int_1 |   float_1 |   int_2 | bool_1   |   int_3 |   float_2 |
|---:|--------:|----------:|--------:|:---------|--------:|----------:|
|  0 |       0 |         0 |      10 | True     |      20 |        10 |
|  1 |       1 |         1 |      11 | True     |      21 |        11 |

---

# How is this stored?

.grid-row[
.grid-6[
|    |   int_1 |   float_1 |   int_2 | bool_1   |   int_3 |   float_2 |
|---:|--------:|----------:|--------:|:---------|--------:|----------:|
|  0 |       0 |         0 |      10 | True     |      20 |        10 |
|  1 |       1 |         1 |      11 | True     |      21 |        11 |
]
.grid-6[
    ![](images/blockmanager-1.png)
]
]

---

# Pandas API For BlockManager

```py
>>>  df._data

BlockManager
Items: Index(['int_1', 'float_1', 'int_2', 'bool_1', 'int_3', 'float_2'], dtype='object')
Axis 1: RangeIndex(start=0, stop=1000000, step=1)
FloatBlock: slice(1, 9, 4), 2 x 1000000, dtype: float64
IntBlock: slice(0, 6, 2), 3 x 1000000, dtype: int64
BoolBlock: slice(3, 4, 1), 1 x 1000000, dtype: bool
```

```py
>>> df._data.nblocks
3
```
---

# Query the dataframe

A view!

```py
>>> df['int_1'].values
array([     0,      1,      2, ..., 999997, 999998, 999999])
```

Underlying numpy array

```py
>>> df['int_1'].values.base
array([[      0,       1,       2, ...,  999997,  999998,  999999],
       [     10,      11,      12, ..., 1000007, 1000008, 1000009],
       [     20,      21,      22, ..., 1000017, 1000018, 1000019]])
```

---

# Add a new column

Recall


.grid[
.grid-6[
```py
>>> df._data.nblocks
3
```

Add new column

```py
df['int_4'] = df['int_1'].values
```

How many blocks now?

```py
>>> df._data.nblocks
4
```

]
.grid-6[
![](images/blockmanager-2.png)
]
]

---

# Add another column for fun

```py
>>> %time df['int_5'] = df['int_1'].values
```

How long does it take?

1. 1.3 seconds
2. 6.1 milliseconds

.grid[
.grid-6[.success.bold.center[
    Press 1 for 1.3 seconds
]]
.grid-6[.alert.bold.center[
    Press 2 for 6.1 milliseconds
]]
]

---

# Add another column for fun

```py
>>> %time df['int_5'] = df['int_1'].values
```

How long does it take?

1. 1.3 seconds<br>
2. 6.1 milliseconds

.grid[
.grid-12[.alert.bold.center[
    Press 2 for 6.1 milliseconds
]]
]

```
Wall time: 6.18 ms
```

```py
>>> df._data.nblocks
5
```

---

# Add 94 more columns

```py
%%time
for i in range(94):
    df[f'int_{i + 6}'] = df['int_1'].values

# Wall time: 560 ms
```

```py
>>> df._data.nblocks
100
```

.width-70[
![](images/blockmanager-scatter.png)
]

---

# Add one more int column

```py
%time df['int_101'] = df['int_1'].values
```

How long does it take?

A. 1.3 seconds<br>
B. 6.1 milliseconds

.grid[
.grid-6[.success.bold.center[
    Press A for 1.3 seconds
]]
.grid-6[.alert.bold.center[
    Press B for 6.1 milliseconds
]]
]

---

# Add one more int column

```py
%time df['int_101'] = df['int_1'].values
```

How long does it take?

A. 1.3 seconds<br>
B. 6.1 milliseconds

.grid[
.grid-12[.success.bold.center[
    Press A for 1.3 seconds
]]
]

```
Wall time: 1.3 s
```

---


# What happened? (Consolidation!)

```py
>>> df._data.nblocks
3
```

![](images/blockmanager-3.png)

---

class: title-slide

# When does pandas consolidate? 🤔

---

# Grep the source!

```bash
git grep -E "\._?consolidate" -- "*.py" :"^pandas/tests" :"^asv_bench"
```

```
pandas/core/frame.py:        self._consolidate_inplace()
pandas/core/generic.py:        self._consolidate_inplace()
pandas/core/generic.py:        self._consolidate_inplace()
pandas/core/generic.py:        self._consolidate_inplace()
pandas/core/generic.py:            self._mgr = self._mgr.consolidate()
pandas/core/generic.py:        f = lambda: self._mgr.consolidate()
pandas/core/generic.py:        self._consolidate_inplace()
pandas/core/generic.py:        self._consolidate_inplace()
pandas/core/generic.py:        self._consolidate_inplace()
pandas/core/groupby/generic.py:            result = result._consolidate()
pandas/core/groupby/generic.py:            result = result._consolidate()
pandas/core/groupby/generic.py:            result = result._consolidate()
pandas/core/internals/managers.py:        bm._consolidate_inplace()
...
```

---

class: middle

# Conclusion

.larger[
- Performance of DataFrame operations depends on context
- [Block Manager Rewrite](https://pandas.pydata.org/docs/development/roadmap.html#block-manager-rewrite)
- Implications for scikit-learn - [array out prototype](https://github.com/scikit-learn/scikit-learn/pull/16772)
]
