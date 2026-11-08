# Subgroup Operations

Also called: warp-level intrinsics, wave intrinsics

## Overview
Subgroup operations are hardware-synchronized collective computations performed across a subgroup (NVIDIA warp, AMD wave) of threads [S1, unofficial]. They solve the problem of high-latency global memory synchronization by allowing threads within a subgroup to exchange data and compute aggregates (sums, ballots) using fast on-chip registers [S1, unofficial]. Unlike atomic operations, which serialize access to global memory, subgroup operations are significantly faster and do not involve global memory synchronization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Per-thread data values (e.g., integers, booleans) and the subgroup context [S1, unofficial].
*   **Output:** Aggregated results (sums, counts) or ballot masks, available to all threads in the subgroup or specific threads [S1, unofficial].

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
# Helper functions implementing hardware intrinsics explicitly
def subgroup_add(value, subgroup_size):
    """Returns the sum of `value` across all threads in the subgroup."""
    return sum(value for _ in range(subgroup_size))

def subgroup_elect(thread_id_in_subgroup):
    """Returns True if the current thread has the lowest ID in the subgroup."""
    return thread_id_in_subgroup == 0

def subgroup_ballot(predicate, subgroup_size):
    """Collects boolean conditions from all threads into a bitmask."""
    mask = 0
    for i in range(subgroup_size):
        if predicate:
            mask |= (1 << i)
    return mask

def subgroup_ballot_bit_count(mask):
    """Counts the number of set bits in the ballot mask."""
    return bin(mask).count('1')

def subgroup_exclusive_add(value, thread_id_in_subgroup):
    """Returns the sum of `value` from threads with lower IDs in the subgroup."""
    return sum(value for _ in range(thread_id_in_subgroup))

# Example 1: Reducing atomicAdd latency using subgroupAdd
# Layout: local_size_x = 64
# Buffers: count (global atomic counter)
def reduce_atomic_latency(count_buffer, subgroup_size, global_id):
    thread_in_subgroup = global_id % subgroup_size
    # subgroupAdd(1): compute subgroup-local sum
    local_sum = subgroup_add(1, subgroup_size)
    # subgroupElect(): gate global write to leader thread only
    if subgroup_elect(thread_in_subgroup):
        count_buffer += local_sum

# Example 2: Counting threads satisfying a condition using Ballot
# Layout: local_size_x = 64
# Buffers: counts (array of uint, one per subgroup)
def ballot_count(counts_buffer, subgroup_size, global_id):
    thread_in_subgroup = global_id % subgroup_size
    value = global_id * global_id
    condition = (value % 100 == 0)
    # subgroupBallot(condition): collect predicates into 128-bit mask
    vote = subgroup_ballot(condition, subgroup_size)
    # subgroupBallotBitCount(vote): count true conditions
    count = subgroup_ballot_bit_count(vote)
    # Index into counts array per subgroup (divisor must match hardware size)
    subgroup_index = global_id // subgroup_size
    counts_buffer[subgroup_index] = count

# Example 3: Generating exclusive indices for scattering
# Layout: local_size_x = 64
# Buffers: array (output)
def exclusive_indices(array_buffer, subgroup_size, global_id):
    thread_in_subgroup = global_id % subgroup_size
    # subgroupExclusiveAdd(1): compute local offset within subgroup
    index = subgroup_exclusive_add(1, thread_in_subgroup)
    # array[gid] = index
    array_buffer[global_id] = index
```

**Edge Cases and Pitfalls:**
*   **Hardware Dependency:** Results depend on the hardware subgroup size. On NVIDIA GPUs (32-thread subgroups), launching 64 threads results in two separate subgroups. Indices reset to 0-31 for each subgroup, and counts are per-subgroup [S1, unofficial].
*   **Ballot Mask Size:** The `uvec4` return type of `subgroupBallot` is 128 bits, which is sufficient for both NVIDIA (32 threads) and AMD (64 threads) subgroups [S1, unofficial].
*   **Division for Subgroup Index:** When indexing into a results array per subgroup, the divisor must match the hardware subgroup size (32 for NVIDIA, 64 for AMD) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Subgroup operations are significantly faster than atomic operations because they do not involve global memory synchronization [S1, unofficial].
*   **Limitations:** Results are confined to the subgroup. Performance characteristics vary between NVIDIA (32 threads) and AMD (64 threads) hardware [S1, unofficial].
*   **When to use:** Use subgroups to reduce the number of global atomic operations, or when performing reductions and ballots within a warp/wave [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
Subgroup operations were introduced in Vulkan and OpenGL as a cross-vendor abstraction for warp-level (NVIDIA) and wave-level (AMD) intrinsics [S1, unofficial]. They provide a standardized way to access hardware-specific collective instructions in GLSL [S1, unofficial].

## Sources
[S1] 【WIP】GPUのatomic演算とsubgroup演算の練習帳(GLSL/OpenGL), https://zenn.dev/nishiki/articles/3db056030aaad1, fetched 2026-10-30 (unofficial)
