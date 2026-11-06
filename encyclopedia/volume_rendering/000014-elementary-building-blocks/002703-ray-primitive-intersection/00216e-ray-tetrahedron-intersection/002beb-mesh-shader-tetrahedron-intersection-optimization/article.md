# Mesh Shader Tetrahedron Intersection Optimization

Also called: Accelerated intersections using wave intrinsics within a mesh shader, Optimized Cyrus-Beck for tetrahedra

## Overview
This method accelerates ray-tetrahedron intersection testing by leveraging mesh shader wave intrinsics to share geometric data across the four vertices of a single tetrahedron, reducing redundant memory loads and arithmetic operations [S1, unofficial]. It improves upon the naive per-vertex loading approach by precomputing plane parameters in the mesh shader and solving the Cyrus-Beck inequalities in the fragment shader, achieving a runtime cost of only 12 FLOPs for the intersection logic [S1, unofficial].

## How it works
The method splits the intersection calculation into two stages: a mesh shader stage that computes plane parameters for each face of the tetrahedron, and a fragment shader stage that solves the linear inequalities of the Cyrus-Beck algorithm using interpolated values [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Ray origin $\mathbf{o}_{ray}$ and direction $\mathbf{d}_{ray}$ [S1, unofficial].
- Vertex positions of the tetrahedron, accessible via wave intrinsics [S1, unofficial].
- `tetVertexId`: The index of the current vertex within the tetrahedron (0–3) [S1, unofficial].

**Outputs:**
- `o.planeN`: A vector of 4 floats containing the numerator of the plane equation for each face [S1, unofficial].
- `o.planeD`: A vector of 4 floats containing the denominator (dot product of normal and ray direction) for each face [S1, unofficial].

### Data Structures
**Static Lookup Table:**
A constant array `kTetTriangles` of 4 `uint4` vectors defines the vertex indices for the 4 faces of a tetrahedron [S1, unofficial]. The indices are ordered to ensure outward-facing normals [S1, unofficial].
$$
\text{kTetTriangles} = \begin{bmatrix}
0, 2, 1, 3 \\
1, 2, 3, 0 \\
0, 3, 2, 1 \\
3, 0, 1, 2
\end{bmatrix}
$$
Each row `kTetTriangles[i]` corresponds to the face opposite vertex `i` [S1, unofficial]. The first three elements define the triangle vertices, and the fourth is unused in the cross product but part of the `uint4` layout [S1, unofficial].

### Algorithm Steps

1. **Retrieve Face Indices:** In the mesh shader, for the current vertex `tetVertexId`, retrieve the face vertex indices from `kTetTriangles[tetVertexId]` [S1, unofficial]. Let these be $v_0, v_1, v_2$ (the first three components) [S1, unofficial].
2. **Access Vertices via Wave Intrinsics:** Use wave read intrinsics to access the positions of the tetrahedron's vertices [S1, unofficial]. Since all 4 vertices of the tetrahedron are processed in the same wave group, each lane can access any vertex's position [S1, unofficial]. Let $\mathbf{P}_k$ be the position of vertex $k$, and the vertices for the current face be $\mathbf{P}_{v_0}, \mathbf{P}_{v_1}, \mathbf{P}_{v_2}$ [S1, unofficial].
3. **Compute Outward Normal:** Calculate the outward-facing normal $\mathbf{n}$ for the current face using the cross product of two edge vectors: $\mathbf{n} = (\mathbf{P}_{v_2} - \mathbf{P}_{v_0}) \times (\mathbf{P}_{v_1} - \mathbf{P}_{v_0})$ [S1, unofficial]. This normal is computed in each lane but represents the same geometric plane for the tetrahedron face [S1, unofficial].
4. **Compute Plane Numerator:** Select the first vertex of the face, $\mathbf{v} = \mathbf{P}_{v_0}$ [S1, unofficial]. Compute the numerator for the plane equation: $\text{num} = \mathbf{n} \cdot (\mathbf{v} - \mathbf{o}_{ray})$ [S1, unofficial].
5. **Broadcast Numerator:** Broadcast the scalar `num` to all lanes in the wave group to populate the output vector `o.planeN` [S1, unofficial]. Specifically, lane $i$ stores `num` in `o.planeN[i]` [S1, unofficial].
6. **Compute Plane Denominator:** For each lane $i$, compute the denominator by dotting the normal $\mathbf{n}$ (which is identical across lanes for this face) with the ray direction $\mathbf{d}_{ray}$: $\text{planeD}_i = \mathbf{n} \cdot \mathbf{d}_{ray}$ [S1, unofficial]. Store these in `o.planeD` [S1, unofficial].
7. **Fragment Shader Normalization:** In the fragment shader, normalize the denominator by the length of the ray direction: $\text{planeD}_i \leftarrow \frac{\text{planeD}_i}{\|\mathbf{d}_{ray}\|}$ [S1, unofficial].
8. **Compute Intersection Parameters:** For each plane $i \in \{0, 1, 2, 3\}$, compute the intersection parameter $t_i$: $t_i = \frac{\text{planeN}_i}{\text{planeD}_i}$ [S1, unofficial].
9. **Determine Entry and Exit Distances:** Initialize $t_{enter} = -\infty$ and $t_{exit} = +\infty$ [S1, unofficial]. For each plane $i$: if $\text{planeD}_i > 0$, update $t_{enter} = \max(t_{enter}, t_i)$; if $\text{planeD}_i < 0$, update $t_{exit} = \min(t_{exit}, t_i)$ [S1, unofficial]. If $\text{planeD}_i = 0$, the ray is parallel to the plane and does not contribute to the bounds [S1, unofficial].
10. **Result:** The ray intersects the tetrahedron if $t_{enter} < t_{exit}$ and $t_{exit} > 0$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper definitions for wave intrinsics (conceptual hardware mapping)
def wave_broadcast(value):
    """Returns the value replicated across all 4 lanes."""
    return [value] * 4

def wave_read_lane_at(value, lane_idx):
    """Hardware wave shuffle; returns the value from the specified lane."""
    return value

# Mesh Shader Stage
def mesh_shader_intersect(ray_origin, ray_dir, tet_vertex_positions, tet_vertex_id):
    # Static lookup table for outward-facing faces
    kTetTriangles = [
        (0, 2, 1, 3),
        (1, 2, 3, 0),
        (0, 3, 2, 1),
        (3, 0, 1, 2)
    ]
    
    # Get face indices for the current tetrahedron vertex
    face_indices = kTetTriangles[tet_vertex_id]
    v0_idx, v1_idx, v2_idx = face_indices[0], face_indices[1], face_indices[2]
    
    # Access vertex positions using wave intrinsics
    P_v0 = wave_read_lane_at(tet_vertex_positions, v0_idx)
    P_v1 = wave_read_lane_at(tet_vertex_positions, v1_idx)
    P_v2 = wave_read_lane_at(tet_vertex_positions, v2_idx)
    
    # Compute outward normal n
    edge1 = P_v2 - P_v0
    edge2 = P_v1 - P_v0
    n = cross(edge1, edge2)
    
    # Compute numerator num = n . (v - rayOrigin)
    v = P_v0
    num = dot(n, v - ray_origin)
    
    # Broadcast num to all lanes for planeN
    planeN = wave_broadcast(num)
    
    # Compute denominator planeD = n . rayDir for each lane
    planeD = [dot(wave_read_lane_at(n, i), ray_dir) for i in range(4)]
    
    return planeN, planeD

def fragment_shader_intersect(planeN, planeD, ray_dir):
    # Normalize denominator by ray direction length
    ray_len = length(ray_dir)
    normalized_planeD = [d / ray_len for d in planeD]
    
    # Compute t for each plane
    all_t = []
    for i in range(4):
        if normalized_planeD[i] == 0.0:
            all_t.append(float('inf'))
        else:
            all_t.append(planeN[i] / normalized_planeD[i])
    
    # Compute t_enter and t_exit
    t_enter = float('-inf')
    t_exit = float('inf')
    
    for i in range(4):
        if normalized_planeD[i] > 0.0:
            if all_t[i] > t_enter:
                t_enter = all_t[i]
        elif normalized_planeD[i] < 0.0:
            if all_t[i] < t_exit:
                t_exit = all_t[i]
                
    return t_enter, t_exit
```

### Parameters
- **kTetTriangles**: Static constant array defining vertex indices for outward-facing normals [S1, unofficial].
- **Wave Size**: The method assumes a wave size of at least 4 to process all 4 vertices of the tetrahedron in a single wave group [S1, unofficial].

### Edge Cases and Pitfalls
- **Redundant Loads**: The naive implementation loads tetrahedron data for each vertex separately [S1, unofficial]. This optimization avoids that by sharing data via wave intrinsics, reducing memory loads by a factor of four [S1, unofficial].
- **Perspective-Correct Interpolation**: The source notes that interpolation between distances must be perspective-correct barycentric interpolation to ensure correctness, although the core Cyrus-Beck logic described here operates on precomputed plane parameters in the mesh shader [S1, unofficial]. The 12 FLOP count applies to the optimized Cyrus-Beck steps [S1, unofficial].
- **Parallel Rays**: If `planeD` is zero, the ray is parallel to the plane [S1, unofficial]. The algorithm handles this by excluding it from the max/min updates, which is correct for convex polyhedra [S1, unofficial].

## Operating characteristics
- **Time Complexity**: The optimized implementation reduces the runtime cost to 12 FLOPs for the intersection calculation [S1, unofficial].
- **Memory Efficiency**: By sharing information between vertices in the mesh shader, the method reduces memory loads by a factor of four compared to the naive per-vertex approach [S1, unofficial].
- **Strengths**: Significant reduction in arithmetic and memory bandwidth usage [S1, unofficial]. Suitable for high-performance volumetric rendering where tetrahedral meshes are dense [S1, unofficial].
- **Limitations**: Requires hardware support for mesh shaders and wave intrinsics [S1, unofficial]. The optimization is specific to tetrahedral primitives [S1, unofficial].

## Background
This method was introduced in a preprint describing an optimized implementation of the Cyrus-Beck algorithm for computing ray-tetrahedron intersections using mesh shaders [S1, unofficial]. It builds on the standard Cyrus-Beck algorithm for ray-convex polygon intersection, adapting it for tetrahedral meshes using modern GPU mesh shader capabilities [S1, unofficial]. The goal was to accelerate the intersection test, which is a bottleneck in raytracing irregular volumes [S1, unofficial].

## Sources
[S1] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
