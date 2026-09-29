---
title: "Cascaded Shadow Maps Without the Shimmer"
date: 2026-09-28T10:00:00Z
draft: false
description: "Splitting the view frustum, stabilizing cascades, and blending between them."
tags: ["rendering", "shadows", "csm", "glsl"]
---

A single shadow map can't cover a large outdoor scene at a useful resolution.
Cascaded shadow maps split the view frustum into slices, each with its own map.

<!--more-->

## Choosing split distances

A mix of uniform and logarithmic splits works well in practice:

```glsl
float splitDepth(int i, int count, float near, float far, float lambda) {
    float p = float(i) / float(count);
    float logSplit = near * pow(far / near, p);
    float uniSplit = near + (far - near) * p;
    return mix(uniSplit, logSplit, lambda);
}
```

`lambda = 0.75` is a reasonable starting point.

## Killing the shimmer

Shadow edges crawl as the camera moves because the cascade's projection moves
by sub-texel amounts. Two fixes:

1. **Fit a bounding sphere** instead of a box, so the cascade's size doesn't
   change when the camera rotates.
2. **Snap the origin to texel increments** in light space.

```glsl
vec2 texelSize = vec2(2.0 * radius / float(shadowMapSize));
origin.xy = floor(origin.xy / texelSize) * texelSize;
```

## Blending cascades

Near the edge of a cascade, sample both it and the next one, then blend. That
hides the visible seam where resolution suddenly drops.
