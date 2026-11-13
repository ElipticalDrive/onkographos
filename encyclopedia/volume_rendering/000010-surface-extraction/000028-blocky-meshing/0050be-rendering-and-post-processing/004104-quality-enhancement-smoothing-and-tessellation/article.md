# Quality Enhancement (Smoothing and Tessellation)
Also called: Detail tessellation, Post-processing smoothing

## Overview
Quality enhancement refines coarse, blocky meshes extracted from volumetric data to improve visual fidelity by reducing jagged, stair-stepped artifacts and adding geometric detail [S1, unofficial][S2]. It distinguishes itself from pure smoothing methods by including synthetic detail generation and explicit mesh connectivity optimization, and from pure tessellation methods by its focus on correcting topological artifacts like T-junctions [S1, unofficial]. It builds on [Rendering and Post-Processing](14298-rendering-and-post-processing.md).

## How it works
**Inputs:** A coarse quadrilateral or triangular mesh derived from voxel data.
**Outputs:** A refined mesh with smoother surfaces and higher tessellation density, ready for rasterization.

**Data Structures:**
*   **Mesh:** A collection of vertices, edges, and faces (quads or tris).
*   **Displacement Maps:** Textures storing height information for detail tessellation.
*   **Texture Maps:** Standard surface color/normal maps.

**Steps:**

1.  **Smoothing (Surface Diffuse Flow):**
    For quadrilateral meshes, apply surface diffuse flow to smooth the geometry [S1, unofficial]. This involves moving vertices along the surface tangent plane to reduce high-frequency noise while preserving the overall shape [S1, unofficial].
    *   *Formula:* Let $v_i$ be a vertex. The new position $v_i'$ is computed by averaging neighboring vertices projected onto the tangent plane at $v_i$.
    *   *Parameter:* `smoothmesh` (Mesh Smooth). Higher values increase smoothing but slow down mesh building [S2].
    *   *Note:* This can be implemented efficiently in parallel [S1, unofficial].

2.  **Tangent Movement:**
    Add tangent movement to further improve quality on quadrilateral meshes [S1, unofficial]. This adjusts vertex positions along tangent directions to correct distortions introduced by the initial extraction [S1, unofficial].

3.  **Detail Tessellation (Catmull-Clark Approximation):**
    Approximate a smooth Catmull-Clark subdivision surface using bicubic patches [S1, unofficial].
    *   *Mechanism:* Use a tessellation shader to subdivide the coarse mesh into smaller patches [S1, unofficial].
    *   *Displacement:* Apply displacement maps to the bicubic patches to add fine geometric detail [S1, unofficial].
    *   *Mapping:* Use tri-planar texture mapping for both displacement maps and texture maps to avoid distortion on complex or blocky surfaces [S1, unofficial][S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quality_enhance(mesh, smoothness_factor, displacement_maps, texture_maps):
    """
    Applies smoothing and detail tessellation to a coarse mesh.
    
    Args:
        mesh: Coarse quadrilateral or triangular mesh.
        smoothness_factor: Float controlling smoothing intensity.
        displacement_maps: Dict of tri-planar displacement textures.
        texture_maps: Dict of tri-planar color/normal textures.
        
    Returns:
        Refined mesh.
    """
    # Step 1: Smoothing via Surface Diffuse Flow
    # For quadrilateral meshes, apply surface diffuse flow
    if mesh.is_quadrilateral:
        for vertex in mesh.vertices:
            # Calculate tangent plane at vertex
            tangent_plane = compute_tangent_plane(vertex)
            
            # Average neighbors projected onto tangent plane
            neighbors = get_adjacent_vertices(vertex)
            smoothed_pos = vertex.position
            for neighbor in neighbors:
                delta = neighbor.position - vertex.position
                # Project delta onto tangent plane
                projected_delta = project_onto_plane(delta, tangent_plane.normal)
                smoothed_pos += projected_delta
            
            # Normalize and scale by smoothness factor
            smoothed_pos = vertex.position + (smoothed_pos - vertex.position) * smoothness_factor
            vertex.position = smoothed_pos
            
    # Step 2: Tangent Movement
    # Add tangent movement for quality improvements
    for vertex in mesh.vertices:
        tangent_movement = compute_tangent_movement(vertex)
        vertex.position += tangent_movement

    # Step 3: Detail Tessellation
    # Approximate Catmull-Clark subdivision with bicubic patches
    tessellated_mesh = tessellate_with_bicubic_patches(mesh)
    
    # Apply displacement maps using tri-planar mapping
    for face in tessellated_mesh.faces:
        for vertex in face.vertices:
            # Tri-planar projection for displacement
            disp_value = tri_planar_sample(displacement_maps, vertex.position)
            # Displace vertex along normal
            vertex.position += vertex.normal * disp_value
            
    # Apply texture maps using tri-planar mapping
    for face in tessellated_mesh.faces:
        for vertex in face.vertices:
            tex_color = tri_planar_sample(texture_maps, vertex.position)
            vertex.color = tex_color
            
    return tessellated_mesh

def compute_tangent_plane(vertex):
    # [G] Compute normal from adjacent faces, then derive tangent basis
    normal = compute_vertex_normal(vertex)
    # Return normal and two orthogonal tangent vectors
    return {'normal': normal, 'tangent1': orthogonal_vector(normal), 'tangent2': cross(normal, orthogonal_vector(normal))}

def project_onto_plane(delta, normal):
    # [G] Project vector delta onto plane perpendicular to normal
    return delta - dot(delta, normal) * normal

def compute_tangent_movement(vertex):
    # [G] Placeholder for specific tangent movement algorithm from [S1, unofficial]
    # Implementation details not fully specified in sources beyond "add tangent movement"
    return vec3(0, 0, 0) # Placeholder

def tessellate_with_bicubic_patches(mesh):
    # [G] Subdivide mesh into bicubic patches suitable for tessellation shader
    # Implementation details depend on specific GPU API
    return mesh # Simplified return for pseudocode completeness

def tri_planar_sample(textures, position):
    # [G] Sample texture using X, Y, Z projections and blend
    # Standard tri-planar mapping logic
    return 0.0 # Placeholder for displacement value
```

**Parameters:**
*   `smoothmesh` (Mesh Smooth): Controls smoothing intensity. Larger values increase smoothing but slow down mesh building [S2]. Typical values depend on the desired smoothness; no specific range given [S2].
*   `surflevel` (Isosurface Level): Threshold for geometry generation. Cells below this value are ignored. For Phoenix liquids (range 0-1), 0.5 is recommended [S2].

**Edge Cases:**
*   **Precision:** Tri-planar mapping avoids distortion but may introduce seams if not blended correctly [S1, unofficial].
*   **Boundaries:** Smoothing may shrink the mesh slightly; boundary handling is not explicitly detailed [S1, unofficial].
*   **Degenerate Input:** Coarse meshes with poor topology may not smooth well; tangent movement helps but does not fix severe artifacts [S1, unofficial].

## Subtopics

### [Geometry Post-Processing](13056-geometry-post-processing.md)
Geometry Post-Processing refers to a suite of techniques applied to blocky meshes extracted from volumetric data to improve visual fidelity and rendering performance. It addresses the jagged, stair-stepped appearance of initial voxel-based surface extractions and optimizes the resulting geometry for the GPU. It is distinguished from pure smoothing methods by its inclusion of synthetic detail generation and explicit mesh connectivity optimization, and from pure tessellation methods by its focus on correcting topological artifacts like T-junctions. See [Geometry Post-Processing](13056-geometry-post-processing.md).

### Mesh Smooth
Used when Mode is set to Mesh, Ocean Mesh, or Cap Mesh to reduce the roughness of the mesh. The larger the value, the slower the mesh builds [S2]. See [Geometry Post-Processing](13056-geometry-post-processing.md) for broader context.

## Operating characteristics
*   **Time Complexity:** Smoothing is $O(V)$ where $V$ is the number of vertices, as it processes each vertex independently in parallel [S1, unofficial]. Tessellation increases vertex count, so complexity depends on the tessellation factor.
*   **Memory Complexity:** Increases with tessellation due to additional vertices and texture map storage [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Smoothing scales well with parallel implementation [S1, unofficial]. Tessellation performance depends on GPU tessellation capabilities and displacement map resolution [S1, unofficial].
*   **Strengths:** Produces smooth, high-quality surfaces; real-time capable with tessellation shaders [S1, unofficial].
*   **Limitations:** Smoothing can be slow with high values [S2]; tri-planar mapping may have seams [S1, unofficial].
*   **When to Use:** When visual fidelity is critical and the input mesh is blocky [S1, unofficial][S2].
*   **When Not to Use:** When performance is paramount and blocky appearance is acceptable [S2].

## Background
Originating from research on generating smooth high-quality isosurfaces for interactive modeling and visualization of complex terrains [S1, unofficial], this method combines techniques from Zhang et al. [ZHK04] for smoothing and Loop and Schaefer [LS08] for tessellation. It is used in volume rendering pipelines to enhance the visual quality of extracted surfaces [S1, unofficial].

## Sources
[S1] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
[S2] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
