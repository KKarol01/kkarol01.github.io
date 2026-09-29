---
title: "False Sharing: When Your Threads Fight Over a Cache Line"
date: 2026-09-20T10:00:00Z
draft: false
description: "Why two independent counters can make a multithreaded loop 10x slower, and how padding fixes it."
tags: ["cpu", "cache", "concurrency", "c++"]
---

Two threads, two counters, zero shared state — and yet the program runs slower
than the single-threaded version. The culprit is almost always **false sharing**.

<!--more-->

## The setup

```cpp
struct Counters {
    std::atomic<uint64_t> a;
    std::atomic<uint64_t> b;
};

Counters c;

void worker_a() { for (int i = 0; i < 100'000'000; ++i) c.a.fetch_add(1, std::memory_order_relaxed); }
void worker_b() { for (int i = 0; i < 100'000'000; ++i) c.b.fetch_add(1, std::memory_order_relaxed); }
```

Logically `a` and `b` are unrelated. Physically they sit 8 bytes apart, which
puts them on the same 64-byte cache line.

## What the hardware does

Cache coherence (MESI and friends) works on whole lines, not variables. Every
time core 0 writes `a`, it must own the line exclusively, which invalidates
core 1's copy. Core 1 then writes `b`, steals the line back, and so on. The line
ping-pongs between cores on every single increment.

## The fix

Give each hot variable its own line:

```cpp
struct Counters {
    alignas(std::hardware_destructive_interference_size) std::atomic<uint64_t> a;
    alignas(std::hardware_destructive_interference_size) std::atomic<uint64_t> b;
};
```

On my (entirely imaginary) benchmark machine this took the loop from
**1.9 s** down to **0.18 s**.

## Takeaways

- Profile with `perf c2c` — it points straight at contended lines.
- Pad per-thread data, or better, keep it in thread-local storage and merge at the end.
- Remember that 64 bytes is typical, but Apple M-series cores use 128-byte lines.
