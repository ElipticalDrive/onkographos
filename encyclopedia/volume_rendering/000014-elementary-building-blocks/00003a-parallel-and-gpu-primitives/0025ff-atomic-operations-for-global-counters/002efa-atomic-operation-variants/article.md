# Atomic Operation Variants
Also called: atomicAdd, atomicCAS, atomicMin

## Overview
Atomic operation variants provide specific computational primitives (addition, minimum, logical operations, etc.) that execute as indivisible units on global memory, extending the basic read-modify-write guarantee of atomic operations. While the parent article [Atomic Operations for Global Counters](3684-atomic-operations-for-global-counters.md) establishes the mechanism for race-free updates, these variants define *what* computation occurs during the update. They are distinguished from sibling techniques like the [Thread-Safe Circular Buffer for Brick Allocation](4567-thread-safe-circular-buffer-for-brick-allocation.md) by operating directly on scalar memory locations rather than managing complex data structures or indices.

## Shared principles
All variants share the fundamental property of atomicity: the read, computation, and write phases occur without interruption from other threads accessing the same memory location. This ensures that concurrent invocations from multiple GPU threads result in a deterministic final state consistent with some serial execution order. The interface generally follows the pattern `result = operation(inout memory, operand)`, where `memory` is updated and the previous value is returned.

## Subtopics

### Arithmetic and Comparison
These variants perform numerical accumulation or extremum finding.
- **atomicAdd**: Performs addition. `atomicAdd(inout int mem, int data)` adds `data` to `mem` [S1, unofficial].
- **atomicMin**: Computes the minimum. `atomicMin(inout int mem, int data)` updates `mem` to `min(mem, data)` [S1, unofficial].
- **atomicMax**: Computes the maximum. `atomicMax(inout int mem, int data)` updates `mem` to `max(mem, data)` [S1, unofficial].

### Logical Operations
These variants perform bitwise operations, useful for flag management or bitmasks.
- **atomicAnd**: Logical AND. `atomicAnd(inout int mem, int data)` updates `mem` to `mem & data` [S1, unofficial].
- **atomicOr**: Logical OR. `atomicOr(inout int mem, int data)` updates `mem` to `mem | data` [S1, unofficial].
- **atomicXor**: Logical XOR. `atomicXor(inout int mem, int data)` updates `mem` to `mem ^ data` [S1, unofficial].

### Swap and Compare-Exchange
These variants allow conditional updates or value exchange.
- **atomicExchange**: Swaps the value. `atomicExchange(inout int mem, int data)` sets `mem = data` and returns the original value of `mem` [S1, unofficial].
- **atomicCompSwap**: Compare-and-swap (CAS). `atomicCompSwap(inout int mem, uint compare, uint data)` checks if `mem == compare`. If true, it sets `mem = data`. It always returns the original value of `mem` regardless of whether the swap occurred [S1, unofficial].

Implementation logic for `atomicCompSwap`:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atomicCompSwap(mem, compare, data):
    old = mem
    if mem == compare:
        mem = data
    return old
```

### Deprecated Counters
- **atomicCounter**: Dedicated counter functions (`atomicCounterIncrement`, `atomicCounterDecrement`) existed in Vulkan but are deprecated [S1, unofficial]. They can be replaced by `atomicAdd` [S1, unofficial].

## Background
The specific variants listed are documented in practice guides for GPU programming using GLSL and OpenGL [S1, unofficial]. The distinction between atomic operations and subgroup operations (warp/wave intrinsics) is noted, with atomic operations targeting global memory consistency across threads, whereas subgroup operations coordinate within a hardware thread group [S1, unofficial].

## Sources
[S1] 【WIP】GPUのatomic演算とsubgroup演算の練習帳(GLSL/OpenGL), https://zenn.dev/nishiki/articles/3db056030aaad1, fetched 2026-10-30 (unofficial)
