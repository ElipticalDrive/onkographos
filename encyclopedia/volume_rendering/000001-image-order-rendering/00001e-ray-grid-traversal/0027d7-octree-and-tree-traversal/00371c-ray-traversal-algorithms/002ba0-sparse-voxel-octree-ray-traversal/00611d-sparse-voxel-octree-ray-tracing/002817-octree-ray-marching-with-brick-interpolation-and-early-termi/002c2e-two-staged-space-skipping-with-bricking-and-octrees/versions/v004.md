# Two-staged Space-Skipping with Bricking and Octrees

Also called: Bricking with octree refinement, Two-stage space-skipping

## Overview
This method balances GPU rendering bottlenecks by dividing space-skipping into two distinct stages: coarse division using bricking to address bus and texture cache limitations, and fine division using octrees within bricks to address the rasterization bottleneck [S1, unofficial]. It solves the problem of inefficient memory access and load balancing in large volume rendering by allowing runtime transfer function changes without texture regeneration, unlike Tong et al.'s bricking technique which requires regenerating textures for every transfer function change [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar voxel volume dataset, potentially exceeding available texture memory [S1, unofficial].
*   **Output:** A rendered image with optical properties (RGB$\alpha$) mapped via a transfer function [S1, unofficial].

**Data Structures:**
1.  **Bricks:** The volume is divided into 3D chunks called bricks [S1, unofficial].
    *   **Layout:** Each brick contains unmodified scalar voxel values [S1, unofficial].
    *   **Size:** Brick dimension $b$ (in voxels) must be $\le$ the size of available texture memory [S1, unofficial]. If volume dimensions do not divide evenly by $b$, additional partially empty bricks or smaller rest-bricks are added [S1, unofficial].
    *   **Overlap:** Adjacent bricks overlap by $n$ voxels in every direction to support interpolation [S1, unofficial].
        *   Nearest neighbor: $n=0$ [S1, unofficial].
        *   Trilinear interpolation: $n=1$ [S1, unofficial].
        *   Pre-integrated rendering or on-the-fly gradient calculation: $n \ge 2$ (increased by another voxel) [S1, unofficial].
    *   **Memory Overhead:** Approximately $(3 \cdot n / b) \cdot 100\%$ [S1, unofficial].
2.  **Octrees:** Stored within each brick [S1, unofficial].
    *   **Layout:** Sparse representation of the brick's interior, marking empty regions [S1, unofficial].
    *   **Purpose:** Enables skipping empty regions at a finer granularity than the brick level [S1, unofficial].
3.  **Transfer Function:** Applied in the fragment shader [S1, unofficial].
    *   **Mechanism:** RGB$\alpha$ values are looked up post-interpolatively based on the current scalar value, allowing runtime changes without regenerating textures [S1, unofficial].

**Numbered Steps:**
1.  **Stage 1: Coarse Division (Bricking)**
    *   Divide the volume dataset into bricks of size $b \times b \times b$ [S1, unofficial].
    *   Load bricks sequentially from main memory into texture memory if the volume exceeds available texture memory [S1, unofficial].
    *   Apply early ray termination to each brick to alleviate fragment shader load, particularly beneficial for highly-filled datasets [S1, unofficial].
2.  **Stage 2: Fine Division (Octree Traversal)**
    *   Employ an octree within each loaded brick [S1, unofficial].
    *   Use the octree to skip empty regions at a finer granularity than the brick level [S1, unofficial].
    *   Perform ray casting within the non-empty octree nodes [S1, unofficial].
3.  **Rendering and Transfer Function Application**
    *   Store unmodified scalar voxel values in the brick textures [S1, unofficial].
    *   In the fragment shader, lookup RGB$\alpha$ values post-interpolatively based on the scalar value and the current transfer function [S1, unofficial].

**Formulas:**
*   Memory Overhead: $\text{Overhead} \approx \frac{3 \cdot n}{b} \cdot 100\%$ [S1, unofficial].
    *   $b$: Brick dimension in voxels [S1, unofficial].
    *   $n$: Overlap in voxels in every direction [S1, unofficial].

**Parameters:**
*   **Brick Size ($b$):** Must be equal to or smaller than the size of the available texture memory [S1, unofficial].
*   **Overlap ($n$):** Depends on the interpolation kernel. For trilinear, $n=1$. For pre-integration/gradient calculation, $n \ge 2$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Triangle Throughput Bottleneck:** Space-skipping techniques may increase the amount of triangles considerably [S1, unofficial]. If the triangle count becomes too high, it becomes a limiting factor for frame rate, potentially overloading the triangle throughput bottleneck when the rasterization bottleneck is lifted [S1, unofficial].
*   **Bottleneck Balance:** The two stages (bricking and octree) must be balanced because lifting one bottleneck may overload another (e.g., rasterization vs. triangle throughput) [S1, unofficial].
*   **Texture Memory Limits:** If the amount of data in textures exceeds available texture memory, textures are swapped between main memory and texture memory, which can impact performance [S1, unofficial].
*   **Power of 2 Constraints:** Many graphics cards require 3D texture sizes to be a power of 2 in every direction [S1, unofficial]. Non-divisible dimensions require additional layers or rest-bricks [S1, unofficial].

## Subtopics

### [Quaternary Tree Volume Traversal](13509-quaternary-tree-volume-traversal.md)
Quaternary Tree Volume Traversal is a hybrid rendering technique that offloads coarse spatial partitioning to the CPU and fine-grained ray-object intersection to the GPU. It solves the problem of inefficient memory access and load balancing in volume rendering by dividing the rendering pipeline into two distinct stages: geometric culling on the CPU and ray casting on the GPU. Unlike pure GPU ray tracers, this method leverages the CPU's flexibility to build a quaternary tree for early space-skipping, reducing the number of rays processed by the GPU.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Memory overhead for bricks is approximately $(3 \cdot n / b) \cdot 100\%$ [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:**
    *   Addresses bus and texture cache bottlenecks via bricking [S1, unofficial].
    *   Addresses rasterization bottleneck via fine-grained octree skipping [S1, unofficial].
    *   Allows runtime transfer function changes without texture regeneration by storing scalar values and using fragment shader lookup [S1, unofficial].
    *   Early ray termination alleviates fragment shader load [S1, unofficial].
*   **Limitations:**
    *   Risk of triangle throughput bottleneck if space-skipping increases triangle count too much [S1, unofficial].
    *   Requires balancing between coarse and fine division stages to avoid overloading other bottlenecks [S1, unofficial].
    *   Texture compression methods (S3TC, FXT1, etc.) are unsuitable for 12- or 16-bit scalar medical data, necessitating this approach for large sparse datasets [S1, unofficial].
*   **When to Use:** Large volume datasets exceeding texture memory, especially those with sparse regions (e.g., vessels occupying 1% to 8% of the volume) where space-skipping provides significant benefit [S1, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
*   **Origin:** The method was introduced in 2006 in Plzen, Czech Republic [S1, unofficial].
*   **History:** It builds upon earlier bricking techniques by Eckhardt [Eck98] and Wang et al. [WWE04], and specifically addresses limitations in Tong et al.'s bricking technique [TWTT99] which required texture regeneration for transfer function changes [S1, unofficial]. It also responds to the unsuitability of standard texture compression for high-bit scalar data [S1, unofficial].
*   **Usage:** Used in GPU volume rendering to handle large datasets and optimize performance by addressing specific graphics pipeline bottlenecks (bus, texture cache, rasterization, triangle throughput) [S1, unofficial].

## Sources
[S1] Optimizing GPU Volume Rendering - Danny Ruijters, https://dannyruijters.nl/docs/OptimizingGPUVolumeRendering_final.pdf, fetched 2026-10-26 (unofficial)
