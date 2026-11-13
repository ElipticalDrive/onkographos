# Out-of-Core Frustum Clipping

Also called: CPU Frustum Culling

## Overview
Out-of-Core Frustum Clipping is a visibility culling stage in distributed volume rendering where each rendering node determines which parts of the dataset to display based on the current view, loading only the required subvolume at the maximum resolution fitting into texture memory [S1]. It sets itself apart from sibling methods by explicitly coupling the geometric frustum test with a resolution selection algorithm that ensures the loaded data fits within a strict voxel count limit per texture, enabling out-of-core streaming from hard disk [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Octree brick bounding boxes (defined in object space), camera projection matrix $P$, inverse texture matrix $T^{-1}$, viewport dimensions $V$, and a maximum voxel count limit $n$ [S1].
*   **Outputs:** A set of rectangular subvolumes (defined by their spatial extent and wavelet level) to be loaded from disk for each monitor/screen connected to the node [S1].

**Data Structures:**
*   **Octree Brick:** Represented by its 8 corner vertices in object space $v = (ox, oy, oz, 1.0)$ [S1].
*   **Viewport $V$:** A 4-element array or structure defining the tiled display size, typically $[x_{min}, y_{min}, width, height]$ [S1].
*   **Texture Limit $n$:** An integer constant defining the maximum number of voxels allowed in a single 3D texture (e.g., $16.7 \times 10^6$) [S1].

**Steps:**
1.  **Project Brick Corners:** For each octree brick, calculate the 2D window coordinates of its eight corners by projecting them into screen space [S1]. This is done by transforming the object-space vertex $v$ using the inverse texture matrix $T^{-1}$ and the projection matrix $P$, rather than the standard modelview matrix, because the volume is manipulated via texture coordinates [S1].
    *   Compute the intermediate point $v'$:
        $$v' = P \cdot T^{-1} \cdot v$$
    *   Calculate window coordinates $(w_x, w_y)$ using the viewport $V$:
        $$w_x = V[0] + V[2] \cdot \frac{v'[0] + 1}{2}$$
        $$w_y = V[1] + V[3] \cdot \frac{v'[1] + 1}{2}$$
    *   Note: $v'[0]$ and $v'[1]$ correspond to the clip-space x and y coordinates, normalized to $[-1, 1]$ before mapping to the viewport [S1].

2.  **Test Intersection:** Determine the bounding box of the projected 2D points $(w_x, w_y)$ for the eight corners [S1]. Test this 2D bounding box for intersection with the rectangular area of each LCD (monitor) in the tiled display grid [S1].

3.  **Define Subvolume:** If an intersection exists, define a rectangular subvolume corresponding to the intersection area on the screen [S1]. This subvolume represents the portion of the dataset that needs to be loaded [S1].

4.  **Calculate Wavelet Level:** Determine the highest detail level (wavelet level) for the subvolume such that the total number of voxels does not exceed the predefined limit $n$ [S1].
    *   Let $dx, dy, dz$ be the dimensions of the subvolume in the finest resolution (level 0) [S1].
    *   Initialize $level = 0$ and $cMod = 1$ [S1].
    *   Increment $level$ and $cMod$ (doubling $cMod$) until the voxel count fits:
        $$(dx/cMod) \cdot (dy/cMod) \cdot (dz/cMod) \le n$$
    *   The resulting $level$ is the wavelet level for loading [S1].

5.  **Load Data:** Load the content of the defined subvolume from hard disk at the calculated wavelet level into main memory [S1]. The data is prepared as a 3D RGBA texture [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mat_vec_mul(M, v):
    """Standard 4x4 matrix-vector multiplication."""
    return [sum(M[i][j] * v[j] for j in range(4)) for i in range(4)]

def out_of_core_frustum_clipping(bricks, P, T_inv, V, n):
    """
    bricks: List of objects, each having 8 corners in object space (ox, oy, oz, 1.0)
    P: Projection matrix (4x4)
    T_inv: Inverse texture matrix (4x4)
    V: Viewport [x_min, y_min, width, height]
    n: Max voxels per texture (e.g., 16.7e6)
    Returns: List of subvolumes to load, each defined by (subvolume_dims, wavelet_level)
    """
    loaded_subvolumes = []

    for brick in bricks:
        projected_corners = []
        
        # Step 1: Project all 8 corners
        for corner in brick.corners:
            v_prime = mat_vec_mul(P, mat_vec_mul(T_inv, corner))
            wx = V[0] + V[2] * (v_prime[0] + 1.0) / 2.0
            wy = V[1] + V[3] * (v_prime[1] + 1.0) / 2.0
            projected_corners.append((wx, wy))

        # Step 2: Test intersection with screen area
        min_x = min(p[0] for p in projected_corners)
        max_x = max(p[0] for p in projected_corners)
        min_y = min(p[1] for p in projected_corners)
        max_y = max(p[1] for p in projected_corners)

        # Check intersection with viewport V
        if (max_x >= V[0] and min_x <= V[0] + V[2] and
            max_y >= V[1] and min_y <= V[1] + V[3]):
            
            # Step 3: Define subvolume
            # The sources do not specify the exact geometric mapping from
            # 2D screen intersection back to 3D subvolume dimensions [S1].
            # We assume dx, dy, dz represent the finest-resolution dimensions
            # of the rectangular subvolume defined by the intersection.
            dx, dy, dz = brick.dx, brick.dy, brick.dz

            # Step 4: Calculate wavelet level
            level = 0
            c_mod = 1
            while (dx / c_mod) * (dy / c_mod) * (dz / c_mod) > n:
                level += 1
                c_mod = 2 * c_mod
            
            # Step 5: Add to load list
            loaded_subvolumes.append({
                'dimensions': (dx, dy, dz),
                'wavelet_level': level,
                'brick_id': brick.id
            })

    return loaded_subvolumes
```

**Parameters:**
*   **Viewport $V$:** Defined by the size of the tiled display [S1]. Example: $25,600 \times 8,000$ pixels [S1].
*   **$n$:** Maximum number of voxels per texture [S1]. Set to $16.7 \times 10^6$ (approx. 16.7 million) [S1].
*   **Texture Size:** A 3D RGBA texture with $16.7 \times 10^6$ voxels occupies 64MB of memory [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation relies on floating-point projection. Degenerate cases where a brick is exactly on the boundary of the frustum or screen may require epsilon checks to avoid flickering or missed loads [G].
*   **Texture Memory:** The method assumes a fixed maximum voxel count $n$ to fit within texture memory [S1].
*   **Fallback Texture:** The system requires storing three textures per node: one for each of the two monitors and one common low-resolution fallback texture containing the entire dataset [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of octree bricks, as each brick is projected and tested independently [S1].
*   **Memory Complexity:** Bounded by the texture memory limit [S1]. Each node stores at most three textures: two high-resolution textures for the monitors and one low-resolution fallback texture [S1].
*   **Performance:** The method enables rendering of gigascale datasets by streaming only visible, resolution-appropriate chunks from disk [S1].
*   **Strengths:** Efficiently manages memory by limiting texture size; adapts resolution to view distance and screen area [S1].
*   **Limitations:** Requires pre-computed octree/wavelet structure; relies on CPU for culling and projection [S1].

## Background
The method originates from the implementation of "Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters" [S1]. It was developed to handle massive datasets on distributed systems with tiled displays, where each node manages its own monitors and data loading [S1].

## Sources
[S1] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
