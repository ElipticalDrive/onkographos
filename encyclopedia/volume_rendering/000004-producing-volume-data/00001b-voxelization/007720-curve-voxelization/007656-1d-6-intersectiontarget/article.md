# 1D_6_intersectionTarget

Also called: 1D mesh ensuring 6-connectivity, Diagonal mesh target for 1D inputs

## Overview
The `1D_6_intersectionTarget` method generates a specific 3D mesh geometry used as an "intersection target" within each candidate voxel during the voxelization of 1D inputs (lines, polylines, or curves) [S1]. Its primary purpose is to guarantee 6-connectivity (face-adjacency) in the resulting voxel set by ensuring that any curve passing through a voxel intersects this internal mesh [S1]. Unlike the cube outline target, which traces the voxel edges, this method uses a diagonal mesh structure composed of triangles to determine intersection [S1]. It is a core component of the Curve Voxelization pipeline, specifically designed to resolve the topological connectivity of the output volume data [S1].

## How it works
The algorithm constructs a static mesh defined relative to the center of a voxel. This mesh consists of 9 vertices and 12 triangular faces. The construction relies on the voxel's center point and half-dimensions along the X, Y, and Z axes.

### Inputs and Outputs
*   **Input:**
    *   `vSize`: A 3D vector representing the voxel dimensions $(X, Y, Z)$ [S1].
    *   `VoxelPoint`: A 3D point representing the center of the voxel [S1].
*   **Output:**
    *   `IntersectionTarget`: A mesh object containing the defined vertices and faces [S1].

### Data Structures
The mesh is defined by two arrays:
1.  **Vertices**: An array of 9 3D points (`Point3d[9]`) [S1].
2.  **Faces**: A list of 12 triangles, where each triangle is defined by three indices into the `Vertices` array [S1].

### Parameters
*   $u = \frac{vSize.X}{2}$
*   $v = \frac{vSize.Y}{2}$
*   $w = \frac{vSize.Z}{2}$

These represent the half-width, half-height, and half-depth of the voxel, respectively [S1].

### Steps
1.  **Initialize**: Create an empty mesh object `IntersectionTarget`. Calculate $u, v, w$. Initialize the `Vertices` array of size 9. Set `Vertices[0]` to `VoxelPoint` [S1].
2.  **Create Vertices**: Compute the remaining 8 vertices by adding or subtracting the half-dimensions from `VoxelPoint`. The indices correspond to specific corners/edges relative to the center:
    *   `Vertices[1]` = `VoxelPoint` $- (u, v, w)$
    *   `Vertices[2]` = `VoxelPoint` $- (-u, v, w)$
    *   `Vertices[3]` = `VoxelPoint` $- (-u, -v, w)$
    *   `Vertices[4]` = `VoxelPoint` $- (u, -v, w)$
    *   `Vertices[5]` = `VoxelPoint` $+ (-u, -v, w)$
    *   `Vertices[6]` = `VoxelPoint` $+ (u, -v, w)$
    *   `Vertices[7]` = `VoxelPoint` $+ (u, v, w)$
    *   `Vertices[8]` = `VoxelPoint` $+ (-u, v, w)$
    [S1]
3.  **Create Faces**: Add 12 triangles to the mesh using the vertex indices. The faces are defined as follows:
    *   $(0, 1, 2)$
    *   $(0, 2, 6)$
    *   $(0, 6, 5)$
    *   $(0, 5, 1)$
    *   $(0, 4, 1)$
    *   $(0, 5, 8)$
    *   $(0, 8, 7)$
    *   $(0, 7, 3)$
    *   $(0, 3, 4)$
    *   $(0, 4, 8)$
    *   $(0, 7, 6)$
    *   $(0, 2, 3)$
    [S1]
4.  **Return**: Attach the `Vertices` and `Faces` to the `IntersectionTarget` mesh and return it [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def 1D_6_intersectionTarget(vSize, VoxelPoint):
    # 1. Initialize
    u = vSize.X / 2.0
    v = vSize.Y / 2.0
    w = vSize.Z / 2.0
    
    # Vertices array of size 9
    Vertices = [None] * 9
    Vertices[0] = VoxelPoint
    
    # 2. Create vertices
    # Note: Vector3d operations are standard vector arithmetic
    Vertices[1] = VoxelPoint - Vector3d(+u, +v, +w)
    Vertices[2] = VoxelPoint - Vector3d(-u, +v, +w)
    Vertices[3] = VoxelPoint - Vector3d(-u, -v, +w)
    Vertices[4] = VoxelPoint - Vector3d(+u, -v, +w)
    Vertices[5] = VoxelPoint + Vector3d(-u, -v, +w)
    Vertices[6] = VoxelPoint + Vector3d(+u, -v, +w)
    Vertices[7] = VoxelPoint + Vector3d(+u, +v, +w)
    Vertices[8] = VoxelPoint + Vector3d(-u, +v, +w)
    
    # 3. Create faces
    # Each tuple represents a triangle (v1, v2, v3)
    Faces = [
        (0, 1, 2), (0, 2, 6),
        (0, 6, 5), (0, 5, 1),
        (0, 4, 1), (0, 5, 8),
        (0, 8, 7), (0, 7, 3),
        (0, 3, 4), (0, 4, 8),
        (0, 7, 6), (0, 2, 3)
    ]
    
    # 4. Return
    IntersectionTarget = Mesh()
    IntersectionTarget.Vertices = Vertices
    IntersectionTarget.Faces = Faces
    return IntersectionTarget
```

### Edge Cases and Pitfalls
*   **Triangle Count**: The source explicitly states that this target produces 12 triangles and that this number "cannot be reduced" for this specific 6-connected result type [S1].
*   **Quadrangular vs Triangular**: The source notes that a naive implementation might produce 12 quadrangular faces (24 triangles), but the algorithm described here specifically constructs 12 triangles [S1]. Implementers should ensure they use the triangular decomposition provided, not a quadrangular one, for efficiency and correctness as defined in the source [S1].
*   **Connectivity**: This target is strictly for 6-connectivity. It is not suitable for 26-connectivity requirements without modification [S1].

## Subtopics

### Cube Outline Target
An alternative intersection target that uses the edges of the voxel cube. It is compared against the diagonal mesh target in terms of performance and connectivity guarantees [S1].

### 2D Diagonal Mesh Target
A 2D analog of this method, used for 2D inputs. It uses 8 lines compared to 12 lines for the outline, offering a 1.5x speedup in that specific 2D context [S1].

## Operating characteristics
*   **Complexity**: The construction of the mesh is $O(1)$ per voxel, as it involves a fixed number of arithmetic operations and array assignments (9 vertices, 12 faces) [S1].
*   **Performance**: The source compares the diagonal mesh target to the cube outline target. While a specific speedup factor for the 1D case is not explicitly quantified in the same way as the 2D case (where diagonal was 1.5x faster), the source notes that the diagonal target has 12 triangles and cannot be reduced, implying a fixed computational cost per voxel for intersection testing [S1]. The 2D comparison suggests that fewer primitives (lines/triangles) generally lead to faster intersection solving [S1].
*   **Memory**: Requires storage for 9 vertices and 12 faces per voxel candidate [S1].
*   **Strengths**: Guarantees 6-connectivity for 1D inputs [S1].
*   **Limitations**: The fixed number of triangles (12) means it cannot be optimized further by reducing face count for this connectivity type [S1].

## Background
The `1D_6_intersectionTarget` algorithm is described in the context of voxelization algorithms for geospatial applications [S1]. It is part of a broader methodology for converting 1D geometric primitives (lines, polylines) into 3D voxel grids with guaranteed topological connectivity [S1]. The specific algorithm is labeled as "Algorithm 5" in the source literature [S1].

## Sources
[S1] Voxelization algorithms for geospatial applications, https://www.sciencedirect.com/science/article/pii/S2215016116000029, fetched 2026-10-29
