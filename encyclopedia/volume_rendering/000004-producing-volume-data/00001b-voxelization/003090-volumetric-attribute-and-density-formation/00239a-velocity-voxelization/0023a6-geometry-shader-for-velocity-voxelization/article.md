# Geometry Shader for Velocity Voxelization


## Overview
This method computes the obstacle velocity at each grid cell containing part of the obstacle's boundary by projecting 3D triangle intersections onto 2D slice planes [S1, unofficial]. It differs from standard voxelization by outputting a 1-voxel wide quadrilateral for each intersecting triangle edge, explicitly encoding the interpolated velocity vector at the intersection point [S1, unofficial]. This allows the subsequent rasterization or accumulation step to directly populate velocity fields without post-processing interpolation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A single triangle defined by three vertices (`input[0]`, `input[1]`, `input[2]`), each containing a position `Pos` (with z-component) and a velocity `Velocity` [S1, unofficial]. The current slice index `sliceIdx` and the slice plane's z-coordinate `sliceZ` [S1, unofficial].
*   **Output:** A stream of vertices forming two triangles (a quadrilateral) if the input triangle intersects the slice plane, or no output if it does not [S1, unofficial]. The quadrilateral is 1-voxel wide in screen space [S1, unofficial].

**Data Structures:**
*   `VsGenVelOutput`: Struct containing `Pos` (float4, xy from intersection, z=0, w=1) and `Velocity` (interpolated) [S1, unofficial].
*   `GsGenVelIntVtx`: Struct containing `Pos` (float2) and `Velocity` (interpolated) [S1, unofficial].
*   `intersections[2]`: Array of two `GsGenVelIntVtx` elements to store the two intersection points found on the triangle edges [S1, unofficial].

**Parameters:**
*   `projSpacePixDim`: The pixel dimension in projection space [S1, unofficial]. The source uses a constant `sqrtOf2 = 1.414` as a proxy for the diagonal of a pixel, implying the width is derived from this diagonal magnitude [S1, unofficial].

**Algorithm Steps:**
1.  **Bounding Box Check:** Calculate `minZ` and `maxZ` of the three input triangle vertices [S1, unofficial]. If `sliceZ` is outside `[minZ, maxZ]`, the triangle does not intersect the slice; return immediately [S1, unofficial].
2.  **Intersection Finding:** Initialize an `intersections` array and an index `idx = 0` [S1, unofficial]. Iterate through the three edges of the triangle (0-1, 1-2, 2-0) [S1, unofficial]. For each edge, call `GetEdgePlaneIntersection` [S1, unofficial].
3.  **Edge Intersection Logic (`GetEdgePlaneIntersection`):**
    *   Calculate the parameter $t$ for the line-plane intersection:
        $$ t = \frac{sliceZ - vA.Pos.z}{vB.Pos.z - vA.Pos.z} $$
    *   If $t < 0$ or $t > 1$, the intersection is not within the segment endpoints; return without modifying the array [S1, unofficial].
    *   Otherwise, compute the 2D position and velocity by linear interpolation:
        $$ Pos = \text{lerp}(vA.Pos, vB.Pos, t).xy $$
        $$ Velocity = \text{lerp}(vA.Velocity, vB.Velocity, t) $$
    *   Store the result in `intersections[idx]` and increment `idx` [S1, unofficial].
4.  **Validation:** If fewer than 2 intersections are found (`idx < 2`), the triangle does not form a valid segment across the slice (or is tangent/degenerate); return [S1, unofficial].
5.  **Normal Calculation:** Compute the 2D normal for offsetting the segment to create the quadrilateral width [S1, unofficial]. The source specifies:
    $$ \text{normal} = \sqrt{2} \cdot \text{normalize}(\text{cross}((input[1].Pos - input[0].Pos), (input[2].Pos - input[0].Pos)).xy) $$
    *Note: The cross product is taken in 3D, but only the xy components are used for the 2D offset direction* [S1, unofficial].
6.  **Output Generation:** For each of the two intersection points:
    *   Append a vertex with `Pos = float4(intersections[i].Pos, 0, 1)` and `Velocity = intersections[i].Velocity` [S1, unofficial].
    *   Append a second vertex with `Pos = float4(intersections[i].Pos + (normal \cdot projSpacePixDim), 0, 1)` and the same `Velocity` [S1, unofficial].
7.  **Primitive Restart:** Call `triStream.RestartStrip()` to end the primitive strip [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def GS_GEN_BOUNDARY_VELOCITY(input, sliceIdx, sliceZ, projSpacePixDim, triStream):
    min_z = min(input[0].Pos.z, input[1].Pos.z, input[2].Pos.z)
    max_z = max(input[0].Pos.z, input[1].Pos.z, input[2].Pos.z)
    if sliceZ < min_z or sliceZ > max_z:
        return

    intersections = [None, None]
    idx = 0
    if idx < 2: idx = GetEdgePlaneIntersection(input[0], input[1], sliceZ, intersections, idx)
    if idx < 2: idx = GetEdgePlaneIntersection(input[1], input[2], sliceZ, intersections, idx)
    if idx < 2: idx = GetEdgePlaneIntersection(input[2], input[0], sliceZ, intersections, idx)
    if idx < 2: return

    edge1 = (input[1].Pos.x - input[0].Pos.x, input[1].Pos.y - input[0].Pos.y, input[1].Pos.z - input[0].Pos.z)
    edge2 = (input[2].Pos.x - input[0].Pos.x, input[2].Pos.y - input[0].Pos.y, input[2].Pos.z - input[0].Pos.z)
    cross_x = edge1[1]*edge2[2] - edge1[2]*edge2[1]
    cross_y = edge1[2]*edge2[0] - edge1[0]*edge2[2]
    length = (cross_x**2 + cross_y**2)**0.5
    normal = (cross_x/length * 1.414, cross_y/length * 1.414)

    for i in range(2):
        v1 = Vertex()
        v1.Pos = (intersections[i].Pos.x, intersections[i].Pos.y, 0.0, 1.0)
        v1.Velocity = intersections[i].Velocity
        triStream.Append(v1)

        v2 = Vertex()
        v2.Pos = (intersections[i].Pos.x + normal[0]*projSpacePixDim, intersections[i].Pos.y + normal[1]*projSpacePixDim, 0.0, 1.0)
        v2.Velocity = intersections[i].Velocity
        triStream.Append(v2)

    triStream.RestartStrip()

def GetEdgePlaneIntersection(vA, vB, sliceZ, intersections, idx):
    t = (sliceZ - vA.Pos.z) / (vB.Pos.z - vA.Pos.z)
    if t < 0 or t > 1: return idx
    intersections[idx].Pos = (vA.Pos.x + t*(vB.Pos.x - vA.Pos.x), vA.Pos.y + t*(vB.Pos.y - vA.Pos.y))
    intersections[idx].Velocity = vA.Velocity + t*(vB.Velocity - vA.Velocity)
    return idx + 1
```

**Edge Cases and Pitfalls:**
*   **Parallel Edges:** If an edge is parallel to the slice plane ($vB.Pos.z = vA.Pos.z$), the denominator in $t$ is zero. The source code checks `t < 0 || t > 1` but does not explicitly handle division by zero. In practice, if $t$ is undefined, the intersection is not within the segment bounds for a single plane cut unless the edge lies *on* the plane, which is a degenerate case not explicitly handled in the provided snippet [S1, unofficial].
*   **Tangent Triangles:** If the triangle is tangent to the slice, only one intersection point might be found (or two coincident points), resulting in `idx < 2` and no output. This is correct behavior as a tangent does not create a volume segment [S1, unofficial].
*   **Precision:** The use of `sqrtOf2 = 1.414` is an approximation. For exact 1-voxel width, the exact diagonal length $\sqrt{2}$ should be used if `projSpacePixDim` represents the side length [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per triangle, as it performs a fixed number of arithmetic operations and edge checks [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per invocation [S1, unofficial].
*   **Strengths:** Efficiently generates velocity-carrying primitives directly on the GPU. Avoids CPU-side ray-triangle intersection calculations [S1, unofficial].
*   **Limitations:** Only handles triangle inputs. Does not resolve overlaps or multiple intersections within a single slice cell (relies on rasterization/accumulation). The fixed width (`sqrt(2) * pixDim`) may cause aliasing or gaps depending on the projection and resolution [S1, unofficial].

## Background
This method is described in the context of real-time 3D fluid simulation and rendering, specifically for voxelizing obstacle velocities [S1, unofficial]. It is part of a pipeline that uses geometry shaders to accelerate the voxelization process by generating stream output buffers of transformed vertices [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
