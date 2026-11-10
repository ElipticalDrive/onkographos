# Embree Mesh Acceleration Structure

Also called: Embree RTC, Embree-based Volume Rendering for Unstructured Meshes, RTC Mesh

## Overview
The Embree Mesh Acceleration Structure is a method for volume rendering unstructured mesh data by wrapping Intel's Embree ray-tracing engine to build a triangle mesh acceleration structure and perform ray tracing [S1, unofficial][S2]. It solves the problem of efficiently traversing complex, non-uniform geometric domains by converting unstructured mesh elements (such as hexahedral or tetrahedral cells) into triangular polygons [S2]. It differs from sibling methods by relying on hardware-accelerated BVH traversal via Embree's RTC API rather than custom software ray marchers or grid-based traversal, enabling both surface plots and traditional accumulation rendering [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Unstructured mesh data (vertices and connectivity) and field data associated with the mesh [S2].
*   **Output:** Rendered image (surface values or accumulated field values) and hit data (intersection points, primitive IDs) [S1, unofficial][S2].

**Data Structures:**
The implementation relies on a `RenderSource` object that encapsulates the logic for creating the Embree polygon mesh from the `data_source` and performing ray tracing calls [S2]. Internally, the acceleration structure is built using the following buffers:
*   **Vertex Buffer:** Allocated with `RTC_FORMAT_FLOAT3`, storing vertex coordinates as 32-bit floating-point vectors [S1, unofficial].
*   **Index Buffer:** Allocated with `RTC_FORMAT_UINT3`, storing triangle indices as 32-bit unsigned integer triplets [S1, unofficial].
*   **Geometry:** Created via `rtcNewGeometry` (implied by Embree API) and committed [S1, unofficial].

**Steps:**
1.  **Mesh Generation:** Generate a triangular polygon mesh from the unstructured mesh data. If the source elements are not triangles (e.g., hexahedra or tetrahedra), break up faces into multiple triangles [S2].
2.  **Embree Initialization:** Initialize the Embree device and scene. Set the build quality to `HIGH` [S1, unofficial].
3.  **Geometry Creation:** Create triangle geometry. Allocate the vertex buffer (`RTC_FORMAT_FLOAT3`) and index buffer (`RTC_FORMAT_UINT3`). Copy the mesh data into these buffers [S1, unofficial].
4.  **Commit Geometry:** Commit the geometry and attach it to the Embree scene [S1, unofficial].
5.  **Commit Scene:** Commit the scene to finalize the acceleration structure build [S1, unofficial].
6.  **Ray Tracing:**
    *   **Single Ray Closest Hit:** Use `rtcIntersect1` with a coherent context to find the nearest intersection for a single ray [S1, unofficial].
    *   **Single Ray Any Hit (Occlusion):** Use `rtcOccluded1` with a coherent context to determine if a ray intersects any geometry [S1, unofficial].
    *   **Multiple Ray Closest Hit:** Use `rtcIntersect1M` with a coherent context flag determined by a parameter to trace multiple rays [S1, unofficial].
    *   **Multiple Ray Any Hit:** Use `rtcOccluded1M` with a coherent context flag [S1, unofficial].
7.  **Rendering:**
    *   **Surface Plots:** Calculate the value of the field at the intersection point for each ray using the hit data computed by Embree [S2].
    *   **Accumulation Rendering:** Accumulate values along each ray for every element the rays intersect, allowing operations such as computing maximum intensity along the ray [S2].

**Formulas and Parameters:**
*   **Coherence Flag:** A boolean parameter `coherent` passed to trace functions. If `true`, uses `RTC_INTERSECT_CONTEXT_FLAG_COHERENT`. If `false`, the same flag is used in the implementation (noted as a potential bug in the source, but documented as is) [S1, unofficial].
*   **Intra-element Interpolation:** Implementations of various intra-element interpolation functions are required to support meshes of various types and orders, as ray intersections may occur inside elements rather than just on surfaces [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_embree_mesh(vertices: List[Vec3], indices: List[Vec3]) -> EmbreeScene:
    """
    Build an Embree acceleration structure from vertex and index data.
    vertices: List of 3D coordinates (float3).
    indices: List of triangle indices (uint3).
    """
    # 1. Initialize Embree device and scene
    device = embree_new_device()
    scene = embree_new_scene(device)
    
    # Set build quality to HIGH
    embree_set_build_quality(scene, EMBREE_BUILD_QUALITY_HIGH)
    
    # 2. Create triangle geometry
    geometry = embree_new_geometry(device, EMBREE_GEOMETRY_TRIANGLE, len(indices))
    
    # 3. Allocate and copy vertex buffer
    vertex_buffer = embree_new_buffer(device, EMBREE_BUFFER_TYPE_VERTEX, EMBREE_FORMAT_FLOAT3, len(vertices))
    embree_buffer_write(vertex_buffer, 0, vertices)
    embree_set_geometry_vertex_buffer(geometry, vertex_buffer, 0, 0)
    
    # 4. Allocate and copy index buffer
    index_buffer = embree_new_buffer(device, EMBREE_BUFFER_TYPE_INDEX, EMBREE_FORMAT_UINT3, len(indices))
    embree_buffer_write(index_buffer, 0, indices)
    embree_set_geometry_index_buffer(geometry, index_buffer, 0, 0)
    
    # 5. Commit geometry and attach to scene
    embree_commit_geometry(geometry)
    embree_add_geometry(scene, geometry, 0)
    
    # 6. Commit scene
    embree_commit_scene(scene)
    
    return scene

def trace_closest(scene: EmbreeScene, ray: Ray, coherent: bool) -> HitRecord:
    """
    Trace a single ray for the closest hit.
    """
    # Use rtcIntersect1 with coherent context
    context_flags = EMBREE_INTERSECT_CONTEXT_FLAG_COHERENT if coherent else 0
    hit = embree_intersect1(scene, ray, context_flags)
    return hit

def trace_any(scene: EmbreeScene, ray: Ray, coherent: bool) -> bool:
    """
    Trace a single ray for occlusion (any hit).
    """
    # Use rtcOccluded1 with coherent context
    context_flags = EMBREE_INTERSECT_CONTEXT_FLAG_COHERENT if coherent else 0
    occluded = embree_occluded1(scene, ray, context_flags)
    return occluded

def trace_closest_batch(scene: EmbreeScene, rays: List[Ray], coherent: bool) -> List[HitRecord]:
    """
    Trace multiple rays for closest hits.
    """
    # Use rtcIntersect1M with coherent context flag
    context_flags = EMBREE_INTERSECT_CONTEXT_FLAG_COHERENT if coherent else 0
    hits = embree_intersect1m(scene, rays, context_flags)
    return hits

def render_surface(scene: EmbreeScene, rays: List[Ray], field_data: FieldData) -> Image:
    """
    Surface plot rendering: calculate field value at intersection point.
    """
    hits = trace_closest_batch(scene, rays, coherent=True)
    image = empty_image(len(rays))
    for i, hit in enumerate(hits):
        if hit.is_valid:
            # Calculate value of the field at the intersection point
            value = interpolate_field(field_data, hit.intersection_point)
            image[i] = value
    return image

def render_accumulation(scene: EmbreeScene, rays: List[Ray], field_data: FieldData) -> Image:
    """
    Accumulation rendering: accumulate values along each ray.
    """
    # Note: Embree RTC primarily supports closest/any hit. 
    # For accumulation, multiple intersections or a custom any-hit shader is needed.
    # This pseudocode assumes a hypothetical multi-hit capability or repeated tracing.
    image = empty_image(len(rays))
    for i, ray in enumerate(rays):
        accumulated_value = 0.0
        # Hypothetical loop for multiple intersections
        # hits = trace_all_hits(scene, ray) 
        # for hit in hits:
        #     value = interpolate_field(field_data, hit.intersection_point)
        #     accumulated_value = max(accumulated_value, value) # Example: max intensity
        image[i] = accumulated_value
    return image
```

**Edge Cases and Pitfalls:**
*   **Intra-element Interpolation:** Ray intersections may occur inside elements rather than just on surfaces. Intra-element interpolation functions must be implemented to support meshes of various types and orders [S2].
*   **Coherence Flag Bug:** The source notes a potential bug where the `coherent` flag logic might be inverted or incorrectly applied in the `false` case, but it is documented as implemented [S1, unofficial].
*   **Mesh Type Support:** Currently implemented for Hexahedral and Tetrahedral mesh elements. Adding support for other mesh types requires breaking up faces into triangles [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the BVH traversal depth, typically $O(\log N)$ for ray intersection, where $N$ is the number of triangles [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of vertices and triangles in the mesh, plus the overhead of the BVH structure [S1, unofficial].
*   **Performance:** Embree is described as a "fast ray-tracing code" [S2]. Hardware acceleration via RTX BVH (as per parent article) further improves performance for large scenes.
*   **Strengths:** Supports both surface plots and traditional accumulation rendering; flexible integration with camera paths and multiple render sources [S2].
*   **Limitations:** Requires conversion of unstructured mesh to triangular polygons; dependent on external Embree library (though a future replacement with custom ray-tracing code is planned) [S2].
*   **When to use:** When rendering unstructured mesh data with complex geometry where hardware-accelerated ray tracing provides a performance benefit over software rasterization or marching.
*   **When not to use:** When the mesh elements are simple and regular, or when intra-element interpolation is too complex to implement efficiently.

## Background
The method originates from the yt project's enhancement proposal for unstructured mesh volume rendering [S2]. It leverages Intel's Embree ray-tracing engine, which is known for high-performance software ray tracing [S2]. The implementation was developed to support both surface plots and accumulation rendering for unstructured mesh data, integrating with the yt volume rendering refactor to allow flexible camera paths and multiple render sources [S2]. The code examples are from an unofficial repository by Mike-Leo-Smith, demonstrating a C++ implementation using glm and Embree [S1, S1, unofficial].

## Sources
[S1] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
[S2] YTEP-0031: Unstructured Mesh — yt Enhancement Proposals 1.0 documentation, https://ytep.readthedocs.io/en/master/YTEPs/YTEP-0031.html, fetched 2026-10-27
