---
title: "Designing a Compact G-Buffer for Deferred Shading"
date: 2026-09-25T10:00:00Z
draft: false
description: "Packing albedo, normals, roughness and metalness into as few bytes per pixel as possible."
tags: ["rendering", "deferred", "gbuffer", "vulkan"]
---

Deferred shading trades memory bandwidth for flexible lighting. The G-buffer is
where that bandwidth goes, so every byte per pixel counts.

<!--more-->

## A naive layout

| Target | Format              | Contents                | Bytes |
|--------|---------------------|-------------------------|-------|
| RT0    | `R32G32B32A32_SFLOAT` | world position          | 16    |
| RT1    | `R32G32B32A32_SFLOAT` | normal                  | 16    |
| RT2    | `R8G8B8A8_UNORM`      | albedo + alpha          | 4     |
| RT3    | `R8G8B8A8_UNORM`      | roughness, metal, AO    | 4     |

That's **40 bytes per pixel** — about 330 MB of traffic per frame at 4K before
you've shaded anything.

## Cutting it down

1. **Drop position entirely.** Reconstruct it from the depth buffer and the
   inverse view-projection matrix.
2. **Octahedral normals.** Encode the unit normal into two 16-bit channels
   (`R16G16_SNORM`) with negligible visible error.
3. **Pack material params** into the spare channels.

```glsl
vec2 octEncode(vec3 n) {
    n /= abs(n.x) + abs(n.y) + abs(n.z);
    vec2 p = n.z >= 0.0 ? n.xy : (1.0 - abs(n.yx)) * sign(n.xy);
    return p;
}
```

## The result

| Target | Format            | Contents                      | Bytes |
|--------|-------------------|-------------------------------|-------|
| RT0    | `R8G8B8A8_SRGB`     | albedo, AO                    | 4     |
| RT1    | `R16G16_SNORM`      | octahedral normal             | 4     |
| RT2    | `R8G8_UNORM`        | roughness, metalness          | 2     |
| Depth  | `D32_SFLOAT`        | depth (already needed)        | 4     |

**14 bytes per pixel**, down from 40. On tile-based mobile GPUs, mark these as
transient attachments in a single render pass with subpasses and most of that
traffic never leaves on-chip memory at all.

Next time: clustered light culling on top of this G-buffer.
