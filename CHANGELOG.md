# Changelog

All notable changes to libvulkan-sys are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-24

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-16

The first release: fifty entry points of the Vulkan loader, one `@ffi`
declaration each, and no logic.

### Added

- `libvulkan` — the whole surface, in seven groups.
  - The instance: `vkEnumerateInstanceVersion`,
    `vkEnumerateInstanceExtensionProperties`,
    `vkEnumerateInstanceLayerProperties`, `vkCreateInstance`,
    `vkDestroyInstance` and `vkGetInstanceProcAddr`.
  - The physical device: `vkEnumeratePhysicalDevices` and the six
    queries beside it — the properties, the features, the memory
    types and heaps, the queue families, one format's capabilities and
    the device extensions.
  - The device and its queues: `vkCreateDevice`, `vkDestroyDevice`,
    `vkGetDeviceProcAddr`, `vkGetDeviceQueue`, `vkQueueSubmit`,
    `vkQueueWaitIdle` and `vkDeviceWaitIdle`.
  - Memory: `vkAllocateMemory`, `vkFreeMemory`, `vkMapMemory`,
    `vkUnmapMemory`, `vkFlushMappedMemoryRanges`,
    `vkInvalidateMappedMemoryRanges` and
    `vkGetDeviceMemoryCommitment`.
  - Buffers: `vkCreateBuffer`, `vkDestroyBuffer`,
    `vkGetBufferMemoryRequirements` and `vkBindBufferMemory`.
  - Command pools and command buffers: the three pool calls, the two
    allocation calls, the three recording calls, and the four
    commands — copy, fill, update and pipeline barrier.
  - Fences and semaphores: `vkCreateFence`, `vkDestroyFence`,
    `vkWaitForFences`, `vkResetFences`, `vkGetFenceStatus`,
    `vkCreateSemaphore` and `vkDestroySemaphore`.
- `tests/libvulkan_tests.nv` — seven tests over the signatures. They
  call the C library, so they need the Vulkan loader installed. Each
  test that needs a physical device asks for the count first and stops
  when it is zero, so the suite passes on a machine with no driver.

### The whole interface is a structure laid out by hand

Vulkan passes and returns nothing by value, so the struct-by-value rule
takes nothing away. What it takes instead is convenience: every
creation call reads a create-info record and every query call fills one
in, and the caller writes each one byte by byte. A record begins with a
four-byte structure type and, after four bytes of padding, an
eight-byte pointer to the next record in a chain. The README's rule 5
carries the layout of every record this package's entry points take,
for the `x86_64` System V layout the loader is compiled to.

`ptr.alloc` does not clear, and a create-info record must be zero apart
from the fields the caller sets, so the test suite reserves every
record through a helper that writes a zero into every word first.

### A queue priority is a float behind a pointer

`VkDeviceQueueCreateInfo` names an array of 32-bit floats, and
`ptr.write_f32` writes one. That is the only floating point value in
this package's surface, and it is reachable exactly because it is
behind a pointer: an `@ffi` declaration's `Float` is a C `double`, so a
Vulkan call taking a `float` argument directly could not have been
bound. None does.

### What a caller can do with it

Everything up to a pipeline. A program can list the physical devices
and read their names, vendors, memory heaps and queue families; open a
logical device and take a queue; allocate device memory and map it into
its own address space; create buffers, learn what memory they need and
bind several of them into one allocation; record a command buffer that
copies, fills and updates buffers with a barrier between the stages;
submit it; and wait on a fence for the device to finish. The tests do
all of that, in order, against whatever device the machine has.

What it cannot do is compute or draw. That needs a pipeline, a pipeline
needs a shader module, and a shader module needs SPIR-V, which
novo-lang has no way to produce. The pipeline, descriptor, render pass
and image surfaces are left out of the first release for that reason
rather than for a rule of the foreign function interface.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Named as missing

**The allocation callbacks.** Every call in Vulkan that creates or
destroys an object takes a `VkAllocationCallbacks *`, whose members are
C FUNCTION POINTERS. Each of those arguments is an `Int` in this
package's signatures and the only value a novo-lang program can pass is
0, which selects the driver's own allocator. Nothing is lost that a
program would ordinarily want.

**Everything reached through `vkGetInstanceProcAddr`.** The two
proc-address calls are here, and what they answer is an address a
novo-lang program CANNOT CALL. They are useful only to ask whether an
entry point exists. Every extension whose entry points the loader does
not export as symbols of its own is therefore out of reach, and so is
`VK_EXT_debug_utils`, whose messenger takes a callback in any case.

**The pipeline and the descriptors.** `vkCreateComputePipelines`,
`vkCreateGraphicsPipelines`, `vkCreateShaderModule`,
`vkCreatePipelineLayout`, `vkCreateDescriptorSetLayout`,
`vkCreateDescriptorPool`, `vkAllocateDescriptorSets`,
`vkUpdateDescriptorSets` and the `vkCmdBindPipeline` /
`vkCmdBindDescriptorSets` / `vkCmdDispatch` / `vkCmdDraw` commands are
left out of the first release. A pipeline is worth nothing without a
SPIR-V module, and there is no novo-lang path to one.

**Images, image views, samplers, render passes and framebuffers.**
They are the drawing half, and they follow the pipeline.

**The window system integration.** `vkCreateSwapchainKHR`,
`vkCreateXlibSurfaceKHR` and their neighbours are exported by the
loader and are left out all the same: a surface needs a window, and
getting one is `libglfw-sys`'s job or the platform's.

**Events, queries, sparse binding and the pipeline cache.** They are
left out of the first release.
