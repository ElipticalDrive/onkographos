# Block-based UV Parameterization

Also called: Block-based charting, Texture parametrization

## Overview
Block-based UV Parameterization is a method for generating coherent texture maps for 3D geometry compression, specifically designed to work with block-based TSDF (Truncated Signed Distance Function) representations. It solves the problem of efficiently encoding high-quality textures (e.g., 4096×4096 pixels) alongside compressed geometry without the computational overhead of tracking features over time or transmitting UV coordinates. Unlike methods that sacrifice spatial coherence for efficiency (e.g., Motion2Fusion) or require expensive temporal tracking for consistency, this approach inherently encourages spatio-temporal coherence by deriving UVs deterministically from the local block structure.

## How it works
The method operates on a per-block basis, where each occupied 3D block is treated as an independent unit for texture mapping. The process ensures that the receiver can reconstruct the UV coordinates without explicit transmission, relying on the deterministic nature of the block extraction and Morton ordering.

**Inputs:**
- Reconstructed TSDF block (volume data).
- Texture encoder $T_{enc}$ and decoder $T_{dec}$ blocks.

**Outputs:**
- Morton packed chart $M_x$ (2D texture representation).
- Encoded bitstream for the chart.

**Data Structures:**
- **TSDF Block:** A 3D grid of voxel values representing the signed distance field.
- **Mesh:** Extracted surface geometry from the TSDF block using Marching Cubes.
- **Morton Packed Chart ($M_x$):** A 2D texture atlas region where vertices and faces are ordered according to Morton codes derived from their 3D spatial positions.

**Steps:**
1. **TSDF Reconstruction:** Obtain the TSDF block $\mathbf{x}$ for the current spatial region.
2. **Mesh Extraction:** Apply the Marching Cubes algorithm to the TSDF block $\mathbf{x}$ to extract the surface mesh. Let this mesh be denoted as $\mathcal{M}$.
3. **Morton Ordering:** Assign a 3D Morton code to each vertex in $\mathcal{M}$ based on its spatial coordinates. Sort the vertices and faces of $\mathcal{M}$ according to these Morton codes.
4. **Chart Generation:** Map the sorted mesh $\mathcal{M}$ to a 2D texture chart $M_x$. The mapping preserves the Morton order, ensuring that spatially adjacent vertices in 3D are likely to be adjacent in the 2D chart.
5. **Encoding:** Encode the chart $M_x$ using the texture encoder $T_{enc}$. The resulting bitstream is transmitted alongside the geometry data.
6. **Decoding:** The receiver decodes the bitstream using $T_{dec}$ to reconstruct $M_x$. The receiver independently generates the same Morton codes from the decoded geometry (or TSDF) to interpret the UV coordinates correctly.

**Formulas:**
- **Morton Code:** For a vertex with 3D coordinates $(x, y, z)$, the Morton code $m$ is interleaved bits of $x, y, z$.
  $$ m = \text{interleave}(x, y, z) $$
- **Texture Mapping:** The UV coordinate $(u, v)$ for a vertex is determined by its rank in the sorted list of vertices based on $m$. If vertex $i$ is the $k$-th vertex in the Morton-sorted list, its UV is mapped to a position in the 2D chart corresponding to index $k$.

**Parameters:**
- **Texture Resolution:** Typically 4096×4096 pixels for high-quality compression [S1].
- **Block Size:** Not given in the sources.

**Edge Cases and Pitfalls:**
- **Degenerate Geometry:** If the TSDF block results in no surface (empty or fully inside/outside), no chart is generated.
- **Precision:** The Morton code generation must be consistent between encoder and decoder to ensure UV coherence.
- **Boundary Conditions:** Vertices on the boundary of the TSDF block must be handled consistently to avoid gaps or overlaps in the chart.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def block_based_uv_parameterization(tsdf_block, texture_encoder, texture_decoder):
    """
    Generates a Morton-packed texture chart from a TSDF block.
    
    Args:
        tsdf_block: 3D numpy array of TSDF values.
        texture_encoder: Function to encode 2D texture data.
        texture_decoder: Function to decode 2D texture data.
        
    Returns:
        encoded_chart: Encoded bitstream of the texture chart.
    """
    # Step 1: Extract mesh from TSDF block using Marching Cubes
    # vertices: Nx3 array of vertex coordinates
    # faces: Mx3 array of face indices
    vertices, faces = marching_cubes(tsdf_block)
    
    if len(vertices) == 0:
        return None  # No geometry in this block
    
    # Step 2: Compute Morton codes for each vertex
    # Morton code interleaves bits of x, y, z coordinates
    morton_codes = compute_morton_codes(vertices)
    
    # Step 3: Sort vertices and faces by Morton code
    # Create a permutation index that sorts vertices by morton_codes
    sort_indices = np.argsort(morton_codes)
    
    # Reorder vertices and faces
    sorted_vertices = vertices[sort_indices]
    # Remap face indices to the sorted vertex order
    sorted_faces = np.zeros_like(faces)
    for i in range(3):
        sorted_faces[:, i] = sort_indices[faces[:, i]]
        
    # Step 4: Generate 2D texture chart
    # Map sorted vertices to 2D UV coordinates
    # Assuming a simple rasterization or packing strategy based on order
    # For implementation, this might involve creating a 2D image where
    # each vertex's color/texture is placed at a position corresponding to its rank
    chart_width = 4096
    chart_height = 4096
    chart = np.zeros((chart_height, chart_width, 3), dtype=np.uint8)
    
    # Placeholder for actual texture mapping logic
    # In practice, this step involves projecting the 3D mesh to 2D
    # while preserving the Morton order for coherence.
    # This is a simplified representation.
    chart = map_mesh_to_chart(sorted_vertices, sorted_faces, chart)
    
    # Step 5: Encode the chart
    encoded_chart = texture_encoder(chart)
    
    return encoded_chart

def compute_morton_codes(vertices):
    """
    Computes Morton codes for a set of 3D vertices.
    """
    # Normalize vertices to integer range if necessary
    # Interleave bits of x, y, z
    codes = []
    for v in vertices:
        x, y, z = v
        # Simplified Morton code calculation
        code = 0
        for i in range(16):  # Assuming 16-bit precision
            code |= ((x >> i) & 1) << (3 * i)
            code |= ((y >> i) & 1) << (3 * i + 1)
            code |= ((z >> i) & 1) << (3 * i + 2)
        codes.append(code)
    return np.array(codes)

def map_mesh_to_chart(vertices, faces, chart):
    """
    Maps a sorted mesh to a 2D chart.
    This is a placeholder for the actual texture mapping algorithm.
    """
    # In a real implementation, this would involve:
    # 1. Projecting vertices to 2D UV space.
    # 2. Ensuring that the Morton order is preserved in the UV layout.
    # 3. Filling the chart with texture data (e.g., vertex colors, normals).
    # For this article, we assume a deterministic mapping exists.
    return chart
```

## Subtopics

### [Block-level Charting](8424-block-level-charting.md)
A UV parametrization method for texture compression that creates one 2D chart per occupied 3D block, ensuring deterministic UV coordinates inferred on the receiver side without transmitting UVs.

## Operating characteristics
- **Time Complexity:** The method is fast because it avoids temporal tracking and complex global optimization. The primary cost is the Marching Cubes extraction and Morton code computation, which are linear in the number of vertices.
- **Memory Complexity:** Memory usage is proportional to the size of the TSDF block and the resulting mesh. The texture chart size is fixed (e.g., 4096×4096 pixels).
- **Performance:** Enables efficient compression of textures using existing image-based techniques (e.g., video codecs) [S1].
- **Strengths:** Inherently encourages temporal consistency without tracking; removes the need for UV coordinate compression and streaming; maintains spatial coherence better than arbitrary mapping methods.
- **Limitations:** May not achieve the same level of temporal consistency as methods that explicitly track features over time, though it avoids their high computational cost.
- **When to use:** Suitable for real-time applications where computational efficiency is critical, and high-quality texture compression is needed alongside geometry compression.
- **When not to use:** Not suitable for applications requiring the highest possible temporal consistency if the computational cost of tracking is acceptable.

## Background
Block-based UV Parameterization was introduced in the context of deep 3D compression methods, specifically in the paper "Deep Implicit Volume Compression" [S1]. It addresses the challenge of compressing high-quality texture maps alongside TSDF geometry. The method was developed to complement TSDF compression algorithms by providing a fast, coherent texture parametrization that works well with existing image compression standards. It is part of a broader effort to create end-to-end learnable compression models for 3D data.

## Sources
[S1] Deep Implicit Volume Compression, https://doi.org/10.1109/cvpr42600.2020.00137, fetched 2026-10-26
