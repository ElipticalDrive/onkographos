# CPU-Based Splat Sorting

Also called: Splat Sort, WebWorker Splat Sort

## Overview
CPU-Based Splat Sorting is a method for reordering 3D Gaussian splats by depth on the central processing unit to enable correct front-to-back alpha blending for transparency [S2, unofficial]. It solves the problem of order-independent transparency in volume rendering by decoupling the sorting phase from the GPU rendering pipeline, allowing the GPU to focus exclusively on rasterization and blending [S1, unofficial]. It differs from GPU-based sorting methods (like bitonic sort) by avoiding the overhead of complex GPU sorting algorithms and instead relying on CPU-side sorting algorithms to produce a sorted order buffer [S1, unofficial][S2, unofficial].

## How it works
The method computes a depth metric for each splat, sorts the splats based on this metric, and outputs an index buffer for the GPU to consume.

**Inputs and Outputs:**
*   **Inputs:**
    *   `viewProj`: A $4 \times 4$ view-projection matrix ($glm::mat4$ or equivalent) [S2, unofficial].
    *   `positions`: A pointer to an array of splat positions, interleaved as $(x, y, z)$ floats [S2, unofficial].
    *   `vertexCount`: The number of splats ($uint32_t$) [S2, unofficial].
*   **Outputs:**
    *   `depthIndex`: A vector of $uint32_t$ indices, mapping the sorted order to the original splat positions [S2, unofficial].

**Data Structures:**
1.  **Splat Pair:** A structure or tuple containing:
    *   `depth`: A floating-point value representing the splat's depth in view space.
    *   `original_index`: The index of the splat in the original `positions` array.
2.  **Index Vector:** A dynamic array (e.g., `std::vector<uint32_t>`) storing the `original_index` of splats in sorted order.

**Steps:**
1.  **Compute Depth:** For each splat $i$ from $0$ to $vertexCount - 1$:
    *   Retrieve the 3D position $\mathbf{p}_i = (x_i, y_i, z_i)$ from `positions`.
    *   Transform the position into clip space using the view-projection matrix:
        $$ \mathbf{v}_{clip} = \text{viewProj} \cdot \begin{bmatrix} x_i \\ y_i \\ z_i \\ 1 \end{bmatrix} $$
    *   Extract the depth value $d_i$ by perspective division:
        $$ d_i = \frac{v_{clip,z}}{v_{clip,w}} $$
        [S2, unofficial]
    *   Store the pair $(d_i, i)$ in a temporary list.
2.  **Sort:** Sort the list of pairs in ascending order of $d_i$ (smallest depth first, i.e., closest to camera) using a comparison-based sort (e.g., quicksort, mergesort) or radix sort [S1, unofficial][S2, unofficial].
    *   Comparator: $a < b$ if $a.\text{depth} < b.\text{depth}$ [S2, unofficial].
3.  **Extract Indices:** Iterate through the sorted list and extract the `original_index` from each pair into the output `depthIndex` vector.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_splat_sort(view_proj_matrix, positions, vertex_count):
    """
    Sorts splats by depth on the CPU.
    
    Args:
        view_proj_matrix: 4x4 matrix (list of lists or array)
        positions: List of 3-tuples (x, y, z)
        vertex_count: Integer number of splats
        
    Returns:
        depth_index: List of integers representing sorted indices
    """
    # Step 1: Compute depth for each splat
    splat_data = []
    for i in range(vertex_count):
        x, y, z = positions[i]
        
        # Transform position to clip space
        # Assuming row-major matrix multiplication: v_clip = M * v
        # v = [x, y, z, 1]
        wx = view_proj_matrix[0][0]*x + view_proj_matrix[1][0]*y + view_proj_matrix[2][0]*z + view_proj_matrix[3][0]
        wy = view_proj_matrix[0][1]*x + view_proj_matrix[1][1]*y + view_proj_matrix[2][1]*z + view_proj_matrix[3][1]
        wz = view_proj_matrix[0][2]*x + view_proj_matrix[1][2]*y + view_proj_matrix[2][2]*z + view_proj_matrix[3][2]
        ww = view_proj_matrix[0][3]*x + view_proj_matrix[1][3]*y + view_proj_matrix[2][3]*z + view_proj_matrix[3][3]
        
        # Perspective divide to get depth
        if ww == 0:
            depth = 0.0 # Guard against division by zero [G]
        else:
            depth = wz / ww
            
        splat_data.append((depth, i))
        
    # Step 2: Sort by depth (ascending)
    # Using a stable sort is not strictly required for correctness of depth,
    # but ensures deterministic behavior for equal depths.
    splat_data.sort(key=lambda item: item[0])
    
    # Step 3: Extract indices
    depth_index = [item[1] for item in splat_data]
    
    return depth_index
```

**Parameters:**
*   **Sorting Algorithm:** Typically $O(n \log n)$ comparison sorts (quicksort/mergesort) or radix sort [S1, unofficial].
*   **Precision:** Floating-point depth calculation is standard. Integer-based sorting may be used for performance but can fail in scenes with large dimensions or high density [S3, unofficial].
*   **Distance Map Precision:** A parameter `splatSortDistanceMapPrecision` can adjust the precision of the distance map used in some implementations to alleviate artifacts, at the cost of performance [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Rapid Camera Movement:** Rapid changes in camera orientation (e.g., jumping between opposite sides) cause momentary artifacts because the Z-order does not update fast enough relative to the render loop [S1, unofficial][S3, unofficial].
*   **Large Scenes:** Scenes with very large dimensions may cause 'Index out of bounds' errors during sorting. Changing precision or integer-based sort settings usually does not help [S3, unofficial].
*   **Performance Bottleneck:** Sorting is significantly slower than rendering. For ~1M splats, sorting takes ~150ms on a typical CPU, resulting in a sorting rate of ~4fps, while the main render runs at 60fps [S1, unofficial].

## Subtopics

### [Splat Sorting Key Calculation](2106-splat-sorting-key-calculation.md)
This method calculates a scalar key for each 3D Gaussian splat to determine the rendering order, typically for front-to-back alpha blending. It combines the splat's projected size and opacity to prioritize "important" splats (large and opaque) to be drawn first, ensuring correct accumulation of color and alpha. It differs from depth-based sorting by ignoring camera distance in favor of visual contribution, which is more robust for Gaussian splats where size varies significantly per splat [2106-splat-sorting-key-calculation.md].

### [CPU Fallback Sort Path](5586-cpu-fallback-sort-path.md)
The CPU Fallback Sort Path is a rendering strategy for volume splatting that executes the sorting of splats on the CPU rather than the GPU. It is designed for environments where GPU compute shaders are unavailable, disabled, or insufficient, allowing the GPU to focus exclusively on rasterization and blending. It differs from GPU-based sorting methods (like bitonic sort) by avoiding the overhead of complex GPU sorting algorithms and instead relying on CPU-side sorting algorithms (such as radix sort or $O(n \log n)$ sorts) to produce a sorted order buffer.

### WASM/SIMD CPU Splat Sorting
This variant implements CPU-based splat sorting using WebAssembly (WASM) with SIMD instructions for improved performance [S3, unofficial]. It sorts splats by distance from the camera for correct alpha blending. It includes parameters `integerBasedSort` (to force floating-point sorting if integer-based fails for large scenes) and `splatSortDistanceMapPrecision` (to adjust precision and reduce artifacts) [S3, unofficial]. Artifacts are still visible during rapid movement due to CPU latency [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(n \log n)$ for comparison-based sorts, where $n$ is the number of splats [S1, unofficial].
*   **Memory Complexity:** $O(n)$ to store the depth-index pairs and the output index buffer [S2, unofficial].
*   **Measured Performance:** Sorting ~1M splats takes approximately 150ms on a standard CPU, yielding a sorting throughput of ~4fps [S1, unofficial]. The main rendering loop typically runs at 60fps, meaning the sort is performed asynchronously and infrequently [S1, unofficial].
*   **Strengths:**
    *   Offloads sorting from the GPU, allowing the GPU to focus on rasterization and blending [S1, unofficial].
    *   Avoids the complexity and potential overhead of implementing GPU-side sorting algorithms like bitonic sort [S1, unofficial].
    *   Works in environments where GPU compute shaders are unavailable (e.g., WebGL 1.0, older hardware) [S2, unofficial].
*   **Limitations:**
    *   **Latency:** The sorting process is slow relative to the frame rate, leading to visual artifacts during rapid camera movement [S1, unofficial][S3, unofficial].
    *   **Scalability:** Performance degrades with increasing splat count; 1M splats take 150ms, which is unacceptable for real-time if sorted every frame [S1, unofficial].
    *   **Precision Issues:** Integer-based sorting may fail in scenes with large dimensions, and floating-point precision can cause artifacts if not tuned [S3, unofficial].
*   **When to Use:** When GPU compute capabilities are limited or unavailable, or when the scene is static/slow-moving enough that infrequent sorting updates are acceptable [S1, unofficial].
*   **When Not to Use:** For high-speed camera movement or real-time applications requiring frame-by-frame sorting accuracy [S1, unofficial][S3, unofficial].

## Background
CPU-based splat sorting was explored as an alternative to GPU-based sorting in early web-based 3D Gaussian Splatting implementations. The approach was highlighted in the `antimatter15/splat` repository, where the author noted that while GPU sorting (e.g., bitonic sort) is parallelizable, it splits the GPU's time between rendering and sorting, potentially reducing overall efficiency [S1, unofficial]. The author settled on running the sort asynchronously on the CPU in a WebWorker, accepting the lower throughput (~4fps) in exchange for simpler GPU code and acceptable visual quality during normal navigation [S1, unofficial]. This method is also discussed in `mkkellogg/GaussianSplats3D`, which implements WASM SIMD optimizations for CPU sorting and provides viewer parameters to handle precision issues [S3, unofficial].

## Sources
[S1] antimatter15/splat, https://github.com/antimatter15/splat, fetched 2026-10-27 (unofficial)
[S2] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
[S3] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
