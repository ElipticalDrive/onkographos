# Texture Atlas UV Mapping

Also called: Per-face lookup table UV mapping

## Overview
This method maps block types to specific sub-regions of a shared texture atlas using a per-face lookup table, ensuring correct UV coordinates for voxel face meshes [S1, unofficial]. It differs from its parent [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) by explicitly handling the UV coordinate generation and atlas indexing logic, while its sibling [Face Direction and Quad Geometry](4881-face-direction-and-quad-geometry.md) focuses on vertex position and normal calculation. The primary advantage is the elimination of redundant texture lookups by sharing a single atlas across all block types, though it requires careful handling of UV inversion to maintain correct winding order.

## How it works
**Inputs and Outputs:**
*   **Input:** Block type identifier (integer or enum), face direction (normal vector or axis), and the global texture atlas dimensions.
*   **Output:** UV coordinates ($u, v$) for the quad vertices, and the material index or sampler state associated with the block type.

**Data Structures:**
*   **Lookup Table:** A 2D array or hash map indexed by `block_type` and `face_direction`. Each entry contains:
    *   `atlas_u`: The starting U coordinate in the atlas.
    *   `atlas_v`: The starting V coordinate in the atlas.
    *   `width`: The width of the sub-region in atlas units.
    *   `height`: The height of the sub-region in atlas units.
    *   `invert_u`: Boolean flag indicating if the U coordinate must be inverted for correct winding.
    *   `invert_v`: Boolean flag indicating if the V coordinate must be inverted for correct winding.

**Steps:**
1.  **Retrieve Block Type:** Identify the block type for the current voxel face.
2.  **Identify Face Direction:** Determine the normal direction of the face (e.g., +X, -Y, +Z).
3.  **Lookup UV Parameters:** Access the per-face lookup table using the block type and face direction to retrieve `atlas_u`, `atlas_v`, `width`, `height`, and inversion flags.
4.  **Calculate Base UVs:** Compute the base UV coordinates for the quad. Assuming a standard quad with vertices $(0,0), (1,0), (1,1), (0,1)$ in local space:
    $$ u_{base} = atlas\_u + vertex\_x \times width $$
    $$ v_{base} = atlas\_v + vertex\_y \times height $$
5.  **Apply Inversion:** If `invert_u` is true, set $u_{final} = atlas\_u + atlas\_v + width - u_{base} + atlas\_u$ (or simply flip the vertex order in the index buffer). If `invert_v` is true, apply similar logic to $v$. [S1, unofficial]
6.  **Assign Material:** Assign the material instance associated with the block type to the mesh data. Note that a single material instance is applied to all 16 sections of a chunk [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_uv(block_type, face_direction, atlas_width, atlas_height):
    # Retrieve parameters from the per-face lookup table
    # The lookup table is pre-populated based on block definitions
    params = lookup_table[block_type][face_direction]
    
    atlas_u = params['atlas_u']
    atlas_v = params['atlas_v']
    width = params['width']
    height = params['height']
    invert_u = params['invert_u']
    invert_v = params['invert_v']
    
    # Define local quad vertices (0,0 to 1,1)
    # Vertex 0: (0,0), Vertex 1: (1,0), Vertex 2: (1,1), Vertex 3: (0,1)
    local_vertices = [
        (0.0, 0.0),
        (1.0, 0.0),
        (1.0, 1.0),
        (0.0, 1.0)
    ]
    
    uv_coords = []
    for lx, ly in local_vertices:
        u = atlas_u + lx * width
        v = atlas_v + ly * height
        
        if invert_u:
            u = atlas_u + width - (lx * width)
        if invert_v:
            v = atlas_v + height - (ly * height)
            
        uv_coords.append((u, v))
        
    return uv_coords, params['material_index']
```

**Parameters:**
*   **Atlas Size:** Not given in the sources.
*   **Block Type Count:** Not given in the sources.
*   **Face Directions:** Typically 6 (±X, ±Y, ±Z).

**Edge Cases and Pitfalls:**
*   **Winding Order:** Incorrect UV inversion leads to flipped textures. The method explicitly requires inversion for certain face directions to maintain consistent winding [S1, unofficial].
*   **Atlas Bounds:** UV coordinates must not exceed the atlas dimensions.
*   **Material Assignment:** A single material instance is applied to all 16 sections of a chunk, which may limit per-block material variation [S1, unofficial].

## Subtopics

### [Material Sampling](6218-material-sampling.md)
Adds block-type-specific logic such as grass top/side mixing and leaf tinting, extending the basic UV mapping with semantic texturing rules.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per face for UV lookup and calculation.
*   **Memory Complexity:** $O(B \times F)$ for the lookup table, where $B$ is the number of block types and $F$ is the number of face directions.
*   **Performance:** Efficient due to shared atlas and simple arithmetic. No dynamic texture generation required.
*   **Strengths:** Reduces texture memory usage, simplifies shader logic by using a single atlas.
*   **Limitations:** Requires careful atlas layout management. Limited flexibility for complex per-block materials without additional logic.
*   **When to Use:** Suitable for voxel-based games with a fixed set of block types and a need for efficient rendering.
*   **When Not to Use:** When dynamic texture generation or highly varied per-block materials are required.

## Background
This method is part of the voxel meshing pipeline in Unreal Engine 5 Minecraft clones, as implemented in the [Manolo94/UE5MinecraftClone](https://github.com/Manolo94/UE5MinecraftClone) project [S1, unofficial]. It builds on the principles of texture atlasing and per-face geometry generation to optimize rendering performance.

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
