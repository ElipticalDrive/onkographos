# Parallel Vector Multiplication

Also called: CUDA Vector Multiplication, OpenCL Parallel Vector Multiplication, Vector Add/Multiply Kernel

## Overview
Parallel Vector Multiplication is a fundamental GPU primitive where each thread independently multiplies corresponding elements from two input arrays to produce an output array. It solves the problem of element-wise vector scaling or Hadamard product computation at massive scale by mapping one thread per data element. It is distinct from sibling primitives like [Parallel Prefix Sum](2593-parallel-prefix-sum-scan-orchestration.md) (which requires inter-thread dependency) or [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md) (which requires synchronization), as it involves no communication between threads.

## How it works
**Inputs and Outputs:**
- Input: Two arrays of floating-point values, $A$ and $B$, of size $N$.
- Output: One array $C$ of size $N$, where $C[i] = A[i] \times B[i]$.

**Data Structures:**
- Arrays are stored in global memory.
- Data types: `float` (32-bit) is standard. Vector types `float2`, `float3`, `float4`, `float8`, and `float16` are supported to increase arithmetic throughput per instruction [S2, unofficial].
- In specific applications like compressive rendering, the vectors may represent sampling matrices where $S^T S x$ operations involve inserting zeros for missing pixels [S1, unofficial].

**Steps:**
1. The GPU kernel is launched with a grid of threads sized to cover the entire vector length $N$.
2. Each thread calculates its unique global index $id$.
3. The thread reads $A[id]$ and $B[id]$ from global memory.
4. The thread computes the product $P = A[id] \times B[id]$.
5. The thread writes $P$ to $C[id]$ in global memory.

**Pseudocode (OpenCL/CUDA style):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel entry point
def vector_multiply_kernel(global_id, a, b, c):
    # 1. Retrieve global unique ID
    id = global_id
    
    # 2. Bounds check (G: Standard guard to prevent out-of-bounds access)
    if id < length_of_arrays:
        # 3. Load inputs
        val_a = a[id]
        val_b = b[id]
        
        # 4. Multiply
        result = val_a * val_b
        
        # 5. Store output
        c[id] = result
```

**Parameters:**
- **Vector Width:** Using `float4` or `float8` can improve performance by coalescing memory accesses and utilizing SIMD units effectively [S2, unofficial].
- **Memory Coalescing:** Threads in a warp (32 threads in CUDA) should access contiguous memory locations to maximize bandwidth [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point multiplication is associative but not distributive over addition in finite precision; however, for element-wise multiplication, this is rarely an issue unless combined with other ops.
- **Boundaries:** If $N$ is not a multiple of the grid size, a bounds check is required [G].
- **Missing Pixels:** In compressive rendering contexts, "multiplication" may involve sparse matrices where missing pixels are treated as zero [S1, unofficial].

## Subtopics

### Tiled Parallel Compositing
Uses vector multiplication for opacity/color blending within tiles.

### Parallel Direct Compositing
Relies on element-wise multiplication for ray-marching contributions.

### Subgroup Operations
Can optimize vector multiplication within a warp using shuffle instructions if further reduction is needed.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of elements. The operation is memory-bound rather than compute-bound for simple multiplication, as the arithmetic intensity is low (1 mul, 2 loads, 1 store).
- **Memory Complexity:** $O(N)$ for input and output arrays.
- **Performance:** Massive calculations (e.g., $S^T S x$ in compressive rendering) can be done in milliseconds [S1, unofficial]. Modern GPUs with 16 to 480+ stream processors scale linearly with hardware capability [S2, unofficial].
- **Strengths:** Extremely high throughput for independent element-wise operations; minimal synchronization overhead.
- **Limitations:** Does not handle dependencies between elements; memory bandwidth can become a bottleneck if vector width is not optimized.

## Background
The method originates from early GPU computing frameworks like CUDA and OpenCL, leveraging the SIMT (Single Instruction, Multiple Threads) architecture. It is widely used in volume rendering for tasks such as compressive rendering, where large sampling matrices are applied [S1, unofficial], and in standard raycasting for independent pixel processing [S2, unofficial].

## Sources
[S1] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S2] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
