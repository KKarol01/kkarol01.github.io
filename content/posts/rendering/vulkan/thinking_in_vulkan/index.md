---
title: "Rendering a Triangle Backwards"
date: 2026-10-11
draft: false
description: "A Vulkan mental model built by starting from the finished triangle and working back to the objects that put it on the screen."
aliases:
  - /posts/2026/rendering/vulkan/thinking_in_vulkan/
---

This post is a reference for Vulkan mental model and how to render your first triangle --- but backwards.
<!--more-->

I often see tutorials teaching Vulkan first from setting up a project, then showcasing all the Vulkan objects, ranging from VkDevice to VkCommandBuffers and so on, and the triangle comes up last. I think it quickly becomes abstract as you just don't see how it relates to the final goal and you might lose focus. So I thought: let's try the other way.

## Your First Triangle!

{{< figure src="./triangle.png" alt="Your first triangle!" class="flex" >}}
And there we go; that was easy! Now, how did it get there?

The triangle must be drawn somehow, right? And the pixel values must be stored in some kind of texture. And that texture must be shown on the computer screen --- also somehow... Let's tackle the first problem:

## The Triangle Must Be Drawn Somehow
Let's assume we already have a texture and now we only want to draw a triangle to it. A special piece of code is written to access the texture's memory. That code is compiled into a shader program; Vulkan stores it in `VkShaderModule`. Let's go over the shader code:
```HLSL
struct VSOutput
{
    float4 pos : SV_Position;
    float2 uv : POSITION0;
};
VSOutput main(uint vid : SV_VertexID)
{
    static const float3 vertices[] = { float3(-1.0f, 0.0f, 0.0f),
                                       float3(1.0f, 0.0f, 0.0f),
                                       float3(0.0, -1.0f, 0.0f) };
    static const float2 uvs[] = { float2(0.0f, 0.0f),
                                  float2(1.0f, 0.0f),
                                  float2(0.0f, 1.0f) };
    VSOutput output;
    output.pos = float4(vertices[vid].xyz, 1.0);
    output.uv = uvs[vid];
    return output;
}
```

```HLSL
struct VSOutput
{
    float4 pos : SV_Position;
    float2 uv : POSITION0;
};
float4 main(VSOutput vso) : SV_Target0
{
    return float4(vso.uv.xy, 0.0, 1.0);
}
```

As you can see, there are two blocks of code. Both of them are written in a language called HLSL. Most popular shader languages are: [HLSL](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-reference), [GLSL](https://registry.khronos.org/OpenGL/specs/gl/GLSLangSpec.4.60.pdf) and more recently [SLANG](https://shader-slang.org/).

The first block of code is compiled into something called Vertex Shader, and the second one is compiled into Fragment/Pixel Shader (OpenGL and Vulkan use fragment shader version whereas DirectX is using pixel shader in its terminology). Vertex Shader is used for manipulating the vertices: it gets positions in nominal form (model space) and can manipulate them to move them from the starting position, animate them, scale and rotate, and more. This is the place where you apply perspective or player's camera movement. Our shader does nothing fancy: it just outputs raw vertices. As you may have noticed, the triangle perfectly stretches to the absolute edges of the screen, and has a nice gradient. Everything that will become visible, and rendered, is within the range `[-1, +1]` for X and Y, and `[0, 1]` for Z (depth), called Normalized Device Coordinates (NDC), and anything outside of that is [clipped](https://en.wikipedia.org/wiki/Clipping_(computer_graphics)). Notice that the tip of the triangle has coordinates `(0.0, -1.0, 0.0)`. In Vulkan top-left corner has coordinates `(-1.0, -1.0)`, so +X points right, +Y points down, and +Z points into the screen (away from the viewer). Another observation is we have those vertices right in the shader code and access the array with them using some kind of input integer. The integer comes from the rendering command that we will later execute in which we will specify the number of vertices (3) to draw, and each one, starting from zero, gets assigned `SV_VertexID` index. Lastly, even though we output only three vertices, we get a triangle represented by many pixels. This helpful diagram of vulkan graphics pipeline shows the entire process:
{{<figure src="./pipelinemesh.svg" alt="Block diagram of the Vulkan graphics pipeline" class="flex flex-column" caption="Block diagram of the Vulkan graphics pipeline. Source:" attr="Vulkan Specification, Khronos Group" attrlink="https://docs.vulkan.org/spec/latest/chapters/pipelines.html">}}

After the Vertex Shader is run --- skipping some optional stages -- vertices arrive at Rasterization Stage, where individual pixels are tested if they fall inside the triangle by testing a sample point --- usually the pixel's center (or several sample positions within the pixel, with multisampling). Those pixels, along with their state, are passed into the Fragment Shader that can do all the fancy math for your lighting calculations or discard the pixels altogether ([with a possible performance penalty](https://therealmjp.github.io/posts/to-earlyz-or-not-to-earlyz/)). In our example, it just uses colors passed from the Vertex Stage. And this is also where the gradient comes: we only pass --- for the base of the triangle --- color `(0, 0)` and, going to the right, color `(1, 0)`. These get smoothly (and perspective-correct) interpolated in the Rasterization Stage and sent to the Fragment Shader. You can use `nointerpolation` to disable that behaviour so that the triangle keeps the value consistent across its surface. Fragment shader then outputs the final color for a pixel to the resulting texture (`SV_Target0`).
  
To compile the shader, you typically would use GLSLC to compile GLSL shader code, or DXC to compile HLSL, to intermediate representation called SPIR-V, which then can be passed to [VkShaderModuleCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkShaderModuleCreateInfo.html) (it only accepts SPIR-V), and then call [vkCreateShaderModule](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateShaderModule.html) with it. You will get an opaque handle `VkShaderModule` which will be used later when setting up a pipeline.

## The Pixel Values Must Be Stored In Some Kind Of Texture
Let us tackle our assumption from the previous section and actually realise it. Our Fragment Shader code gets executed in this case once for each pixel, and writes the result into `SV_Target0`, which is our texture. A texture usually (see [sparse resources](https://docs.vulkan.org/spec/latest/chapters/sparsemem.html)) is a contiguous chunk of memory. Textures, depending on the texel (texture element) format, can have different requirements, like memory alignment, and may even be restricted to live only in particular [memory types](https://docs.vulkan.org/refpages/latest/refpages/source/VkMemoryPropertyFlagBits.html) (say, the ones that are not CPU-visible). 

To create a texture, you fill out [VkImageCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkImageCreateInfo.html). The most important bits to fill out are: dimensionality (here we use 2D), format of texels (you can use a 4-channel 16-bit floating-point one), size (span in texels -- you can pass resolution of the screen, with depth set to 1), number of [mipmaps](https://en.wikipedia.org/wiki/Mipmap) and layers (should be 1 here), and samples per texel (`VK_SAMPLE_COUNT_1_BIT`, since we don't use multisampling). [Tiling](https://en.wikipedia.org/wiki/Z-order_curve#Texture_mapping) should be optimal, because we don't care about reading it on the host (CPU); sharing mode should be exclusive, queue family data should be ignored, initial layout should be undefined (we will change the layout later) and usage should be `VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT` and `VK_IMAGE_USAGE_TRANSFER_SRC_BIT`, since we want to render to it (Color Attachment), and then copy it to another (swapchain) texture for screen presentation purposes. After filling up the structure, you can pass it to [vkCreateImage](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateImage.html) to obtain an opaque `VkImage` handle.

Ok, we got a handle. Unfortunately, this is not the end of the story. We need to procure a chunk of memory to store it somewhere (ay, caramba!). First, we have to check the memory requirements of said image. Pass the opaque handle to [VkImageMemoryRequirementsInfo2](https://docs.vulkan.org/refpages/latest/refpages/source/VkImageMemoryRequirementsInfo2.html) and call [vkGetImageMemoryRequirements2](https://docs.vulkan.org/refpages/latest/refpages/source/vkGetImageMemoryRequirements2.html) with it. In return, you should get filled out [VkMemoryRequirements2](https://docs.vulkan.org/refpages/latest/refpages/source/VkMemoryRequirements2.html). It will contain size that will be occupied by the texture, required memory address alignment, and memory type bits. The memory types supported by the physical device (see my previous [post]({{< relref "posts/rendering/vulkan/setting_up" >}})) can be queried with [vkGetPhysicalDeviceMemoryProperties](https://docs.vulkan.org/refpages/latest/refpages/source/vkGetPhysicalDeviceMemoryProperties.html), which fills out [VkPhysicalDeviceMemoryProperties](https://docs.vulkan.org/spec/latest/chapters/memory.html#VkPhysicalDeviceMemoryProperties). Memory type bits is actually a bit-field: bit `i` is set if the memory type at index `i` can hold our image. Iterate over the memory types, and pick the first index whose bit is set *and* whose property flags contain what you need --- here `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`, the fast GPU memory. Otherwise you may end up with a slow, CPU-visible memory type. After that, you have your Memory Type Index that you can pass to [VkMemoryAllocateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkMemoryAllocateInfo.html) with the required size, and call [vkAllocateMemory](https://docs.vulkan.org/refpages/latest/refpages/source/vkAllocateMemory.html) with it. After that, you get your own memory block. Now the only thing left to do is to bind it to the opaque handle with [vkBindImageMemory2](https://docs.vulkan.org/refpages/latest/refpages/source/vkBindImageMemory2.html). Notice that [VkBindImageMemoryInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkBindImageMemoryInfo.html) that you need to fill out contains Memory Offset that is a byte offset from the start of the memory block. Now, why is that? You see, it's not as easy as to just allocate memory whenever you please. In fact, you may quickly run into the limit of those, given by [maxMemoryAllocationCount](https://docs.vulkan.org/spec/latest/chapters/limits.html#limits-maxMemoryAllocationCount) inside VkPhysicalDeviceLimits. In reality, you want to allocate bigger chunk (not only because of the limit, but because vkAllocateMemory is just a slow kernel call), and manually suballocate from it. Sounds like a lot of work? Well, it is, and that's why we have beautiful [VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator) that does it all for us.

## The Triangle Texture Must Be Shown On The Screen
To show the triangle on our screen, we must somehow tell the GPU to do so. You do that with list of recorded GPU commands called a command buffer (`VkCommandBuffer`) and then submit them to `VkQueue` which is a command queue that processes those commands.

To get a `VkCommandBuffer`, you allocate it from `VkCommandPool`. Fill out [VkCommandPoolCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkCommandPoolCreateInfo.html) and call [vkCreateCommandPool](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateCommandPool.html) with it. `VkCommandPoolCreateInfo` takes queue family index: command buffers from this pool can only be submitted to command queues of that family. Command Buffers can have commands recorded once and then be reused, or can be reset, and re-recorded with new commands. You can pass `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT` to be able to reset command buffers individually, or have the entire pool reset at once.

Lastly, fill out [VkCommandBufferAllocateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkCommandBufferAllocateInfo.html) and call [vkAllocateCommandBuffers](https://docs.vulkan.org/refpages/latest/refpages/source/vkAllocateCommandBuffers.html). Level should be primary (secondary command buffers can only be executed from inside primary ones, and we don't need that).

Recording starts with [vkBeginCommandBuffer](https://docs.vulkan.org/refpages/latest/refpages/source/vkBeginCommandBuffer.html) (pass `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`, since we'll re-record it every frame), then actual commands get recorded (they start with `vkCmd`), and ends with [vkEndCommandBuffer](https://docs.vulkan.org/refpages/latest/refpages/source/vkEndCommandBuffer.html).

### Preparing The Command Buffer
The command that does the drawing is [vkCmdDraw](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdDraw.html):
```cpp
vkCmdDraw(cmd, 3, 1, 0, 0); // 3 vertices, 1 instance, starting from vertex 0 and instance 0
```
Remember the `SV_VertexID` from the vertex shader? This is where it comes from: three vertices, numbered starting from `firstVertex`, which is 0 here. But we can't draw just yet, because the shaders and output texture were not set. The shaders are set in the pipeline state object. We will be using Graphics Pipeline which stores the state used to control the rendering.

To create a Graphics Pipeline, you fill out [VkGraphicsPipelineCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkGraphicsPipelineCreateInfo.html). It's probably the biggest structure you're gonna have to fill out in Vulkan, so let's quickly go over it.

One thing before we start: whenever I say "fill it out with `0`s" below, the `sType` field is the exception --- every Vulkan structure that has one needs it set, even if everything else stays zero.

First, you give it the shaders: Vertex Shader and Fragment Shader in an array of `VkPipelineShaderStageCreateInfo`. 

Next, you fill out [VkPipelineVertexInputStateCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineVertexInputStateCreateInfo.html), which describes memory locations and how to interpret the bytes inside to fetch when running Vertex Shader. It's called Vertex Fetching, which is optional for our purposes, but alternatively can be replaced with Vertex Pulling, which lets you not fill it out and manually pull data inside the shader using variables like `SV_VertexID` or `SV_InstanceID`. Its main use is to fetch vertex data like position, normal vector, UVs and per instance data like transform matrix. We don't use any attributes, except for position, but we get it from our local shader array, so just fill that structure out with `0`s and NULLs.

After that you configure input assembly [VkPipelineInputAssemblyStateCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineInputAssemblyStateCreateInfo.html), where you can configure the type of primitive that is supposed to be constructed out of the vertices -- we will be drawing triangles mostly, so a Triangle List should be used. 

Tessellation step [`VkPipelineTessellationStateCreateInfo`](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineTessellationStateCreateInfo.html) describes surface subdivision logic which lets you dynamically increase the resolution of the mesh for close-ups for example. We don't use it, so just pass `NULL` --- it's ignored when the pipeline has no tessellation shaders.

Next state is important, because it is a viewport state. `VkPipelineViewportStateCreateInfo` describes the size of our render region. It should be set to the size of our texture, which should match the resolution of your window you have created with depth being ignored, since we won't be using depth test. Remember how rasterization only runs Fragment Shader for locations that are within `[-1, +1]` (NDC)? Those will be scaled and offset by whatever you pass in the `VkViewport` to span it fully. Apart from `VkViewport` you set-up a scissor test with matching `VkRect2D`. Using that, you can only render to a rectangular portion of the image (framebuffer). Instead of specifying those values upfront, we enable both `VK_DYNAMIC_STATE_VIEWPORT_WITH_COUNT` and `VK_DYNAMIC_STATE_SCISSOR_WITH_COUNT` in `VkPipelineDynamicStateCreateInfo`, leave `VkPipelineViewportStateCreateInfo` filled with `0`s (with those dynamic states, both counts must be `0`), and provide the values at draw time with `vkCmdSetViewportWithCount` and `vkCmdSetScissorWithCount` --- that way when your application's window resizes, you don't have to recreate the whole pipeline, which is a slow operation, especially if you have more than one.

Another important step is to configure the rasterizer --- the stage that decides which fragments are covered by our triangle and hands them, along with the interpolated vertex outputs, to the Fragment Shader. Here we tell it how to fill polygons (`VK_POLYGON_MODE_FILL`), which triangles (if any) to cull (`VK_CULL_MODE_NONE` for now), and which winding order counts as front-facing. Also set `lineWidth` to `1.0f` --- it's the one field that can't be left at `0`, even though we don't draw lines. If you take a look at the Vertex Shader, and check the array with vertices, you will notice that the vertices go from: left, to right, to the tip. If you trace your finger in that order, you will trace a circle in a counter-clockwise fashion, which is your winding order. The choice here is arbitrary, but this is what I am using.

We don't use multisampling at this point so fill [VkPipelineMultisampleStateCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineMultisampleStateCreateInfo.html) out with defaults and 1 sample per pixel.

[VkPipelineDepthStencilStateCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineDepthStencilStateCreateInfo.html) should be filled with defaults, with depth tests and writes disabled, because we're not using that feature too for now, which normally lets us discard fragments hidden behind the ones already drawn, to save processing power. 

`renderPass` should be set to `VK_NULL_HANDLE`, since we use dynamic rendering instead: chain [VkPipelineRenderingCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineRenderingCreateInfo.html) into `pNext`, with the format of our texture as the only color attachment format. `basePipelineHandle` should be `VK_NULL_HANDLE` and `basePipelineIndex` set to `-1` (both are ignored anyway unless you create derivative pipelines).

What's left is [VkPipelineColorBlendStateCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineColorBlendStateCreateInfo.html) and [VkPipelineLayout](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineLayout.html). [VkPipelineColorBlendStateCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineColorBlendStateCreateInfo.html) describes logic applied to the color that the Fragment Shader outputs to a texture. You can do alpha blending here: with blending enabled, the resulting color is `src.rgb * srcFactor <op> dst.rgb * dstFactor`, where `src` is a color that our Fragment Shader tries to write, and `dst` is a color that already is inside the texture at the position we're trying to write to. Classic alpha blending, `src.rgb * src.a + dst.rgb * (1 - src.a)`, uses `VK_BLEND_FACTOR_SRC_ALPHA`, `VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA` and `VK_BLEND_OP_ADD` (to play around with those, check out [https://www.andersriggelsen.dk/glblendfunc.php](https://www.andersriggelsen.dk/glblendfunc.php)). We don't need any of that for our triangle, so set everything to false/`0` (the default values), and fill out one [VkPipelineColorBlendAttachmentState](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineColorBlendAttachmentState.html) for our one output texture. With `blendEnable` set to `VK_FALSE`, the resulting color is simply `src`, written as-is. There is one exception to "everything to 0", though: `colorWriteMask` bitmask has to have all four channels (`R | G | B | A`) set, or nothing gets written at all.

The last thing is [VkPipelineLayout](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineLayout.html) which describes the layout of locations for external resources that the pipeline --- and the shaders --- will use; those would be additional textures (`RWTexture2D<float4>`) and buffers (`RWByteAddressBuffer`) that you'd like to access inside your shader, together with push constants which is a small fast-access memory stored in the command buffer accessible to shaders. Here we don't use any external resources, so the pipeline layout can be simple: fill out [VkPipelineLayoutCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineLayoutCreateInfo.html) with default values (`0`s and NULLs), and call [vkCreatePipelineLayout](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreatePipelineLayout.html).

With all that behind us, we can finally call [vkCreateGraphicsPipelines](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateGraphicsPipelines.html). Pipelines can be [cached](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineCache.html) and saved on disk. Once we have it, binding is a single call to [vkCmdBindPipeline](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdBindPipeline.html), followed by the dynamic state we promised to provide, via [vkCmdSetViewportWithCount](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdSetViewportWithCount.html) and [vkCmdSetScissorWithCount](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdSetScissorWithCount.html):
```cpp
vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
auto viewport = VkViewport{ .x = 0.0f, .y = 0.0f, .width = (float)width, .height = (float)height, .minDepth = 0.0f, .maxDepth = 1.0f };
auto scissor = VkRect2D{ .offset = { 0, 0 }, .extent = { width, height } };
vkCmdSetViewportWithCount(cmd, 1, &viewport);
vkCmdSetScissorWithCount(cmd, 1, &scissor);
```
Before the draw command, we need to setup the output texture. Draw commands must be recorded between [vkCmdBeginRendering](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdBeginRendering.html) and [vkCmdEndRendering](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdEndRendering.html), because we're using [dynamic rendering](https://docs.vulkan.org/samples/latest/samples/extensions/dynamic_rendering/README.html), and this is where we finally point at our texture. [VkRenderingInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkRenderingInfo.html) takes the render area and a list of [VkRenderingAttachmentInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkRenderingAttachmentInfo.html) --- one per `SV_TargetN` --- those don't want a `VkImage`. They want a `VkImageView`. What is a VkImageView, you ask? It is a way to interpret the image: you can select a subset of mips and layers, change the type from 3D to a series of 2D slices (2D_ARRAY), select a particular aspect (color, or only depth or stencil from otherwise depth-stencil-combined image), and how to interpret the format (from int to float for example, if [mutable format](https://docs.vulkan.org/refpages/latest/refpages/source/VkImageCreateFlagBits.html) was set. See [compatible formats](https://docs.vulkan.org/spec/latest/chapters/formats.html#formats-compatibility-classes)). Our case is the simplest one possible: fill out [VkImageViewCreateInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkImageViewCreateInfo.html) with 2D view type, the same format as the image, color aspect, and one mip and one layer, then call [vkCreateImageView](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateImageView.html).

The attachment info also asks what to do with the texture's contents when rendering starts (load op) and ends (store op). We want to clear it (otherwise we would draw on top of whatever garbage was there before) and store the result (otherwise the GPU is allowed to throw it away).

And then there is the image layout, which the attachment info also asks about. When creating the texture, we set its initial layout to undefined and promised to change it later. Well, later is now. GPUs like to store images differently depending on what they're used for --- compressed, swizzled, in a format the render units prefer, or one the copy engine prefers. A layout is how you tell Vulkan the image is about to be used in a particular way, so that it can be rearranged accordingly. For rendering, that's `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`.

Layouts are changed with pipeline barriers. With synchronization2 (also enabled in the previous post), you fill out [VkImageMemoryBarrier2](https://docs.vulkan.org/refpages/latest/refpages/source/VkImageMemoryBarrier2.html), wrap it in [VkDependencyInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkDependencyInfo.html) and record [vkCmdPipelineBarrier2](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdPipelineBarrier2.html). Besides the layout transition, a barrier does two more things, and both are equally important. The GPU doesn't execute your commands one after another --- it overlaps them, and different caches may hold different versions of the same memory. A barrier says: commands before me must finish *this* stage (source stage), before commands after me start *that* stage (destination stage), and *these* writes (source access) must be made visible to *those* reads or writes (destination access). Our first transition goes from `VK_IMAGE_LAYOUT_UNDEFINED` (meaning: we don't care about the old contents, which is fine, since we clear them anyway) to `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`. Nothing has to happen before it (`VK_PIPELINE_STAGE_2_NONE` with `VK_ACCESS_2_NONE`), and the clear and the draw both write to the attachment in the color attachment output stage (`VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT` with `VK_ACCESS_2_COLOR_ATTACHMENT_WRITE_BIT`). 

Putting it all together:
```cpp
image_barrier_pseudo_code(cmd, render_image,
              VK_PIPELINE_STAGE_2_NONE, VK_ACCESS_2_NONE,
              VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT, VK_ACCESS_2_COLOR_ATTACHMENT_WRITE_BIT,
              VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL);
auto color_attachment = VkRenderingAttachmentInfo{
    .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
    .imageView = render_view,
    .imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
    .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
    .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
    .clearValue = { .color = { .float32 = { 0.0f, 0.0f, 0.0f, 1.0f } } },
};
auto rendering = VkRenderingInfo{
    .sType = VK_STRUCTURE_TYPE_RENDERING_INFO,
    .renderArea = { .offset = { 0, 0 }, .extent = { width, height } },
    .layerCount = 1,
    .colorAttachmentCount = 1,
    .pColorAttachments = &color_attachment,
};
vkCmdBeginRendering(cmd, &rendering);
vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
vkCmdSetViewportWithCount(cmd, 1, &viewport);
vkCmdSetScissorWithCount(cmd, 1, &scissor);
vkCmdDraw(cmd, 3, 1, 0, 0);
vkCmdEndRendering(cmd);
```

### Render The Triangle
We have a recorded command buffer, now we need it executed. To do that, it has to be submitted to a queue (`VkQueue`). A GPU has several hardware engines that can work in parallel --- the graphic ones, asynchronous compute ones, copy (DMA) ones --- and Vulkan exposes them as queues, grouped into families by what they are capable of (see the glossary in the previous post). We already retrieved a graphics queue there, and our command pool was created for its family, so we're all set. Wrap the command buffer in [VkCommandBufferSubmitInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkCommandBufferSubmitInfo.html), put that into [VkSubmitInfo2](https://docs.vulkan.org/refpages/latest/refpages/source/VkSubmitInfo2.html), and call [vkQueueSubmit2](https://docs.vulkan.org/refpages/latest/refpages/source/vkQueueSubmit2.html).

OK, we have sent our triangle for rendering. `vkQueueSubmit2` returns immediately --- it does not wait until it's finished. However, even when it will be done, the image still won't show on our screen. To make that happen, we have to use a [VkSwapchainKHR](https://docs.vulkan.org/refpages/latest/refpages/source/VkSwapchainKHR.html), which is an abstraction over an array of images that are presentable (onto screen). We don't own the memory for them, so we don't have to create them (yay).

To create a swapchain, fill out [VkSwapchainCreateInfoKHR](https://docs.vulkan.org/refpages/latest/refpages/source/VkSwapchainCreateInfoKHR.html). Pass in window surface (see previous post), set min image count to 2 ([double buffering](https://en.wikipedia.org/wiki/Multiple_buffering)), image extent should match the window and output texture, there should only be 1 layer, usage should be `VK_IMAGE_USAGE_TRANSFER_DST_BIT`, because we're gonna copy our output texture to it before presenting, sharing mode should be set to VK_SHARING_MODE_EXCLUSIVE, because those images are not going to be used by other queue families and queue family indices should be then set to NULL; preTransform to `VK_SURFACE_TRANSFORM_IDENTITY_BIT_KHR`, compositeAlpha to `VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR`, and clipped can be set to `VK_TRUE` to allow for more efficient presentation methods and, for example, not running Fragment Shader for pixels that are obscured by other windows. You can query for supported image formats and colorspaces with [vkGetPhysicalDeviceSurfaceFormatsKHR](https://docs.vulkan.org/refpages/latest/refpages/source/vkGetPhysicalDeviceSurfaceFormatsKHR.html). I usually use `VK_FORMAT_R8G8B8A8_SRGB` with `VK_COLOR_SPACE_SRGB_NONLINEAR_KHR` for regular non-HDR presentation. [Color space](https://docs.vulkan.org/refpages/latest/refpages/source/VkColorSpaceKHR.html) specifies encoding which final pixels have been stored with, and how a presentation engine should interpret them. With [vkGetPhysicalDeviceSurfacePresentModesKHR](https://docs.vulkan.org/refpages/latest/refpages/source/vkGetPhysicalDeviceSurfacePresentModesKHR.html) you can query for supported presentation modes, which control how presentation engine handles presenting the images. I usually pick `VK_PRESENT_MODE_FIFO_KHR`. You can read about their behaviour [here](https://docs.vulkan.org/refpages/latest/refpages/source/VkPresentModeKHR.html). Now we can call [vkCreateSwapchainKHR](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateSwapchainKHR.html) with the filled out structure and get an opaque handle. To get the presentable images, you call [vkGetSwapchainImagesKHR](https://docs.vulkan.org/refpages/latest/refpages/source/vkGetSwapchainImagesKHR.html).

### Present The Triangle
Swapchain images belong to the presentation engine, so every frame we have to ask for one with [vkAcquireNextImageKHR](https://docs.vulkan.org/refpages/latest/refpages/source/vkAcquireNextImageKHR.html). It gives us the index of the available image, but that image might still be in use. So instead of waiting, the function returns right away and later signals a semaphore ([VkSemaphore](https://docs.vulkan.org/refpages/latest/refpages/source/VkSemaphore.html), created with [vkCreateSemaphore](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateSemaphore.html)) once the image is actually free. Semaphores are for GPU-to-GPU waits: one operation signals it, another one waits on it.

Our texture and the swapchain image have different formats which are not [size-compatible](https://docs.vulkan.org/spec/latest/chapters/formats.html#formats-size-compatibility) (`VK_FORMAT_R16G16B16A16_SFLOAT` and `VK_FORMAT_R8G8B8A8_SRGB`), so [vkCmdCopyImage](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdCopyImage.html) won't do. [vkCmdBlitImage2](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdBlitImage2.html) copies and converts the format (it can scale the source image too). Before the blit, both images need to be transitioned to correct layouts, so we need to issue another image barrier: our texture goes to `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL` once the rendering writes are done, and the swapchain image goes from `VK_IMAGE_LAYOUT_UNDEFINED` (we overwrite it anyway) to `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`. After the blit, the swapchain image needs one more transition, to `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`, which is the layout the presentation engine requires:
```cpp
image_barrier_pseudo_code(cmd, render_image,
              VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT, VK_ACCESS_2_COLOR_ATTACHMENT_WRITE_BIT,
              VK_PIPELINE_STAGE_2_ALL_TRANSFER_BIT, VK_ACCESS_2_TRANSFER_READ_BIT,
              VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL, VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL);
image_barrier_pseudo_code(cmd, swapchain_images[image_index],
              VK_PIPELINE_STAGE_2_ALL_TRANSFER_BIT, VK_ACCESS_2_NONE,
              VK_PIPELINE_STAGE_2_ALL_TRANSFER_BIT, VK_ACCESS_2_TRANSFER_WRITE_BIT,
              VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL);
auto region = VkImageBlit2{
    .sType = VK_STRUCTURE_TYPE_IMAGE_BLIT_2,
    .srcSubresource = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 0, 1 },
    .srcOffsets = { { 0, 0, 0 }, { (int32_t)width, (int32_t)height, 1 } },
    .dstSubresource = { VK_IMAGE_ASPECT_COLOR_BIT, 0, 0, 1 },
    .dstOffsets = { { 0, 0, 0 }, { (int32_t)width, (int32_t)height, 1 } },
};
auto blit = VkBlitImageInfo2{
    .sType = VK_STRUCTURE_TYPE_BLIT_IMAGE_INFO_2,
    .srcImage = render_image,
    .srcImageLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL,
    .dstImage = swapchain_images[image_index],
    .dstImageLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    .regionCount = 1,
    .pRegions = &region,
    .filter = VK_FILTER_LINEAR,
};
vkCmdBlitImage2(cmd, &blit);
image_barrier_pseudo_code(cmd, swapchain_images[image_index],
              VK_PIPELINE_STAGE_2_ALL_TRANSFER_BIT, VK_ACCESS_2_TRANSFER_WRITE_BIT,
              VK_PIPELINE_STAGE_2_NONE, VK_ACCESS_2_NONE,
              VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_PRESENT_SRC_KHR);
```

To finally show the image, we can call [vkQueuePresentKHR](https://docs.vulkan.org/refpages/latest/refpages/source/vkQueuePresentKHR.html) with the image index. Presenting must also wait until the blit is done, which is another GPU-to-GPU wait: the submit signals a second semaphore, and present waits on it. Keep one of those per swapchain image, not one per frame.

### Rendering Loop
One last problem: we re-record the same command buffer every frame, but we can't touch it while the GPU is still executing it. We need the GPU to tell the CPU it's done, and that's what a fence is for ([VkFence](https://docs.vulkan.org/refpages/latest/refpages/source/VkFence.html), created with [vkCreateFence](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateFence.html)). Pass it to the submit, wait on it with [vkWaitForFences](https://docs.vulkan.org/refpages/latest/refpages/source/vkWaitForFences.html), and reset it with [vkResetFences](https://docs.vulkan.org/refpages/latest/refpages/source/vkResetFences.html). Create it with `VK_FENCE_CREATE_SIGNALED_BIT`, or the very first wait will hang forever.

We'd also like the CPU and the GPU to work at the same time: while the GPU is still rendering the previous frame, the CPU can already start recording the next one. For that to happen, we need two sets of rendering primitives and resources: two command pools, two command buffers, two fences, two acquire semaphores and two color textures (with their image views) --- and alternate between them each frame, so the CPU records frame N+1 while the GPU is still busy with frame N. These are called frames in flight. Note that their number is independent of the number of swapchain images --- you could have three swapchain images and two frames in flight. We just happen to use two of both, which lets us keep one `render_finished` semaphore per swapchain image in an array of `FRAMES_IN_FLIGHT` size. Here's the final rendering loop:
```cpp
constexpr uint32_t FRAMES_IN_FLIGHT = 2;
struct FrameData
{
    VkCommandPool cmd_pool;
    VkCommandBuffer cmd;
    VkFence fence; // created with VK_FENCE_CREATE_SIGNALED_BIT
    VkSemaphore image_acquired;
    VkImage render_image; // our color texture, one per frame
    VkImageView render_view;
};
FrameData frames[FRAMES_IN_FLIGHT];
VkSemaphore render_finished[FRAMES_IN_FLIGHT]; // one per swapchain image (we have two of those too)
uint64_t frame_number = 0;

while(!glfwWindowShouldClose(window))
{
    glfwPollEvents();

    FrameData& frame = frames[frame_number % FRAMES_IN_FLIGHT];
    vkWaitForFences(device, 1, &frame.fence, VK_TRUE, ~0ull); // this frame's resources are no longer in use
    uint32_t image_index = 0;
    vkAcquireNextImageKHR(device, swapchain, ~0ull, frame.image_acquired, VK_NULL_HANDLE, &image_index);
    vkResetFences(device, 1, &frame.fence);

    VkCommandBuffer cmd = frame.cmd;
    vkResetCommandPool(device, frame.cmd_pool, 0);
    vkBeginCommandBuffer(cmd, &begin_info);
    // ...barriers, rendering and copy from above, using frame.render_image and frame.render_view
    vkEndCommandBuffer(cmd);

    auto cmd_info = VkCommandBufferSubmitInfo{ .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_SUBMIT_INFO, .commandBuffer = cmd };
    auto wait_info = VkSemaphoreSubmitInfo{ .sType = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
                                            .semaphore = frame.image_acquired,
                                            .stageMask = VK_PIPELINE_STAGE_2_ALL_TRANSFER_BIT };
    auto signal_info = VkSemaphoreSubmitInfo{ .sType = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
                                              .semaphore = render_finished[image_index],
                                              .stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT };
    auto submit = VkSubmitInfo2{ .sType = VK_STRUCTURE_TYPE_SUBMIT_INFO_2,
                                 .waitSemaphoreInfoCount = 1, .pWaitSemaphoreInfos = &wait_info,
                                 .commandBufferInfoCount = 1, .pCommandBufferInfos = &cmd_info,
                                 .signalSemaphoreInfoCount = 1, .pSignalSemaphoreInfos = &signal_info };
    // signal semaphore when done
    vkQueueSubmit2(gfx_queue, 1, &submit, frame.fence); 

    auto present = VkPresentInfoKHR{ .sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR,
                                     .waitSemaphoreCount = 1, .pWaitSemaphores = &render_finished[image_index],
                                     .swapchainCount = 1, .pSwapchains = &swapchain,
                                     .pImageIndices = &image_index }; 
    // wait for render semaphore, then present
    vkQueuePresentKHR(gfx_queue, &present);

    ++frame_number;
}
```

And that would be all. It’s a long article, and yet it’s still simplified --- we don’t handle window resizes, depth testing, or vertex buffers. I hope it can be of use to someone and provides a good mental model of the rendering process and all the work required to render a triangle. The good thing is that when you have all of this set up, adding more things and rendering even more complex geometries with actual lighting doesn't take that much work anymore.

## References
- [https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-reference](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-reference)
- [https://registry.khronos.org/OpenGL/specs/gl/GLSLangSpec.4.60.pdf](https://registry.khronos.org/OpenGL/specs/gl/GLSLangSpec.4.60.pdf)
- [https://shader-slang.org/](https://shader-slang.org/)
- [https://en.wikipedia.org/wiki/Clipping_(computer_graphics)](https://en.wikipedia.org/wiki/Clipping_(computer_graphics))
- [https://docs.vulkan.org/spec/latest/](https://docs.vulkan.org/spec/latest/)
- [https://therealmjp.github.io/posts/to-earlyz-or-not-to-earlyz/](https://therealmjp.github.io/posts/to-earlyz-or-not-to-earlyz/)
- [https://docs.vulkan.org/refpages/latest/refpages/source/VkMemoryPropertyFlagBits.html](https://docs.vulkan.org/refpages/latest/refpages/source/VkMemoryPropertyFlagBits.html)
- [https://en.wikipedia.org/wiki/Mipmap](https://en.wikipedia.org/wiki/Mipmap)
- [https://en.wikipedia.org/wiki/Z-order_curve#Texture_mapping](https://en.wikipedia.org/wiki/Z-order_curve#Texture_mapping)
- [https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)
- [https://www.andersriggelsen.dk/glblendfunc.php](https://www.andersriggelsen.dk/glblendfunc.php)
- [https://docs.vulkan.org/samples/latest/samples/extensions/dynamic_rendering/README.html](https://docs.vulkan.org/samples/latest/samples/extensions/dynamic_rendering/README.html)
- [https://en.wikipedia.org/wiki/Multiple_buffering](https://en.wikipedia.org/wiki/Multiple_buffering)
- [https://docs.vulkan.org/refpages/latest/refpages/source/VkColorSpaceKHR.html](https://docs.vulkan.org/refpages/latest/refpages/source/VkColorSpaceKHR.html)