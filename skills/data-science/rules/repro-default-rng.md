---
title: Use an Explicit Generator for Randomness
impact: HIGH
impactDescription: Reproducible experiments; no hidden global state shared across modules and tests
tags: [numpy, random, reproducibility, seed, generator]
---

# Use an Explicit Generator for Randomness [HIGH]

## Description
`np.random.seed()` and the module-level functions (`np.random.rand`, `np.random.choice`, ...) share one hidden global `RandomState`. Any library, test, or helper that touches it changes the sequence every caller sees, so results depend on call order and stop being reproducible the moment code is refactored or parallelized. The legacy API is also frozen at an older, slower algorithm. Create a `Generator` with `np.random.default_rng(seed)` at the composition root and pass it to whatever needs randomness, exactly as `coding-standards/design-no-global-singleton` prescribes for other shared state.

## Bad Example
```python
import numpy as np

np.random.seed(42)  # global, affects every module in the process


def make_split(n: int) -> np.ndarray:
    return np.random.permutation(n)


def add_noise(x: np.ndarray) -> np.ndarray:
    return x + np.random.normal(scale=0.1, size=x.shape)


# Order-dependent: swapping these two lines changes both results
train_idx = make_split(1000)
noisy = add_noise(features)
```

## Good Example
```python
import numpy as np


def make_split(n: int, rng: np.random.Generator) -> np.ndarray:
    return rng.permutation(n)


def add_noise(x: np.ndarray, rng: np.random.Generator) -> np.ndarray:
    return x + rng.normal(scale=0.1, size=x.shape)


def run_experiment(seed: int) -> Result:
    rng = np.random.default_rng(seed)
    # Independent streams so each stage is reproducible on its own
    split_rng, noise_rng = rng.spawn(2)
    train_idx = make_split(1000, split_rng)
    noisy = add_noise(features, noise_rng)
    ...


# Tests inject a fixed generator
def test_add_noise_preserves_shape() -> None:
    rng = np.random.default_rng(0)
    assert add_noise(np.zeros((3, 4)), rng).shape == (3, 4)
```

## Notes
- `Generator` methods (`rng.integers`, `rng.random`, `rng.choice`, `rng.normal`) replace the legacy `np.random.*` functions one-to-one; note `integers` instead of `randint`.
- Use `rng.spawn(n)` (NumPy 1.25+) or `SeedSequence` to derive independent child generators for parallel workers or pipeline stages; never reuse one seed across workers.
- Pass the `Generator` explicitly or accept `seed: int | None` and call `default_rng(seed)` at the top of the entry point. `default_rng(None)` draws fresh entropy, which is the right default for production.
- Other libraries have their own state: seed `random`, `torch`, and `tensorflow` explicitly and independently; a NumPy seed does not cover them.
- pandas `sample(random_state=rng)` accepts a `Generator`. scikit-learn's `random_state=` does not; derive an int from the generator (`int(rng.integers(2**32))`) so the whole run still descends from one seed.

## References
- [NumPy - Random Generator](https://numpy.org/doc/stable/reference/random/generator.html)
- [NumPy - Parallel random number generation](https://numpy.org/doc/stable/reference/random/parallel.html)
