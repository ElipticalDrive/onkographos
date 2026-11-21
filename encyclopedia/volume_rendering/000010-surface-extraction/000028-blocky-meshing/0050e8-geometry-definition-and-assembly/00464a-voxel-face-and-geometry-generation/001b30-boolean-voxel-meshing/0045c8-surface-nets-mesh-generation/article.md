# Surface Nets Mesh Generation

Also called: Marching Cubes Variant, Signed Distance Field Meshing via Surface Nets, Smooth Mesh from Density Field

## Overview
Surface Nets generates a smooth triangle mesh from a binary density field or Signed Distance Field (SDF) by computing vertex positions at voxel centers based on local gradients, producing significantly fewer vertices and smoother geometry than blocky voxel-based methods [S1, S2, unofficial]. It serves as a bridge between raw voxel occupancy and high-fidelity surface extraction, offering a computationally efficient alternative to Marching Cubes when the input data is already structured as a grid [S1, unofficial].

## How it works
**Inputs:**
1.  A 3D grid of scalar values (density field $d$ or SDF).
2.  An iso-value threshold (typically $0.5$ for binary density, $0.0$ for SDF) [S1, S2, unofficial].
3.  Optional bounding box definition (`bounding_box.lo`) for world-space translation [S1, unofficial].
4.  Optional material data and chunk dimensions for post-processing [S2, unofficial].

**Outputs:**
1.  A triangular mesh consisting of vertex positions, indices, and optionally normals [S1, S2, unofficial].
2.  Post-processed mesh attributes including material blending, face culling, and collider octants if required [S2, unofficial].

**Data Structures:**
*   **Density Field:** A 3D array or texture where each cell $(i, j, k)$ holds a scalar value.
*   **Mesh:** Standard triangle list format with vertex buffers (position, normal) and index buffers.

**Algorithm Steps:**

1.  **Initialize Mesh Generation:**
    Call the core generation routine with the density field and iso-value.
    $$ \text{mesh} = \text{SurfaceNets::GenerateMesh}(d, \text{iso\_value}) $$
    Typical iso-value is $0.5$ for binary fields [S1, unofficial].

2.  **Vertex Position Calculation (Core Logic):**
    For each voxel in the grid, determine the surface intersection. Unlike Marching Cubes which interpolates along edges, Surface Nets typically computes the vertex position at the center of the voxel or cell by averaging the positions of neighboring voxels weighted by their distance to the iso-surface.
    *Note: The specific gradient-based weighting formula is not explicitly detailed in the provided sources, but the method is defined as generating a "smooth triangle mesh" from the density field [S1, unofficial].*

3.  **World Space Translation:**
    Translate all generated mesh vertices to world space by adding the lower corner of the bounding box to each vertex position.
    $$ v_{world} = v_{local} + \text{bounding\_box.lo} $$
    This ensures the mesh is correctly positioned relative to the scene origin [S1, unofficial].

4.  **Post-Processing (Optional but Common):**
    If the mesh contains indices, apply post-processing to handle material blending and collision detection.
    $$ \text{prepMesh}(\text{mesh}, \text{chunk\_coords}) $$
    This step handles:
    *   Material blending based on adjacent voxel materials [S2, unofficial].
    *   Face culling to remove internal faces [S2, unofficial].
    *   Bounds calculation [S2, unofficial].
    *   Generation of collider octants for physics engines [S2, unofficial].

5.  **Data Transfer:**
    Transfer the generated buffers (vertices, indices, normals) and any auxiliary data (SDF, material buffers, octant buffers) back to the main thread or rendering pipeline [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_surface_nets_mesh(density_field, iso_value=0.5, bounding_box_lo=None, material_data=None):
    """
    Generates a smooth triangle mesh from a density field using Surface Nets.
    
    Args:
        density_field: 3D array of scalar values.
        iso_value: Threshold for surface extraction (default 0.5).
        bounding_box_lo: 3D vector for world-space translation.
        material_data: Optional material information for post-processing.
        
    Returns:
        mesh: A mesh object containing vertices, indices, and attributes.
    """
    # Step 1: Core mesh generation
    # The specific vertex computation logic (gradient averaging) is implementation-specific.
    # We assume a standard Surface Nets implementation that computes vertex positions
    # based on the density field and iso-value.
    mesh = surface_nets_core_generate(density_field, iso_value)
    
    # Step 2: World space translation
    if bounding_box_lo is not None:
        for vertex in mesh.vertices:
            vertex.position += bounding_box_lo
            
    # Step 3: Post-processing (if indices and material data are present)
    if mesh.has_indices() and material_data is not None:
        # PrepMesh handles material blending, face culling, bounds, and collider octants
        prep_mesh(mesh, material_data)
        
    return mesh

def surface_nets_core_generate(density_field, iso_value):
    """
    Placeholder for the core Surface Nets algorithm.
    In a full implementation, this would:
    1. Iterate over each voxel.
    2. Compute the surface vertex position based on local density gradients.
    3. Connect vertices to form triangles.
    """
    # Implementation detail: Not fully specified in sources, but defined as 
    # generating a smooth mesh from density field [S1, unofficial].
    pass

def prep_mesh(mesh, material_data):
    """
    Post-processes the mesh for material blending and collision.
    """
    # Handles material blending, face culling, bounds calculation, and collider octants [S2, unofficial].
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in density field sampling can affect vertex placement accuracy [S1, unofficial].
*   **Boundaries:** Ensure the density field is padded or handled correctly at grid boundaries to avoid artifacts [S2, unofficial].
*   **Degenerate Input:** Empty or fully occupied fields should return empty meshes or full volumes appropriately [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels $N$, $O(N)$, as each voxel is processed once to determine vertex positions [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the number of vertices and triangles generated, which is typically less than voxel-based methods [S1, unofficial].
*   **Performance:** Produces smoother meshes with fewer vertices compared to blocky voxel-based mesh generation, leading to lower rendering and physics overhead [S1, unofficial].
*   **Strengths:** Efficiency, smoothness, and compatibility with SDF data [S1, S2, unofficial].
*   **Limitations:** Less geometric detail than Marching Cubes in complex topologies; requires well-behaved density fields [S1, unofficial].
*   **When to Use:** When a smooth, low-poly mesh is needed from a voxel or SDF input, and performance is critical [S1, S2, unofficial].
*   **When Not to Use:** When high-fidelity geometric detail is required, or when the input data is not a structured grid [S1, unofficial].

## Background
Surface Nets was introduced as an alternative to Marching Cubes for surface extraction from volumetric data. It is particularly useful in procedural generation and voxel-based games where smooth geometry is desired without the computational cost of Marching Cubes [S1, S2, unofficial]. It has been implemented in various open-source projects for floating island generation and voxel-based multiplayer games [S1, S2, unofficial].

## Sources
[S1] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
[S2] sh1ftmaker/voxel-wars, https://github.com/sh1ftmaker/voxel-wars, fetched 2026-10-28 (unofficial)
