# Geometry Shader Triangle Slicing (Z-Pass)

Also called: GS Triangle Slicing, Triangle Slicing Method

## Overview
Geometry Shader Triangle Slicing is a single-pass GPU method for multi-channel surface voxelization that explicitly clips input triangles against volume slice boundaries within the geometry shader, emitting only the valid fragments for rasterization into the appropriate volume layer [S1, unofficial]. It differs from its sibling [Pixel Shader Fragment Clipping](1685-pixel-shader-fragment-clipping.md) by performing clipping in the geometry shader rather than the pixel shader, which avoids fragment rejection overhead on architectures with limited bandwidth or non-unified cores [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Triangle vertices $v_1, v_2, v_3$ in Eye Coordinate Space (ECS).
*   **Data Parameters:** $z_{slice}$ (thickness of the volume slice in volume sweep ECS), $z_{min}$ (minimum Z depth of the volume in volume sweep ECS).
*   **Output:** Triangle strips rasterized to specific volume texture layers via the built-in variable `gl_Layer` [S1, unofficial].

**Data Structures:**
The geometry shader processes a single triangle input primitive. It generates output vertices for triangle strips. The number of vertices emitted is $3 + 4n$, where $n$ is the number of slices the triangle spans [S1, unofficial]. The method detects triangle emittance and does not produce degenerate quads [S1, unofficial].

**Algorithm Steps:**
The method assumes the triangle is aligned with the Z-axis; if not, it returns immediately [S1, unofficial].

1.  **Sort Vertices:** Sort $v_1, v_2, v_3$ according to their Z-depth in ECS such that $z(v_1) \le z(v_2) \le z(v_3)$ [S1, unofficial].
2.  **Calculate Bounds:** Compute the minimum and maximum slice indices ($s_{min}, s_{max}$) that the triangle intersects [S1, unofficial]:
    $$ s_{min} = \lfloor \frac{\min(z(v_1, v_2, v_3) - z_{min})}{z_{slice}} \rfloor $$
    $$ s_{max} = \lfloor \frac{\max(z(v_1, v_2, v_3) - z_{min})}{z_{slice}} \rfloor $$
    *Note: The source text writes `min(triangle z - z volume min) / z slice thickness`. We assume floor or integer truncation for slice indexing.*
3.  **Initialize:** Set `layer` to $s_{min}$. Set `slice` depth to the start of the current slice in ECS [S1, unofficial].
4.  **Iterate Slices:** Loop while $z(v_3) \ge slice$ [S1, unofficial]:
    *   **Case A:** If $z(v_3) \ge slice$ and this is the first iteration (or specific condition where the triangle fits entirely within the current slice bounds relative to previous logic, though the pseudocode implies a specific check):
        *   If $z(v_3) \ge slice$ (initial check in source): Emit the original triangle $(v_1, v_2, v_3)$ to `layer` and return. *Note: The source pseudocode structure is complex. A more robust interpretation of the loop logic below follows the iterative strip generation.*
    *   **Determine Left/Right Intersections:**
        *   If $z(v_2) \ge slice$:
            *   If $z(v_1) \ge slice$: Set $v_{1L} = v_{1R} = v_1$.
            *   Else: Set $v_{1L} = v_2, v_{1R} = v_1$.
            *   Calculate $v_{2L}, v_{2R}$ by intersecting edges $(v_1-v_2)$ and $(v_1-v_3)$? No, the source says "Intersect Edges (slice)". Given the sorted vertices, the edges crossing the slice plane are typically $(v_1, v_2)$ and $(v_1, v_3)$ if $v_1$ is below and $v_2, v_3$ are above, or $(v_1, v_2)$ and $(v_2, v_3)$ depending on exact positions. The source specifies: "Emit the quad strip $(v_{1R}, v_{1L}, v_{2L}, v_{2R})$".
        *   **Loop Update:** Increment `slice` by $z_{slice}$ and increment `layer`.
        *   **Update State:** Set $v_{1L} = v_{2L}, v_{1R} = v_{2R}$ (from previous iteration's right-side intersections) [S1, unofficial].
        *   **Case B:** If $z(v_2) \ge slice$:
            *   Calculate $v_{2L}, v_{2R}$ by intersecting edges with the current slice plane.
            *   Proceed to next iteration (continue loop).
        *   **Case C/D (Triangle ends in this slice):** If $z(v_3) < slice$:
            *   If $z(v_2)$ was $\ge slice$ (in previous step): Set $v_{2L} = v_2, v_{2R} = v_3$ (Case C).
            *   Else: Set $v_{2L} = v_{2R} = v_3$ (Case D).
            *   Emit the quad strip $(v_{1R}, v_{1L}, v_{2L}, v_{2R})$ to `layer`.
        *   **Case E/F (Triangle continues but vertex status changes):** Else if $z(v_2)$ was $< slice$:
            *   If $z(v_2)$ was $< slice$ (meaning $v_2$ is below current slice): Calculate $v_{2L}, v_{2R}$ by intersecting edges with slice (Case E).
            *   Else (meaning $v_2$ is exactly at slice depth or handled by Case F logic): Calculate $v_{2L}, v_{2R}$ by intersecting edges with $v_2$ depth (Case F).
            *   Emit the quad strip $(v_{1R}, v_{1L}, v_{2L}, v_{2R})$ to `layer`.
            *   Update $v_{1L} = v_{2L}, v_{1R} = v_{2R}$.
            *   Calculate new $v_{2L}, v_{2R}$ by intersecting edges with current slice.
            *   Emit the quad strip $(v_{1R}, v_{1L}, v_{2L}, v_{2R})$ to `layer`.
    *   Repeat until $z(v_3) < slice$ [S1, unofficial].

**Pseudocode:**
The following pseudocode implements Algorithm 1 from [S1, unofficial]. It assumes helper functions `IntersectEdge(vA, vB, zPlane)` which returns the point on the segment $v_A-v_B$ at depth $zPlane$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def GeometryShaderTriangleSlicing(v1, v2, v3, z_slice_thickness, z_volume_min):
    # 1. Check alignment
    if not is_aligned_with_z(v1, v2, v3):
        return

    # 2. Sort vertices by Z depth
    vertices = sort_by_z([v1, v2, v3])
    v1, v2, v3 = vertices[0], vertices[1], vertices[2]

    # 3. Calculate slice bounds
    z_min_tri = min(v1.z, v2.z, v3.z)
    z_max_tri = max(v1.z, v2.z, v3.z)
    
    s_min = floor((z_min_tri - z_volume_min) / z_slice_thickness)
    s_max = floor((z_max_tri - z_volume_min) / z_slice_thickness)

    # 4. Initialize loop variables
    layer = s_min
    current_slice_z = z_volume_min + s_min * z_slice_thickness
    
    # Previous intersection points for the "right" side of the strip
    # Initially undefined or handled by Case A
    v1L = None
    v1R = None
    v2L = None
    v2R = None
    
    # Flag to track if v2 was above the previous slice
    v2_was_above = False

    for s in range(s_min, s_max + 1):
        current_slice_z = z_volume_min + s * z_slice_thickness
        
        # Case A: Triangle fits entirely in the first slice or simple case
        # The source pseudocode has a specific early return if v3 >= slice at start.
        # However, the loop structure suggests iterative processing.
        # We follow the iterative strip generation logic described in the source.
        
        if v3.z >= current_slice_z:
            # Determine v1L, v1R based on v1 and v2 position relative to slice
            if v2.z >= current_slice_z:
                if v1.z >= current_slice_z:
                    v1L = v1
                    v1R = v1
                else:
                    v1L = v2
                    v1R = v1
                
                # Intersect edges (v1-v2) and (v1-v3)? 
                # Source says "Intersect Edges (slice)". 
                # Given v1 < slice <= v2 <= v3, the edges crossing are (v1,v2) and (v1,v3)?
                # Actually, standard triangle slicing intersects the two edges that cross the plane.
                # If v1 < slice and v2, v3 >= slice, edges are (v1,v2) and (v1,v3).
                # But the source output is (v1R, v1L, v2L, v2R).
                # Let's assume IntersectEdges returns points on the two edges crossing the plane.
                v2L, v2R = intersect_edges_crossing_plane(v1, v2, v3, current_slice_z)
                
                emit_triangle_strip([v1R, v1L, v2L, v2R], layer)
                
                # Update for next iteration
                v1L = v2L
                v1R = v2R
                v2_was_above = True
            else:
                # v2 < current_slice_z. This implies v1 < v2 < slice <= v3.
                # Edges crossing are (v2, v3) and (v1, v3)? No, (v1,v3) and (v2,v3).
                # Source Case E/F logic applies here in subsequent iterations.
                # For the first slice, if v2 < slice, we might need different handling.
                # The source pseudocode flow is tricky. 
                # Let's stick to the explicit Case B/C/D/E/F structure.
                pass

        # The source pseudocode is a single loop with internal state.
        # Re-implementing strictly per Algorithm 1 text:
        
        # Reset state for the specific iteration logic if needed, 
        # but the source maintains v1L/v1R across iterations.
        
        # Check Case A condition again as per source: "if v3 depth is >= slice then CASE A ... return"
        # This implies if the triangle is fully contained in the current slice (and it's the start?), emit and exit.
        # But the loop continues. The source likely means:
        # If we are at the start and v3 >= slice, we enter the loop.
        
        # Let's follow the exact sequence of the source pseudocode block:
        
        if v3.z >= current_slice_z:
            if v2.z >= current_slice_z:
                # Case B logic (or initial Case A if first iter)
                if v1.z >= current_slice_z:
                    v1L = v1
                    v1R = v1
                else:
                    v1L = v2
                    v1R = v1
                
                v2L, v2R = intersect_edges(v1, v2, v3, current_slice_z) # Intersect edges with slice
                emit_triangle_strip([v1R, v1L, v2L, v2R], layer)
                
                # Prepare for next slice
                v1L = v2L
                v1R = v2R
                v2_was_above = True
            else:
                # v2 < current_slice_z
                # This falls into Case E/F logic in the source's "else if" chain
                if v2_was_above: # Was v2 >= previous slice?
                    # Case E
                    v2L, v2R = intersect_edges(v1, v2, v3, current_slice_z)
                else:
                    # Case F
                    v2L, v2R = intersect_edges(v1, v2, v3, v2.z) # Intersect with v2 depth?
                
                emit_triangle_strip([v1R, v1L, v2L, v2R], layer)
                v1L = v2L
                v1R = v2R
                
                # Calculate new v2L, v2R for next step?
                v2L, v2R = intersect_edges(v1, v2, v3, current_slice_z)
                emit_triangle_strip([v1R, v1L, v2L, v2R], layer)
                
                v2_was_above = False # v2 is below current slice
        
        # Increment slice
        layer += 1
        current_slice_z += z_slice_thickness
        
        # Check termination
        if v3.z < current_slice_z:
            break

    # Helper to find intersection points of triangle edges with a Z-plane
    def intersect_edges(va, vb, vc, z_plane):
        # Find which two edges cross z_plane
        # Edges: (va,vb), (vb,vc), (vc,va)
        pts = []
        if (va.z - z_plane) * (vb.z - z_plane) <= 0:
            pts.append(intersect_point(va, vb, z_plane))
        if (vb.z - z_plane) * (vc.z - z_plane) <= 0:
            pts.append(intersect_point(vb, vc, z_plane))
        if (vc.z - z_plane) * (va.z - z_plane) <= 0:
            pts.append(intersect_point(vc, va, z_plane))
        return pts[0], pts[1]

    def intersect_point(va, vb, z_plane):
        t = (z_plane - va.z) / (vb.z - va.z)
        return va + t * (vb - va)

    def is_aligned_with_z(v1, v2, v3):
        # Check if triangle normal is parallel to Z axis or vertices have same Z?
        # Source says "if triangle not aligned with Z-axis then return".
        # This likely means the triangle's plane is perpendicular to the XY plane (vertical) 
        # or simply that the sweep direction is Z.
        # For Z-pass, we assume the triangle has extent in Z.
        # A stricter interpretation: The triangle must not be parallel to the Z-axis (i.e. not a vertical wall in XY?).
        # Actually, "aligned with Z-axis" usually means the triangle's normal is perpendicular to Z?
        # Or that the triangle lies in a plane parallel to Z?
        # Given the context of slicing, it likely means the triangle has non-zero depth in Z.
        # If all Zs are equal, it's a degenerate slice case.
        return not (v1.z == v2.z == v3.z)
```

**Parameters:**
*   $z_{slice}$: Thickness of the volume slice in volume sweep ECS [S1, unofficial].
*   $z_{min}$: Minimum Z depth of the volume in volume sweep ECS [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Triangles not aligned with the Z-axis are ignored (returned) in this specific Z-Pass implementation [S1, unofficial].
*   The method detects triangle emittance and does not produce degenerate quads [S1, unofficial].
*   On GPU architectures with small bandwidth, the alternative Pixel Shader Clipping method produces many fragments that are rejected in the pixel shader, which is an issue. The Geometry Shader Slicing method is favored here because it produces exactly the fragments rasterized in the final volume slices [S1, unofficial].

## Subtopics

### [Pixel Shader Fragment Clipping](1685-pixel-shader-fragment-clipping.md)
A single-pass GPU method for multi-channel surface voxelization where the geometry shader rasterizes triangles into all intersecting volume slices without clipping, and the pixel shader discards fragments that fall outside the slice boundaries. It avoids the computational overhead of geometry shader clipping by relying on fragment rejection in the pixel shader, achieving performance comparable to geometry slicing while being slightly more efficient for small triangle counts.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles and the number of slices each triangle intersects. The number of vertices emitted is $3 + 4n$, where $n$ is the number of slices [S1, unofficial].
*   **Memory Complexity:** Depends on the volume grid size and the number of multi-channel render targets (MRTs). The method uses layered rendering to write directly to volume texture layers [S1, unofficial].
*   **Performance:** Tested on an Intel Core i7 860 @ 2.80GHz, 8 GB RAM, nVIDIA GeForce GTX 285 GPU (1 GB memory). For the Bunny model (69,451 triangles) at a $256^3$ grid size, Geometry Slicing took 6.43 ms (15 vertices out) and 6.98 ms (19 vertices out) [S1, unofficial].
*   **Strengths:** Produces exactly the fragments rasterized in the final volume slices, avoiding fragment rejection overhead. Preferred on non-unified architectures if pixel shader cores are intensively used, and on architectures with small bandwidth [S1, unofficial].
*   **Limitations:** Requires geometry shader support. Triangles not aligned with the Z-axis are ignored. Performance varies with the number of output vertices allowed by the GPU [S1, unofficial].

## Background
The method was introduced by Athanasios Gaitatzes, Pavlos Mavridis, and Georgios Papaioannou in the paper "Two Simple Single-pass GPU methods for Multi-channel Surface Voxelization of Dynamic Scenes" [S1, unofficial]. It is part of a suite of real-time surface voxelization algorithms designed for deferred rendering and global illumination effects, utilizing OpenGL layered rendering extensions to write to arbitrary volume texture layers [S1, unofficial].

## Sources
[S1] Two Simple Single-pass GPU methods for Multi-channel Surface Voxelization of Dynamic Scenes, https://doi.org/10.2312/pe/pg/pg2011short/031-036, fetched 2026-10-26 (unofficial)
