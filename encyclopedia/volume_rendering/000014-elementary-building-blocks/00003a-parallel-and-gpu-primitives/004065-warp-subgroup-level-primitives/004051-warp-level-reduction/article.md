# Warp-level Reduction

Also called: Butterfly Reduction, Local Reduction with Warp-Level Primitives, Serialized Reduction, Subgroup reduction

## Overview
Warp-level reduction is a GPU optimization technique that aggregates update requests from threads within the same warp (or subgroup) before issuing global memory operations. It solves the problem of redundant atomic operations, such as `atomicMin` or `atomicAdd`, when multiple threads in a warp target the same memory location (e.g., the same pixel in a framebuffer). It sets itself apart from sibling methods like [Warp Shuffling](4949-warp-shuffling.md) by focusing specifically on the reduction of contention and overhead for atomic primitives, and from [Subgroup Operations](3683-subgroup-operations.md) by providing concrete algorithmic patterns (serialized, butterfly, pairwise) for deduplication.

## How it works
**Inputs:**
- `pixelID`: The target pixel index for each thread.
- `depth`: The depth value (e.g., z-buffer value) associated with the point being rendered.
- `point`: The data (color, normal, etc.) to be written.
- `framebuffer`: The global memory buffer storing the current minimum depth and associated point data.

**Outputs:**
- Updated `framebuffer` entries for pixels that received a valid, non-redundant update.

**Data Structures:**
- No specific complex data structures are required beyond standard registers and shared memory (if used for intermediate storage, though warp primitives often operate directly in registers).
- The core state is held in thread-local registers: `minDepth`, `idXOr`, and the original `depth`/`pixelID`.

**Prerequisites:**
- The GPU architecture must support warp/subgroup primitives (e.g., NVIDIA `__shfl_sync`, AMD `__builtin_amdgcn_ds_bpermute`, or OpenGL `subgroup*` functions).
- Threads in a warp must be active or explicitly masked to handle inactive threads correctly.

**Steps:**

1.  **Full Subgroup Deduplication (Fast Path for Dense Pixels):**
    Check if all threads in the subgroup target the same pixel. If true, compute the minimum depth across the entire subgroup. Only the thread holding the minimum depth performs the global atomic write.
    $$ \text{if } (\text{subgroupAllEqual}(\text{pixelID})) \quad \text{minDepth} = \text{subgroupMin}(\text{depth}) $$
    $$ \text{if } (\text{minDepth} == \text{depth}) \quad \text{atomicMin}(\text{framebuffer}[\text{pixelID}], \text{point}) $$

2.  **Fine-Granular Pairwise Reduction (Local Optimization):**
    For cases where not all threads target the same pixel, perform a reduction on neighboring thread pairs (stride 2). This reduces atomic traffic even for close-up viewpoints depending on vertex order.
    $$ \text{idXOr} = \text{subgroupClusteredXor}(\text{pixelID}, 2) $$
    $$ \text{minDepth} = \text{subgroupClusteredMin}(\text{depth}, 2) $$
    $$ \text{if } (\text{idXOr} \neq 0 \lor \text{minDepth} == \text{depth}) \quad \text{atomicMin}(\text{framebuffer}[\text{pixelID}], \text{point}) $$

3.  **Serialized Reduction (For Multiple Parameters):**
    Identify threads updating the same parameter (memory location). One designated thread iterates through the gradients/values from all other threads in the group, accumulating the sum (or min). Finally, perform a single regular atomic operation with the accumulated result. This is inefficient due to serialization but allows parallelization across multiple distinct parameters.

4.  **Butterfly Reduction (Tree-Based Summation):**
    Check if all threads in the warp update the same primitive. If so, use a reduction tree (butterfly pattern) to sum gradients. Inactive threads must contribute zero values to maintain correctness. This introduces redundant computation for inactive threads but is efficient when one parameter is updated by most active threads.

**Pseudocode:**
The following pseudocode combines the fast-path full subgroup check and the pairwise local reduction. It assumes the availability of subgroup primitives.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: pixelID, depth, point (per thread)
# Global: framebuffer

# Step 1: Check if all threads in the subgroup target the same pixel
if subgroup_all_equal(pixelID):
    # Compute minimum depth across the entire subgroup
    min_depth = subgroup_min(depth)
    
    # Only the thread with the global minimum depth writes to memory
    if depth == min_depth:
        atomic_min(framebuffer[pixelID], point)
else:
    # Step 2: Fine-granular pairwise reduction
    # Compute XOR of pixel IDs for neighboring threads (stride 2)
    # If idXOr is 0, both threads target the same pixel
    id_xor = subgroup_clustered_xor(pixelID, 2)
    
    # Compute minimum depth for the pair
    min_depth_pair = subgroup_clustered_min(depth, 2)
    
    # Update if:
    # 1. They target different pixels (id_xor != 0), OR
    # 2. They target the same pixel, and this thread has the lower depth
    if id_xor != 0 or depth == min_depth_pair:
        atomic_min(framebuffer[pixelID], point)
```

**Parameters:**
- `stride`: Typically 2 for pairwise reduction. Larger strides can be used for multi-level reduction but increase register pressure and complexity. Not given in the sources for general cases, but 2 is specified for the pairwise method.

**Edge Cases and Pitfalls:**
- **Inactive Threads:** In Butterfly Reduction, inactive threads must explicitly contribute zero values (or identity elements for min/max) to avoid corrupting the reduction result [S2].
- **Control Flow Divergence:** Butterfly reduction requires no control flow divergence within the warp; all threads must be active or masked appropriately [S2].
- **Precision:** Standard integer or float comparisons apply. Ensure `atomicMin` semantics match the data type (e.g., unsigned int for depth).

## Subtopics

### [Adaptive Atomic ReduCtion (ARC)](12357-adaptive-atomic-reduction-arc.md)
ARC is a primitive that accelerates atomic operations in differentiable rendering by leveraging intra-warp locality (threads in a warp update the same memory location) and varying atomic traffic (subset of threads active). It reduces atomic requests sent to L2 by performing warp-level reduction in registers/SM cores and dynamically distributing remaining atomic computation between SM cores and L2 atomic units.

## Operating characteristics
- **Time Complexity:** Reduces the number of global atomic operations from $O(W)$ to $O(1)$ or $O(W/2)$ per pixel, where $W$ is the warp size (typically 32). The reduction itself is $O(\log W)$ for butterfly or $O(W)$ for serialized, but these are register-bound and extremely fast compared to global memory latency.
- **Memory Complexity:** $O(1)$ additional register space per thread for intermediate values (`minDepth`, `idXOr`).
- **Measured Performance:** Significant speedup in dense point clouds and zoomed-out views where many points project to the same pixel [S1]. Pairwise reduction helps even in close-up viewpoints depending on vertex memory order [S1].
- **Scaling:** Effectiveness scales with the density of the scene and the degree of pixel overlap. In sparse scenes with little overlap, the overhead of subgroup primitives may outweigh the benefits of reduced atomic calls.
- **Strengths:** Drastically reduces global memory contention and atomic operation overhead. Simple to implement with modern subgroup primitives.
- **Limitations:** Serialized reduction is inefficient due to its serial nature [S2]. Butterfly reduction requires all threads to be active or zero-initialized, introducing redundant computation [S2].
- **When to use:** When rendering dense point clouds, volume slices, or any scenario where multiple threads in a warp target the same memory location.
- **When not to use:** When threads in a warp target entirely different pixels with no overlap, or when the overhead of subgroup primitives exceeds the cost of atomic operations (rare on modern GPUs).

## Background
Warp-level reduction was introduced to address the bottleneck of atomic operations in GPU rendering, particularly for point clouds and differentiable rendering. The concept leverages the SIMT (Single Instruction, Multiple Threads) nature of NVIDIA GPUs and similar architectures in AMD and OpenGL. The specific techniques of full subgroup deduplication and pairwise reduction were detailed in "Rendering Point Clouds with Compute Shaders and Vertex Order Optimization" [S1]. The broader context of warp-level adaptive atomic reduction, including serialized and butterfly approaches, was further explored in "ARC: Warp-level Adaptive Atomic Reduction in GPUs to Accelerate Differentiable Rendering" [S2].

## Sources
[S1] Rendering Point Clouds with Compute Shaders and Vertex Order Optimization, https://doi.org/10.1111/cgf.14345, fetched 2026-10-26
[S2] ARC: Warp-level Adaptive Atomic Reduction in GPUs to Accelerate Differentiable Rendering, https://doi.org/10.1145/3669940.3707238, fetched 2026-11-03
