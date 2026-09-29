---
title: "GPU-Driven Frustum Culling with Compute Shaders"
date: 2026-07-05T10:00:00Z
draft: false
description: "Moving per-object visibility tests to the GPU and feeding the results into indirect draws."
tags: ["rendering", "compute", "vulkan", "culling"]
---

Once a scene has tens of thousands of objects, CPU culling and draw submission
become the bottleneck. Moving both to the GPU removes that bottleneck.

<!--more-->

## The pipeline

1. Upload every object's bounding sphere and draw arguments once.
2. A compute shader tests each sphere against the six frustum planes.
3. Visible objects append their draw command to a buffer.
4. `vkCmdDrawIndexedIndirectCount` consumes that buffer.

## The culling shader

```glsl
layout(local_size_x = 64) in;

void main() {
    uint id = gl_GlobalInvocationID.x;
    if (id >= objectCount) return;

    vec4 s = spheres[id];                 // xyz = center, w = radius
    bool visible = true;
    for (int i = 0; i < 6; ++i)
        visible = visible && dot(planes[i].xyz, s.xyz) + planes[i].w > -s.w;

    if (visible) {
        uint slot = atomicAdd(drawCount, 1);
        outDraws[slot] = inDraws[id];
    }
}
```

## Next steps

- Add Hi-Z occlusion culling using last frame's depth pyramid.
- Cull per meshlet instead of per object for finer granularity.

In my (fictional) test scene with 80k objects, CPU frame time went from 11 ms
to under 1 ms.
