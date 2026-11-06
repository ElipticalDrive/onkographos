# Atomic Operations for Global Counters

Also called: atomic operations, atomicAdd

## Overview
Atomic operations for global counters provide a mechanism to safely write to global memory from multiple GPU threads simultaneously, avoiding race conditions (collisions) that occur when concurrent threads attempt to update the same memory location [S1, unofficial]. This method solves the problem of generating unique, thread-safe indices or accumulating values in a shared buffer without requiring coarse-grained locks or serialization. It sets itself apart from sibling techniques like [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by operating directly on scalar memory locations rather than managing complex data structures or spatial partitions, and from [Parallel Prefix Sum (Scan) Orchestration](2593-parallel-prefix-sum-scan-orchestration.md) by providing immediate, per-thread unique identifiers rather than aggregated range information.

## How it works
**Inputs and Outputs:**
*   **Input:** A global memory buffer (e.g., a `std430` storage buffer) containing a counter variable (typically `uint`) initialized to zero, and a target array buffer for storing results.
*   **Output:** The counter buffer holds the total number of invocations; the target array holds unique indices or accumulated values assigned to each thread.

**Data Structures:**
1.  **Count Buffer:** A `std430` storage buffer binding (e.g., binding 0) containing a single `uint` variable named `count`.
    *   Layout: `[uint count]`
2.  **Array Buffer:** A `std430` storage buffer binding (e.g., binding 1) containing a dynamic array of `uint`.
    *   Layout: `[uint array[]]`

**Parameters:**
*   `local_size_x`: The number of threads per workgroup. A typical value used in examples is 64 [S1, unofficial].

**Steps:**
1.  Initialize the global counter buffer to 0 on the host or via a clear operation.
2.  In the compute shader, each thread calculates its global ID (`gid`).
3.  The thread invokes `atomicAdd(count, 1)`. This operation atomically adds 1 to the global counter and returns the *pre-increment* value. This returned value serves as a unique index for the calling thread [S1, unofficial].
4.  The thread stores its result (e.g., the returned index or a computed value) into the global array at the position determined by the returned index.
5.  For conditional packing, the thread evaluates a condition (e.g., `value % 100 == 0`). If true, it calls `atomicAdd` to get an index and stores the value at that index in the array. If false, it skips the atomic operation and storage [S1, unofficial].

**Pseudocode:**
The following Python-like pseudocode demonstrates the basic counter and index assignment pattern. It assumes a compute shader environment where `atomicAdd` is a built-in intrinsic.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for basic atomic counter and index assignment
def compute_shader_main():
    # 1. Get global thread ID
    gid = get_global_invocation_id_x()
    
    # 2. Atomically increment global counter and get unique index
    # atomicAdd returns the value of 'count' BEFORE the addition
    index = atomic_add(global_count_buffer, 1)
    
    # 3. Store the unique index into the result array at the thread's global ID
    # Note: Storing at 'gid' maps the result to the thread ID, 
    # while storing at 'index' packs results contiguously.
    # The example in [S1, unofficial] stores at 'gid' for verification, 
    # but conditional packing (see below) uses 'index' for compact storage.
    result_array[gid] = index
    
    # Conditional Packing Variant (from [S1, unofficial]):
    # value = gid * gid
    # if value % 100 == 0:
    #     compact_index = atomic_add(global_count_buffer, 1)
    #     result_array[compact_index] = value
```

**Edge Cases and Pitfalls:**
*   **Race Conditions:** Using non-atomic increment (e.g., `count++`) on global memory results in incorrect counts due to race conditions where threads read stale values [S1, unofficial].
*   **Ordering:** The order of indices returned by `atomicAdd` is not guaranteed to be sorted, even if it appears sorted in small, single-workgroup examples [S1, unofficial]. Implementations must not rely on the returned indices being sequential or sorted relative to thread IDs.
*   **Buffer Size:** The target array must be large enough to hold the maximum possible number of writes. If using conditional packing, the array size must accommodate the worst-case scenario (all threads passing the condition) or dynamic resizing must be handled.

## Subtopics

### [Thread-Safe Circular Buffer for Brick Allocation](4567-thread-safe-circular-buffer-for-brick-allocation.md)
A thread-safe mechanism for allocating and recycling brick indices on the CPU side, using a circular buffer with atomic head/tail pointers and prime-stepped recycling to prevent cache false sharing. It solves the problem of concurrent brick management in multi-threaded voxel rendering, where multiple threads claim and release bricks simultaneously. It sets itself apart from standard circular buffers by using a prime-stepped recycling stride to mitigate false sharing, a critical optimization for CPU cache performance.

### [Atomic Operation Variants](5912-atomic-operation-variants.md)
Atomic operation variants provide specific computational primitives (addition, minimum, logical operations, etc.) that execute as indivisible units on global memory, extending the basic read-modify-write guarantee of atomic operations. While the parent article [Atomic Operations for Global Counters](3684-atomic-operations-for-global-counters.md) establishes the mechanism for race-free updates, these variants define *what* computation occurs during the update. They are distinguished from sibling techniques like the [Thread-Safe Circular Buffer for Brick Allocation](4567-thread-safe-circular-buffer-for-brick-allocation.md) by operating directly on scalar memory locations rather than managing complex data structures or indices.

### Atomic Compare-and-Swap
Uses atomic compare-and-swap to handle concurrent writes to the same voxel. It takes a pointer to the data, an expected value, and a new value. The write only occurs if the actual data matches the expected value. The return value is checked against the assumed value to determine if the write was successful [S2, unofficial]. This is useful for scenarios where multiple threads might attempt to update the same voxel, ensuring only one write succeeds based on state consistency.

### Atomic Bit Setter
A device function to set a specific bit in a global memory array of unsigned integers using atomic OR operations [S3, unofficial].
**Steps:**
1.  Calculate the integer array index: `size_t int_location = index / 32;`
2.  Calculate the bit position within the integer (Right-to-Left counting): `unsigned int bit_pos = 31 - (index % 32);`
3.  Create mask: `unsigned int mask = 1 << bit_pos;`
4.  Perform atomic OR: `atomicOr(&(voxel_table[int_location]), mask);`
**Pitfall:** Bit ordering counts Right-to-Left (31 down to 0) for a given index, while array indices are Left-to-Right. This is handled by `31 - (index % 32)` [S3, unofficial].
**Comparison:** An alternative `bufferedBitSetter` buffers bits in a register and flushes them with a single atomic OR per 32-bit word to reduce contention. The active `setBit` performs an atomic OR for every bit, which is simpler but potentially slower due to atomic overhead [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per atomic operation, but contention can lead to serialization. Performance degrades as the number of threads contending for the same counter increases.
*   **Memory Complexity:** $O(1)$ for the counter buffer, $O(N)$ for the result array where $N$ is the number of threads or conditional writes.
*   **Performance:** Atomic operations can reduce performance due to serialization and cache coherency traffic [S2, unofficial]. However, they are often sufficient for moderate contention levels.
*   **Strengths:** Simple to implement; guarantees uniqueness and race-free updates; no need for complex synchronization primitives like barriers for simple counting.
*   **Limitations:** High contention can become a bottleneck; order of operations is not guaranteed; not suitable for complex data structure updates without careful design.
*   **When to Use:** When generating unique IDs, counting events, or packing data into a global array from parallel threads.
*   **When Not to Use:** When high contention is expected and performance is critical (consider [Parallel Prefix Sum (Scan) Orchestration](2593-parallel-prefix-sum-scan-orchestration.md) or [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md)); when ordered output is required without post-processing.

## Background
Atomic operations are a fundamental primitive in parallel computing, particularly on GPUs, where they ensure data consistency in global memory. The technique is widely used in graphics programming, as demonstrated by examples in GLSL/OpenGL contexts [S1, unofficial]. It is also employed in voxel simulation and rendering, such as in ray-marched voxel simulations [S2, unofficial] and CUDA voxelizers [S3, unofficial], to manage shared resources like counters and occupancy flags.

## Sources
[S1] 【WIP】GPUのatomic演算とsubgroup演算の練習帳(GLSL/OpenGL), https://zenn.dev/nishiki/articles/3db056030aaad1, fetched 2026-10-30 (unofficial)
[S2] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S3] GitHub - Forceflow/cuda_voxelizer: CUDA Voxelizer to convert polygon ..., https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
