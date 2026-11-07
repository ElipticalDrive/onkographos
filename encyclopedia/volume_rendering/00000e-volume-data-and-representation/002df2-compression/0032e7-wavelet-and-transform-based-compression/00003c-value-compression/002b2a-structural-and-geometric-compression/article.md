# Structural and Geometric Compression

Also called: Data Structure and Compression

## Overview
Structural and Geometric Compression is a lossless or lossy compression method for volumetric data that exploits the spatial connectivity and geometric regularity of the underlying structures, such as isosurfaces or particle boundaries, rather than relying solely on statistical redundancy in scalar values. It sets itself apart from sibling methods like [Value compression](6-value-compression.md) by encoding the topological relationships (e.g., surface links) and geometric attributes (e.g., normals) directly, allowing for efficient on-the-fly rendering without full volume reconstruction. Unlike [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) which operates in the frequency domain, this method operates in the spatial domain, preserving exact geometric boundaries.

## How it works
The method encodes volumetric data by storing depth values, surface connectivity links, and surface normals in a compact, structured format. It is particularly effective for particle-based or isosurface data where the surface geometry is the primary rendering target.

### Inputs and Outputs
- **Input:** A voxelized representation of a volume, typically derived from particle data or scalar fields, containing depth values, surface normals, and connectivity information.
- **Output:** A compressed bitstream or memory structure containing packed depth, link, and normal data.

### Data Structures and Layout
The data is organized per voxel column (pixel). For each voxel, three 32-bit words are stored:

1.  **Depth Value:** Stored in 32-bit floating-point format ($f32$). This represents the distance to the surface along the viewing ray or voxel axis [S1].
2.  **Surface Links:** A 32-bit word storing links to the four direct neighbors (up, down, left, right) of the current depth value. Each neighbor link is stored in 1 byte ($u8$), totaling 4 bytes. These links allow traversal of the surface connectivity without storing a full adjacency graph [S1].
3.  **Normals:** A 32-bit word storing the surface normal vector. This is compressed using octahedron normal encoding, which maps the 3D normal $(x, y, z)$ to two 16-bit normalized integers ($u16, u16$). This encoding projects the normal onto an octahedron, reducing the storage from 96 bits (three $f32$) to 32 bits [S1].

### Numbered Steps
1.  **Voxelization and Depth Extraction:** For each voxel column, identify the surface intersection. Store the depth value $d$ as a 32-bit float [S1].
2.  **Surface Link Generation:** Determine the four direct spatial neighbors of the current voxel. For each neighbor, if a surface connection exists, store a reference or index in 1 byte. If no connection exists, store a null value. This creates a 32-bit word of links [S1].
3.  **Normal Computation:** Compute the surface normal $\mathbf{n} = (n_x, n_y, n_z)$ at the surface point. This can be done using a Sobel operator on the depth values to estimate derivatives in two directions [S1].
4.  **Normal Encoding:** Apply octahedron encoding to $\mathbf{n}$. Let $n_{max} = \max(|n_x|, |n_y|, |n_z|)$. Normalize the vector by $n_{max}$ and map the resulting coordinates to two 16-bit integers. Store these two integers in the 32-bit normal word [S1].
5.  **Parallel Storage:** Store the depth, links, and normal words in parallel for each voxel column to allow easy access during rendering [S1].

### Formulas
- **Octahedron Normal Encoding:**
  Given a normalized normal vector $\mathbf{n} = (n_x, n_y, n_z)$, let $n_{max} = \max(|n_x|, |n_y|, |n_z|)$.
  The encoded coordinates $(u, v)$ are derived from:
  $$ u = \text{encode}(n_x / n_{max}) $$
  $$ v = \text{encode}(n_y / n_{max}) $$
  where $\text{encode}(x)$ maps a value in $[-1, 1]$ to a 16-bit integer range $[0, 65535]$. The sign of $n_z$ determines the quadrant [S1].

- **Smoothing (Optional Pre-processing):**
  To smooth the surface, iteratively apply a $3 \times 3$ kernel to the reciprocal of the depth values $1/d$. The number of iterations $k$ depends on the distance of the surface point to the camera $D$:
  $$ k \propto D $$
  This ensures a constant world-space radius for the filter [S1].

### Parameters
- **Depth Precision:** 32-bit float [S1].
- **Link Size:** 1 byte per neighbor [S1].
- **Normal Precision:** 16-bit integer per component [S1].
- **Smoothing Kernel:** $3 \times 3$ [S1].

### Edge Cases and Pitfalls
- **Precision Loss:** Octahedron encoding introduces quantization error in normals. For high-precision rendering, this may be noticeable [S1].
- **Boundary Conditions:** Links at the volume boundary must be handled carefully to avoid out-of-bounds access. Null links should be used for non-existent neighbors [S1].
- **Coherency:** In parallel implementations, data coherency (shared vertices) can be disregarded as it has a small impact on quality, but data balancing is critical [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_voxel_column(depth_value, neighbors, normal_vector):
    """
    Compresses a single voxel column's data.
    
    Args:
        depth_value (float): 32-bit float depth to surface.
        neighbors (list of 4 bools): True if surface link exists for each neighbor.
        normal_vector (tuple of 3 floats): Normal (nx, ny, nz).
        
    Returns:
        tuple: (depth_word, link_word, normal_word)
    """
    # 1. Depth is stored directly as 32-bit float
    depth_word = depth_value
    
    # 2. Links are stored as 1 byte per neighbor
    # Assuming neighbors are ordered: up, down, left, right
    link_word = 0
    for i, has_link in enumerate(neighbors):
        if has_link:
            # Store a simple flag or index. Here we assume a flag for simplicity.
            # In practice, this might be an offset or neighbor ID.
            link_word |= (1 << (i * 8))
        else:
            link_word |= (0 << (i * 8))
            
    # 3. Normal encoding using octahedron projection
    nx, ny, nz = normal_vector
    n_max = max(abs(nx), abs(ny), abs(nz))
    
    # Avoid division by zero
    if n_max == 0:
        n_max = 1e-9
        
    # Normalize
    nx_norm = nx / n_max
    ny_norm = ny / n_max
    
    # Map to 16-bit integers
    # Assuming a simple linear mapping from [-1, 1] to [0, 65535]
    u = int((nx_norm + 1) * 32767)
    v = int((ny_norm + 1) * 32767)
    
    # Pack into 32-bit word
    normal_word = (u << 16) | v
    
    return (depth_word, link_word, normal_word)

def decompress_normal(normal_word):
    """
    Decompresses a normal from a 32-bit word.
    
    Args:
        normal_word (int): 32-bit packed normal.
        
    Returns:
        tuple: (nx, ny, nz) normalized normal vector.
    """
    u = (normal_word >> 16) & 0xFFFF
    v = normal_word & 0xFFFF
    
    # Map back to [-1, 1]
    nx_norm = (u / 32767.0) - 1.0
    ny_norm = (v / 32767.0) - 1.0
    
    # Reconstruct nz from octahedron geometry
    # |nx| + |ny| <= 1 for the octahedron surface
    # nz = 1 - |nx| - |ny| (for the upper hemisphere, sign depends on original nz)
    # Since we lost the sign of nz in the encoding, we assume the upper hemisphere
    # or store the sign separately if needed. For this method, we assume standard octahedron mapping.
    nz_norm = 1.0 - abs(nx_norm) - abs(ny_norm)
    
    # Normalize to unit length
    length = sqrt(nx_norm**2 + ny_norm**2 + nz_norm**2)
    nx = nx_norm / length
    ny = ny_norm / length
    nz = nz_norm / length
    
    return (nx, ny, nz)
```

## Subtopics

### Parallel Edgebreaker
A parallel compression algorithm for isosurfaces where each part of the mesh is encoded independently from the others, based on the Edgebreaker algorithm [S2]. Unlike sequential compression, it allows for concurrent processing of mesh parts. It is chosen when high throughput is required and slight increases in file size are acceptable. Data balancing is critical because the cost to process each triangle is constant time, whereas data coherency can be disregarded as it has a small impact on quality [S2].

## Operating characteristics
- **Time Complexity:** Encoding is linear with respect to the number of voxels or triangles. Parallel versions reduce wall-clock time significantly but may increase total CPU time due to overhead.
- **Memory Complexity:** Storage is compact: 96 bits per voxel column (32 for depth, 32 for links, 32 for normals) [S1].
- **Measured Performance:** For dataset XT-52 (374 MB uncompressed), sequential compression reduced size to 31 MB, and parallel compression (16 processes) reduced it to 32 MB [S2]. Transmission over a 1.14 Mbps network took 44min:37sec for uncompressed, 17min:36sec for sequential, and 4min:31sec for parallel [S2].
- **Strengths:** Efficient on-the-fly rendering, compact storage, parallelizable.
- **Limitations:** Octahedron encoding introduces normal quantization error. Parallel compression increases file size by almost 10% compared to sequential [S2].

## Background
Structural and Geometric Compression originates from the need to efficiently store and render complex volumetric data, particularly particle-based simulations and isosurfaces. It was developed to address the limitations of storing full scalar fields, which are often sparse or redundant. The method leverages the geometric structure of the data, storing only the surface information and its connectivity. Key papers include "Memory-Efficient On-The-Fly Voxelization of Particle Data" [S1] and "Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments" [S2].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
[S2] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
