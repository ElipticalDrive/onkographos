# Hierarchical Spatial Enumeration Optimization

Also called: Complete octree representation, Hierarchical Spatial Enumeration (Octree Pyramid), Pyramid of binary volumes

## Overview
Hierarchical Spatial Enumeration Optimization is a data structure and rendering acceleration technique for image-order volume rendering that encodes spatial coherence (specifically, regions of zero opacity) using a pyramid of binary volumes [S1][S2, unofficial]. It solves the problem of efficiently skipping empty space during ray traversal by representing the volume data as a complete octree (pyramid) rather than a condensed sparse structure [S1]. Unlike sibling methods that may use condensed representations or object-order traversal, this approach enables fast image-order ray tracing by allowing rays to jump over large empty regions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset of size $N \times N \times N$ voxels, where $N = 2^M + 1$ for some integer $M$, with each voxel having an opacity value $\alpha$ [S1].
*   **Output:** A pyramid of $M+1$ binary volumes $V_0, \dots, V_M$, where each volume $V_m$ contains binary flags indicating the presence of non-zero opacity in the corresponding spatial region [S1].

**Data Structures:**
The structure is a pyramid of $M+1$ binary volumes indexed by level $m \in \{0, \dots, M\}$ [S1].
*   **Volume Dimensions:**
    *   $V_0$ has dimensions $(N-1) \times (N-1) \times (N-1)$ cells [S1].
    *   $V_m$ has dimensions $\frac{N-1}{2^m} \times \frac{N-1}{2^m} \times \frac{N-1}{2^m}$ cells [S1].
    *   $V_M$ is a single cell ($1 \times 1 \times 1$) [S1].
*   **Cell Content:** Each cell $V_m(\mathbf{i})$ stores a binary value (0 or 1) [S1].
    *   $1$: The region contains at least one voxel with non-zero opacity [S1].
    *   $0$: The region contains only voxels with zero opacity (empty) [S1].
*   **Coordinate System:**
    *   Voxels are treated as point samples, while cells fill the space between them [S1].
    *   Cell $(1,1,1)$ at level $0$ encloses the space between voxel $(1,1,1)$ and voxel $(2,2,2)$ [S1].
    *   Voxel $(1,1,1)$ is placed at the front-lower-right corner of cell $(1,1,1)$ at level $0$ [S1].
    *   The size of a cell on level $m$ is $2^m$ times the spacing between voxels [S1].

**Construction Algorithm:**
The pyramid is constructed recursively from the base level $V_0$ up to $V_M$ [S1].

1.  **Base Level ($V_0$) Construction:**
    For each cell index $\mathbf{i} = (i,j,k)$ where $i,j,k \in \{1, \dots, N-1\}$:
    $$
    V_0(\mathbf{i}) = \begin{cases} 
    1 & \text{if } \alpha(\mathbf{i} + \Delta\mathbf{i}) = 1 \text{ for any } \Delta\mathbf{i} \in \{0, 1\}^3 \\
    0 & \text{otherwise}
    \end{cases}
    $$
    This means cell $\mathbf{i}$ in $V_0$ is $1$ if any of the $8$ voxels at its vertices (defined by offsets $\Delta\mathbf{i}$) has non-zero opacity [S1].

2.  **Recursive Level Construction ($m > 0$):**
    For each cell index $\mathbf{i}$ in volume $V_m$:
    $$
    V_m(\mathbf{i}) = \begin{cases} 
    1 & \text{if } V_{m-1}(\mathbf{j}) = 1 \text{ for any child cell } \mathbf{j} \text{ of } \mathbf{i} \\
    0 & \text{otherwise}
    \end{cases}
    $$
    Specifically, cell $\mathbf{i}$ in $V_m$ contains $0$ if all $8$ cells on level $m-1$ that form its octants contain zeros; otherwise, it contains $1$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_pyramid(opacity_voxels, N):
    """
    Builds the hierarchical spatial enumeration pyramid.
    
    Args:
        opacity_voxels: 3D array of size N x N x N containing opacity values.
                        Uses 1-based indexing to match the source notation.
        N: Side length of the voxel grid. Must satisfy N = 2^M + 1.
        
    Returns:
        pyramid: List of 3D binary arrays [V_0, V_1, ..., V_M].
    """
    import math
    M = int(math.log2(N - 1))
    pyramid = []
    
    # Step 1: Construct Base Volume V_0
    # V_0 dimensions: (N-1) x (N-1) x (N-1)
    dim_0 = N - 1
    # Initialize 1-based padded arrays
    V_0 = [[[0 for _ in range(dim_0 + 1)] for _ in range(dim_0 + 1)] for _ in range(dim_0 + 1)]
    
    for i in range(1, dim_0 + 1):
        for j in range(1, dim_0 + 1):
            for k in range(1, dim_0 + 1):
                has_nonzero = False
                for di in (0, 1):
                    for dj in (0, 1):
                        for dk in (0, 1):
                            if opacity_voxels[i + di][j + dj][k + dk] != 0:
                                has_nonzero = True
                                break
                        if has_nonzero: break
                    if has_nonzero: break
                V_0[i][j][k] = 1 if has_nonzero else 0
    pyramid.append(V_0)
    
    # Step 2: Construct higher levels V_1 to V_M
    current_vol = V_0
    for m in range(1, M + 1):
        prev_vol = current_vol
        prev_dim = dim_0 // (2 ** (m - 1))
        curr_dim = prev_dim // 2
        
        V_m = [[[0 for _ in range(curr_dim + 1)] for _ in range(curr_dim + 1)] for _ in range(curr_dim + 1)]
        
        for i in range(1, curr_dim + 1):
            for j in range(1, curr_dim + 1):
                for k in range(1, curr_dim + 1):
                    has_nonzero = False
                    for di in (0, 1):
                        for dj in (0, 1):
                            for dk in (0, 1):
                                if prev_vol[2*i + di][2*j + dj][2*k + dk] == 1:
                                    has_nonzero = True
                                    break
                            if has_nonzero: break
                        if has_nonzero: break
                    V_m[i][j][k] = 1 if has_nonzero else 0
                    
        pyramid.append(V_m)
        current_vol = V_m
        
    return pyramid
```

**Usage in Rendering:**
The pyramid is used to efficiently compute intersections between viewing rays and regions of interest [S2, unofficial].
*   **Ray Tracing:** Rays are traced through the pyramid; if a cell in the current level is $0$, the entire region is empty and the ray can skip it [S1].
*   **Hybrid Ray Tracer:** The pyramid speeds up tracing through segments of volumetric data lying between successive polygon intersections [S2, unofficial].
*   **Scan Conversion:** The composite dataset including sampled polygons can be used during pyramid construction to improve efficiency for mixed geometric and volumetric data [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Coordinate Alignment:** Voxel $(1,1,1)$ is at the front-lower-right corner of cell $(1,1,1)$ at level $0$, and cell $(1,1,1)$ at level $0$ encloses the space between voxels $(1,1,1)$ and $(2,2,2)$ [S1]. Misalignment here will cause incorrect empty region detection [G].
*   **Dataset Size:** The method requires $N = 2^M + 1$ [S1]. Not given in the sources.
*   **Opacity Threshold:** The binary volume uses a threshold of $0$; voxels with opacity exactly $0$ are considered empty, while any non-zero opacity marks the region as occupied [S1].

## Subtopics

### [Hierarchical Volume Traversal with Adaptive Termination](1682-hierarchical-volume-traversal-with-adaptive-termination.md)
A ray tracing algorithm that uses this pyramidal structure to skip empty regions and an adaptive termination criterion to stop tracing when opacity changes become negligible. It differs from geometric ray tracing by using a hierarchical binary volume structure, which is more efficient for volume data with tens of millions of spatially ordered cells.

## Operating characteristics
*   **Time Complexity:** The cost of ray tracing volumetric data using this hierarchical enumeration grows linearly with the size of the image, logarithmically with the length of each ray, and linearly with the depth complexity of the data [S2, unofficial].
*   **Memory Complexity:** The pyramid stores $M+1$ binary volumes [S1]. The total number of cells is $\sum_{m=0}^{M} (\frac{N-1}{2^m})^3$ [S1]. Since $N-1 = 2^M$, the base volume has $(2^M)^3$ cells [S1]. The total size is roughly $\frac{1}{7}(N-1)^3$ cells, which is significantly less than the original $N^3$ voxels if only the hierarchy is stored, but the base volume $V_0$ is still large [G]. Since $V_m$ are binary, they can be packed efficiently [G].
*   **Strengths:**
    *   Efficiently skips large empty regions in volume data [S1][S2, unofficial].
    *   Enables fast image-order rendering [S1].
    *   Can be integrated with hybrid ray tracing for mixed geometric/volumetric scenes [S2, unofficial].
*   **Limitations:**
    *   Requires dataset size $N = 2^M + 1$ [S1].
    *   Performance depends on the depth complexity of the data [S2, unofficial].
    *   Adaptive termination is needed to reduce dependence on depth complexity [S2, unofficial].
*   **When to Use:** Suitable for image-order volume rendering of large datasets with significant empty space (spatial coherence) [S1][S2, unofficial].
*   **When Not to Use:** If the dataset is small, or if the data has no spatial coherence (all voxels have non-zero opacity), the overhead of the pyramid may not be justified [G].

## Background
*   **Origin:** The method was introduced by Marc Levoy in the paper "Efficient ray tracing of volume data" [S1].
*   **History:** It is based on the observation that many volumetric datasets contain coherent regions of uninteresting voxels (opacity zero) [S2, unofficial]. The technique uses a pyramid of binary volumes to encode this spatial coherence [S2, unofficial].
*   **Comparison:** It differs from prior work by rendering in image order (tracing rays through the octree) rather than object order (depth-first traversal) [S1]. It also differs by using a pyramid of binary volumes/complete octree rather than a condensed representation [S1].
*   **Applications:** Medical imaging and molecular graphics [S1].

## Sources
[S1] Efficient ray tracing of volume data, https://doi.org/10.1145/78964.78965, fetched 2026-10-25
[S2] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
