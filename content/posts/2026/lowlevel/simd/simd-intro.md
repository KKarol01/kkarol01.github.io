---
title: "SIMD by Hand: Vectorizing a Dot Product"
date: 2026-09-27T10:00:00Z
draft: false
description: "Writing AVX2 intrinsics for a dot product and seeing where the compiler already beats you."
tags: ["cpu", "simd", "avx2", "c++"]
---

The compiler auto-vectorizes a lot, but not everything. Writing one kernel by
hand is the fastest way to learn what it can and can't do for you.

<!--more-->

## The scalar baseline

```cpp
float dot(const float* a, const float* b, size_t n) {
    float sum = 0.0f;
    for (size_t i = 0; i < n; ++i) sum += a[i] * b[i];
    return sum;
}
```

Without `-ffast-math` the compiler must keep the additions in order, so this
loop stays scalar.

## The AVX2 version

```cpp
#include <immintrin.h>

float dot_avx2(const float* a, const float* b, size_t n) {
    __m256 acc = _mm256_setzero_ps();
    size_t i = 0;
    for (; i + 8 <= n; i += 8)
        acc = _mm256_fmadd_ps(_mm256_loadu_ps(a + i), _mm256_loadu_ps(b + i), acc);

    alignas(32) float lanes[8];
    _mm256_store_ps(lanes, acc);
    float sum = 0.0f;
    for (float l : lanes) sum += l;
    for (; i < n; ++i) sum += a[i] * b[i];
    return sum;
}
```

## Results (made up, as usual)

| Version        | 1M floats |
|----------------|-----------|
| Scalar         | 0.92 ms   |
| AVX2, 1 acc    | 0.21 ms   |
| AVX2, 4 accs   | 0.09 ms   |

The last row is the trick: one accumulator serializes on FMA latency. Use four
independent accumulators and the CPU can keep several FMAs in flight.
