---
title: Build Arrays and DataFrames Once, Not Incrementally
impact: CRITICAL
impactDescription: O(n) instead of O(n²); avoids a full copy on every append
tags: [vectorization, numpy, pandas, concat, append, memory]
---

# Build Arrays and DataFrames Once, Not Incrementally [CRITICAL]

## Description
`np.ndarray` and `pd.DataFrame` are fixed-size, contiguous buffers. `np.append`, `np.concatenate`, `pd.concat`, or `df.loc[len(df)] = row` inside a loop allocates and copies the entire structure on every iteration, giving quadratic time and memory churn. Collect results in a Python list (or preallocate a NumPy array of known size) and construct the final object once.

## Bad Example
```python
import numpy as np
import pandas as pd

# Copies the whole array every iteration
samples = np.array([])
for _ in range(10_000):
    samples = np.append(samples, draw_sample())

# Copies the whole DataFrame every iteration
results = pd.DataFrame(columns=["id", "score"])
for item in items:
    results = pd.concat([results, pd.DataFrame([{"id": item.id, "score": score(item)}])])

# Row assignment by growing index also reallocates
for item in items:
    results.loc[len(results)] = [item.id, score(item)]
```

## Good Example
```python
import numpy as np
import pandas as pd

# Known size: preallocate and fill
samples = np.empty(10_000)
for i in range(10_000):
    samples[i] = draw_sample()

# Unknown size: collect in a list, build once
chunks = [load_chunk(path) for path in paths]
data = np.concatenate(chunks)

# DataFrame from a list of records
records = [{"id": item.id, "score": score(item)} for item in items]
results = pd.DataFrame.from_records(records)

# Many DataFrames: one concat at the end
frames = [pd.read_csv(path) for path in paths]
combined = pd.concat(frames, ignore_index=True)
```

## Notes
- Prefer generating the whole array in one vectorized call (`rng.normal(size=n)`, `np.arange`, `np.linspace`) over filling it in a loop at all.
- `pd.DataFrame(records)` from a list of dicts infers dtypes once; pass `dtype=`/`astype()` afterwards if inference is not what you want (see `dtype-explicit-io`).
- `pd.concat` accepts any iterable, so a generator expression works when the frames are large.
- For streaming data that truly never ends, batch: accumulate N rows in a list, flush to a DataFrame/Parquet, then clear the list.
- `DataFrame.append` was removed in pandas 2.0; code that still uses it should switch to the list-then-concat pattern.

## References
- [pandas User Guide - Merge, join, concatenate](https://pandas.pydata.org/docs/user_guide/merging.html)
- [NumPy - numpy.concatenate](https://numpy.org/doc/stable/reference/generated/numpy.concatenate.html)
