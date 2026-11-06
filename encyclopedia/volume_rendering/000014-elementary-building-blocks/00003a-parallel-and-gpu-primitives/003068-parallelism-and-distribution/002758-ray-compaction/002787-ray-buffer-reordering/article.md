# Ray Buffer Reordering


## Overview
Ray Buffer Reordering is a GPU kernel primitive that compacts a list of active rays by moving them from a current buffer to a next buffer, effectively removing terminated rays. It solves the problem of maintaining a dense, contiguous list of alive rays for subsequent volume rendering steps, such as ray marching or feature sampling. It sets itself apart from its siblings by using a double-buffering strategy with atomic counters to assign new indices, ensuring thread-safe reordering without requiring a prior sort or global reduction.

## How it works
**Inputs and Outputs:**
- Input: `alive_indices` array of size $2 \times B$, containing ray indices. Indices $< 0$ denote terminated rays.
- Input: `B`, the batch size or number of rays.
- Input: `c_index`, the index of the current buffer (0 or 1).
- Output: `alive_indices` array of size $2 \times B$, with compacted alive rays in the next buffer.
- Output: `counter`, an atomic counter tracking the number of compacted rays.

**Data Structures:**
- `alive_indices`: A flat array of integers. The layout is double-buffered: indices $[0, B-1]$ correspond to buffer 0, and $[B, 2B-1]$ correspond to buffer 1.
- `counter`: A single integer value, atomically incremented.

**Steps:**
1. Reset the atomic counter to 0.
2. Determine the next buffer index: $n\_index = (c\_index + 1) \pmod 2$.
3. Update the global state to reflect the new current buffer index.
4. For each ray $i$ in $[0, B-1]$:
   a. Read the ray index from the current buffer: $src\_idx = alive\_indices[i \times 2 + c\_index]$.
   b. If $src\_idx \ge 0$ (ray is alive):
      i. Atomically increment the counter to get a unique destination index $dest\_idx$.
      ii. Write the source index to the next buffer at $dest\_idx$: $alive\_indices[dest\_idx \times 2 + n\_index] = src\_idx$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_buffer_reorder(alive_indices, B, c_index, counter):
    # Step 1: Reset counter
    counter[None] = 0
    
    # Step 2: Determine next index
    n_index = (c_index + 1) % 2
    
    # Step 3: Update current index state (assumed to be a global variable or passed by reference)
    # In a real kernel, this might be a separate atomic store or a simple assignment if single-threaded
    # Here we assume the caller manages the state update or it's done via atomicAdd(0, 0) for synchronization
    # For the purpose of this method, we focus on the reordering logic
    
    # Step 4: Iterate and reorder
    for i in range(B):
        # Read from current buffer
        src_idx = alive_indices[i * 2 + c_index]
        
        if src_idx >= 0:
            # Atomically increment counter to get destination index
            dest_idx = atomic_add(counter, 1)
            
            # Write to next buffer
            alive_indices[dest_idx * 2 + n_index] = src_idx
```

**Parameters:**
- $B$: Batch size. Typical values depend on the scene and GPU memory, often in the range of thousands to millions.
- $c\_index$: 0 or 1.

**Edge Cases and Pitfalls:**
- **Precision:** Atomic operations on integers are precise. No floating-point issues.
- **Boundaries:** Ensure $B$ does not exceed half the size of `alive_indices`.
- **Degenerate Input:** If all rays are terminated ($src\_idx < 0$), the counter remains 0, and the next buffer is empty.

## Subtopics

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(B)$, where $B$ is the number of rays. Each ray is processed once.
- Memory: $O(B)$ for the double-buffered `alive_indices` array.

**Measured Performance:**
- Performance depends on the GPU's atomic operation throughput and memory bandwidth. For large $B$, the atomic contention on `counter` can become a bottleneck.

**Scaling:**
- Scales linearly with $B$.

**Strengths and Limitations:**
- **Strengths:** Simple, thread-safe, no sorting required.
- **Limitations:** Atomic contention on the counter can reduce performance for very large $B$. Requires double the memory for the buffer.

**When to Use:**
- Use when compacting rays is a frequent operation and the number of alive rays is significant.

**When Not to Use:**
- Avoid if memory is extremely constrained or if the number of alive rays is very small, as the overhead of atomic operations may outweigh the benefits.

## Background
Ray Buffer Reordering is a technique used in GPU-based volume rendering and ray tracing to manage the state of active rays. It is commonly found in implementations of neural radiance fields (NeRF) and other ray-marching algorithms where efficient management of alive rays is crucial for performance. The specific implementation described here is derived from unofficial sources related to Taichi-based NGP renderers.

## Sources
[S1] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
