# Wavelet-based Multiresolution Rendering

Also called: Wavelet-based Multresolution Isosurface Rendering

## Overview
An interactive rendering method for isosurfaces in a voxel grid that represents the underlying trivariate function as a spline wavelet hierarchy [S1]. It enables adaptive, view-dependent selection of level-of-detail by superimposing appropriately weighted basis functions, allowing distant parts of the volume to be rendered at lower resolution to improve performance with minimal visual impact [S1]. It distinguishes itself from sibling methods by explicitly combining GPU-accelerated wavelet reconstruction with a view-dependent error metric to selectively activate octree chunks, ensuring continuity by reconstructing necessary neighbors [S1].

## How it works
**Inputs:**
*   Voxel grid dataset $f(x,y,z)$.
*   Isovalue $c$ defining the surface $f(x,y,z) = c$.
*   Camera position and viewport parameters.
*   User-defined error threshold $\varepsilon$.
*   Interpolation degree $d$ (typically $d=1$ for tri-linear or $d=3$ for tri-cubic).
*   Chunk size $k \times k \times k$ (typically $k=8$).

**Outputs:**
*   Isosurface geometry rendered to the viewport.

**Data Structures:**
*   **Octree:** Nested vector spaces where nodes represent regions of the volume. Leaf nodes store min/max values estimated using the convex combination property of B-spline basis functions, propagated from leaves to root [S1].
*   **Wavelet Coefficients:** Result of the polynomial spline wavelet transform. Coefficients are stored in a structure supporting chunk-based access.
*   **Chunks:** Groups of $k \times k \times k$ octree nodes. Each chunk is processed as a unit for activation and reconstruction [S1].

**Algorithm Steps:**

1.  **Wavelet Transform:** Perform a 3D wavelet transform on the volume dataset using polynomial spline wavelets [S1]. To optimize for GPU scattering penalties, the transform is implemented by traversing the octree hierarchy and *gathering* contributions of wavelets intersecting each node, exploiting the separability of the B-spline basis to sequentially transform in x-, y-, and z-directions [S1].

2.  **Octree Traversal and Chunk Activation:** Traverse the octree hierarchy starting from the root. Expand nodes as needed until the vector space spanned by expanded nodes contains the desired approximation [S1]. Process the volume in chunks of $k \times k \times k$ nodes. For each chunk, evaluate an error metric to determine if it needs reconstruction:
    *   **Isovalue Check:** The chunk's min/max values (from octree nodes) must bracket the isovalue $c$.
    *   **View-Dependent Check:** The chunk is activated if $s \cdot c > \varepsilon$, where $s$ is the node diameter, $c$ is the distance from the camera to the chunk, and $\varepsilon$ is the user-defined threshold [S1].

3.  **Continuity Handling:** Identify neighboring chunks that must be reconstructed to maintain surface continuity. If a wavelet's support in an activated chunk extends into a neighboring chunk, that neighbor must also be reconstructed, even if it did not pass the activation metric [S1].

4.  **GPU Reconstruction:** For each activated or continuity-required chunk, execute a CUDA kernel to reconstruct the function values.
    *   The kernel consists of blocks of $k \times k$ threads.
    *   Each thread computes a 1D convolution with wavelet filter kernels (low pass and high pass) along one dimension.
    *   One block is responsible for a beam of $k \times k$ voxels along the current convolution direction [S1].
    *   Reconstruction is performed sequentially for x, y, and z dimensions.
    *   Data fetched for the previous chunk is reused where possible. All needed data is loaded into register space before convolution [S1].

5.  **Root Finding:** Solve for the isosurface within the reconstructed chunks. The function is represented as a piecewise algebraic surface:
    $$f(x, y, z) = \sum_{0 \leq i, j, k \leq d} f_{ijk} x^i y^j z^k = 0$$
    where $f_{ijk}$ are the reconstructed coefficients and $d$ is the surface degree along each dimension, resulting in an overall degree of $3d$ [S1]. Different root finding techniques are compared for precision and efficiency [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
K = 8  # Chunk dimension
EPSILON = 0.1  # User-defined threshold
D = 3  # Interpolation degree (tri-cubic)

# Data Structures
class OctreeNode:
    min_val: float
    max_val: float
    children: list[OctreeNode] # 8 children for octree
    wavelet_coeffs: list[float] # Detail coefficients

class Chunk:
    nodes: list[OctreeNode] # K*K*K nodes
    activated: bool
    needed_for_continuity: bool

def wavelet_transform(volume: Volume) -> OctreeNode:
    """
    Performs 3D wavelet transform using gathering strategy.
    Exploits separability of B-spline basis.
    """
    root = build_octree(volume)
    # Gather contributions: For each node, sum influences from wavelets
    # intersecting it. Sequentially process x, y, z dimensions.
    # This avoids scattering atomic operations.
    return root

def evaluate_error_metric(chunk: Chunk, camera_pos: Vec3, isovalue: float) -> bool:
    """
    Checks if chunk contains isovalue and satisfies view-dependent error.
    """
    # Check min/max bracketing
    min_v = min(node.min_val for node in chunk.nodes)
    max_v = max(node.max_val for node in chunk.nodes)
    if not (min_v <= isovalue <= max_v):
        return False
    
    # View-dependent check: s * c > epsilon
    # s: node diameter (approximated by chunk size or node size)
    # c: distance from camera to chunk center
    chunk_center = get_chunk_center(chunk)
    dist = distance(camera_pos, chunk_center)
    node_diameter = get_node_diameter(chunk)
    
    if node_diameter * dist > EPSILON:
        return True
    return False

def find_continuity_neighbors(activated_chunks: list[Chunk], all_chunks: list[Chunk]) -> list[Chunk]:
    """
    Identifies neighbors whose reconstruction is needed for continuity
    because wavelet support extends into them.
    """
    neighbors = []
    for chunk in activated_chunks:
        # Check spatial neighbors (6 faces, 12 edges, 8 corners)
        # If any wavelet in 'chunk' has support overlapping 'neighbor',
        # mark 'neighbor' for reconstruction.
        for neighbor in get_spatial_neighbors(chunk):
            if not neighbor.activated and has_wavelet_overlap(chunk, neighbor):
                neighbor.needed_for_continuity = True
                neighbors.append(neighbor)
    return neighbors

def reconstruct_chunk(chunk: Chunk, wavelet_coeffs: dict) -> VolumeChunk:
    """
    CUDA kernel logic executed on GPU.
    """
    # Load data into registers
    data = load_chunk_data(chunk, wavelet_coeffs)
    
    # 1D Convolution in X
    for thread in threads(K*K):
        # Thread computes 1D convolution along X
        result_x = convolve_1d(data, thread, axis=0)
    
    # 1D Convolution in Y
    for thread in threads(K*K):
        result_y = convolve_1d(result_x, thread, axis=1)
        
    # 1D Convolution in Z
    for thread in threads(K*K):
        result_z = convolve_1d(result_y, thread, axis=2)
        
    return result_z

def render_isosurface(root: OctreeNode, camera: Camera, isovalue: float):
    chunks = extract_chunks(root, K)
    activated_chunks = []
    
    for chunk in chunks:
        if evaluate_error_metric(chunk, camera.pos, isovalue):
            chunk.activated = True
            activated_chunks.append(chunk)
            
    continuity_chunks = find_continuity_neighbors(activated_chunks, chunks)
    
    # Combine activated and continuity chunks
    render_chunks = activated_chunks + continuity_chunks
    
    # Sort chunks by distance for painter's algorithm or depth buffering
    render_chunks.sort(key=lambda c: distance(camera.pos, get_chunk_center(c)))
    
    for chunk in render_chunks:
        if chunk.activated or chunk.needed_for_continuity:
            reconstructed = reconstruct_chunk(chunk, root.wavelet_coeffs)
            solve_isosurface(reconstructed, isovalue, D)
```

**Edge Cases and Pitfalls:**
*   **Continuity Cracks:** If neighbors are not reconstructed when wavelet support extends into them, cracks appear in the isosurface. The algorithm explicitly checks for this [S1].
*   **Atomic Operations:** Direct scattering of wavelet coefficients during transform causes bus locking on GPUs. The gathering approach is essential for performance [S1].
*   **Precision:** Root finding precision varies by technique. The algebraic surface degree is $3d$ [S1].

## Subtopics

### [Wavelet Transform](2285-wavelet-transform.md)
Provides sparse representation via dilated/translated basis functions, solving fixed resolution limits of Fourier transforms. Local support eliminates block-based processing.

### [Wavelet-based Adaptive Resolution Volume Rendering](1171-wavelet-based-adaptive-resolution-volume-rendering.md)
Focuses on adaptive resolution for volume rendering, potentially using different activation criteria or rendering techniques (e.g., ray casting) compared to isosurface extraction.

### [Hierarchical Wavelet Transform on Omnitrees](2216-hierarchical-wavelet-transform-on-omnitrees.md)
Uses Omnitrees for hierarchical wavelet representation, differing in the tree structure and traversal method compared to the standard octree used here.

### [L2-optimal wavelet splatting](2816-l2-optimal-wavelet-splatting.md)
Uses L2-optimal splatting for rendering, likely focusing on direct volume rendering or splatting primitives rather than isosurface root finding.

### [Point-wise Inverse Discrete Wavelet Transform (PW-IDWT) via Generalized Projection](3394-point-wise-inverse-discrete-wavelet-transform-pw-idwt-via-ge.md)
Focuses on a specific inverse transform method (PW-IDWT) which may be used for reconstruction but differs in the projection approach and computational model.

## Operating characteristics
*   **Time Complexity:** Reconstruction is performed per chunk. The gathering transform is $O(N)$ where $N$ is the number of voxels, but with a higher constant due to gathering logic. Reconstruction is $O(M)$ where $M$ is the number of activated chunks.
*   **Memory Complexity:** Requires storage for the octree, wavelet coefficients, and chunk data. The octree structure adds overhead compared to a flat grid.
*   **Performance:** Tested with datasets up to $512^3$ voxels and viewport sizes up to $1024 \times 768$ pixels, achieving interactive frame rates [S1]. Performance scales with the number of activated chunks and the complexity of the root finding algorithm.
*   **Strengths:** Interactive frame rates for large datasets; view-dependent LOD reduces computational load; continuous surfaces due to B-spline properties.
*   **Limitations:** Requires GPU implementation for optimal performance (CUDA); root finding can be computationally intensive; error metric tuning ($\varepsilon$) affects quality and performance.
*   **When to Use:** Interactive isosurface visualization of large volumetric datasets where view-dependent LOD is beneficial.
*   **When Not to Use:** When global volume rendering is required instead of isosurfaces; when CPU-only implementation is necessary.

## Background
Originated by Markus Steinberger and Markus Grabner from the Institute for Computer Graphics and Vision, Graz University of Technology, Austria [S1]. The method builds on the wavelet transform [Chu92] and polynomial spline wavelets [CDF92], leveraging the nested vector space properties of octrees [SDS96]. It addresses the increasing size of volumetric datasets from technologies like CT, where full-resolution rendering is infeasible [S1].

## Sources
[S1] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
