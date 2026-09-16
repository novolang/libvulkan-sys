# libvulkan-sys

Vulkan is a low-level interface to a graphics and compute device. A
program describes what it wants in explicit detail, and the driver does
almost nothing on its behalf: it allocates no memory the program did
not ask for and validates nothing the program did not enable. It is
specified by the Khronos Group in the
[Vulkan specification](https://registry.khronos.org/vulkan/specs/1.3/html/),
and on a Linux or Windows machine every program reaches it through a
shared library called the **loader**, which finds the installed drivers
and dispatches into them. This package declares fifty of that loader's
entry points to novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in the Vulkan loader. The package contains no
logic of its own, it does nothing without the C library installed, and
it does nothing useful without a driver installed beside it. The fifty
entry points are the ones a program needs to find a device, give it
memory and make it move bytes; the section "What is not included" says
what a program still cannot do with them alone.

## What it is

The **loader** is the shared library this package binds. It is not a
driver. It enumerates the drivers installed on the machine and passes
every call through to one of them.

An **instance** is the program's connection to the loader. It is made
first and released last, and everything else hangs off it.

A **physical device** is one piece of hardware the loader found. It is
not created and not released: it is enumerated, and it is asked what it
is and what it can do.

A **logical device** is the program's connection to one physical
device, opened with a list of the queue families it wants. Everything
below is created from it.

A **queue** is where work is submitted. A physical device groups its
queues into **families**, and every queue in a family has the same
capabilities: graphics, compute, transfer, or a combination. A queue is
not created; it is taken out of a logical device that asked for it.

**Device memory** is allocated from one of the physical device's
**memory types**, each of which belongs to a **heap** and carries a set
of property flags. A type that is *host visible* can be mapped into the
program's own address space. A type that is *device local* is the fast
memory on the card. A type that is both exists on some hardware and not
on others, which is why a program reads the types rather than assuming
them.

A **buffer** is a named region with no memory of its own. It is
created, asked what memory it needs, and then bound to an allocation at
an offset. Several buffers can share one allocation.

A **command buffer** is a recording. Every call whose name begins
`vkCmd` writes into it rather than doing anything, and the recording
runs when it is submitted to a queue.

A **fence** tells the host that the device has finished. A
**semaphore** orders work between queues on the device. A **pipeline
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
    // clear and a create-info must be zero apart from what is set.
    let app = ptr.alloc(48)
    for i in 0..6
        ptr.write_word(app + i * 8, 0)
    ptr.write_i32(app, 0)            // VK_STRUCTURE_TYPE_APPLICATION_INFO
    ptr.write_i32(app + 40, 4194304) // VK_API_VERSION_1_0

    let info = ptr.alloc(64)
    for i in 0..8
        ptr.write_word(info + i * 8, 0)
    ptr.write_i32(info, 1)           // VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO
    ptr.write_word(info + 24, app)

    let slot = ptr.alloc_word()
    if libvulkan.vk_create_instance(info, 0, slot) as i32 != 0
        println("no Vulkan loader")
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

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls are in
`tests/libvulkan_tests.nv`, where the answers are asserted.

## What the package contains

| Module | Contents |
| --- | --- |
| `libvulkan` | Every entry point, in seven groups: the instance, the physical device query, the logical device and its queues, device memory, buffers, command pools and their commands, and the fences and semaphores. |

The seven groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| The instance | 6 | Reports the loader's version and what is installed, and opens the connection. |
| The physical device | 7 | Lists the hardware and asks each piece what it is and what it can do. |
| The device and its queues | 7 | Opens a logical device, takes a queue, submits work and waits for it. |
| Memory | 7 | Allocates on the device, maps it into the process, and keeps the two views in step. |
| Buffers | 4 | Creates a region, learns what memory it needs, and binds it. |
| Command pools and commands | 12 | Records a copy, a fill, an update and a barrier, and resets what it recorded into. |
| Fences and semaphores | 7 | Tells the host that the device has finished, and orders work on the device. |

## How to choose an entry point

Every enumeration in Vulkan is a **two-call sequence**: call it with a
null buffer to learn the count, reserve that many records, and call it
again. `vkEnumerateInstanceExtensionProperties`,
`vkEnumerateInstanceLayerProperties`, `vkEnumeratePhysicalDevices`,
`vkGetPhysicalDeviceQueueFamilyProperties` and
`vkEnumerateDeviceExtensionProperties` all work that way.

`vkQueueWaitIdle` waits for one queue and `vkDeviceWaitIdle` for all of
them. A fence is the finer instrument: it says which submission
finished, and it can be waited on with a timeout.

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
   dispatchable handle — an instance, a physical device, a device, a
   queue, a command buffer — is a pointer. A non-dispatchable handle —
   a buffer, an allocation, a command pool, a fence, a semaphore — is a
   64-bit number. Both fit an `Int`.
2. **A result code is signed and a failure is negative.** Bind the
   answer to a local with `as i32` before comparing it with zero.

   | Code | Name | What it means |
   | --- | --- | --- |
   | 0 | `VK_SUCCESS` | it worked |
   | 1 | `VK_NOT_READY` | the fence is not signalled; not an error |
   | 2 | `VK_TIMEOUT` | the wait ran out of time; not an error |
   | 5 | `VK_INCOMPLETE` | the buffer held fewer records than there are |
   | -1 | `VK_ERROR_OUT_OF_HOST_MEMORY` | the host allocator failed |
   | -2 | `VK_ERROR_OUT_OF_DEVICE_MEMORY` | the device allocator failed |
   | -6 | `VK_ERROR_LAYER_NOT_PRESENT` | the layer is not installed |
   | -7 | `VK_ERROR_EXTENSION_NOT_PRESENT` | the extension is not there |
   | -9 | `VK_ERROR_INCOMPATIBLE_DRIVER` | no driver supports the version asked for |

3. **An out-parameter is the address of a caller-owned slot.** Every
   call that produces a handle writes it into one. `ptr.alloc_word`
   reserves a slot and `ptr.read_word` reads it back.
4. **An enumeration is called twice.** Once with a null buffer to get
   the count, once with a buffer of that size. A count slot is a
   four-byte unsigned value, so write it with `ptr.write_i32` and read
   it with `ptr.read_word(a) & 4294967295`.
5. **Every record is laid out by hand, and this is the layout.** The
   offsets are those of the `x86_64` System V layout the loader is
   compiled to. Every record begins with a four-byte structure type at
   0 and an eight-byte `pNext` at 8; the four bytes at 4 are padding
   and must be zero.

   | Record | Type | Size | Fields after `pNext` |
   | --- | --- | --- | --- |
   | `VkApplicationInfo` | 0 | 48 | name at 16, version at 24, engine at 32, engine version at 40 — and the API version at 40 |
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

   `VkApplicationInfo`'s application version sits at 24 and its engine
   name at 32, and its API version — the only field most callers set —
   at 40.

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

7. **`ptr.alloc` does not clear the bytes it reserves.** A create-info
   record must be zero apart from the fields the caller sets, so write
   every word of it, including the padding and the fields that stay
   zero, before the call. The test suite does it with a helper that
   writes a zero into every eight bytes.
8. **A memory type is chosen, not assumed.** Read the types from
   `vkGetPhysicalDeviceMemoryProperties`, keep only the ones the
   resource's memory requirements allow, and pick one whose property
   flags carry what is needed.

   | Bit | Flag | What it means |
   | --- | --- | --- |
   | 1 | `DEVICE_LOCAL` | the fast memory on the card |
   | 2 | `HOST_VISIBLE` | it can be mapped into the process |
   | 4 | `HOST_COHERENT` | a mapped write needs no flush |
   | 8 | `HOST_CACHED` | a mapped read is cached |
   | 16 | `LAZILY_ALLOCATED` | the device commits it on use |

9. **A binding offset is a multiple of the reported alignment.**
   `vkGetBufferMemoryRequirements` answers a size that is at least the
   size asked for and an alignment the offset must be a multiple of.
   Round up: `(size + align - 1) / align * align`.
10. **A queue priority is a 32-bit float behind a pointer.**
    `VkDeviceQueueCreateInfo` names an array of them, and
    `ptr.write_f32` writes one. It is the only float in this package's
    surface.
11. **A fence starts unsignalled, the device signals it and the host
    resets it.** `vkGetFenceStatus` answers 0 when it is signalled and
    1 when it is not, and neither is an error. `vkWaitForFences`
    answers 2 on a timeout, which is not an error either.
12. **A command buffer records; it does not run.** Every `vkCmd` call
    between `vkBeginCommandBuffer` and `vkEndCommandBuffer` writes into
    the buffer, and a malformed recording is reported by
    `vkEndCommandBuffer` rather than by the call that caused it.
13. **`vkCmdUpdateBuffer` copies the bytes at record time.** The
    address it is given need not outlive the call. `vkCmdCopyBuffer`
    does not: its buffers must still exist when the recording runs.
14. **`vkCmdFillBuffer` writes one four-byte value over and over.**
    The offset and the length must both be multiples of four.
15. **The allocator argument is always 0.** Every create and destroy
    call takes a `VkAllocationCallbacks *`, whose members are C
    function pointers. Passing 0 selects the driver's own allocator.
16. **Release in the reverse order of creation, and wait first.**
    Command buffers before their pool, buffers before the memory they
    are bound to, everything before the device, the device before the
    instance. `vkDeviceWaitIdle` comes before the first release.
17. **`VK_WHOLE_SIZE` is -1.** It is the largest unsigned 64-bit value,
    and `vkMapMemory` and `VkMappedMemoryRange` both take it.
18. **A version is packed into 32 bits.** The major version is the top
    ten bits, the minor the next ten and the patch the low twelve.
    `bits.shr(v, 22)` is the major version.

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
  `vkCmdDraw` commands. A pipeline is worth nothing without a SPIR-V
  module, and novo-lang has no way to produce one.
- **Images, image views, samplers, render passes and framebuffers.**
  They are the drawing half, and they follow the pipeline.
- **The window system integration.** `vkCreateSwapchainKHR`,
  `vkCreateXlibSurfaceKHR` and their neighbours are exported by the
  loader and are left out all the same: a surface needs a window.
- **Events, query pools, sparse binding and the pipeline cache.** They
  are left out of the first release.

## Related packages

There is no Vulkan port in novo-lang and there will not be one. Vulkan
is not a format or an algorithm; it is the interface an operating
system and a driver present to a program, and the only implementation
is the one on the machine.

`libglfw-sys` creates the window a swapchain would present to, and
answers the instance extensions a surface needs on the current
platform.

`libcuda-sys` and `libhip-sys` are the other two ways to reach a device
in this registry. They are compute-only and vendor-specific; Vulkan is
neither, and its compute support needs a SPIR-V module this package
does not help produce.

## Tests

`tests/libvulkan_tests.nv` holds seven tests written against the
signatures. They call the C library, so `novo test` needs the Vulkan
loader installed and linkable:

```
novo test tests/libvulkan_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The loader is present once the library is; a driver may not be. Every
test that needs a physical device asks for the count first and stops
when it is zero, so the suite passes on a machine with no device as
well as on one with a device. Nothing here reads or writes a file, and
nothing needs privileges.

With a device, the suite asserts the loader's version and its extension
and layer lists, that a missing layer is reported rather than ignored,
that an instance is created from a record laid out by hand, the first
device's version, vendor, type and name, its memory types and heaps and
its queue families, a logical device with one transfer queue, an
allocation mapped and read back through `ptr`, two buffers bound into
one allocation at an aligned offset, and finally a command buffer that
copies four bytes, fills four words and updates one, submitted to the
queue and waited for on a fence — with the result read back through the
mapping.

`novo --leak-check` reports five leaked objects at the end of the run.
They are the `ptr.read_str` and `ptr.read_bytes_n` copies the tests
make out of the device's own strings; both are declared untracked,
which is a defect in the toolchain and not in this package.

## Implementation status

| Group | State |
| --- | --- |
| The instance | Complete for the core, with no extensions or layers enabled. |
| The physical device | Complete for the six 1.0 queries. |
| The device and its queues | Complete for one submission path. |
| Memory | Complete, including the mapping and the two coherency calls. |
| Buffers | Complete. |
| Command pools and commands | Complete for the four transfer commands. |
| Fences and semaphores | Complete for fences; a semaphore is created and released only. |
| The allocation callbacks | Absent. They are C function pointers. |
| Anything behind a proc address | Absent. The address cannot be called. |
| Pipelines and descriptors | Absent. They need a SPIR-V module. |
| Images and render passes | Absent. They follow the pipeline. |
| Swapchains and surfaces | Absent. A surface needs a window. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

The Vulkan loader itself is distributed under the Apache 2.0 licence,
and installing it and a driver is the reader's own step.
