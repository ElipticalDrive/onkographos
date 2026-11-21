# Wavelet-based Multiresolution Rendering

Also called: Wavelet-based Multiresolution Isosurface Rendering

## Overview
An interactive rendering method for isosurfaces in voxel grids that represents the underlying trivariate function as a spline wavelet hierarchy [S1]. This allows for adaptive, view-dependent selection of level-of-detail by superimposing appropriately weighted basis functions, decreasing resolution of distant parts to improve performance with little impact on visual quality [S1]. The method implements both wavelet reconstruction and root finding in CUDA to utilize high computational performance [S1].

## How it works
**Inputs:**
*   Volumetric dataset $V$ of size $N \times N \times N$ (tested up to $512^3$ voxels) [S1].
*   Isovalue $c$ defining the surface $f(x,y,z) = c$ [S1].
*   Camera position and viewport parameters (tested up to $1024 \times 768$ pixels) [S1].
*   Parameters:
    *   Interpolation degree $d$: Surface degree along each dimension. Overall degree is $3d$. Common choices: tri-linear ($d=1$) or tri-cubic ($d=3$) [S1].
    *   Wavelet type: Polynomial spline wavelets with the least possible number of vanishing moments to minimize filter support [S1].
    *   Chunk size: $k \times k \times k$ nodes, where $k=8$ [S1].
    *   Error threshold $\varepsilon$: User-defined threshold for view-dependent activation [S1].

**Outputs:**
*   Rendered isosurface image [S1].

**Data Structures:**
*   **Octree:** Nested vector spaces where nodes represent regions of the volume. Leaf nodes store min/max values estimated using the convex combination property of the B-spline basis, propagated towards the root [S1].
*   **Wavelet Coefficients:** Stored in a hierarchy corresponding to the octree levels. Detail coefficients are set to zero for deactivated nodes [S1].
*   **Chunks:** Groups of $k \times k \times k$ octree nodes processed together for GPU efficiency [S1].

**Steps:**

1.  **Wavelet Transform:** Perform a 3D wavelet transform on the volume dataset using polynomial spline wavelets [S1].
    *   Due to the separability of the B-spline wavelet basis, the transformation is performed sequentially in the x-, y-, and z-directions [S1].
    *   This reduces computation to three times the 1D case (e.g., for tri-cubic, $3 \times 7$ coefficients are considered per dimension) [S1].
    *   Implementation uses a "gather" approach: traverse the octree hierarchy and gather all contributions of wavelets intersecting the current node, avoiding atomic operations associated with scattering [S1].

2.  **Octree Traversal and Chunk Activation:**
    *   Traverse the octree hierarchy starting from the root [S1].
    *   Expand octree nodes as needed until the vector space spanned by expanded nodes contains the desired approximation [S1].
    *   Evaluate an error metric for each chunk of $k \times k \times k$ nodes [S1].
    *   A chunk is activated if:
        1.  It can possibly contain the current isovalue (min/max comparison with the render tree entry) [S1].
        2.  $s \cdot c > \varepsilon$, where $s$ is the node diameter, $c$ is the distance from the camera, and $\varepsilon$ is the user-defined threshold [S1].
    *   If all detail coefficients for a chunk are zero, the output corresponds to the next lower resolution level [S1].

3.  **Selective Reconstruction (CUDA):**
    *   Reconstruction is applied to one chunk after another. Deactivated chunks are skipped [S1].
    *   Chunks that are neither activated nor needed for continuity are omitted [S1].
    *   If a wavelet's support in an activated chunk extends into neighboring chunks, those neighbors are also reconstructed to maintain continuity [S1].
    *   **Kernel Execution:**
        *   CUDA kernels consist of blocks of $k \times k$ threads [S1].
        *   Each thread computes a 1D convolution with wavelet filter kernels (low pass and high pass) [S1].
        *   One block is responsible for a beam of $k \times k$ voxels along the current convolution direction [S1].
        *   At each reconstruction step, ensure all needed data resides in register space before convolution [S1].
        *   Reuse data fetched for the reconstruction of the previous chunk [S1].

4.  **Root Finding:**
    *   For each activated chunk, solve for the isosurface $f(x,y,z) = c$ [S1].
    *   The function is represented as:
        $$f(x, y, z) = \sum_{0 \leq i, j, k \leq d} f_{ijk} x^i y^j z^k = 0 \quad (1)$$
        where $f_{ijk}$ are the wavelet coefficients and $d$ is the surface degree [S1].
    *   Different root finding techniques are compared for precision and efficiency [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wavelet_multiresolution_rendering(volume, isovalue, camera_pos, threshold):
    # 1. Wavelet Transform
    # Perform separable 3D wavelet transform using polynomial spline wavelets
    # Sequentially transform along x, y, and z directions using a gather approach
    coeffs = wavelet_transform_separable(volume)

    # 2. Build Octree with Min/Max Bounds
    # Estimate leaf min/max using convex combination of B-spline basis
    # Propagate bounds to root
    octree = build_octree_bounds(coeffs)

    # 3. View-dependent Chunk Activation
    k = 8
    active_chunks = []
    for chunk in octree.get_chunks(k):
        min_val, max_val = chunk.get_bounds()
        if not (min_val <= isovalue <= max_val):
            continue
        s = chunk.node_diameter
        c = distance(camera_pos, chunk.center)
        if s * c > threshold:
            active_chunks.append(chunk)

    # 4. Selective Reconstruction & Rendering
    for chunk in active_chunks:
        # Ensure continuity: reconstruct neighbors if wavelet support extends into them
        neighbor_chunks = get_continuity_neighbors(chunk, k)
        chunks_to_process = list(set([chunk] + neighbor_chunks))

        reconstruction_data = []
        for c in chunks_to_process:
            # If all detail coefficients are zero, output corresponds to next lower resolution
            data = reconstruct_chunk(c, coeffs, k)
            reconstruction_data.extend(data)

        # Root finding for isosurface
        points = find_isosurface_points(reconstruction_data, isovalue, degree=3)

        # Render points (safely handles empty lists to prevent unpacking errors)
        render_surface(points)

def wavelet_transform_separable(volume):
    # 1D convolution along x, then y, then z
    # Uses low-pass and high-pass filter kernels
    # Gather approach: traverse octree and accumulate wavelet contributions
    data = volume.copy()
    for dim in [0, 1, 2]:
        data = convolve_1d_gather(data, dim)
    return data

def build_octree_bounds(coeffs):
    # Create octree hierarchy from coefficients
    # Leaf bounds from convex combination of B-spline basis
    # Propagate min/max up to root
    return Octree(coeffs)

def get_continuity_neighbors(chunk, k):
    # Return adjacent chunks whose wavelet support overlaps the current chunk
    return chunk.get_adjacent_chunks()

def reconstruct_chunk(chunk, coeffs, k):
    # CUDA kernel: blocks of k x k threads
    # Each thread computes 1D convolution with wavelet filters
    # One block handles k x k voxel beam along convolution direction
    # Data reused from previous chunk, kept in registers
    return chunk.apply_wavelet_reconstruction(coeffs, k)

def find_isosurface_points(data, isovalue, degree):
    # Solve f(x,y,z) = sum(f_ijk * x^i * y^j * z^k) - isovalue = 0
    # Returns list of (x, y, z) tuples on the surface
    points = []
    for sample in data:
        # [G] Standard numerical root finding (e.g., Newton-Raphson or grid traversal)
        # Returns list of 3D points to prevent unpacking errors when empty
        roots = solve_polynomial_surface(sample, isovalue, degree)
        points.extend(roots)
    return points

def solve_polynomial_surface(sample, isovalue, degree):
    # [G] Standard numerical method to find intersections of polynomial surface
    return []

def render_surface(points):
    # Project and rasterize points to viewport
    pass
```

## Subtopics

### [Wavelet Transform](2285-wavelet-transform.md)
The mathematical decomposition engine providing spatially localized frequency information and sparse representation.

### [Wavelet-based Adaptive Resolution Volume Rendering](1171-wavelet-based-adaptive-resolution-volume-rendering.md)
Focuses on volume accumulation rather than isosurface extraction.

### [Hierarchical Wavelet Transform on Omnitrees](2216-hierarchical-wavelet-transform-on-omnitrees.md)
Uses omnitrees instead of standard octrees for hierarchical storage.

### [L2-optimal wavelet splatting](2816-l2-optimal-wavelet-splatting.md)
Uses L2-optimal splatting techniques for rendering.

### [Point-wise Inverse Discrete Wavelet Transform (PW-IDWT) via Generalized Projection](3394-point-wise-inverse-discrete-wavelet-transform-pw-idwt-via-ge.md)
An alternative inverse transform method.

## Operating characteristics
*   **Time Complexity:** Depends on the number of active chunks and the support size of the wavelet [S1]. The separable transform reduces complexity compared to a full 3D transform [S1]. GPU implementation utilizes high computational performance [S1].
*   **Memory Complexity:** Scales with the volume size and the octree structure [S1]. Detail coefficients for deactivated nodes are set to zero, saving memory [S1].
*   **Performance:** Tested with datasets up to $512^3$ voxels and viewport sizes up to $1024 \times 768$ pixels, demonstrating interactive frame rates [S1].
*   **Strengths:** Interactive rendering, view-dependent level-of-detail, continuous surfaces (no cracks) due to B-spline continuity [S1].
*   **Limitations:** Complexity of root finding for higher degrees; dependency on GPU hardware for performance [S1].

## Background
Originated by Markus Steinberger and Markus Grabner from the Institute for Computer Graphics and Vision, Graz University of Technology, Austria [S1]. The method addresses the increasing size of volumetric datasets from technologies like computed tomography (CT), where displaying full resolution is often infeasible [S1]. It builds on the wavelet transformation's ability to decorrelate data and allow selective removal of irrelevant data [S1].

## Sources
[S1] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
