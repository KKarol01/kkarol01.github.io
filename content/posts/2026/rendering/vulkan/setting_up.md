---
title: "Quick Vulkan Setup"
date: 2026-10-01
draft: false
description: ""
tags: ["rendering", "vulkan", "tutorial"]
---

A reference for setting up Vulkan 1.3 with vk-bootstrap, volk and VMA; creating a logical device, retrieving a graphics queue and initializing a memory allocator.
<!--more-->

## Libraries I use
- [https://github.com/charles-lunarg/vk-bootstrap](https://github.com/charles-lunarg/vk-bootstrap) --- keeps the setup code short and avoids hand-written selection loops,
- [https://github.com/zeux/volk](https://github.com/zeux/volk) --- loads the system Vulkan loader and lets you skip the loader's dispatch overhead,
- [https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator) --- for easy GPU memory management.

## Library Setup
Volk and VMA are single-header libraries: define `VOLK_IMPLEMENTATION` and `VMA_IMPLEMENTATION` in exactly one translation unit for the implementation to be compiled. Because volk defines `VK_NO_PROTOTYPES`, VMA can't call Vulkan functions directly, so set `VMA_STATIC_VULKAN_FUNCTIONS=0` and `VMA_DYNAMIC_VULKAN_FUNCTIONS=1`. VMA then fetches the function pointers it needs through the `vkGetInstanceProcAddr`.

CMake config (I configure my translation unit as a static library):
```cmake
target_compile_definitions(build_header_only
    PRIVATE
        VOLK_IMPLEMENTATION
        VMA_IMPLEMENTATION
        VMA_STATIC_VULKAN_FUNCTIONS=0
        VMA_DYNAMIC_VULKAN_FUNCTIONS=1
    PUBLIC
        VK_USE_PLATFORM_WIN32_KHR  # exposes the Win32 surface API in vulkan.h and volk
)
```

## VK Structure Glossary
- **`VkInstance`** ---  all per-application state is stored in a VkInstance object. Creating a VkInstance object initializes the Vulkan library and allows the application to pass information about itself to the implementation.
- **`VkPhysicalDevice`** --- Vulkan separates the concept of physical and logical devices. A physical device usually represents a single complete implementation of Vulkan (excluding instance-level functionality) available to the host.
- **`VkDevice`** (logical device) --- the application's own "session" with a physical device. Device extensions and features are enabled when it's created, and almost every other object is created from it.
- **`VkQueue`** --- the queue that command buffers are submitted to for execution on the GPU.
- **Queue family** --- a group of queues with the same capabilities (graphics, compute, transfer, presentation). Queues are requested per family when creating the device.
- **`VkSurfaceKHR`** --- an abstraction of the native window that Vulkan can present to.
- **`VkSwapchainKHR`** --- a swapchain is an abstraction for an array of presentable images that are associated with a surface.

## Init process
### Init Volk
First initialize volk. This loads the Vulkan loader and the global functions (`vkCreateInstance`):
```cpp
if(volkInitialize() != VK_SUCCESS) { ... }
```

### Build VkInstance
Then prepare the Vulkan instance object:
```cpp
vkb::InstanceBuilder builder;
auto inst_ret = builder
    .set_app_name("Example Vulkan Application")
#ifdef ENG_DEBUG_BUILD
    .request_validation_layers()
    .use_default_debug_messenger()
#endif
    .require_api_version(VK_API_VERSION_1_3)
    .build();

if(!inst_ret) { ... }
```

### Load instance function pointers via volk
Next load instance-level function pointers via volk, using the `VkInstance` object:
```cpp
vkb::Instance vkb_inst = inst_ret.value();
volkLoadInstance(vkb_inst.instance);
```

### Create window surface
After that you can create the window surface (`VkSurfaceKHR`):
```cpp
VkWin32SurfaceCreateInfoKHR vkwin32surfinfo = {};
vkwin32surfinfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
vkwin32surfinfo.hinstance = GetModuleHandle(nullptr);
vkwin32surfinfo.hwnd = glfwGetWin32Window(window->window);
VkSurfaceKHR window_surface = VK_NULL_HANDLE;
VkResult surf_res = vkCreateWin32SurfaceKHR(vkb_inst.instance, &vkwin32surfinfo, nullptr, &window_surface);
if(surf_res != VK_SUCCESS) { ... }
```

### Pick physical device
Next up is the quest for a physical device. Thankfully we have vk-bootstrap to aid us. You can also add required extensions that a physical device needs to support in order to be picked:
```cpp
vkb::PhysicalDeviceSelector selector{ vkb_inst };
auto phys_rets = selector.require_present()
    .set_surface(window_surface)
    .set_minimum_version(1, 3)
    .add_required_extension(VK_KHR_DYNAMIC_RENDERING_EXTENSION_NAME)        // for imgui
    .add_required_extension(VK_KHR_SWAPCHAIN_MUTABLE_FORMAT_EXTENSION_NAME) // for imgui
    .add_required_extension(VK_KHR_MAINTENANCE_5_EXTENSION_NAME)
    .prefer_gpu_device_type(vkb::PreferredDeviceType::discrete)
    .allow_any_gpu_device_type(true)
    .select_devices();
if(!phys_rets)
{
    ENG_ERROR("Failed to select Vulkan Physical Device.");
    return;
}
```

Once queried, you can inspect and pick one (or more):
```cpp
vkb::PhysicalDevice phys_ret = [&phys_rets] {
    for(auto& pd : *phys_rets)
    {
        if(pd.properties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) { return pd; }
    }
    return phys_rets->front();
}();
```

To add optional extensions you can call:
```cpp
phys_ret.enable_extensions_if_present({
    VK_KHR_ACCELERATION_STRUCTURE_EXTENSION_NAME,
    VK_KHR_RAY_TRACING_PIPELINE_EXTENSION_NAME,
    VK_KHR_DEFERRED_HOST_OPERATIONS_EXTENSION_NAME,
    VK_KHR_RAY_QUERY_EXTENSION_NAME,
});
```
And to check the support:
```cpp
caps.supports_raytracing = phys_ret.is_extension_present(VK_KHR_RAY_TRACING_PIPELINE_EXTENSION_NAME)
    && phys_ret.is_extension_present(VK_KHR_ACCELERATION_STRUCTURE_EXTENSION_NAME)
    && phys_ret.is_extension_present(VK_KHR_DEFERRED_HOST_OPERATIONS_EXTENSION_NAME)
    && phys_ret.is_extension_present(VK_KHR_RAY_QUERY_EXTENSION_NAME);
```
### Creating the VkDevice object
Before creating the object itself, you can enable features your application needs:
```cpp
auto synch2_features = VkPhysicalDeviceSynchronization2Features{};
synch2_features.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_SYNCHRONIZATION_2_FEATURES;
synch2_features.synchronization2 = VK_TRUE;

auto dyn_features = VkPhysicalDeviceDynamicRenderingFeatures{};
dyn_features.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DYNAMIC_RENDERING_FEATURES;
dyn_features.dynamicRendering = VK_TRUE;

auto bda_features = VkPhysicalDeviceBufferDeviceAddressFeatures{};
bda_features.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_BUFFER_DEVICE_ADDRESS_FEATURES;
bda_features.bufferDeviceAddress = VK_TRUE;

auto maint5_features = VkPhysicalDeviceMaintenance5FeaturesKHR{};
maint5_features.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_MAINTENANCE_5_FEATURES_KHR;
maint5_features.maintenance5 = VK_TRUE;

auto dev_2_features = VkPhysicalDeviceFeatures2{};
dev_2_features.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2;
dev_2_features.features.multiDrawIndirect = VK_TRUE;

// Ray tracing needs its features enabled too, not only the extensions
auto accel_features = VkPhysicalDeviceAccelerationStructureFeaturesKHR{};
accel_features.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_ACCELERATION_STRUCTURE_FEATURES_KHR;
accel_features.accelerationStructure = VK_TRUE;

auto rt_pipe_features = VkPhysicalDeviceRayTracingPipelineFeaturesKHR{};
rt_pipe_features.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_RAY_TRACING_PIPELINE_FEATURES_KHR;
rt_pipe_features.rayTracingPipeline = VK_TRUE;

auto ray_query_features = VkPhysicalDeviceRayQueryFeaturesKHR{};
ray_query_features.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_RAY_QUERY_FEATURES_KHR;
ray_query_features.rayQuery = VK_TRUE;

// Chain the structures
vkb::DeviceBuilder device_builder{ phys_ret };
device_builder.add_pNext(&dev_2_features);
device_builder.add_pNext(&dyn_features);
device_builder.add_pNext(&synch2_features);
device_builder.add_pNext(&bda_features);
device_builder.add_pNext(&maint5_features);
if(caps.supports_raytracing)
{
    device_builder.add_pNext(&accel_features);
    device_builder.add_pNext(&rt_pipe_features);
    device_builder.add_pNext(&ray_query_features);
}
```

Now build the device and load the function pointers associated with it via volk:
```cpp
auto dev_ret = device_builder.build();
if(!dev_ret) { ... }
vkb::Device vkb_device = dev_ret.value();
volkLoadDevice(vkb_device.device);
```

### Querying physical device properties
To query properties used throughout the engine (ray tracing structures are chained only when supported):
```cpp
auto rt_props = VkPhysicalDeviceRayTracingPipelinePropertiesKHR{};
rt_props.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_RAY_TRACING_PIPELINE_PROPERTIES_KHR;
auto rt_acc_props = VkPhysicalDeviceAccelerationStructurePropertiesKHR{};
rt_acc_props.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_ACCELERATION_STRUCTURE_PROPERTIES_KHR;
rt_props.pNext = &rt_acc_props;

auto pdev_props = VkPhysicalDeviceProperties2{};
pdev_props.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2;
if(caps.supports_raytracing) { pdev_props.pNext = &rt_props; }
vkGetPhysicalDeviceProperties2(phys_ret.physical_device, &pdev_props);
```

### Getting command queues
Queues are created together with the device, so you only need to retrieve them via `vkb::Device`:
```cpp
VkQueue gfx_queue = vkb_device.get_queue(vkb::QueueType::graphics).value();
uint32_t gfx_queue_family = vkb_device.get_queue_index(vkb::QueueType::graphics).value();
```

### Initializing VMA
To initialize the memory allocator, you need to pass it `vkGetInstanceProcAddr` and `vkGetDeviceProcAddr`, because volk removed the static prototypes and we set `VMA_DYNAMIC_VULKAN_FUNCTIONS=1`:
```cpp
VmaVulkanFunctions vulkanFunctions = {
    .vkGetInstanceProcAddr = vkb_inst.fp_vkGetInstanceProcAddr,
    .vkGetDeviceProcAddr = vkb_inst.fp_vkGetDeviceProcAddr,
};

VmaAllocatorCreateInfo allocatorCreateInfo{};
allocatorCreateInfo.flags = VMA_ALLOCATOR_CREATE_BUFFER_DEVICE_ADDRESS_BIT | VMA_ALLOCATOR_CREATE_KHR_MAINTENANCE5_BIT;
allocatorCreateInfo.physicalDevice = phys_ret.physical_device;
allocatorCreateInfo.device = vkb_device.device;
allocatorCreateInfo.pVulkanFunctions = &vulkanFunctions;
allocatorCreateInfo.instance = vkb_inst.instance;
allocatorCreateInfo.vulkanApiVersion = VK_API_VERSION_1_3;
VmaAllocator vma = VK_NULL_HANDLE;
VK_CHECK(vmaCreateAllocator(&allocatorCreateInfo, &vma));
```

## References
[https://docs.vulkan.org/spec/latest/index.html](https://docs.vulkan.org/spec/latest/index.html)
[https://github.com/KKarol01/vkrt/tree/main](https://github.com/KKarol01/vkrt/tree/main)