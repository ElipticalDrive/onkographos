# Geometry Shader for Velocity Voxelization

## Overview
This method computes the obstacle velocity at each grid cell containing part of the obstacle's boundary by projecting 3D triangle intersections onto 2D slice planes. It differs from standard voxelization by outputting a 1-voxel wide quadrilateral for each intersecting triangle edge, explicitly encoding the interpolated velocity vector at the intersection point. This allows the subsequent rasterization or accumulation step to directly populate velocity fields without post-processing interpolation.

## How it works
**Inputs and Outputs:**
*   **Input:** A single triangle defined by three vertices (`input[0]`, `input[1]`, `input[2]`), each containing a position `Pos` (with z-component) and a velocity `Velocity`. The current slice index `sliceIdx` and the slice plane's z-coordinate `sliceZ`.
*   **Output:** A stream of vertices forming two triangles (a quadrilateral) if the input triangle intersects the slice plane, or no output if it does not. The quadrilateral is 1-voxel wide in screen space.

**Data Structures:**
*   `VsGenVelOutput`: Struct containing `Pos` (float4, xy from intersection, z=0, w=1) and `Velocity` (float2 or float3, interpolated).
*   `GsGenVelIntVtx`: Struct containing `Pos` (float2) and `Velocity` (interpolated).
*   `intersections[2]`: Array of two `GsGenVelIntVtx` elements to store the two intersection points found on the triangle edges.

**Parameters:**
*   `projSpacePixDim`: The pixel dimension in projection space. The source uses a constant `sqrtOf2 = 1.414` as a proxy for the diagonal of a pixel, implying the width is derived from this diagonal magnitude [S1, unofficial].

**Algorithm Steps:**
1.  **Bounding Box Check:** Calculate `minZ` and `maxZ` of the three input triangle vertices. If `sliceZ` is outside `[minZ, maxZ]`, the triangle does not intersect the slice; return immediately.
2.  **Intersection Finding:** Initialize an `intersections` array and an index `idx = 0`. Iterate through the three edges of the triangle (0-1, 1-2, 2-0). For each edge, call `GetEdgePlaneIntersection`.
3.  **Edge Intersection Logic (`GetEdgePlaneIntersection`):**
    *   Calculate the parameter $t$ for the line-plane intersection:
        $$ t = \frac{sliceZ - vA.Pos.z}{vB.Pos.z - vA.Pos.z} $$
    *   If $t < 0$ or $t > 1$, the intersection is not within the segment endpoints; return without modifying the array.
    *   Otherwise, compute the 2D position and velocity by linear interpolation:
        $$ Pos = \text{lerp}(vA.Pos, vB.Pos, t).xy $$
        $$ Velocity = \text{lerp}(vA.Velocity, vB.Velocity, t) $$
    *   Store the result in `intersections[idx]` and increment `idx`.
4.  **Validation:** If fewer than 2 intersections are found (`idx < 2`), the triangle does not form a valid segment across the slice (or is tangent/degenerate); return.
5.  **Normal Calculation:** Compute the 2D normal for offsetting the segment to create the quadrilateral width. The source specifies:
    $$ \text{normal} = \sqrt{2} \cdot \text{normalize}(\text{cross}((input[1].Pos - input[0].Pos), (input[2].Pos - input[0].Pos)).xy) $$
    *Note: The cross product is taken in 3D, but only the xy components are used for the 2D offset direction.*
6.  **Output Generation:** For each of the two intersection points:
    *   Append a vertex with `Pos = float4(intersections[i].Pos, 0, 1)` and `Velocity = intersections[i].Velocity`.
    *   Append a second vertex with `Pos = float4(intersections[i].Pos + (normal \cdot projSpacePixDim), 0, 1)` and the same `Velocity`.
7.  **Primitive Restart:** Call `triStream.RestartStrip()` to end the primitive strip.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for GS_GEN_BOUNDARY_VELOCITY
# Input: input[3] (triangle vertices with Pos.z and Velocity), sliceIdx, sliceZ
# Output: TriangleStream triStream

def GS_GEN_BOUNDARY_VELOCITY(input, sliceIdx, sliceZ):
    # 1. Bounding Box Check
    min_z = min(input[0].Pos.z, input[1].Pos.z, input[2].Pos.z)
    max_z = max(input[0].Pos.z, input[1].Pos.z, input[2].Pos.z)
    
    if sliceZ < min_z or sliceZ > max_z:
        return

    # 2. Initialize Intersections
    intersections = [None, None]
    idx = 0
    
    # 3. Find Intersections on Edges
    # Edge 0-1
    if idx < 2:
        idx = GetEdgePlaneIntersection(input[0], input[1], sliceZ, intersections, idx)
    # Edge 1-2
    if idx < 2:
        idx = GetEdgePlaneIntersection(input[1], input[2], sliceZ, intersections, idx)
    # Edge 2-0
    if idx < 2:
        idx = GetEdgePlaneIntersection(input[2], input[0], sliceZ, intersections, idx)

    # 4. Validation
    if idx < 2:
        return

    # 5. Compute Normal for Width
    # Cross product of two edge vectors in 3D, take xy
    edge1 = input[1].Pos - input[0].Pos
    edge2 = input[2].Pos - input[0].Pos
    cross_xy = cross(edge1, edge2).xy # Vector cross product in 3D, result is vector, take xy components
    normal = 1.414 * normalize(cross_xy) # sqrt(2) approx 1.414

    # 6. Output Vertices
    for i in range(2):
        # First vertex: intersection point
        v1 = OutputVertex()
        v1.Pos = float4(intersections[i].Pos, 0, 1)
        v1.Velocity = intersections[i].Velocity
        triStream.Append(v1)

        # Second vertex: offset point
        v2 = OutputVertex()
        offset = normal * projSpacePixDim
        v2.Pos = float4(intersections[i].Pos + offset, 0, 1)
        v2.Velocity = intersections[i].Velocity
        triStream.Append(v2)

    # 7. Restart Strip
    triStream.RestartStrip()

def GetEdgePlaneIntersection(vA, vB, sliceZ, intersections, idx):
    denom = vB.Pos.z - vA.Pos.z
    # Avoid division by zero if edge is parallel to slice plane
    if denom == 0:
        return idx
        
    t = (sliceZ - vA.Pos.z) / denom
    
    if t < 0 or t > 1:
        return idx
        
    # Linear interpolation
    # lerp(a, b, t) = a + t * (b - a)
    pos_xy = lerp_2d(vA.Pos.xy, vB.Pos.xy, t)
    vel = lerp_vec(vA.Velocity, vB.Velocity, t)
    
    intersections[idx].Pos = pos_xy
    intersections[idx].Velocity = vel
    return idx + 1
```

**Edge Cases and Pitfalls:**
*   **Parallel Edges:** If an edge is parallel to the slice plane ($vB.Pos.z = vA.Pos.z$), the denominator in $t$ is zero. The source code checks `t < 0 || t > 1` but does not explicitly handle division by zero. In practice, if $t$ is undefined, the intersection is not within the segment bounds for a single plane cut unless the edge lies *on* the plane, which is a degenerate case not explicitly handled in the provided snippet [S1, unofficial].
*   **Tangent Triangles:** If the triangle is tangent to the slice, only one intersection point might be found (or two coincident points), resulting in `idx < 2` and no output. This is correct behavior as a tangent does not create a volume segment.
*   **Precision:** The use of `sqrtOf2 = 1.414` is an approximation. For exact 1-voxel width, the exact diagonal length $\sqrt{2}$ should be used if `projSpacePixDim` represents the side length.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per triangle, as it performs a fixed number of arithmetic operations and edge checks.
*   **Memory Complexity:** $O(1)$ auxiliary memory per invocation.
*   **Strengths:** Efficiently generates velocity-carrying primitives directly on the GPU. Avoids CPU-side ray-triangle intersection calculations.
*   **Limitations:** Only handles triangle inputs. Does not resolve overlaps or multiple intersections within a single slice cell (relies on rasterization/accumulation). The fixed width (`sqrt(2) * pixDim`) may cause aliasing or gaps depending on the projection and resolution.

## Background
This method is described in the context of real-time 3D fluid simulation and rendering, specifically for voxelizing obstacle velocities. It is part of a pipeline that uses geometry shaders to accelerate the voxelization process by generating stream output buffers of transformed vertices [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
