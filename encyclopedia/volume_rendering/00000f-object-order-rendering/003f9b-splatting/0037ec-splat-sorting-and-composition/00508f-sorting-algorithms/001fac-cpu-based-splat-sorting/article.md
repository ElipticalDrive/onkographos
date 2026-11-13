# CPU-Based Splat Sorting

Also called: Splat Sort, WebWorker Splat Sort

## Overview
CPU-Based Splat Sorting is a method for ordering volumetric primitives (specifically 3D Gaussian splats) by depth to enable correct front-to-back alpha blending [S2, unofficial]. It solves the problem of rendering transparency in splat-based volume visualization by ensuring that occluded splats do not incorrectly contribute to the final pixel color [G]. It differs from GPU-based sorting methods (like [Cascaded 2-way Merge](7689-cascaded-2-way-merge-for-global-splat-sorting.md)) by offloading the $O(n \log n)$ sorting computation to the CPU, allowing the GPU to dedicate its resources exclusively to rasterization and blending [S1, unofficial][S2, unofficial]. This approach is particularly useful in web environments or when GPU compute shaders are unavailable, though it introduces latency that can cause visual artifacts during rapid camera movement [S1, unofficial][S3, unofficial].

## How it works
The method transforms splat positions into view space, extracts a depth metric, sorts the splats based on this metric, and outputs a permutation index buffer for the GPU renderer [S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   `viewProj`: A $4 \times 4$ Model-View-Projection matrix transforming volume space to clip space [S2, unofficial].
    *   `positions`: An array of $N$ 3D positions $\mathbf{p}_k = (x_k, y_k, z_k)$, typically interleaved [S2, unofficial].
    *   `vertexCount`: The number of splats $N$ [S2, unofficial].
*   **Outputs:**
    *   `depthIndex`: A vector of $N$ unsigned integers representing the original indices of the splats, sorted from front (smallest depth) to back (largest depth) [S2, unofficial].

### Data Structures
1.  **Splat Pair:** A structure or tuple containing:
    *   `depth`: A floating-point value representing the projected depth [S2, unofficial].
    *   `index`: The original integer index of the splat in the input array [S2, unofficial].
2.  **Index Buffer:** A contiguous array of integers used by the GPU vertex shader to fetch splat attributes in the sorted order [S2, unofficial].

### Steps
1.  **Project Positions:** For each splat $k$ with position $\mathbf{p}_k$:
    *   Compute the homogeneous clip position:
        $$ \mathbf{v}_{clip} = \text{viewProj} \cdot \begin{bmatrix} x_k \\ y_k \\ z_k \\ 1 \end{bmatrix} $$
    *   Extract the depth component $z_{clip}$ and the perspective divide factor $w_{clip}$ from $\mathbf{v}_{clip}$ [S2, unofficial].
    *   Compute the normalized depth:
        $$ d_k = \frac{z_{clip}}{w_{clip}} $$
        *Note: Division by $w$ is required for perspective correctness, though some pipelines may use raw $z_{clip}$ if the projection matrix preserves order [S2, unofficial].*
2.  **Create Pairs:** Store the pair $(d_k, k)$ for all $k \in [0, N-1]$ [S2, unofficial].
3.  **Sort:** Sort the array of pairs using a comparison function that orders by depth ascending (front-to-back):
    $$ \text{compare}(a, b) \implies a.\text{depth} < b.\text{depth} $$
    Standard CPU sorting algorithms (e.g., Quicksort, Mergesort) are used [S1, unofficial][S2, unofficial].
4.  **Extract Indices:** Create the output `depthIndex` vector by extracting the original index $k$ from each sorted pair [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_splat_sort(view_proj_matrix, positions, vertex_count):
    """
    Sorts splats by depth on the CPU.
    
    Args:
        view_proj_matrix: 4x4 matrix (column-major layout, list of lists or numpy array)
        positions: List of tuples (x, y, z)
        vertex_count: Integer N
        
    Returns:
        sorted_indices: List of integers representing original indices sorted front-to-back
    """
    depth_index_pairs = []
    
    for k in range(vertex_count):
        x, y, z = positions[k]
        
        # Column-major matrix multiplication: v_clip = M * v
        wx = view_proj_matrix[0][0]*x + view_proj_matrix[1][0]*y + view_proj_matrix[2][0]*z + view_proj_matrix[3][0]
        wy = view_proj_matrix[0][1]*x + view_proj_matrix[1][1]*y + view_proj_matrix[2][1]*z + view_proj_matrix[3][1]
        wz = view_proj_matrix[0][2]*x + view_proj_matrix[1][2]*y + view_proj_matrix[2][2]*z + view_proj_matrix[3][2]
        ww = view_proj_matrix[0][3]*x + view_proj_matrix[1][3]*y + view_proj_matrix[2][3]*z + view_proj_matrix[3][3]
        
        # Compute normalized depth
        if ww == 0:
            depth = float('inf')
        else:
            depth = wz / ww
            
        depth_index_pairs.append((depth, k))
        
    # Sort by depth ascending (front to back)
    depth_index_pairs.sort(key=lambda pair: pair[0])
    
    # Extract indices
    sorted_indices = [pair[1] for pair in depth_index_pairs]
    
    return sorted_indices
```

### Parameters and Typical Values
*   **Sorting Algorithm:** Standard $O(N \log N)$ comparison sort (e.g., `std::sort` in C++, `Array.sort` in JS) [S1, unofficial][S2, unofficial].
*   **Precision:** Floating-point (32-bit) is standard [S3, unofficial].
*   **Asynchrony:** In web contexts, this is often executed in a Web Worker to prevent blocking the main render thread [S1, unofficial].

### Edge Cases and Pitfalls
*   **Camera Latency:** Because sorting is asynchronous and slower than rendering, rapid camera movements (e.g., jumping to the opposite side of the scene) cause momentary Z-order artifacts where splats are rendered in the previous frame's order [S1, unofficial][S3, unofficial].
*   **Large Dimensions:** Scenes with very large coordinate values may cause "Index out of bounds" errors or precision loss in the depth calculation, particularly if integer-based sorting is used [S3, unofficial].
*   **Precision Tuning:** For high-density or large-scale scenes, the `splatSortDistanceMapPrecision` parameter may need adjustment to balance performance and artifact reduction [S3, unofficial].

## Subtopics

### [Splat Sorting Key Calculation](2106-splat-sorting-key-calculation.md)
This variant calculates a scalar key for each 3D Gaussian splat to determine rendering order, prioritizing splats based on projected size and opacity rather than pure geometric depth. It is chosen when visual contribution (importance) is more critical than strict geometric occlusion, which is common in Gaussian Splatting where splat sizes vary significantly. It differs from standard depth sorting by ignoring camera distance in favor of a composite "importance" metric [G].

### [CPU Fallback Sort Path](5586-cpu-fallback-sort-path.md)
This strategy explicitly executes sorting on the CPU when GPU compute shaders are unavailable, disabled, or insufficient, allowing the GPU to focus solely on rasterization. It differs from GPU-based sorting (like bitonic sort) by avoiding the overhead of complex GPU sorting kernels and relying on CPU-side algorithms (radix or $O(n \log n)$). It is chosen for compatibility with older hardware or web environments where WebGPU compute is not supported.

### WASM/SIMD CPU Splat Sorting
An optimization of the CPU sort path implemented in C++ using WebAssembly (WASM) SIMD instructions to accelerate the sorting process [S3, unofficial]. It differs from generic CPU sorting by leveraging vectorized instructions for parallel comparison and swapping [S3, unofficial]. It includes parameters `integerBasedSort` (to force floating-point sorting for large scenes) and `splatSortDistanceMapPrecision` [S3, unofficial]. It is chosen for web-based viewers requiring better performance than standard JS sorting while maintaining GPU offloading for rendering [S1, unofficial][S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ for the sorting phase, where $N$ is the number of splats [S1, unofficial]. The GPU rendering phase remains $O(P)$ where $P$ is the number of pixels, but the CPU sort is a separate bottleneck [S1, unofficial].
*   **Memory Complexity:** $O(N)$ to store the depth-index pairs and the output index buffer [S2, unofficial].
*   **Measured Performance:** Sorting ~1 million splats on a typical CPU takes approximately 150ms, resulting in a sort rate of ~4fps, while the GPU render loop runs at 60fps [S1, unofficial].
*   **Strengths:**
    *   Simplifies GPU shader logic (no need for complex sorting shaders or transform feedback) [S1, unofficial][S2, unofficial].
    *   Compatible with any hardware that supports basic OpenGL/WebGL rasterization [S2, unofficial].
    *   Effective for static or slowly moving scenes [S1, unofficial].
*   **Limitations:**
    *   High latency during rapid camera movement causes visible sorting artifacts [S1, unofficial][S3, unofficial].
    *   CPU becomes a bottleneck for very large $N$ (>1M splats) if not optimized with SIMD [S3, unofficial].
    *   Not suitable for real-time interactive applications requiring instant Z-order updates during fast navigation [S1, unofficial][S3, unofficial].
*   **When to Use:** Web-based viewers, legacy hardware, or scenes with static camera viewpoints [S1, unofficial][S3, unofficial].
*   **When Not to Use:** High-performance real-time VR/AR with rapid head movement, or scenes with >10M splats where CPU sort latency exceeds frame budget [S1, unofficial][S3, unofficial].

## Background
CPU-based splat sorting originated as a pragmatic solution for early web-based 3D Gaussian Splatting viewers, where GPU compute shaders were not universally available or efficient for sorting tasks [S1, unofficial]. The approach was popularized by implementations like `antimatter15/splat`, which demonstrated that offloading sorting to a Web Worker allowed for acceptable visual quality despite the latency [S1, unofficial]. It serves as a fallback to GPU-based methods like [Cascaded 2-way Merge](7689-cascaded-2-way-merge-for-global-splat-sorting.md) when hardware acceleration for sorting is not feasible [S1, unofficial][S2, unofficial].

## Sources
[S1] antimatter15/splat, https://github.com/antimatter15/splat, fetched 2026-10-27 (unofficial)
[S2] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
[S3] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
