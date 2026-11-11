# Triangle Rendering with Thickness

Also called: Triangle Thickness Rasterization

## Overview
Triangle Rendering with Thickness renders a triangle as a volumetric shell by voxelizing grid points that lie within a specified distance from the triangle's plane or edges. It differs from solid triangle rasterization by including points outside the triangle boundary (up to the thickness limit) and from sphere-based thickness methods by using planar geometry defined by three vertices. This method is a specialized case of mesh rasterization where the primitive is a polygon.

## How it works
**Inputs**: Triangle vertices $v_1, v_2, v_3$ (3D vectors), thickness $T$ (scalar), grid resolution and bounds (implicit from context).
**Outputs**: A list of voxel indices or a density field where points within the thickness are marked.

**Data Structures**:
- `Vec3<T>`: 3D vector with components `x, y, z`.
- `Bucket`: A subset of the grid defined by integer bounds $[min_i, max_i] \times [min_j, max_j] \times [min_k, max_k]$.

**Steps**:
1. **Bounding Box**: Compute the axis-aligned bounding box (AABB) of the triangle vertices $v_1, v_2, v_3$.
2. **Bucket Selection**: Calculate the grid indices for the AABB extended by the thickness $T$ in all directions. Let this set of indices be $B$.
3. **Geometry Precomputation**:
   - Compute edge vectors: $e_{21} = v_2 - v_1$, $e_{32} = v_3 - v_2$, $e_{13} = v_1 - v_3$.
   - Compute squared magnitudes: $m_{21} = \|e_{21}\|^2$, $m_{32} = \|e_{32}\|^2$, $m_{13} = \|e_{13}\|^2$.
   - Compute the triangle normal: $n = e_{21} \times e_{13}$.
   - Compute squared normal magnitude: $m_n = \|n\|^2$.
   - Compute cross products for edge orientation checks: $c_{21} = e_{21} \times n$, $c_{32} = e_{32} \times n$, $c_{13} = e_{13} \times n$.
4. **Voxel Evaluation**: For each voxel center $p$ in bucket $B$:
   - Compute vectors from vertices: $p_1 = p - v_1$, $p_2 = p - v_2$, $p_3 = p - v_3$.
   - **Inside Check**: Determine if $p$ projects inside the triangle using barycentric sign checks. The source code uses a specific (and potentially buggy) implementation:
     $$ inside = (\text{sgn}(c_{21} \cdot p_1) + \text{sgn}(c_{21} \cdot p_1) + \text{sgn}(c_{21} \cdot p_1)) < 2.0 $$
     *Note: The source code repeats $c_{21}$ three times, which effectively checks the sign of one edge three times. A correct implementation would use $c_{21}, c_{32}, c_{13}$ respectively [S1, unofficial].*
   - **Distance Calculation**:
     - If $inside$ is true: Calculate the squared distance to the closest edge.
       $$ d^2 = \min \left( \begin{aligned} &\| e_{21} \cdot \text{clamp}\left(\frac{e_{21} \cdot p_1}{m_{21}}, 0, 1\right) - p_1 \|^2, \\ &\| e_{32} \cdot \text{clamp}\left(\frac{e_{32} \cdot p_2}{m_{32}}, 0, 1\right) - p_2 \|^2, \\ &\| e_{13} \cdot \text{clamp}\left(\frac{e_{13} \cdot p_3}{m_{13}}, 0, 1\right) - p_3 \|^2 \end{aligned} \right) $$
     - If $inside$ is false: Calculate the squared distance to the triangle's plane.
       $$ d^2 = \frac{(n \cdot p_1)^2}{m_n} $$
   - **Thickness Test**: If $d^2 < T^2$, add the voxel to the render list.

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_triangle_thickness(v1, v2, v3, thickness, grid_bounds):
    # 1. Bounding Box
    min_v = min(v1, v2, v3)
    max_v = max(v1, v2, v3)
    
    # 2. Bucket Indices (extended by thickness)
    # calculateIndicesFromBB returns list of (i, j, k)
    bucket_indices = calculate_indices_from_bb(min_v, max_v, thickness, grid_bounds)
    
    # 3. Precompute Geometry
    e21 = v2 - v1
    e32 = v3 - v2
    e13 = v1 - v3
    
    m21 = dot(e21, e21)
    m32 = dot(e32, e32)
    m13 = dot(e13, e13)
    
    normal = cross(e21, e13)
    mag_normal = dot(normal, normal)
    
    c21 = cross(e21, normal)
    c32 = cross(e32, normal)
    c13 = cross(e13, normal)
    
    thickness_sq = thickness * thickness
    
    render_list = []
    
    for pt in bucket_indices:
        p1 = pt - v1
        p2 = pt - v2
        p3 = pt - v3
        
        # Inside Check (Source Implementation)
        # Note: Source repeats c21 three times. Correct logic would use c21, c32, c13.
        s1 = sign(dot(c21, p1))
        s2 = sign(dot(c21, p1)) # Bug in source: should be c32
        s3 = sign(dot(c21, p1)) # Bug in source: should be c13
        
        inside = (s1 + s2 + s3) < 2.0
        
        dist_sq = 0.0
        
        if inside:
            # Distance to edges
            # Edge 1: v1-v2
            proj1 = clamp(dot(e21, p1) / m21, 0.0, 1.0)
            vec1 = e21 * proj1 - p1
            d1_sq = dot(vec1, vec1)
            
            # Edge 2: v2-v3
            p2_local = pt - v2
            proj2 = clamp(dot(e32, p2_local) / m32, 0.0, 1.0)
            vec2 = e32 * proj2 - p2_local
            d2_sq = dot(vec2, vec2)
            
            # Edge 3: v3-v1
            p3_local = pt - v3
            proj3 = clamp(dot(e13, p3_local) / m13, 0.0, 1.0)
            vec3 = e13 * proj3 - p3_local
            d3_sq = dot(vec3, vec3)
            
            dist_sq = min(d1_sq, d2_sq, d3_sq)
        else:
            # Distance to plane
            dist_sq = (dot(normal, p1) ** 2) / mag_normal
            
        if dist_sq < thickness_sq:
            render_list.append(pt)
            
    return render_list
```

**Parameters**:
- `thickness`: The distance from the triangle surface. Typical values depend on grid resolution; often $0.5$ to $1.0$ grid units for anti-aliasing or shell rendering.

**Edge Cases**:
- **Degenerate Triangle**: If $m_n \approx 0$, the normal is zero, causing division by zero in the plane distance calculation. Guard against this by checking $m_n < \epsilon$.
- **Precision**: Floating point errors in `clamp` or dot products may cause jitter at boundaries.

## Subtopics

### [Sphere Rendering with Thickness](12912-sphere-rendering-with-thickness.md)
Sphere Rendering with Thickness renders a sphere as a hollow shell by voxelizing grid points that lie between an inner and outer radius. It differs from solid sphere rendering by excluding the core volume, and from triangle-based thickness methods by using a spherical coordinate system defined by a center point and a single radius parameter. This method is a specialized case of mesh rasterization where the primitive is a sphere rather than a polygon.

## Operating characteristics
- **Time Complexity**: $O(N \cdot T^3)$, where $N$ is the number of triangles and $T$ is the thickness in grid units. The complexity is driven by the bounding box expansion.
- **Memory Complexity**: $O(V)$, where $V$ is the number of voxels in the expanded bounding boxes.
- **Strengths**: Simple to implement; produces smooth volumetric edges; suitable for thin-shell rendering.
- **Limitations**: The source implementation contains a bug in the inside/outside test (repeating the same edge check three times) [S1, unofficial]. Performance degrades with large thickness values relative to triangle size.
- **When to use**: When rendering wireframe-like structures or thin shells from triangle meshes.
- **When not to use**: For solid volume rendering where interior density is required; for high-performance applications where the bug in the inside check leads to incorrect geometry.

## Background
This method is derived from an unofficial volumetric display implementation found in a GitHub repository [S1, unofficial]. It represents a direct rasterization approach to creating volumetric primitives from mesh data, specifically focusing on the "thickness" aspect to create hollow or shell-like structures.

## Sources
[S1] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
