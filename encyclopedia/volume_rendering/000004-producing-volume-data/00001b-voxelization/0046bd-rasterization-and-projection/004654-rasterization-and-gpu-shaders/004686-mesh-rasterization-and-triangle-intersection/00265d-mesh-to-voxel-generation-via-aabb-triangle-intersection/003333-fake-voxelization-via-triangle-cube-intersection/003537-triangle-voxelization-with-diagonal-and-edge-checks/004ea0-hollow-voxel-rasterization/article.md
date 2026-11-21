# Hollow Voxel Rasterization

Also called: Hollow Voxelization, Voxel Shell Calculation, hollow

## Overview
Hollow Voxel Rasterization is a fast, memory-efficient method for generating a voxel shell (hollow interior) from an arbitrary polyhedron, rather than a solid volume [S1, unofficial]. It is specifically suited for shapes intended to be hollow or meshes with fine structures at low target resolutions [S1, unofficial]. Unlike solid voxelization methods that fill the interior, this approach inverts the containment test: it marks a voxel as part of the mesh if any mesh face lies inside the voxel's bounding box, effectively capturing the surface geometry without occupying internal voxels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** An arbitrary polyhedron defined by a vertex array and an index buffer (triangles), and a target voxel grid dimension `dim` (`glm::ivec3`) [S1, unofficial].
- **Output:** A 3D buffer of `uint8_t` values (`_voxels`) of size `dim`, where non-zero values indicate shell voxels [S1, unofficial]. The output structure also retains the original polyhedron data [S1, unofficial].

**Data Structures:**
The output is encapsulated in a struct `raster_results<buffer3d<uint8_t>, mesh::polyhedron<base_t>>` containing:
- `_arr_dim`: The dimensions of the voxel grid (`glm::vec3`) [S1, unofficial].
- `_voxels`: A 3D buffer of `uint8_t` representing the voxel shell [S1, unofficial].
- `_mesh`: The input `mesh::polyhedron<base_t>`, which contains `_vertices._vertex_arr` (positions) and `_indices._buffer` (triangle indices) [S1, unofficial].

**Parameters:**
- `dim`: Target voxel grid dimensions. Used to scale the polyhedron and allocate the output buffer [S1, unofficial].

**Algorithm Steps:**

1.  **Normalization and Scaling:**
    Normalize the input polyhedron and scale it to fit the target voxel dimensions `dim` [S1, unofficial]. This ensures the mesh coordinates align with the voxel grid [S1, unofficial].

2.  **Iteration Setup:**
    Iterate over each triangle face of the polyhedron [S1, unofficial]. For each face, compute its 3D bounding box in voxel space:
    $$ l_{min} = \lfloor \min(\text{face vertices}) \rfloor - 0.5 $$
    $$ l_{max} = \lceil \max(\text{face vertices}) \rceil + 0.5 $$
    $$ l_{steps} = l_{max} - l_{min} $$
    Shift the face vertices relative to $l_{min}$ to work in local bounding box coordinates [S1, unofficial].

3.  **Voxel Loop:**
    Iterate over all voxels within the computed bounding box using three nested loops for $x, y, z$ from $0$ to $l_{steps}.x/y/z$ [S1, unofficial].
    Calculate global voxel coordinates:
    $$ n_x = x + l_{min}.x $$
    $$ n_y = y + l_{min}.y $$
    $$ n_z = z + l_{min}.z $$

4.  **Skip Check:**
    Check if the voxel at $(n_x, n_y, n_z)$ is already marked (non-zero) [S1, unofficial]. If it is, skip further processing for this voxel to avoid double-counting [S1, unofficial].

5.  **Shell Intersection Test:**
    Determine if the current face intersects the voxel hexahedron [S1, unofficial]. This is done by checking if the face is inside the voxel's bounding box [S1, unofficial].
    - Calculate `half_box_size` = voxel size / 2 [S1, unofficial].
    - Use a separating axis test (SAT) or similar geometric check (`face_in_hexahedron`) with a half-voxel offset to determine if the face penetrates the voxel volume [S1, unofficial].
    - If the test returns true, mark the voxel as a shell voxel (set `_voxels[n_x, n_y, n_z]` to a non-zero value, e.g., 1) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hollow_voxel_rasterization(polyhedron, dim):
    """
    polyhedron: object with _vertices._vertex_arr (list of vec3) and _indices._buffer (list of ints)
    dim: tuple (x, y, z) of integers
    Returns: 3D array of uint8_t
    """
    # 1. Normalize and scale polyhedron to dim
    polyhedron.normalize()
    polyhedron.scale(dim)
    
    vertices = polyhedron._vertices._vertex_arr
    indices = polyhedron._indices._buffer
    stride = polyhedron._indices._stride
    num_faces = len(indices) // stride
    
    # Initialize output buffer
    voxels = [[[0 for _ in range(dim.z)] for _ in range(dim.y)] for _ in range(dim.x)]
    
    half_box_size = vec3(0.5, 0.5, 0.5)
    
    for face_id in range(num_faces):
        idx_start = face_id * stride
        v1 = vertices[indices[idx_start]]
        v2 = vertices[indices[idx_start + 1]]
        v3 = vertices[indices[idx_start + 2]]
        
        # 2. Compute bounding box in voxel space
        min_coords = vec3(min(v1.x, v2.x, v3.x), min(v1.y, v2.y, v3.y), min(v1.z, v2.z, v3.z))
        max_coords = vec3(max(v1.x, v2.x, v3.x), max(v1.y, v2.y, v3.y), max(v1.z, v2.z, v3.z))
        lmin = vec3(floor(min_coords.x), floor(min_coords.y), floor(min_coords.z)) - vec3(0.5, 0.5, 0.5)
        lmax = vec3(ceil(max_coords.x), ceil(max_coords.y), ceil(max_coords.z)) + vec3(0.5, 0.5, 0.5)
        lsteps = lmax - lmin
        
        # 3. Iterate over voxels in bounding box
        for x in range(int(lsteps.x)):
            for y in range(int(lsteps.y)):
                for z in range(int(lsteps.z)):
                    nx = int(x + lmin.x)
                    ny = int(y + lmin.y)
                    nz = int(z + lmin.z)
                    
                    # 4. Skip if already marked
                    if voxels[nx][ny][nz] != 0:
                        continue
                    
                    # 5. Test if face is inside voxel bounding box
                    voxel_center = vec3(nx + 0.5, ny + 0.5, nz + 0.5)
                    if face_in_hexahedron(v1, v2, v3, voxel_center, half_box_size):
                        voxels[nx][ny][nz] = 1
                        
    return voxels

def face_in_hexahedron(v1, v2, v3, voxel_center, half_box_size):
    """
    Checks if a triangle face is inside the voxel hexahedron defined by center and half_size.
    Uses Separating Axis Theorem (SAT) logic as described in sources.
    """
    # Translate vertices relative to voxel center
    p1 = v1 - voxel_center
    p2 = v2 - voxel_center
    p3 = v3 - voxel_center
    
    edges = [v2 - v1, v3 - v2, v1 - v3]
    
    # Test face normals (X, Y, Z axes)
    for axis_idx in range(3):
        max_val = max(p1[axis_idx], p2[axis_idx], p3[axis_idx])
        min_val = min(p1[axis_idx], p2[axis_idx], p3[axis_idx])
        if min_val > half_box_size[axis_idx] or max_val < -half_box_size[axis_idx]:
            return False
            
    # Test edge axes (cross products of edges with X, Y, Z axes)
    axes = []
    for e in edges:
        axes.append(cross(e, vec3(1, 0, 0)))
        axes.append(cross(e, vec3(0, 1, 0)))
        axes.append(cross(e, vec3(0, 0, 1)))
        
    for axis in axes:
        if length(axis) == 0:
            continue
            
        # Project triangle vertices
        proj1 = dot(p1, axis)
        proj2 = dot(p2, axis)
        proj3 = dot(p3, axis)
        t_min = min(proj1, proj2, proj3)
        t_max = max(proj1, proj2, proj3)
        
        # Project box
        box_extent = abs(axis.x * half_box_size.x) + abs(axis.y * half_box_size.y) + abs(axis.z * half_box_size.z)
        
        if t_min > box_extent or t_max < -box_extent:
            return False
            
    return True
```

**Edge Cases and Pitfalls:**
- **Double-counting:** Voxels intersected by multiple faces are marked only once. The check `if voxels[nx, ny, nz] != 0` prevents redundant processing [S1, unofficial].
- **Precision:** Floating-point precision in bounding box calculations and intersection tests can lead to gaps or overlaps in the shell, especially at low resolutions [G].
- **Degenerate Input:** Triangles with zero area or collinear vertices may cause issues in the intersection test. The source implies robust handling via the SAT approach, but degenerate cases should be filtered or handled explicitly [G].

## Operating characteristics
- **Time Complexity:** $O(F \cdot V_{local})$, where $F$ is the number of faces and $V_{local}$ is the average number of voxels in the bounding box of a face. In the worst case, this is $O(F \cdot N^3)$, but typically much less due to bounding box culling [G].
- **Memory Complexity:** $O(N^3)$ for the output voxel buffer, where $N$ is the grid dimension. This is significantly less than solid voxelization if the interior is not stored, but the buffer size is fixed by $N$ [G].
- **Performance:** On an Intel i7 4770k (4 physical cores), rasterizing a Stanford Bunny mesh (112k faces) at $1024^3$ resolution takes 1956 ms in `--hollow` mode [S1, unofficial].
- **Scaling:** Performance scales linearly with the number of faces and cubically with resolution. The `--hollow` mode is consistently faster than `--fast` (solid) mode at higher resolutions (e.g., 1956 ms vs 2918 ms at $1024^3$) [S1, unofficial].
- **Strengths:** Very fast, memory-efficient for shell-only representations, suitable for fine structures at low resolutions [S1, unofficial].
- **Limitations:** Does not provide interior volume data. Accuracy depends on resolution; fine details may be lost if the voxel grid is too coarse [G].
- **When to use:** When only the surface shell is needed, or for meshes with fine structures where solid voxelization would be too memory-intensive [S1, unofficial].
- **When not to use:** When interior volume information is required, or for high-fidelity solid models [G].

## Background
Hollow Voxel Rasterization is implemented in the `3DStuff/voxelizer` project [S1, unofficial]. It is presented as an alternative to solid voxelization methods (`--fast`, `--safe`), offering a trade-off between performance and memory usage [S1, unofficial]. The method is particularly useful for applications where the interior of the voxelized object is not needed, such as in certain rendering or simulation contexts where only the surface geometry matters [G].

## Sources
[S1] 3DStuff/voxelizer, https://github.com/3DStuff/voxelizer, fetched 2026-10-27 (unofficial)
