# VoxelGrid Preview Rendering

Also called: Debug Voxel Rendering

## Overview
VoxelGrid Preview Rendering is a visualization technique that converts a scalar field or voxel grid into explicit geometric primitives (cubes, lines, or points) to be rasterized by the graphics pipeline. It serves as a diagnostic and debugging tool, allowing developers to inspect the underlying data structure, density distribution, and spatial bounds of a volume without performing expensive ray-marching or ray-casting integration [S1, S3, unofficial]. Unlike final production rendering methods that approximate lighting and transparency through accumulation, this method prioritizes structural clarity, often using false colors, wireframes, and gizmos to highlight occupancy and boundaries [S1, S3, unofficial].

## How it works
The method converts discrete voxel data into visible geometry. It operates in two primary modes: solid mesh rendering for occupancy visualization and wireframe/gizmo rendering for structural inspection.

### Inputs and Outputs
*   **Input**: A voxel grid data structure (dense or sparse), typically defined by a boundary box ($P_{min}, P_{max}$) and dimensions ($N_{dim}$). It may also include density values per voxel and metadata such as selection states or iso-levels.
*   **Output**: Rasterized primitives (triangles for meshes, lines for edges, points/spheres for density) on the screen.

### Data Structures
The implementation relies on lightweight coordinate structures to manage voxel positions. A common representation is a 3D integer point structure (`Point3i`) using 16-bit signed integers (`short`) for X, Y, and Z components, occupying 12 bytes total [S1, unofficial]. This allows for a coordinate range of -32767 to 32767.

### Steps

1.  **Mesh Cache Construction**:
    *   Iterate through the voxel grid.
    *   Identify occupied voxels.
    *   Generate geometric primitives (typically cubes) for each occupied voxel.
    *   Store these primitives in a mesh cache [S1, unofficial].
    *   *Note*: If the grid is sparse, only generate meshes for non-empty cells.

2.  **Solid Rendering (False Colors)**:
    *   Draw the mesh cache using a shader that applies "false colors" (e.g., uniform color or simple lighting) rather than physically based shading [S1, unofficial].
    *   Optionally, add a fake shadow to the mesh to provide depth cues [S1, unofficial].

3.  **Wireframe and Line Rendering**:
    *   Ensure the line cache is built, generating preview lines for the grid boundaries and internal structures [S1, unofficial].
    *   Determine the wire color based on the selection state of the voxel or object [S1, unofficial].
    *   Draw the solid line cache.
    *   Draw the dotted line cache (if applicable for grid spacing) [S1, unofficial].
    *   Draw the mesh cache again in wireframe mode to overlay edges on the solid faces [S1, unofficial].

4.  **Gizmo and Density Visualization (Optional)**:
    *   Iterate through the 3D grid using indices $i, j, k$ up to `numberOfGeneratedMeshObject.x/y/z` [S3, unofficial].
    *   Calculate the world offset for the current voxel based on `boundaryWorldPos` [S3, unofficial].
    *   Draw a wireframe cube centered at `centre + offset` with size `centre * 2` [S3, unofficial].
    *   If `_pointsDensity` is valid and `showGizmos` is true:
        *   If `showVoxelDensity` is true: Calculate the index `id` for the current voxel grid position and draw two spheres representing voxel corners [S3, unofficial].
        *   If `showVoxelDensity` is false: Iterate through `_pointsDensity` entries. If the density value (`.w`) $\ge$ `isoLevel`, draw a sphere at the point's position (`.x, .y, .z`) with radius `_voxel.size/8` and color based on density [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxel_grid_preview(voxel_grid, camera_view, show_wireframe=True, show_density=False):
    """
    Renders a voxel grid as a preview/debug visualization.
    """
    # 1. Build Mesh Cache if not already built
    if not mesh_cache_built:
        mesh_cache = []
        for i in range(voxel_grid.dim_x):
            for j in range(voxel_grid.dim_y):
                for k in range(voxel_grid.dim_z):
                    if voxel_grid.is_occupied(i, j, k):
                        # Generate cube geometry for this voxel
                        cube = generate_cube_geometry(i, j, k, voxel_grid.voxel_size)
                        mesh_cache.append(cube)
        mesh_cache_built = True

    # 2. Draw Solid Mesh with False Colors
    for mesh in mesh_cache:
        # Apply false color shader (e.g., uniform red or simple diffuse)
        draw_mesh(mesh, shader="false_color")
        # Optional: Add fake shadow for depth perception
        draw_shadow(mesh)

    # 3. Draw Wireframes and Lines
    if show_wireframe:
        # Determine wire color based on selection state
        wire_color = get_selection_color()
        
        # Draw solid line cache (grid boundaries/edges)
        draw_lines(line_cache_solid, color=wire_color)
        
        # Draw dotted line cache (internal grid lines)
        draw_lines(line_cache_dotted, color=wire_color)
        
        # Draw mesh cache in wireframe mode
        for mesh in mesh_cache:
            draw_mesh_wireframe(mesh, color=wire_color)

    # 4. Draw Gizmos and Density Indicators
    if show_density or show_gizmos:
        for i in range(number_of_generated_mesh_object.x):
            for j in range(number_of_generated_mesh_object.y):
                for k in range(number_of_generated_mesh_object.z):
                    # Calculate world position
                    offset = boundary_world_pos + Vector3(i, j, k) * voxel_grid.voxel_size
                    center = offset
                    
                    # Draw wireframe cube for structure
                    draw_wireframe_cube(center, size=voxel_grid.voxel_size * 2)
                    
                    if show_gizmos and points_density_valid:
                        if show_voxel_density:
                            # Draw spheres at voxel corners
                            id = get_voxel_grid_index(i, j, k)
                            draw_sphere(corner_positions[id], radius=small_radius)
                        else:
                            # Check density threshold
                            density_val = get_density_at(i, j, k)
                            if density_val >= iso_level:
                                draw_sphere(
                                    position=points_density[i, j, k].xyz,
                                    radius=voxel_grid.size / 8,
                                    color=density_to_color(density_val)
                                )
```

### Parameters and Edge Cases
*   **IsoLevel**: The threshold for density visualization. Values below this are not drawn in density mode [S3, unofficial].
*   **Voxel Size**: Determines the radius of density spheres (`size/8`) and the scale of wireframe cubes [S3, unofficial].
*   **Selection State**: Influences wire color, allowing users to distinguish selected vs. unselected volumes [S1, unofficial].
*   **Precision**: Using `short` (16-bit) for voxel coordinates limits the grid size to ~32k per dimension. Larger grids require 32-bit integers or sparse addressing [S1, unofficial].
*   **Performance**: Rendering every voxel as a cube is expensive for large grids. Culling (frustum and back-face) is essential [G].

## Subtopics

### Soft Shadows via Cone Integration
An approximation of soft shadows for volumetric sources. Instead of traditional shadow maps, it represents rays towards the light source as a single cone. The blocking contribution of the object inside this cone is estimated by accumulating opacity, using the same mipmapping mechanism as eye-ray traversal to approximate cone integration. The resulting opacity value represents the amount of occlusion. This provides cheap, plausible soft shadows without multisampling [S2].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of occupied voxels, as each voxel typically generates a constant number of primitives (cube faces).
*   **Memory Complexity**: $O(N)$ for storing the mesh cache and line cache.
*   **Performance**: Fast for small to medium grids. Becomes bottlenecked by draw calls and vertex count for large grids.
*   **Strengths**: Immediate visual feedback on data structure; easy to debug occupancy and boundaries; low implementation complexity compared to ray-marching.
*   **Limitations**: Not suitable for final production rendering due to lack of accurate lighting, transparency, and aliasing artifacts from discrete geometry. High vertex count for dense volumes.
*   **When to use**: Debugging voxel generation, visualizing sparse data structures, UI gizmos for volume selection.
*   **When not to use**: Final image synthesis, realistic lighting, transparent/translucent volume rendering.

## Background
VoxelGrid Preview Rendering is a standard technique in voxel-based software development, particularly in tools like Unity and custom voxel editors. It is often implemented as part of a "debug" or "gizmo" system to help developers understand the spatial distribution of voxel data. The technique is closely related to the broader field of voxel visualization and is distinct from production rendering methods like ray-marching or ray-casting. The specific implementation details regarding mesh caching, line drawing, and density gizmos are found in community-driven tools and unofficial repositories [S1, S3, unofficial]. The concept of using mipmaps for soft shadows in voxel grids was introduced in the GigaVoxels pipeline [S2].

## Sources
[S1] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
[S2] Beyond Triangles : GigaVoxels Effects In Video Games, https://doi.org/10.1145/1597990.1598068, fetched 2026-10-26
[S3] sachPico/lineBasedCave_Simplex, https://github.com/sachPico/lineBasedCave_Simplex, fetched 2026-10-27 (unofficial)
