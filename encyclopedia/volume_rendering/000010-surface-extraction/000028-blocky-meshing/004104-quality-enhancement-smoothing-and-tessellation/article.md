# Quality Enhancement (Smoothing and Tessellation)
Also called: Detail tessellation, Post-processing smoothing

## Overview
This method refines blocky voxel meshes by applying geometric smoothing and high-resolution tessellation to approximate smooth surfaces. It addresses the jagged, aliased appearance of raw voxel extractions (such as those from [Blocky meshing](31-blocky-meshing.md)) by replacing flat faces with curved patches. Unlike [Greedy face merging](8935-greedy-face-merging.md) which focuses on topology reduction, this method focuses on visual fidelity through subdivision and displacement.

## How it works
**Inputs:** A coarse quadrilateral mesh generated from voxel data; a smoothing strength parameter; a tessellation level.
**Outputs:** A high-resolution mesh with smooth curvature and displaced surface details.

**Data Structures:**
*   **Mesh:** Collection of vertices and quadrilateral faces.
*   **Displacement Maps:** Textures storing height information for detail tessellation.
*   **Texture Maps:** Standard color/albedo maps.

**Steps:**

1.  **Smoothing (Surface Diffuse Flow):**
    For quadrilateral meshes, apply surface diffuse flow to reduce roughness [S1, unofficial]. This is preferred over standard discrete Laplacian smoothing for maintaining mesh quality in parallel implementations [S1, unofficial].
    *   Calculate the surface normal $n_i$ for each vertex $i$.
    *   Compute the tangent movement to improve quality [S1, unofficial].
    *   Update vertex positions $v_i$ iteratively. The smoothing strength controls the convergence speed; higher values reduce roughness but slow down mesh building [S2].

2.  **Tessellation (Catmull-Clark Approximation):**
    Approximate a smooth Catmull-Clark subdivision surface using a tessellation shader [S1, unofficial].
    *   Subdivide each quadrilateral face into smaller patches.
    *   Use bicubic patches to represent the surface geometry [S1, unofficial].
    *   Apply displacement maps to the patches to add fine geometric detail [S1, unofficial].

3.  **Texturing (Tri-planar Mapping):**
    Apply textures and displacement maps using tri-planar texture mapping [S1, unofficial].
    *   Project textures onto the X, Y, and Z axes.
    *   Blend the projections based on the surface normal to avoid distortion on curved surfaces [S1, Gei07, GT07, unofficial].

**Parameters:**
*   **Smoothing Strength:** Controls the aggressiveness of the smoothing. Larger values reduce roughness more effectively but increase computation time [S2].
*   **Tessellation Level:** Determines the density of the bicubic patches.

**Edge Cases:**
*   **Precision:** Parallel implementation requires careful handling of vertex updates to avoid race conditions during smoothing [S1, unofficial].
*   **Boundary:** Tri-planar mapping handles boundaries and corners naturally by blending projections [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quality_enhance(mesh, smoothing_strength, tessellation_level):
    # Step 1: Smoothing via Surface Diffuse Flow
    # [S1, unofficial] Uses surface diffuse flow and tangent movement for quadrilateral meshes
    for iteration in range(max_iterations):
        for vertex in mesh.vertices:
            # Calculate normal
            normal = compute_normal(vertex)
            # Apply diffuse flow update
            # [S1, unofficial] Efficiently implemented in parallel
            vertex.position += smoothing_strength * compute_diffuse_flow(vertex)
            
    # Step 2: Tessellation with Bicubic Patches
    # [S1, unofficial] Approximates Catmull-Clark subdivision
    high_res_mesh = TessellationShader(
        input_mesh=mesh,
        level=tessellation_level,
        patch_type='bicubic'
    )
    
    # Step 3: Displacement and Texturing
    # [S1, unofficial] Uses displacement maps and tri-planar mapping
    for patch in high_res_mesh.patches:
        # [S1, unofficial] Tri-planar texture mapping for displacement and color
        displacement = tri_planar_sample(patch.position, displacement_map)
        color = tri_planar_sample(patch.position, texture_map)
        patch.position += patch.normal * displacement
        patch.color = color
        
    return high_res_mesh
```

## Subtopics

### Mesh Smooth
A specific smoothing mode used in software like Phoenix for 3ds Max [S2]. It is activated when the rendering mode is set to Mesh, Ocean Mesh, or Cap Mesh [S2]. Its primary function is to reduce the roughness of the generated geometry [S2]. The parameter `smoothmesh` controls the intensity; a larger value results in a smoother mesh but increases the time required for mesh building [S2]. It is distinct from general smoothing as it is tightly coupled with the specific mesh generation modes of the host application [S2].

## Operating characteristics
*   **Time Complexity:** Smoothing is iterative; convergence speed depends on the smoothing strength [S2]. Tessellation is proportional to the number of faces and the tessellation level [S1, unofficial].
*   **Memory Complexity:** Increases with tessellation level due to the generation of bicubic patches and associated texture coordinates [S1, unofficial].
*   **Strengths:** Produces visually smooth, high-quality surfaces suitable for interactive visualization [S1, unofficial]. Efficiently parallelizable [S1, unofficial].
*   **Limitations:** Computationally expensive for high tessellation levels [S1, unofficial]. Smoothing strength trade-off between quality and speed [S2].
*   **When to use:** When visual fidelity is prioritized over raw performance, and the input is a quadrilateral mesh [S1, unofficial].

## Background
Originating from research on generating smooth high-quality isosurfaces for interactive modeling [S1, unofficial], this method builds on earlier work in discrete Laplacian smoothing [WMKG08] and surface diffuse flow for quadrilateral meshes [ZHK04]. The tessellation approach follows real-time capable methods by Loop and Schaefer [LS08], adapting Catmull-Clark subdivision for GPU pipelines. Tri-planar mapping was adopted from standard texture mapping techniques [Gei07, GT07].

## Sources
[S1] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
[S2] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
