---
title: "Branchless Code and the Branch Predictor"
date: 2026-08-10T10:00:00Z
draft: false
description: "When replacing an if with arithmetic helps, and when it just makes code harder to read."
tags: ["cpu", "branch-prediction", "performance", "c++"]
---

A mispredicted branch costs roughly 15–20 cycles on a modern core. On
unpredictable data that adds up quickly, which is why "branchless" tricks exist.

<!--more-->

## The classic example

```cpp
long sum_big(const int* v, size_t n) {
    long s = 0;
    for (size_t i = 0; i < n; ++i)
        if (v[i] >= 128) s += v[i];
    return s;
}
```

On sorted input the predictor is nearly perfect. On random input it's right
about half the time.

## Going branchless

```cpp
for (size_t i = 0; i < n; ++i) {
    int mask = -(v[i] >= 128);   // all ones or all zeros
    s += v[i] & mask;
}
```

## When not to bother

- Check the assembly first: compilers often emit `cmov` for you already.
- If the data is predictable, the branch is basically free and often *faster*.
- Branchless code always does both sides of the work. If one side is expensive,
  the branch wins.

Measure with `perf stat -e branch-misses` before and after.
