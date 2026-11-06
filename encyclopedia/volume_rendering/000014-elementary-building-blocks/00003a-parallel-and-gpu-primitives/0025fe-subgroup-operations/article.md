# Subgroup Operations

Also called: warp-level intrinsics, wave intrinsics

## Overview
Subgroup operations are hardware-synchronized collective computations performed across a subgroup (NVIDIA warp, AMD wave) of threads [S1, unofficial]. They solve the problem of high-latency global memory synchronization by allowing threads within a subgroup to exchange data and compute aggregates (sums, ballots) using fast on-chip registers [S1, unofficial]. Unlike atomic operations, which serialize access to global memory, subgroup operations are significantly faster and do not involve global memory synchronization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Per-thread data values (e.g., integers, booleans) and the subgroup ID context.
*   **Output:** Aggregated results (sums, counts) or ballot masks, available to all threads in the subgroup or specific threads.

**Data Structures:**
*   **Ballot Mask:** A `uvec4` (128-bit unsigned integer vector) used to store the boolean state of all threads in the subgroup [S1, unofficial]. This size is sufficient for the maximum subgroup size of 64 threads (AMD) [S1, unofficial].

**Required Extensions (GLSL):**
To use these operations, the following extensions must be enabled [S1, unofficial]:
1.  `GL_KHR_shader_subgroup_basic`
2.  `GL_KHR_shader_subgroup_arithmetic`
3.  `GL_KHR_shader_subgroup_ballot`

**Parameters:**
*   **Subgroup Size:** Hardware-dependent. NVIDIA GPUs use 32 threads per subgroup. AMD GPUs use 64 threads per subgroup [S1, unofficial].

**Algorithm Steps:**
1.  **Enable Extensions:** Include the required GLSL extensions in the shader source [S1, unofficial].
2.  **Compute Aggregates:** Use `subgroupAdd(value)` to compute the sum of `value` across all threads in the subgroup. Every thread receives the same total sum [S1, unofficial].
3.  **Compute Exclusive Indices:** Use `subgroupExclusiveAdd(value)` to compute the sum of `value` from threads with lower IDs than the current thread within the subgroup. This returns unique indices per thread, suitable for scatter/gather operations [S1, unofficial].
4.  **Identify Leader:** Use `subgroupElect()` to determine if the current thread has the lowest ID in the subgroup. This is used to gate global memory writes (e.g., `atomicAdd`) to a single thread to avoid redundant writes [S1, unofficial].
5.  **Ballot Collection:** Use `subgroupBallot(predicate)` to collect boolean conditions from all threads in the subgroup into a `uvec4` bitmask [S1, unofficial].
6.  **Count Bits:** Use `subgroupBallotBitCount(mask)` to count the number of set bits (true conditions) in the ballot result [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Example 1: Reducing atomicAdd latency using subgroupAdd
# Layout: local_size_x = 64
# Buffers: count (global atomic counter), array (output)

def main():
    gid = gl_GlobalInvocationID.x
    
    # 1. Compute subgroup-local sum
    # All threads in the subgroup receive the same total sum
    tmp = subgroupAdd(1)
    
    # 2. Only the thread with the lowest ID in the subgroup writes to global memory
    if subgroupElect():
        atomicAdd(count, tmp)

# Example 2: Counting threads satisfying a condition using Ballot
# Layout: local_size_x = 64
# Buffers: counts (array of uint, one per subgroup), array (output)

def main():
    gid = gl_GlobalInvocationID.x
    
    # 1. Define a predicate
    value = gid * gid
    condition = (value % 100 == 0)
    
    # 2. Collect boolean states from all threads into a 128-bit mask
    vote = subgroupBallot(condition)
    
    # 3. Count the number of true conditions in this subgroup
    # Note: gid / 32 assumes 32-thread subgroups (NVIDIA). 
    # For AMD (64 threads), use gid / 64.
    subgroup_index = gid / 32 
    counts[subgroup_index] = subgroupBallotBitCount(vote)

# Example 3: Generating exclusive indices for scattering
# Layout: local_size_x = 64
# Buffers: array (output), count (global atomic counter)

def main():
    gid = gl_GlobalInvocationID.x
    
    # 1. Compute exclusive sum of 1s
    # This gives the index within the subgroup (0 to N-1)
    index = subgroupExclusiveAdd(1)
    
    # 2. Optionally, compute the global offset using atomicAdd on the subgroup sum
    # (Not shown in source, but implied by the need for global offsets)
    
    # 3. Write to global array using the local index
    # Note: Source snippet implies writing to 'array' using 'index'
    # However, without a global offset, this only fills the first N slots.
    # A complete implementation would add a global offset to 'index'.
    array[index] = gid 
```

**Edge Cases and Pitfalls:**
*   **Hardware Dependency:** Results depend on the hardware subgroup size. On NVIDIA GPUs (32-thread subgroups), launching 64 threads results in two separate subgroups. Indices reset to 0-31 for each subgroup, and counts are per-subgroup [S1, unofficial].
*   **Ballot Mask Size:** The `uvec4` return type of `subgroupBallot` is 128 bits, which is sufficient for both NVIDIA (32 threads) and AMD (64 threads) subgroups [S1, unofficial].
*   **Division for Subgroup Index:** When indexing into a results array per subgroup, the divisor must match the hardware subgroup size (32 for NVIDIA, 64 for AMD) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per subgroup for collective operations. Significantly faster than global atomic operations due to the absence of global memory serialization [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per thread (registers). No additional global memory is required for the collective computation itself [S1, unofficial].
*   **Strengths:** High performance for reductions, ballots, and shuffles within a warp/wave. Reduces contention on global memory atomics [S1, unofficial].
*   **Limitations:** Results are confined to the subgroup. Cross-subgroup synchronization requires global memory atomics or barriers. Performance characteristics vary between NVIDIA (32) and AMD (64) hardware [S1, unofficial].
*   **When to use:** When performing reductions, ballots, or shuffles within a warp/wave. When reducing the number of global atomic operations [S1, unofficial].
*   **When not to use:** When data needs to be shared across multiple subgroups or warps without global memory synchronization [S1, unofficial].

## Background
Subgroup operations were introduced in Vulkan and OpenGL as a cross-vendor abstraction for warp-level (NVIDIA) and wave-level (AMD) intrinsics [S1, unofficial]. They provide a standardized way to access hardware-specific collective instructions in GLSL [S1, unofficial].

## Sources
[S1] 【WIP】GPUのatomic演算とsubgroup演算の練習帳(GLSL/OpenGL), https://zenn.dev/nishiki/articles/3db056030aaad1, fetched 2026-10-30 (unofficial)
