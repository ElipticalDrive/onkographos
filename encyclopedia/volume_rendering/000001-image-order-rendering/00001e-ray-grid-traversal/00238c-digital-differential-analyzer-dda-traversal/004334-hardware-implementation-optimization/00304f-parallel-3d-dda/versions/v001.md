# Parallel 3D-DDA
Also called: SIMD Ray Marching, 并行化改造方案
## Overview
Parallel 3D-DDA is a GPU-accelerated implementation of the Digital Differential Analyzer (DDA) ray traversal algorithm, designed to process large sets of rays simultaneously using Single Instruction, Multiple Data (SIMD) parallelism [S1, unofficial]. It solves the performance bottleneck of sequential ray stepping in volume rendering by distributing rays across GPU warps, leveraging shared memory to mitigate global memory latency [S1, unofficial]. Unlike its sequential parent, it prioritizes throughput over single-ray precision control, making it suitable for real-time applications involving millions of rays [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ rays, each defined by origin $\mathbf{o}_i$ and direction $\mathbf{d}_i$; a 3D voxel grid (volume texture) of dimensions $W \times H \times D$; and a step size $\Delta s$ (typically 0.5 voxel units [S1, unofficial]).
*   **Output:** Transmittance and color accumulation for each ray, or a boolean mask indicating traversal completion.

**Data Structures:**
*   **Ray Buffer:** A structure of arrays (SoA) layout for GPU coalesced access: `float3 origins[N]`, `float3 directions[N]`, `float3 t_maxs[N]`, `float3 current_t[N]`.
*   **Shared Memory Cache:** A tile-sized block of voxel data (e.g., $16 \times 16 \times 16$ voxels) cached per warp to reduce global memory bandwidth pressure [S1, unofficial].
*   **Voxel Grid:** Linearized 3D array or 3D texture object.

**Algorithm Steps:**
1.  **Initialization:** Divide the ray set into warps (e.g., 32 threads per warp). Each thread $t$ in a warp is assigned one ray index $i$. Initialize `current_t[i]` to 0.0.
2.  **Warp-Level Loop:** While any thread in the warp has `current_t[i] < t_maxs[i]`:
    a.  **Voxel Coordinate Calculation:** Compute the 3D grid coordinates $(x, y, z)$ for the current ray position:
        $$ x = \lfloor o_x + d_x \cdot current\_t[i] \rfloor $$
        $$ y = \lfloor o_y + d_y \cdot current\_t[i] \rfloor $$
        $$ z = \lfloor o_z + d_z \cdot current\_t[i] \rfloor $$
        [G] Use floor function to map continuous ray position to discrete voxel index.
    b.  **Boundary Check:** If $(x, y, z)$ is outside the volume bounds $[0, W-1] \times [0, H-1] \times [0, D-1]$, mark the ray as complete.
    c.  **Shared Memory Load:** If the voxel $(x, y, z)$ is not in the warp's shared memory cache, load the relevant tile from global memory into shared memory [S1, unofficial].
    d.  **Sample and Accumulate:** Read the density/value $v$ from the shared memory cache at $(x, y, z)$. Apply the transfer function to compute color $C$ and opacity $\alpha$. Update ray state:
        $$ T_{new} = T_{old} \cdot (1 - \alpha) $$
        $$ C_{new} = C_{old} + C \cdot T_{new} $$
    e.  **Step Forward:** Increment `current_t[i]` by $\Delta s$.
3.  **Termination:** When all rays in the warp are complete or `current_t[i]` exceeds `t_maxs[i]`, the warp exits the loop.

**Parameters:**
*   **Step Size ($\Delta s$):** Typically 0.5 voxel units to balance speed and aliasing [S1, unofficial].
*   **Warp Size:** 32 threads (NVIDIA standard) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision Loss:** Floating-point errors in coordinate calculation can cause jitter or missed voxels. Use mixed precision (see Subtopics) [S1, unofficial].
*   **Divergence:** Rays with different entry/exit points cause warp divergence. Masking inactive rays is required [G].
*   **Empty Voxels:** Traversing empty space wastes cycles. Use occupancy masks if available [S1, unofficial].

## Subtopics

### Mixed Precision Optimization
Uses 32-bit floating-point numbers for initial ray stepping to ensure precision near the entry point, then switches to 16-bit floating-point numbers as the ray penetrates deeper into the medium to accelerate calculation [S1, unofficial]. This strategy increased computational throughput by 40% in a game engine practice while maintaining visual effects [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$, where $N$ is the number of rays and $L$ is the average number of steps per ray. Parallelization reduces wall-clock time by a factor proportional to the number of active warps.
*   **Memory Complexity:** $O(N)$ for ray state storage; $O(V_{tile})$ for shared memory cache per warp, where $V_{tile}$ is the tile size.
*   **Performance:** On an NVIDIA A100 GPU, the parallelized 3D-DDA implementation can process 28 million rays per second [S1, unofficial].
*   **Strengths:** High throughput for large ray counts; efficient use of GPU parallelism; reduced global memory latency via shared memory caching [S1, unofficial].
*   **Limitations:** Warp divergence due to varying ray lengths; precision issues at shallow depths without mixed precision; less efficient for sparse data without occupancy masks [S1, unofficial].
*   **When to Use:** Real-time volume rendering with millions of rays (e.g., clouds, smoke) where throughput is critical [S1, unofficial].
*   **When Not to Use:** Scenes with highly sparse data requiring complex traversal logic (use SVO or occupancy masks); single-ray high-precision requirements [S1, unofficial].

## Background
Parallel 3D-DDA evolved from the traditional DDA traversal algorithm to leverage modern GPU architectures. It was developed to address the scalability limits of sequential ray marching in volume rendering. The approach is part of a broader trend in real-time graphics to optimize voxel-based rendering through SIMD parallelism and memory hierarchy optimization [S1, unofficial]. It is widely used in medical CT visualization, meteorological cloud rendering, and game engines [S1, unofficial].

## Sources
[S1] 体渲染中的体素遍历技术:从理论到实践, https://developer.baidu.com/article/detail.html?id=6975905, fetched 2026-10-25 (unofficial)
