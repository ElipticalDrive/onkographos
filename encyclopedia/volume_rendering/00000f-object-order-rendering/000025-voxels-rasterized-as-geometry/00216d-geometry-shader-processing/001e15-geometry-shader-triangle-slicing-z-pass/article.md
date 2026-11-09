# Geometry Shader Triangle Slicing (Z-Pass)

Also called: GS Triangle Slicing, Triangle Slicing Method

## Overview
Geometry Shader Triangle Slicing is a single-pass GPU method for multi-channel surface voxelization that explicitly clips input triangles against volume slice boundaries within the geometry shader, emitting only the valid geometric fragments for rasterization into the appropriate volume layer [S1, unofficial]. Unlike its sibling [Pixel Shader Fragment Clipping](1685-pixel-shader-fragment-clipping.md), which rasterizes the full triangle to all intersecting slices and discards out-of-bounds fragments in the pixel shader, this method performs clipping in the geometry shader to generate precise triangle strips [S1, unofficial]. This approach is preferred on architectures with small bandwidth or non-unified pipelines where pixel shader cores are intensively used, as it avoids the overhead of generating and rejecting large numbers of unnecessary fragments [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Three triangle vertices $v_1, v_2, v_3$ in Eye Coordinate Space (ECS) [S1, unofficial].
*   **Data Parameters:** $z_{slice}$ (z-slice thickness in volume sweep ECS), $z_{min}$ (z-volume minimum in volume sweep ECS) [S1, unofficial].
*   **Output:** Triangle strips rasterized to specific volume layers via the built-in variable `gl_Layer` [S1, unofficial].

**Data Structures:**
The geometry shader processes a single triangle input primitive. It maintains state variables for the current slice depth, layer index, and intersection points ($v_{1L}, v_{1R}, v_{2L}, v_{2R}$) to construct quad strips [S1, unofficial].

**Algorithm Steps:**
The method assumes the triangle is aligned with the Z-axis (sweep direction). If not, the triangle is ignored [S1, unofficial].

1.  **Sort Vertices:** Sort $v_1, v_2, v_3$ such that their Z-depths are non-decreasing ($z_1 \le z_2 \le z_3$) [S1, unofficial].
2.  **Initialize:**
    *   Calculate the minimum slice index: $layer = \lfloor (z_1 - z_{min}) / z_{slice} \rfloor$ [S1, unofficial].
    *   Set current slice depth: $slice = z_{min} + layer \times z_{slice}$ [S1, unofficial].
3.  **Case A (Single Slice):** If $z_3 \ge slice$, emit the original triangle $(v_1, v_2, v_3)$ to `gl_Layer = layer` and return [S1, unofficial].
4.  **Initial Slice Crossing:**
    *   If $z_2 \ge slice$, set $v_{1L} = v_{1R} = v_1$ [S1, unofficial].
    *   Else, set $v_{1L} = v_2, v_{1R} = v_1$ [S1, unofficial].
    *   Calculate $v_{2L}, v_{2R}$ by intersecting edges $(v_1, v_2)$ and $(v_1, v_3)$ with the plane $z = slice$ [S1, unofficial].
    *   Emit quad strip $(v_{1R}, v_{1L}, v_{2L}, v_{2R})$ to `gl_Layer = layer` [S1, unofficial].
5.  **Loop:** Iterate while $z_3 \ge slice$ [S1, unofficial]:
    *   Increment $slice \leftarrow slice + z_{slice}$ and $layer \leftarrow layer + 1$ [S1, unofficial].
    *   Update $v_{1L} \leftarrow v_{2L}, v_{1R} \leftarrow v_{2R}$ [S1, unofficial].
    *   **Case B:** If $z_2 \ge slice$, calculate $v_{2L}, v_{2R}$ by intersecting edges with $z = slice$ [S1, unofficial].
    *   **Case C/D (Triangle Ends):** Else if $z_3 < slice$:
        *   If $z_2$ was previously $\ge slice$ (Case C), set $v_{2L} = v_2, v_{2R} = v_3$ [S1, unofficial].
        *   Else (Case D), set $v_{2L} = v_{2R} = v_3$ [S1, unofficial].
        *   Emit quad strip $(v_{1R}, v_{1L}, v_{2L}, v_{2R})$ to `gl_Layer = layer` and break [S1, unofficial].
    *   **Case E/F (Intermediate Slices):** Else if $z_2$ was previously $< slice$ (Case E), calculate $v_{2L}, v_{2R}$ by intersecting edges with $z = slice$ [S1, unofficial]. Else (Case F), calculate $v_{2L}, v_{2R}$ by intersecting edges with $z = z_2$ [S1, unofficial].
    *   Emit quad strip $(v_{1R}, v_{1L}, v_{2L}, v_{2R})$ to `gl_Layer = layer` [S1, unofficial].
    *   Update $v_{1L} \leftarrow v_{2L}, v_{1R} \leftarrow v_{2R}$ [S1, unofficial].
    *   Calculate $v_{2L}, v_{2R}$ by intersecting edges with $z = slice$ [S1, unofficial].
    *   Emit quad strip $(v_{1R}, v_{1L}, v_{2L}, v_{2R})$ to `gl_Layer = layer` [S1, unofficial].
6.  **Termination:** The loop repeats until $z_3 < slice$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_shader_triangle_slicing(v1, v2, v3, z_slice_thickness, z_volume_min):
    # Input: v1, v2, v3 in Eye Coordinate Space (ECS)
    # Data: z_slice_thickness, z_volume_min
    
    # Check alignment with Z-axis (triangle plane parallel to Z)
    def is_aligned_with_z(a, b, c):
        # Cross product of two edges; if Z component is non-zero, it's not aligned
        e1 = (b.x - a.x, b.y - a.y, b.z - a.z)
        e2 = (c.x - a.x, c.y - a.y, c.z - a.z)
        cross_z = e1[0]*e2[1] - e1[1]*e2[0]
        return cross_z == 0.0

    if not is_aligned_with_z(v1, v2, v3):
        return

    # Sort vertices by Z depth
    vertices = sorted([v1, v2, v3], key=lambda v: v.z)
    v1, v2, v3 = vertices[0], vertices[1], vertices[2]
    
    # Initialize
    layer = int((v1.z - z_volume_min) // z_slice_thickness)
    slice_depth = z_volume_min + layer * z_slice_thickness
    
    # Case A
    if v3.z >= slice_depth:
        emit_triangle(v1, v2, v3, layer)
        return

    # Initial slice crossing setup
    if v2.z >= slice_depth:
        v1L = v1R = v1
    else:
        v1L = v2
        v1R = v1
        
    v2L, v2R = intersect_edges_with_plane(v1, v2, v1, v3, slice_depth)
    emit_strip([v1R, v1L, v2L, v2R], layer)
    
    # Track state for Cases C/E
    v2_was_ge_slice = (v2.z >= slice_depth)
    
    while v3.z >= slice_depth:
        slice_depth += z_slice_thickness
        layer += 1
        
        v1L = v2L
        v1R = v2R
        
        if v2.z >= slice_depth:
            # Case B
            v2L, v2R = intersect_edges_with_plane(v1, v2, v1, v3, slice_depth)
        elif v3.z < slice_depth:
            # Triangle ends within this slice
            if v2_was_ge_slice:
                # Case C
                v2L = v2
                v2R = v3
            else:
                # Case D
                v2L = v3
                v2R = v3
            emit_strip([v1R, v1L, v2L, v2R], layer)
            break
        else:
            # v3.z >= slice_depth and v2.z < slice_depth
            if v2_was_ge_slice:
                # Case E
                v2L, v2R = intersect_edges_with_plane(v1, v2, v1, v3, slice_depth)
            else:
                # Case F
                v2L, v2R = intersect_edges_with_plane(v1, v2, v1, v3, v2.z)
            emit_strip([v1R, v1L, v2L, v2R], layer)
            
            v1L = v2L
            v1R = v2R
            
            v2L, v2R = intersect_edges_with_plane(v1, v2, v1, v3, slice_depth)
            emit_strip([v1R, v1L, v2L, v2R], layer)
            
        v2_was_ge_slice = (v2.z >= slice_depth)

def intersect_edges_with_plane(v1, v2, v3, v4, z_plane):
    # Intersects edge (v1,v2) and (v3,v4) with plane z=z_plane
    p1 = interpolate(v1, v2, z_plane)
    p2 = interpolate(v3, v4, z_plane)
    return p1, p2

def interpolate(v_start, v_end, z_target):
    # Linear interpolation to find point at z_target
    t = (z_target - v_start.z) / (v_end.z - v_start.z)
    return v_start + t * (v_end - v_start)

def emit_triangle(v1, v2, v3, layer):
    # Emits triangle to layer
    pass

def emit_strip(vertices, layer):
    # Emits quad strip (2 triangles) to layer
    # Vertices: v1R, v1L, v2L, v2R
    pass
```

**Parameters:**
*   $z_{slice}$: Thickness of the volume slice in volume sweep ECS [S1, unofficial].
*   Output Vertex Count: $3 + 4n$, where $n$ is the number of slices the triangle spans [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Z-Axis Alignment:** Triangles not aligned with the Z-axis are ignored (returned immediately) in this specific Z-Pass implementation [S1, unofficial].
*   **Degenerate Quads:** The method detects triangle emittance and does not produce degenerate quads [S1, unofficial].
*   **Precision:** Standard floating-point precision applies for intersection calculations [G].

## Subtopics

### [Pixel Shader Fragment Clipping](1685-pixel-shader-fragment-clipping.md)
A single-pass GPU method where the geometry shader rasterizes triangles into all intersecting volume slices without clipping, and the pixel shader discards fragments that fall outside the slice boundaries. It avoids the computational overhead of geometry shader clipping by relying on fragment rejection in the pixel shader, achieving performance comparable to geometry slicing while being slightly more efficient for small triangle counts.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles and the number of slices each triangle intersects [S1, unofficial].
*   **Memory Complexity:** Low, as it does not require storing intermediate volume data structures on the CPU or GPU beyond the render targets [S1, unofficial].
*   **Performance:** On an Intel Core i7 860 @ 2.80GHz, 8 GB RAM, nVIDIA GeForce GTX 285 GPU (1 GB memory), for the Bunny model (69,451 triangles) at a $256^3$ grid size, Geometry Slicing took 6.43 ms (15 vertices out) and 6.98 ms (19 vertices out) [S1, unofficial].
*   **Strengths:** Produces exactly the fragments rasterized in the final volume slices, avoiding the overhead of fragment rejection on architectures with small bandwidth [S1, unofficial]. Preferred on non-unified architectures if pixel shader cores are intensively used [S1, unofficial].
*   **Limitations:** Requires triangles to be aligned with the Z-axis (sweep direction) in this specific implementation [S1, unofficial]. The vertex output limit ($3+4n$) can be restrictive on some hardware for very high-resolution grids [S1, unofficial].

## Background
The method was introduced by Athanasios Gaitatzes, Pavlos Mavridis, and Georgios Papaioannou in 2011 as part of "Two Simple Single-pass GPU methods for Multi-channel Surface Voxelization of Dynamic Scenes" [S1, unofficial]. It utilizes the OpenGL extension for layered rendering (`gl_Layer`) to write primitives to arbitrary volume texture layers computed at run time, eliminating the need for multiple passes over incoming data [S1, unofficial].

## Sources
[S1] Two Simple Single-pass GPU methods for Multi-channel Surface Voxelization of Dynamic Scenes, https://doi.org/10.2312/pe/pg/pg2011short/031-036, fetched 2026-10-26 (unofficial)
