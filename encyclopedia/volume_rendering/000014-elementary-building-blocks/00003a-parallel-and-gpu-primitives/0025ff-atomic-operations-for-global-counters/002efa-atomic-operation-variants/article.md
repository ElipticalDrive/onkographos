# Atomic Operation Variants

Also called: atomicAdd, atomicCAS, atomicMin

## Overview
Atomic Operation Variants extend the basic atomic write primitive to support specific reduction and comparison operations directly in global memory, enabling thread-safe accumulation of values without explicit locking or fetch-modify-write cycles. Unlike the base [Atomic Operations for Global Counters](3684-atomic-operations-for-global-counters.md) which focuses on the fundamental mechanism of race-condition avoidance, this family provides specialized hardware-supported operations for arithmetic, logical, and conditional updates. It differs from sibling techniques like [Thread-Safe Circular Buffer for Brick Allocation](4567-thread-safe-circular-buffer-for-brick-allocation.md) by operating on scalar or vector memory locations rather than managing buffer indices or allocation states.

## Shared principles
All variants in this family operate on a memory location `mem` and one or more input arguments. They guarantee that the read-modify-write sequence is indivisible with respect to other threads accessing the same memory location. The return value is typically the original value of `mem` before the operation, allowing the caller to detect if a change occurred or to implement retry loops (e.g., in CAS-based algorithms).

The core mathematical operations supported are:
1.  **Arithmetic**: Addition, Minimum, Maximum.
2.  **Logical**: Bitwise AND, OR, XOR.
3.  **Exchange**: Swap with return of old value.
4.  **Conditional Swap**: Compare-and-Swap (CAS).

All operations assume the memory location is of an integer or unsigned integer type, or a floating-point type where the hardware provides specific atomic support (though the sources primarily detail integer variants).

## Subtopics

### atomicAdd
Performs an atomic addition of `data` to `mem`.
- **Definition**: `atomicAdd(inout int mem, int data)` adds `data` to the current value of `mem` [S1, unofficial].
- **Usage**: Commonly used for accumulating counts, sums, or histograms.
- **Note**: Deprecated `atomicCounterIncrement` and `atomicCounterDecrement` functions in Vulkan can be replaced by `atomicAdd` with `+1` or `-1` [S1, unofficial].

### atomicMin
Computes the minimum of `mem` and `data`.
- **Definition**: `atomicMin(inout int mem, int data)` updates `mem` to $\min(\text{mem}, \text{data})$ [S1, unofficial].
- **Usage**: Useful for finding nearest neighbors, minimum distances, or bounding box extents.

### atomicMax
Computes the maximum of `mem` and `data`.
- **Definition**: `atomicMax(inout int mem, int data)` updates `mem` to $\max(\text{mem}, \text{data})$ [S1, unofficial].
- **Usage**: Useful for finding farthest points, maximum intensities, or bounding box extents.

### atomicAnd
Performs a bitwise logical AND.
- **Definition**: `atomicAnd(inout int mem, int data)` updates `mem` to $\text{mem} \ \& \ \text{data}$ [S1, unofficial].
- **Usage**: Bitmask filtering or flag management.

### atomicOr
Performs a bitwise logical OR.
- **Definition**: `atomicOr(inout int mem, int data)` updates `mem` to $\text{mem} \ | \ \text{data}$ [S1, unofficial].
- **Usage**: Setting flags or combining bitmasks.

### atomicXor
Performs a bitwise logical XOR.
- **Definition**: `atomicXor(inout int mem, int data)` updates `mem` to $\text{mem} \ \oplus \ \text{data}$ [S1, unofficial].
- **Usage**: Parity checks or toggling specific bits.

### atomicExchange
Swaps the value of `mem` with `data` and returns the original value.
- **Definition**: `atomicExchange(inout int mem, int data)` sets `mem = data` and returns the old value of `mem` [S1, unofficial].
- **Usage**: Implementing locks or swapping buffers.

### atomicCompSwap
Performs a conditional swap if the current value matches a comparison value.
- **Definition**: `atomicCompSwap(inout int mem, uint compare, uint data)` checks if `mem == compare`. If true, sets `mem = data`. Returns the original value of `mem` regardless of whether the swap occurred [S1, unofficial].
- **Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atomicCompSwap(mem, compare, data):
    # mem is an inout reference to memory
    # compare and data are uint values
    old = mem[0]
    if mem[0] == compare:
        mem[0] = data
    return old
```
- **Usage**: The foundation for lock-free data structures and retry loops.

### Atomic Average Image Store
Performs an atomic weighted average operation on a 3D image buffer, typically used for resolving conflicting values in voxelization or volume rendering.
- **Definition**: Atomically updates a 3D image buffer with the average of the current stored value and a new incoming value [S2, unofficial].
- **Mechanism**:
    1.  Convert the incoming vector value `val` (e.g., `vec3`) to a scaled integer representation (e.g., 255 scale) and pack it into a `uint` (e.g., `newVal`) using a helper function `vecToRgba` [S2, unofficial].
    2.  Enter an infinite loop.
    3.  Perform an atomic compare-swap: `curVal = imageAtomicCompSwap(imgUI, coords, prevVal, newVal)` [S2, unofficial].
    4.  If `curVal == prevVal`, the swap succeeded (no other thread modified the value between read and write). Return.
    5.  If `curVal != prevVal`, another thread modified the value. Update `prevVal = curVal`.
    6.  Convert `curVal` back to a floating-point vector `rval` using `rgbaToVec` [S2, unofficial].
    7.  Calculate the weighted average:
        $$ \text{rval.xyz} = (\text{rval.xyz} \times \text{rval.w}) $$
        $$ \text{curValF} = \text{rval} + \text{val} $$
        $$ \text{curValF.xyz} /= \text{curValF.w} $$
        [S2, unofficial]
    8.  Pack the new average `curValF` into `newVal` using `vecToRgba` and repeat the loop [S2, unofficial].
- **Pitfall**: The implementation using `imageAtomicCompSwap` may not compile on some drivers due to driver bugs or strict type checking [S2, unofficial].
- **Usage**: Resolving multiple ray contributions or particle densities into a single voxel value without race conditions.

## Background
Atomic operations are a fundamental primitive in GPU programming, supported by hardware in NVIDIA (warp-level) and AMD (wave-level) architectures. The specific variants listed (Add, Min, Max, And, Or, Xor, Exchange, CompSwap) are standard in OpenGL/GLSL and Vulkan shading languages. The `atomicCounter` functions were deprecated in favor of `atomicAdd` in Vulkan to unify the API [S1, unofficial]. The Atomic Average Image Store is a specialized application of CAS loops found in voxelization pipelines, such as those in the `gidemo` project, to handle concurrent writes to 3D textures [S2, unofficial].

## Sources
[S1] 【WIP】GPUのatomic演算とsubgroup演算の練習帳(GLSL/OpenGL), https://zenn.dev/nishiki/articles/3db056030aaad1, fetched 2026-10-30 (unofficial)
[S2] righier/gidemo, https://github.com/righier/gidemo, fetched 2026-10-26 (unofficial)
