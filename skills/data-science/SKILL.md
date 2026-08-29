---
name: data-science
description: NumPy and pandas best practices for data science and ML code covering vectorization, DataFrame indexing and mutation under Copy-on-Write, dtypes, schema validation with pandera, reproducible randomness, transformation style, array typing, and numeric testing. Use when writing, reviewing, or refactoring code that uses numpy, pandas, DataFrames, ndarrays, Jupyter notebooks, or data pipelines, or when the user asks about pandas/numpy performance or idioms.
paths:
  - "**/*.ipynb"
  - "**/notebooks/**"
---

# Python Data Science

NumPy and pandas best practices for data science and ML code. Designed for AI agents and LLMs to write fast, correct, and reproducible numeric code.

Numeric code has idioms that differ from general Python: loops are the anti-pattern, data structures are immutable buffers, and dtypes are a design decision. Where a rule here overlaps a `coding-standards` rule, this skill takes precedence for array and DataFrame code.

## Categories

### Vectorization [CRITICAL]
Keep computation in compiled NumPy/pandas code instead of the Python interpreter.

| Rule | Description |
|------|-------------|
| [vec-no-python-loops](rules/vec-no-python-loops.md) | Vectorize instead of Python loops, iterrows, and row-wise apply (10-1000x faster) |
| [vec-no-incremental-growth](rules/vec-no-incremental-growth.md) | Build arrays and DataFrames once, not incrementally (O(n) vs O(n²)) |

### Indexing & Mutation [CRITICAL]
Write to DataFrames in ways that are correct under Copy-on-Write and visible in the data flow.

| Rule | Description |
|------|-------------|
| [mut-loc-not-chained](rules/mut-loc-not-chained.md) | Assign with .loc, never with chained indexing |
| [mut-no-inplace](rules/mut-no-inplace.md) | Avoid inplace=True |

### Data Types [HIGH]
Make dtypes an explicit decision rather than an inference result.

| Rule | Description |
|------|-------------|
| [dtype-explicit-io](rules/dtype-explicit-io.md) | Specify dtypes when loading data |
| [dtype-categorical](rules/dtype-categorical.md) | Use category dtype for low-cardinality strings |
| [dtype-numpy-explicit](rules/dtype-numpy-explicit.md) | Choose NumPy dtypes explicitly |

### Schema Validation [HIGH]
Validate tabular data where it enters the system.

| Rule | Description |
|------|-------------|
| [schema-pandera](rules/schema-pandera.md) | Validate DataFrames at boundaries with pandera |

### Reproducibility [HIGH]
Make experiments repeatable and free of hidden global state.

| Rule | Description |
|------|-------------|
| [repro-default-rng](rules/repro-default-rng.md) | Use an explicit Generator for randomness |

### Transformation Style [MEDIUM]
Write DataFrame transformations that read top-to-bottom as a single pipeline.

| Rule | Description |
|------|-------------|
| [style-method-chaining](rules/style-method-chaining.md) | Express transformations as method chains |
| [style-named-aggregation](rules/style-named-aggregation.md) | Use named aggregation in groupby |

### Typing [MEDIUM]
Type hints that carry dtype and column information.

| Rule | Description |
|------|-------------|
| [type-array-annotations](rules/type-array-annotations.md) | Annotate arrays and frames with precise types |

### Testing [MEDIUM]
Assertions that fit floating-point arrays and DataFrames.

| Rule | Description |
|------|-------------|
| [test-numeric-assertions](rules/test-numeric-assertions.md) | Compare numeric results with tolerance-aware assertions |

## Quick Reference

### Vectorization
```python
# Whole-array operations, not loops / iterrows / apply(axis=1)
df["total"] = df["price"] * df["qty"]
rate = np.select([df["qty"] >= 100, df["qty"] >= 10], [0.2, 0.1], default=0.0)

# Collect, then build once
records = [{"id": item.id, "score": score(item)} for item in items]
results = pd.DataFrame.from_records(records)
combined = pd.concat(frames, ignore_index=True)
```

### Indexing & Mutation
```python
# .loc in one step; chained assignment never reaches df under Copy-on-Write
df.loc[df["status"] == "pending", "priority"] = "high"

# Returning form, not inplace=True
df = df.dropna(subset=["email"]).sort_values("created_at")
```

### Data Types
```python
df = pd.read_csv(
    path,
    dtype={"customer_id": "Int64", "zip_code": "string", "region": "category"},
    parse_dates=["created_at"],
)
SEVERITY = pd.CategoricalDtype(["low", "medium", "high"], ordered=True)
embeddings = np.zeros((n, 768), dtype=np.float32)
```

### Schema Validation
```python
import pandera.pandas as pa
from pandera.typing import DataFrame, Series


class OrderSchema(pa.DataFrameModel):
    order_id: Series[int] = pa.Field(unique=True)
    price: Series[float] = pa.Field(ge=0)

    class Config:
        strict = True


def load_orders(path: str) -> DataFrame[OrderSchema]:
    return OrderSchema.validate(pd.read_csv(path))
```

### Reproducibility
```python
def run_experiment(seed: int) -> Result:
    rng = np.random.default_rng(seed)
    split_rng, noise_rng = rng.spawn(2)
    ...

def add_noise(x: np.ndarray, rng: np.random.Generator) -> np.ndarray:
    return x + rng.normal(scale=0.1, size=x.shape)
```

### Transformation Style
```python
monthly = (
    pd.read_csv("sales.csv", parse_dates=["date"])
    .assign(
        revenue=lambda d: d["price"] * d["qty"],
        month=lambda d: d["date"].dt.to_period("M"),
    )
    .loc[lambda d: d["revenue"] > 0]
    .groupby("month", as_index=False)
    .agg(total_revenue=("revenue", "sum"), orders=("order_id", "count"))
)
```

### Typing
```python
import numpy.typing as npt

def to_float32(values: npt.ArrayLike) -> npt.NDArray[np.float32]:
    return np.asarray(values, dtype=np.float32)
```

### Testing
```python
np.testing.assert_allclose(result, expected, rtol=1e-6, atol=1e-12)
pd.testing.assert_frame_equal(result_df, expected_df, check_like=True)
```

## See Also

- [coding-standards](../coding-standards/SKILL.md) - General Python rules; `validation-pydantic`, `design-no-global-singleton`, and `doc-type-hints` have direct counterparts here
- [testing](../testing/SKILL.md) - pytest structure, fixtures, and mocking that numeric tests still follow
