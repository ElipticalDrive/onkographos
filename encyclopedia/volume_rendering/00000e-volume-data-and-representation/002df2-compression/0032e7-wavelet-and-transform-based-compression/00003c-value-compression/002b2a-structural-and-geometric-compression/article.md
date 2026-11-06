# Structural and Geometric Compression

Also called: Data Structure and Compression

## Overview
Structural and Geometric Compression is a method for compactly storing surface-based volume data by encoding depth values, surface connectivity, and surface normals in a structured data structure [S1]. It solves the problem of high memory consumption in voxel-based rendering by replacing dense voxel grids with sparse surface representations [S1]. It differs from general value compression (like wavelets) by focusing on the geometric topology of the surface rather than the statistical distribution of scalar values, storing explicit links to neighbors to enable on-the-fly connectivity computation during rendering [S1].

## How it works
**Inputs:** A voxelized representation of a volume (e.g., particle spheres) where surface voxels are identified.
**Outputs:** A compressed bitstream or memory structure containing depth values, surface links, and normals.

**Data Structures:**
The method organizes data per voxel column (pixel). For each column, it stores:
1.  **Depth Values:** Stored in 32-bit floating-point format ($f32$) [S1].
2.  **Surface Links:** Links to the four direct neighbors (up, down, left, right) for each depth value. Each link is stored in 1 byte ($u8$), totaling 4 bytes ($32$ bits) per depth entry [S1].
3.  **Normals:** Compressed to a 32-bit word using octahedron normal encoding. This consists of two 16-bit normalized integers ($u16$) [S1].

**Steps:**
1.  **Voxelization and Surface Identification:** Identify voxels that overlap the surface of the volume primitives (e.g., particle spheres) [S1].
2.  **Depth Extraction:** Extract depth values for the surface entries in each voxel column [S1].
3.  **Surface Smoothing (Optional):** To obtain a smooth surface from voxelized primitives, iteratively apply a $3 \times 3$ smoothing kernel to the reciprocal of the extracted depth values. The number of iterations depends on the distance of the surface point to the camera to maintain a constant world-space filter radius [S1].
4.  **Normal Estimation:** Generate normals using the Sobel operator to estimate derivatives in two directions [S1].
5.  **Link Storage:** Store links to the four direct neighbors for each depth value. Each link is 1 byte [S1].
6.  **Normal Encoding:** Compress the estimated normals into a 32-bit word using octahedron encoding with two 16-bit integers [S1].
7.  **Parallel Storage:** Store normals in parallel to the surface depth array to allow easy access during rendering [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for data structure layout and storage
# Note: This describes the storage format, not the full rendering pipeline

def compress_surface_data(depth_values, normals):
    """
    Compresses surface data into the structural format.
    
    Args:
        depth_values: List of 32-bit floats representing surface depths per column.
        normals: List of 3D vectors representing surface normals.
        
    Returns:
        A structured object containing compressed depth, links, and normals.
    """
    compressed_depths = []
    compressed_normals = []
    compressed_links = []
    
    for i, depth in enumerate(depth_values):
        # Store depth as 32-bit float
        compressed_depths.append(depth)
        
        # Store links to 4 neighbors (1 byte each)
        # In a real implementation, neighbor indices would be computed based on grid position
        # Here we assume placeholder logic for neighbor indices as specific addressing
        # is not detailed in the source beyond "four direct neighbors"
        links = [0, 0, 0, 0] # Placeholder for neighbor indices
        compressed_links.append(links)
        
        # Compress normal using octahedron encoding
        # Octahedron encoding maps a 3D normal (nx, ny, nz) to two 16-bit integers
        # This is a standard technique [MSS*10]
        compressed_normal = octahedron_encode(normal)
        compressed_normals.append(compressed_normal)
        
    return {
        "depths": compressed_depths,
        "links": compressed_links,
        "normals": compressed_normals
    }

def octahedron_encode(normal):
    """
    Encodes a 3D normal vector into two 16-bit integers using octahedron encoding.
    This is a standard technique referenced in [S1].
    """
    # Implementation details of octahedron encoding are not provided in the source.
    # Standard implementation involves projecting the normal onto an octahedron
    # and quantizing the resulting 2D coordinates.
    pass
```

**Parameters:**
*   **Smoothing Kernel Size:** $3 \times 3$ [S1].
*   **Depth Precision:** 32-bit float [S1].
*   **Link Size:** 1 byte per neighbor [S1].
*   **Normal Precision:** Two 16-bit integers [S1].

**Edge Cases and Pitfalls:**
*   **Surface Connectivity:** The method relies on surface links to compute connectivity on the fly. If the surface is highly fragmented or disconnected, the link storage may not accurately represent the geometry without additional handling [S1].
*   **Smoothing Artifacts:** Iterative smoothing may blur fine surface details if the number of iterations is not carefully controlled based on camera distance [S1].

## Subtopics

### Fractal Compression
Fractal compression is a variant that relies on self-similarity within the volume [S2]. It works by finding a larger "domain block" in the volume that is most similar to a smaller "range block" and storing only the affine transformations needed to convert the domain block to the range block [S2]. Decompression is performed by iteratively applying these transformations until the volume converges [S2]. If no near-matches are found, blocks are subdivided and the search is repeated [S2]. This method is extremely computationally intensive during compression and problematic for real-time rendering due to iterative reconstruction, making it suitable primarily for off-line volume compression [S2].

## Operating characteristics
*   **Memory Efficiency:** Significantly reduces memory usage compared to dense voxel grids by storing only surface information [S1].
*   **Rendering Speed:** Enables efficient rendering by allowing direct access to surface depth and normals per pixel column [S1].
*   **Compression Ratio:** Depends on the complexity of the surface and the effectiveness of the normal encoding [S1].
*   **Computational Cost:** The smoothing step adds preprocessing cost [S1]. Fractal compression variants are extremely computationally intensive [S2].
*   **Limitations:** Not suitable for real-time rendering if fractal decomposition is used [S2]. The method assumes a surface-based representation, making it less effective for fully volumetric data without surface extraction [S1].

## Background
The structural and geometric compression approach described in [S1] is based on memory-efficient on-the-fly voxelization of particle data [S1]. It builds on techniques for storing surface links and normals in a compact format [S1]. Fractal compression for volumes was proposed by [CHF96] as cited in [S2].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
[S2] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
