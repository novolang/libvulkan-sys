# libvulkan-sys

Vulkan is an explicit, low-overhead programming interface to graphics
and compute devices. A program states what it wants in detail, and the
driver does little error checking on its behalf. Checking is done by
validation layers, which a program enables when it wants them. Vulkan
is specified by the Khronos Group in the
[Vulkan specification](https://registry.khronos.org/vulkan/specs/1.3/html/).
On Linux and Windows a program reaches it through a shared library
called the **loader**, which finds the installed drivers and passes
each call to one of them. This package declares fifty of the loader's
entry points to novo-lang, one declaration each.

Every function here is a declaration of a function in the Vulkan
loader. The package contains no logic of its own. It does nothing
without the loader installed, and nothing useful without a driver
installed beside it. The fifty entry points are the ones a program
needs to find a device, give it memory and make it move bytes. The
section "What is not included" says what a program cannot do with them
alone.

## What it is

The **loader** is the shared library this package binds. It is not a
driver. It enumerates the drivers installed on the machine and passes
every call through to one of them.

An **instance** is the program's connection to the loader. It is made
first and released last, and everything else hangs off it.

A **physical device** is one device a driver reports, usually a piece
of hardware and sometimes a software implementation. It is not created
and not destroyed. It is enumerated, and it is asked what it is and
what it can do.

A **logical device** is the program's connection to one physical
device, opened with a list of the queue families it wants. Everything
below is created from it.

A **queue** is where work is submitted. A physical device groups its
queues into **families**, and every queue in a family has the same
capabilities, such as graphics, compute, transfer or a combination. A
queue is not created. It is taken from a logical device that asked for
it.

**Device memory** is allocated from one of the physical device's
**memory types**, each of which belongs to a **heap** and carries a set
of property flags. A type that is *host visible* can be mapped into the
program's own address space. A type that is *device local* is the fast
memory of the device. A type that is both exists on some hardware and
not on others, so a program reads the types rather than assuming them.

A **buffer** is a named region with no memory of its own. It is
created, asked what memory it needs, and then bound to an allocation at
an offset. Several buffers can share one allocation.

A **command buffer** is a recording. Every call whose name begins
`vkCmd` writes into it rather than doing anything, and the recording
runs when it is submitted to a queue.

A **fence** tells the host that the device has finished. A
**semaphore** orders work between queue submissions on the device. A **pipeline
barrier** orders work inside one command buffer, by saying that what
one stage wrote a later stage may read.

## Install

```
novo pkg add libvulkan-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the loader and its headers come from the system package
`libvulkan-dev`, and a driver comes from `mesa-vulkan-drivers` or from
the hardware vendor:

```
sudo apt install libvulkan-dev mesa-vulkan-drivers
```

On macOS Vulkan is not native and reaches Metal through MoltenVK. On
other systems the loader builds from the Khronos source.

## Example

The name of the first device on the machine:

```novo ignore
use libvulkan

fn main() [io, ffi]
    // Every record is reserved zeroed, because `ptr.alloc` does not
    // clear and the fields a caller does not set must be zero or null.
    let app = ptr.alloc(48)
    for i in 0..6
        ptr.write_word(app + i * 8, 0)
    ptr.write_i32(app, 0)            // VK_STRUCTURE_TYPE_APPLICATION_INFO
    ptr.write_i32(app + 44, 4194304) // apiVersion, VK_API_VERSION_1_0

    let info = ptr.alloc(64)
    for i in 0..8
        ptr.write_word(info + i * 8, 0)
    ptr.write_i32(info, 1)           // VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO
    ptr.write_word(info + 24, app)

    let slot = ptr.alloc_word()
    if libvulkan.vk_create_instance(info, 0, slot) as i32 != 0
        println("no Vulkan driver")
        return
    let instance = ptr.read_word(slot)

    // How many devices, then the devices themselves.
    let count = ptr.alloc_word()
    ptr.write_i32(count, 0)
    let _ = libvulkan.vk_enumerate_physical_devices(instance, count, 0)
    let n = ptr.read_word(count) & 4294967295
    println("${n} device(s)")

    if n > 0
        let devices = ptr.alloc(n * 8)
        let _ = libvulkan.vk_enumerate_physical_devices(instance, count, devices)
        let props = ptr.alloc(824)
        libvulkan.vk_get_physical_device_properties(ptr.read_word(devices), props)
        // The name is a 256-byte string at offset 20.
        println(ptr.read_str(props + 20))
        ptr.free(props)
        ptr.free(devices)

    libvulkan.vk_destroy_instance(instance, 0)
    ptr.free(count)
    ptr.free(slot)
    ptr.free(info)
    ptr.free(app)
```

The example is not compiled, because it links against the Vulkan loader
and the link fails where the loader is not installed. The same calls are in
`tests/libvulkan_tests.nv`, where the answers are asserted.

## What the package contains

| Module | Contents |
| --- | --- |
| `libvulkan` | Every entry point, in seven groups: the instance, the physical device query, the logical device and its queues, device memory, buffers, command pools and their commands, and the fences and semaphores. |

The seven groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| The instance | 6 | Reports the loader's version and what is installed, and creates and destroys an instance. |
| The physical device | 7 | Lists the hardware and asks each piece what it is and what it can do. |
| The device and its queues | 7 | Opens a logical device, takes a queue, submits work and waits for it. |
| Memory | 7 | Allocates on the device, maps it into the process, and keeps the two views in step. |
| Buffers | 4 | Creates a region, learns what memory it needs, and binds it. |
| Command pools and commands | 12 | Records a copy, a fill, an update and a barrier, and resets what it recorded into. |
| Fences and semaphores | 7 | Tells the host that the device has finished, and orders work on the device. |

## How to choose an entry point

Every enumeration in Vulkan is a **two-call sequence**. Call it with a
null buffer to learn the count, reserve that many records, and call it
again. `vkEnumerateInstanceExtensionProperties`,
`vkEnumerateInstanceLayerProperties`, `vkEnumeratePhysicalDevices`,
`vkGetPhysicalDeviceQueueFamilyProperties` and
`vkEnumerateDeviceExtensionProperties` all work that way.

`vkQueueWaitIdle` waits for one queue and `vkDeviceWaitIdle` for all of
them. A fence says which submission finished, and it can be waited on
with a timeout.

`vkCmdFillBuffer` writes one repeated four-byte value and is for
clearing. `vkCmdUpdateBuffer` copies up to 65536 bytes recorded into
the command buffer itself and is for small constant data.
`vkCmdCopyBuffer` moves bytes between two buffers and is for
everything else.

`vkGetInstanceProcAddr` and `vkGetDeviceProcAddr` answer addresses this
package cannot call. Use them to ask whether an extension's entry point
exists, and for nothing else.

## The rules a user needs

1. **A handle is an `Int`, and zero is `VK_NULL_HANDLE`.** A
   dispatchable handle is a pointer. The instance, a physical device, a
   device, a queue and a command buffer are dispatchable. A
   non-dispatchable handle is a 64-bit number. A buffer, an allocation,
   a command pool, a fence and a semaphore are non-dispatchable. Both
   fit an `Int`.
2. **A result code is signed and a failure is negative.** Bind the
   answer to a local with `as i32` before comparing it with zero.

   | Code | Name | What it means |
   | --- | --- | --- |
   | 0 | `VK_SUCCESS` | it worked |
   | 1 | `VK_NOT_READY` | the fence is not signalled, which is not an error |
   | 2 | `VK_TIMEOUT` | the wait ran out of time, which is not an error |
   | 5 | `VK_INCOMPLETE` | the buffer held fewer records than there are |
   | -1 | `VK_ERROR_OUT_OF_HOST_MEMORY` | the host allocator failed |
   | -2 | `VK_ERROR_OUT_OF_DEVICE_MEMORY` | the device allocator failed |
   | -3 | `VK_ERROR_INITIALIZATION_FAILED` | an object could not be initialised |
   | -4 | `VK_ERROR_DEVICE_LOST` | the device was lost |
   | -5 | `VK_ERROR_MEMORY_MAP_FAILED` | the mapping failed |
   | -6 | `VK_ERROR_LAYER_NOT_PRESENT` | the layer is not installed |
   | -7 | `VK_ERROR_EXTENSION_NOT_PRESENT` | the extension is not there |
   | -8 | `VK_ERROR_FEATURE_NOT_PRESENT` | the feature is not there |
   | -9 | `VK_ERROR_INCOMPATIBLE_DRIVER` | no compatible driver was found |

3. **An out-parameter is the address of a caller-owned slot.** Every
   call that produces a handle writes it into one. `ptr.alloc_word`
   reserves a slot and `ptr.read_word` reads it back.
4. **An enumeration is called twice.** The first call passes a null
   buffer and gets the count. The second passes a buffer of that size. A count slot is a
   four-byte unsigned value, so write it with `ptr.write_i32` and read
   it with `ptr.read_word(a) & 4294967295`.
5. **Every record is laid out by hand, and this is the layout.** The
   offsets are those of the `x86_64` System V layout the loader is
   compiled to. Every record begins with a four-byte structure type at
   0 and an eight-byte `pNext` at 8. The four bytes at 4 are padding.
   `pNext` is 0 unless the record extends into a chain.

   | Record | Type | Size | Fields after `pNext` |
   | --- | --- | --- | --- |
   | `VkApplicationInfo` | 0 | 48 | name at 16, version at 24, engine name at 32, engine version at 40, API version at 44 |
   | `VkInstanceCreateInfo` | 1 | 64 | flags at 16, application info at 24, layer count at 32, layer names at 40, extension count at 48, extension names at 56 |
   | `VkDeviceQueueCreateInfo` | 2 | 40 | flags at 16, family index at 20, queue count at 24, priorities at 32 |
   | `VkDeviceCreateInfo` | 3 | 72 | flags at 16, queue info count at 20, queue infos at 24, layer count at 32, layer names at 40, extension count at 48, extension names at 56, features at 64 |
   | `VkSubmitInfo` | 4 | 72 | wait count at 16, wait semaphores at 24, wait stages at 32, command buffer count at 40, command buffers at 48, signal count at 56, signal semaphores at 64 |
   | `VkMemoryAllocateInfo` | 5 | 32 | size at 16, memory type index at 24 |
   | `VkMappedMemoryRange` | 6 | 40 | memory at 16, offset at 24, size at 32 |
   | `VkFenceCreateInfo` | 8 | 24 | flags at 16 |
   | `VkSemaphoreCreateInfo` | 9 | 24 | flags at 16 |
   | `VkBufferCreateInfo` | 12 | 56 | flags at 16, size at 24, usage at 32, sharing mode at 36, family count at 40, family indices at 48 |
   | `VkCommandPoolCreateInfo` | 39 | 24 | flags at 16, queue family index at 20 |
   | `VkCommandBufferAllocateInfo` | 40 | 32 | pool at 16, level at 24, count at 28 |
   | `VkCommandBufferBeginInfo` | 42 | 32 | flags at 16, inheritance info at 24 |
   | `VkMemoryBarrier` | 46 | 24 | source access mask at 16, destination access mask at 20 |

   The API version of `VkApplicationInfo` is at 44, and it is the only
   field most callers set. A value of 0 is taken as Vulkan 1.0.

6. **A record a query fills in has no structure type.** These are the
   plain ones, and their sizes are what `ptr.alloc` must reserve.

   | Record | Size | Fields |
   | --- | --- | --- |
   | `VkPhysicalDeviceProperties` | 824 | API version at 0, driver version at 4, vendor at 8, device at 12, type at 16, a 256-byte name at 20 |
   | `VkPhysicalDeviceFeatures` | 220 | 55 four-byte flags, each 0 or 1 |
   | `VkPhysicalDeviceMemoryProperties` | 520 | type count at 0, 32 eight-byte types from 4, heap count at 260, 16 sixteen-byte heaps from 264 |
   | `VkQueueFamilyProperties` | 24 each | capability flags at 0, queue count at 4, timestamp bits at 8 |
   | `VkFormatProperties` | 12 | linear features at 0, optimal features at 4, buffer features at 8 |
   | `VkMemoryRequirements` | 24 | size at 0, alignment at 8, allowed memory types at 16 |
   | `VkBufferCopy` | 24 each | source offset at 0, destination offset at 8, length at 16 |
   | `VkExtensionProperties` | 260 each | a 256-byte name at 0, the specification version at 256 |
   | `VkLayerProperties` | 520 each | a 256-byte name at 0, two versions at 256 and 260, a 256-byte description at 264 |

7. **`ptr.alloc` does not clear the bytes it reserves.** The fields of
   a create-info record the caller does not set must be zero or null,
   such as the reserved `flags` and an unused `pNext`. Write every word
   of the record before the call, including the fields that stay zero. The test suite does it with a helper that
   writes a zero into every eight bytes.
8. **A memory type is chosen, not assumed.** Read the types from
   `vkGetPhysicalDeviceMemoryProperties`, keep only the ones the
   resource's memory requirements allow, and pick one whose property
   flags carry what is needed.

   | Bit | Flag | What it means |
   | --- | --- | --- |
   | 1 | `DEVICE_LOCAL` | the fastest memory for the device |
   | 2 | `HOST_VISIBLE` | it can be mapped into the process |
   | 4 | `HOST_COHERENT` | a mapped write needs no flush |
   | 8 | `HOST_CACHED` | a mapped read is cached |
   | 16 | `LAZILY_ALLOCATED` | the device may commit it on use, and it is never host visible |

9. **A binding offset is a multiple of the reported alignment.**
   `vkGetBufferMemoryRequirements` answers a size that is at least the
   size asked for and an alignment the offset must be a multiple of.
   Round up with `(size + align - 1) / align * align`.
10. **A queue priority is a 32-bit float behind a pointer.**
    `VkDeviceQueueCreateInfo` names an array of them, and
    `ptr.write_f32` writes one. It is the only float in this package's
    surface.
11. **A fence starts unsignalled unless it is created signalled.** The
    device signals it and the host resets it. `vkGetFenceStatus`
    answers 0 when it is signalled and 1 when it is not, and neither is
    an error. `vkWaitForFences` answers 2 on a timeout, which is not an
    error either.
12. **A command buffer records and does not run.** Every `vkCmd` call
    between `vkBeginCommandBuffer` and `vkEndCommandBuffer` writes into
    the buffer. An error during recording, such as memory running out,
    is reported by `vkEndCommandBuffer` rather than by the call that met
    it. Incorrect use is not reported at all unless a validation layer
    is enabled.
13. **`vkCmdUpdateBuffer` copies the bytes at record time.** The
    address it is given need not outlive the call, and its offset and
    length are multiples of four. `vkCmdCopyBuffer` copies when the
    recording runs, so its buffers must still exist then.
14. **`vkCmdFillBuffer` writes one four-byte value over and over.**
    The offset and the length must both be multiples of four.
15. **The allocator argument is always 0.** Every create and destroy
    call takes a `VkAllocationCallbacks *`, whose members are C
    function pointers. Passing 0 selects the driver's own allocator.
16. **Destroy in the reverse order of creation, and wait first.**
    Command buffers go before their pool, buffers before the memory
    they are bound to, everything before the device, and the device
    before the instance. `vkDeviceWaitIdle` comes before the first
    destruction.
17. **`VK_WHOLE_SIZE` is -1.** It is the largest unsigned 64-bit value,
    and `vkMapMemory` and `VkMappedMemoryRange` both take it.
18. **A version is packed into 32 bits.** The variant is the top three
    bits, the major version the next seven, the minor the next ten and
    the patch the low twelve. For the variant 0 that every desktop
    driver reports, `bits.shr(v, 22)` is the major version.

## What is not included

- **The allocation callbacks.** `VkAllocationCallbacks` is a structure
  of C function pointers, and the novo-lang foreign function interface
  passes integers, floats and strings. Every call that takes one takes
  it as an `Int` here, and 0 selects the driver's own allocator.
- **Everything reached through `vkGetInstanceProcAddr`.** The two
  proc-address calls are here, and the address they answer is one a
  novo-lang program cannot call. An extension whose entry points the
  loader does not export as symbols is therefore unreachable, and so is
  `VK_EXT_debug_utils`, whose messenger takes a callback as well.
- **The pipeline and the descriptors.** `vkCreateShaderModule`,
  `vkCreateComputePipelines`, `vkCreateGraphicsPipelines`, the pipeline
  and descriptor set layouts, the descriptor pool and its sets, and the
  `vkCmdBindPipeline`, `vkCmdBindDescriptorSets`, `vkCmdDispatch` and
  `vkCmdDraw` commands are left out of this release. A pipeline needs a
  shader module in SPIR-V, and novo-lang has no way to produce one. A
  program with these bindings can therefore neither compute nor draw.
- **Images, image views, samplers, render passes and framebuffers.**
  They are the drawing half, and they follow the pipeline.
- **The window system integration.** `vkCreateSwapchainKHR`,
  `vkCreateXlibSurfaceKHR` and their neighbours are exported by the
  loader and are left out of this release. A surface needs a window.
- **Events, query pools, sparse binding and the pipeline cache.** They
  are left out of this release.

## Related packages

No novo-lang package implements Vulkan. Vulkan is the interface a
driver presents to a program, and the implementation is the driver
installed on the machine.

[libglfw-sys](https://novo-lang.org/packages/libglfw-sys) binds GLFW,
which creates the window a swapchain would present to and answers the
instance extensions a surface needs on the current platform.

[libcuda-sys](https://novo-lang.org/packages/libcuda-sys) and
[libhip-sys](https://novo-lang.org/packages/libhip-sys) bind the CUDA
runtime for NVIDIA devices and the HIP runtime for AMD devices. They
declare the device count, device memory, the copies and the
synchronisation call. This package reaches any device with a Vulkan
driver, and moves bytes on it in the same way.

## Tests

`tests/libvulkan_tests.nv` holds seven tests over the fifty entry
points. They call the C library, so `novo test` needs the Vulkan
loader installed and linkable:

```
novo test tests/libvulkan_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The loader is there once the library is, and a driver may not be. The
loader creates an instance only when it finds a driver, so on a machine
with none the two tests that assert an instance fail. Every other test
that needs a physical device stops when there is none. Nothing here
reads or writes a file, and nothing needs privileges.

With a device, the suite asserts what the specification says. The
loader reports its version and its extension and layer lists, and a
missing layer is reported rather than ignored. An instance is created
from a record laid out by hand. The first device reports its version,
vendor, type and name, its memory types and heaps, and its queue
families. A logical device opens with one transfer queue. An
allocation is mapped and read back through `ptr`. Two buffers are
bound into one allocation at an aligned offset. A command buffer copies
four bytes, fills four words and updates one. It is submitted to the
queue and waited for on a fence, and the result is read back through
the mapping.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

The Vulkan loader itself is distributed under the Apache 2.0 licence,
and installing it and a driver is the reader's own step.
