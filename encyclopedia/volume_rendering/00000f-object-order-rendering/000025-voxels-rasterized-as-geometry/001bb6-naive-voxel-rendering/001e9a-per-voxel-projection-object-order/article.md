# Per-voxel Projection (Object-order)

Also called: Forward rendering, Histogram-based sorting, Object Space Approach, Object order voxel projection with value-sorted array, Object-order voxel projection, Voxel projection with value sorting, Voxel space projection

## Overview
Per-voxel projection is an object-order volume rendering technique that iterates through volume data, projects each voxel onto the 2D image plane, and accumulates results to form the final image [S2, S4, unofficial]. It is specifically optimized for Maximum Intensity Projection (MIP) by processing voxels in value-sorted order, allowing simple overwriting of pixel values instead of expensive read-compare-write operations [S1, S3, unofficial]. This method differs from image-order (ray-casting) approaches by projecting geometry into the image space rather than sampling rays through the volume [S2, unofficial].

## How it works
**Inputs:**
*   Volume dataset $V$ (3D grid of scalar values).
*   Camera parameters (view matrix, projection matrix, image resolution $W \times H$).
*   Definition of "interesting" voxels (typically non-zero values) [S1, S3, unofficial].

**Outputs:**
*   Image plane buffer $I$ of size $W \times H$, initialized to the minimum possible value (e.g., 0 or $-\infty$) for MIP [S1, S3, unofficial].

**Data Structures:**
1.  **Value-Sorted Array ($P$):** An array of voxel positions (coordinates) containing only "interesting" voxels. The array is sorted by voxel value in ascending order (low to high) [S1, S3, unofficial].
2.  **Cumulative Histogram Array ($H_{cum}$):** An auxiliary array storing cumulative histogram values to facilitate efficient access or reconstruction if needed [S1, S3, unofficial].
    *   *Note:* In the specific context of MIP rendering described in [S1, S3, unofficial], the primary structure used for the projection loop is the value-sorted array $P$. The histogram array is mentioned as part of the storage scheme but the projection logic relies on the sorted order of $P$.

**Algorithm Steps:**

1.  **Preprocessing (Sorting):**
    *   Identify all "interesting" voxels in $V$. A voxel is interesting if its value $v > 0$ (or a defined threshold) [S1, unofficial].
    *   Create a list of these voxels with their 3D coordinates and values.
    *   Sort this list by value in ascending order (lowest value first) [S1, S3, unofficial].
    *   Store the sorted coordinates in array $P$ [S3, unofficial].

2.  **Initialization:**
    *   Initialize the image buffer $I$ with the minimum value for MIP (e.g., 0) [S1, S3, unofficial].

3.  **Projection Loop:**
    *   Iterate through each voxel position $p_i$ in the sorted array $P$ from index $i = 0$ to $N-1$ (where $N$ is the number of interesting voxels) [S1, S3, unofficial].
    *   For each voxel $p_i$:
        a.  **Transform:** Compute the 3D-to-2D projection of the voxel center $p_i$ using the camera's view and projection matrices. Let the resulting image coordinates be $(u_i, v_i)$ [S2, S4, unofficial].
        b.  **Clipping:** Check if $(u_i, v_i)$ lies within the valid image bounds $[0, W-1] \times [0, H-1]$. If not, skip this voxel [G].
        c.  **Accumulation (MIP):** Since voxels are processed in ascending value order, the current voxel's value $v_i$ is greater than or equal to any previously processed voxel that projected to the same pixel. Therefore, simply overwrite the value at $I[u_i, v_i]$ with $v_i$ [S1, S3, unofficial].
            $$ I[u_i, v_i] \leftarrow v_i $$
            *Note:* This overwriting strategy is valid *only* because of the ascending sort order. If arbitrary order were used, a comparison ($I[u_i, v_i] \leftarrow \max(I[u_i, v_i], v_i)$) would be required, but the sort allows direct assignment [S1, S3, unofficial].

4.  **Completion:**
    *   After all voxels in $P$ are processed, the buffer $I$ contains the Maximum Intensity Projection [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_voxel_projection_mip(volume, camera, width, height):
    """
    volume: 3D array of voxel values
    camera: object with project(point) -> (u, v) method
    width, height: image dimensions
    Returns: 2D array I for MIP
    """
    # 1. Identify interesting voxels (non-zero)
    interesting_voxels = []
    for x in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for z in range(volume.shape[2]):
                val = volume[x, y, z]
                if val > 0: # "Interesting" definition [S1, unofficial]
                    interesting_voxels.append((x, y, z, val))
    
    # 2. Sort by value ascending (low to high) [S1, S3, unofficial]
    # Sort key is the value (4th element of tuple)
    interesting_voxels.sort(key=lambda voxel: voxel[3])
    
    # 3. Initialize image buffer with minimum value for MIP [S1, S3, unofficial]
    # Assuming 0 is the minimum for positive-valued data
    image_buffer = [[0 for _ in range(width)] for _ in range(height)]
    
    # 4. Project and accumulate
    for (x, y, z, val) in interesting_voxels:
        # Transform voxel center to image coordinates [S2, S4, unofficial]
        u, v = camera.project((x, y, z))
        
        # Check bounds [G]
        if 0 <= u < width and 0 <= v < height:
            # Overwrite because of ascending sort order [S1, S3, unofficial]
            image_buffer[v][u] = val
            
    return image_buffer
```

**Edge Cases and Pitfalls:**
*   **Arbitrary Order:** If voxels are projected in arbitrary order, the final image is incorrect because the last projected voxel overwrites previous ones, regardless of depth or value [S4, unofficial]. Strict sorting or explicit max-comparison is required [S4, unofficial].
*   **Non-Axial Views/Holes:** For non-axial viewing angles, undersampling may cause holes in the projection. This requires morphological sampling and closing operations on the projection images to reconstruct a continuous function [S1, unofficial].
*   **Zero Voxels:** Zero-value voxels are excluded from processing as they do not contribute to the maximum in typical positive-valued volumes (e.g., CT scans), providing significant compression [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ for sorting $N$ interesting voxels, plus $O(N)$ for projection. The sorting step is dominant if $N$ is large. However, $N$ is often much smaller than the total voxel count due to zero-voxel removal [S1, unofficial].
*   **Memory Complexity:** $O(N)$ to store the sorted array of interesting voxels and the image buffer. This is significantly less than storing the full 3D volume in GPU memory if the volume is sparse [S1, S3, unofficial].
*   **Performance:** Substantial reduction in processed voxels (often >95%) is achieved by filtering zeros [S1, unofficial].
*   **Strengths:**
    *   Efficient for MIP rendering on GPUs by avoiding read-modify-write cycles (using simple overwrites due to sorting) [S1, S3, unofficial].
    *   Enables compression-domain rendering; MIP can be computed directly on the value-sorted array without full 3D reconstruction [S1, unofficial].
    *   Simple implementation logic for MIP [S3, unofficial].
*   **Limitations:**
    *   Requires pre-sorting, which adds overhead.
    *   Not naturally suited for alpha-blending (transparency) without additional complexity, as the simple overwrite strategy relies on the max operation [S1, S3, unofficial].
    *   Susceptible to aliasing/holes in non-axial views without morphological post-processing [S1, unofficial].
*   **When to use:** Ideal for Maximum Intensity Projection (MIP) of sparse or large volumetric datasets where transparency is not required, and hardware acceleration for simple overwrites is available [S1, S3, unofficial].
*   **When not to use:** When alpha-blending (translucent volume rendering) is needed, or when the volume is dense (few zeros to filter), making sorting overhead unjustified [S1, S3, unofficial].

## Background
*   **Origin:** The object-order approach is a fundamental technique in volume rendering, historically contrasted with image-order (ray-casting) methods [S2, unofficial]. The specific application to MIP with value-sorted arrays for GPU acceleration is detailed in works on multiresolution MIP rendering [S1, S3, unofficial].
*   **History:** Early object-order algorithms, such as those by Upson and Keeler, processed voxels in front-to-back order for binary volumes [S4, unofficial]. The use of value-sorted arrays for MIP leverages the mathematical property that the maximum intensity is determined by the highest value, allowing a simple overwrite if processed in ascending order [S1, S3, unofficial].
*   **Usage:** Commonly used in medical imaging for MIP visualization of CT scans, where bone and contrast agents have high values and background is zero [S1, S2, unofficial].

## Sources
[S1] Comparison of Morphological Pyramids for Multiresolution MIP Volume Rendering, https://doi.org/10.2312/vissym/vissym02/061-070, fetched 2026-10-29 (unofficial)
[S2] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S3] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26 (unofficial)
[S4] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
