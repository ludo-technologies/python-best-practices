---
title: Choose NumPy Dtypes Explicitly
impact: HIGH
impactDescription: Prevents silent integer overflow and precision loss; halves memory with float32 where appropriate
tags: [numpy, dtype, overflow, precision, memory]
---

# Choose NumPy Dtypes Explicitly [HIGH]

## Description
NumPy picks a dtype from the input when you do not say otherwise: `np.array([1, 2])` is `int64`, `np.zeros(n)` is `float64`, and values read from files or other libraries can arrive as `int32`, `uint8`, or `float16`. Fixed-width integers wrap around silently on overflow, and mixing widths promotes in ways that are easy to misjudge. State the dtype at construction and at the boundary where data enters, and check it in tests.

## Bad Example
```python
import numpy as np

pixels = load_image()             # uint8 from the decoder
brightness = pixels + 100         # wraps: 200 + 100 -> 44

counts = np.array(hist, dtype=np.int32)
total = counts * 1_000_000        # overflows silently past 2^31

embeddings = np.zeros((n, 768))   # float64, twice the memory the model needs

ids = np.array(raw_ids)           # int64? object? depends on the input
```

## Good Example
```python
import numpy as np

pixels = load_image().astype(np.int16)     # widen before arithmetic
brightness = np.clip(pixels + 100, 0, 255).astype(np.uint8)

counts = np.asarray(hist, dtype=np.int64)
total = counts * 1_000_000

embeddings = np.zeros((n, 768), dtype=np.float32)

ids = np.asarray(raw_ids, dtype=np.int64)

# Assert assumptions at boundaries
assert embeddings.dtype == np.float32, embeddings.dtype
```

## Notes
- Overflow on NumPy integer arrays does not raise; it wraps. Scalar operations may warn, array operations will not. Use a wider type where the range is uncertain.
- `float32` is usually sufficient for ML features and halves memory/bandwidth; keep `float64` for accumulations, statistics, and anything numerically sensitive.
- Use `np.asarray(x, dtype=...)` at function entry to normalize inputs from lists, other libraries, or pandas; it avoids a copy when the dtype already matches.
- Avoid `dtype=object` arrays: they hold Python objects and lose vectorization entirely. If you have one, it usually means ragged data or mixed types that should be modeled differently.
- Since NumPy 2.0, type promotion follows NEP 50: a Python scalar adapts to the array dtype (`uint8_array + 100` stays `uint8`), so the wraparound in the bad example is exactly what happens.

## References
- [NumPy - Data types](https://numpy.org/doc/stable/user/basics.types.html)
- [NEP 50 - Promotion rules for Python scalars](https://numpy.org/neps/nep-0050-scalar-promotion.html)
