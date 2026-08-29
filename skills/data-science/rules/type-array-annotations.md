---
title: Annotate Arrays and Frames with Precise Types
impact: MEDIUM
impactDescription: Type checkers catch dtype misuse; signatures document what a function accepts
tags: [numpy, pandas, typing, type-hints, mypy]
---

# Annotate Arrays and Frames with Precise Types [MEDIUM]

## Description
`coding-standards/doc-type-hints` requires type hints on public APIs; for numeric code the useful information is *what kind* of array. `np.ndarray` alone says nothing about element type, and a bare `pd.DataFrame` says nothing about columns. Use `numpy.typing.NDArray[dtype]` for arrays, `ArrayLike` for inputs you will normalize with `np.asarray`, and `pandas-stubs` plus pandera `DataFrame[Schema]` hints for frames, so mypy and readers can see the contract.

## Bad Example
```python
import numpy as np
import pandas as pd


def cosine_similarity(a, b):
    return a @ b / (np.linalg.norm(a) * np.linalg.norm(b))


def top_customers(df: pd.DataFrame, n: int) -> pd.DataFrame:
    return df.nlargest(n, "revenue")  # which columns must df have?


def load_matrix(path) -> np.ndarray:  # dtype? shape?
    ...
```

## Good Example
```python
import numpy as np
import numpy.typing as npt
from pandera.typing import DataFrame

from .schemas import CustomerSchema


def cosine_similarity(
    a: npt.NDArray[np.float64],
    b: npt.NDArray[np.float64],
) -> np.float64:
    """Cosine similarity of two 1-D vectors of equal length."""
    return a @ b / (np.linalg.norm(a) * np.linalg.norm(b))


def to_float32(values: npt.ArrayLike) -> npt.NDArray[np.float32]:
    """Accept anything array-like, return a float32 array."""
    return np.asarray(values, dtype=np.float32)


def top_customers(df: DataFrame[CustomerSchema], n: int) -> DataFrame[CustomerSchema]:
    return df.nlargest(n, "revenue")


def load_matrix(path: str) -> npt.NDArray[np.float32]:
    """Load an (n_samples, n_features) float32 matrix."""
    ...
```

## Notes
- `NDArray[np.float64]` is `np.ndarray[Any, np.dtype[np.float64]]`; the shape parameter is unchecked by mypy, so document expected shape in the docstring (`(n_samples, n_features)`).
- Use `npt.ArrayLike` for parameters and `NDArray[...]` for return types: be liberal in what you accept, precise in what you return.
- Install `pandas-stubs` as a dev dependency; without it most of pandas is `Any` to mypy.
- Prefer `np.floating[Any]` / `np.integer[Any]` over concrete widths when a function genuinely works for any float or int dtype.
- For frames, a pandera `DataFrameModel` doubles as the type hint and the runtime validator (see `schema-pandera`); do not maintain a separate `TypedDict` of columns.

## References
- [NumPy - Typing (numpy.typing)](https://numpy.org/doc/stable/reference/typing.html)
- [pandas-stubs](https://github.com/pandas-dev/pandas-stubs)
