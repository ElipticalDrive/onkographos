# Plane Grid Generation

Also called: Axis-Aligned Plane Vertex Generation

## Overview
Plane Grid Generation constructs the vertex connectivity for surfaces aligned with the principal coordinate planes (xy, yz, xz) on a regular 3D grid. It solves the problem of efficiently defining planar geometry for isosurface slicing or structured volume rendering without performing geometric intersection tests. Unlike general isosurface extraction methods such as Marching Cubes, this method assumes fixed axis-aligned topology, enabling $O(1)$ index computation per triangle based on grid coordinates [S1, unofficial].

## How it works
**Inputs:**
- Grid dimensions: $X, Y, Z$ (number of vertices along each axis).
- Current vertex index: $currIndex$ (linear index of the first vertex of the plane).
- Vertex stride: $xVerts$ (number of vertices along the X-axis, typically equal to $X$).

**Outputs:**
- A triangle defined by three vertex indices: $A, B, C$.
- A plane normal vector $N$.

**Data Structures:**
- Vertex indices are integers representing positions in a linear vertex buffer.
- Vectors are 3D Cartesian coordinates.

**Steps:**
1. **Calculate Edge Vector U:** Compute the vector from vertex $A$ to vertex $B$.
   $$ U = \text{Position}(currIndex + xVerts) - \text{Position}(currIndex) $$
   In a regular grid with unit spacing, this vector typically aligns with the X-axis.

2. **Calculate Edge Vector V:** Compute the vector from vertex $A$ to vertex $C$.
   $$ V = \text{Position}(currIndex + xVerts + 1) - \text{Position}(currIndex) $$
   In a regular grid, this vector typically aligns with the Y-axis (for an xy-plane slice).

3. **Compute Normal N:** Calculate the cross product of $U$ and $V$ to determine the plane's orientation.
   $$ N = U \times V $$
   The direction of $N$ depends on the winding order of vertices $A, B, C$.

4. **Define Face Vertices:** Assign the vertex indices for the triangle.
   $$ A = currIndex $$
   $$ B = currIndex + xVerts $$
   $$ C = currIndex + xVerts + 1 $$

5. **Initialize Plane State:** Set the plane's position and rotation parameters to zero or their base grid coordinates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_plane_triangle(curr_index, x_verts, get_vertex_position):
    """
    Generates a single triangle for an axis-aligned plane grid.
    
    Args:
        curr_index (int): The linear index of the first vertex (A).
        x_verts (int): The number of vertices along the X-axis (stride).
        get_vertex_position (function): Returns (x, y, z) for a given linear index.
        
    Returns:
        tuple: (indices, normal) where indices is (A, B, C) and normal is (nx, ny, nz).
    """
    # Step 1: Get positions for vertices A, B, and C
    pos_A = get_vertex_position(curr_index)
    pos_B = get_vertex_position(curr_index + x_verts)
    pos_C = get_vertex_position(curr_index + x_verts + 1)
    
    # Step 2: Calculate edge vectors U and V
    # U is vector from A to B
    u = [pos_B[i] - pos_A[i] for i in range(3)]
    
    # V is vector from A to C
    v = [pos_C[i] - pos_A[i] for i in range(3)]
    
    # Step 3: Compute normal N as cross product of U and V
    # N = U x V
    n = [
        u[1] * v[2] - u[2] * v[1],
        u[2] * v[0] - u[0] * v[2],
        u[0] * v[1] - u[1] * v[0]
    ]
    
    # Step 4: Define face vertices
    indices = (curr_index, curr_index + x_verts, curr_index + x_verts + 1)
    
    # Step 5: Return results
    return indices, n
```

**Parameters:**
- $xVerts$: Must match the grid width. If the grid is not uniform, this value must be adjusted accordingly [G].
- $currIndex$: Must be within valid bounds such that $currIndex + xVerts + 1$ does not exceed the total vertex count.

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Ensure $currIndex$ does not refer to vertices on the far edge of the grid if the plane is meant to be internal.
- **Winding Order:** The order $A, B, C$ determines the normal direction. Reversing $B$ and $C$ flips the normal. Consistency is required for correct culling or lighting [S1, unofficial].
- **Degenerate Input:** If $U$ or $V$ is zero (e.g., grid dimension is 1), the normal will be zero. This should be handled by skipping such planes or using a default normal [G].

## Subtopics

### [Structured Grid Triangle List Generation](8189-structured-grid-triangle-list-generation.md)
Precomputes vertex index arrays for triangle lists representing surfaces parallel to the xy, yz, and xz planes on a regular 3D grid. It solves the problem of efficiently generating the connectivity for isosurface or slice rendering by providing static index buffers that can be offset for different grid depths. It differs from general isosurface extraction (e.g., Marching Cubes) by assuming axis-aligned planar topology, allowing for $O(1)$ index computation per triangle without geometric intersection tests.

## Operating characteristics
- **Time Complexity:** $O(1)$ per triangle for index calculation. Generating a full plane of size $W \times H$ requires $O(W \cdot H)$ operations.
- **Memory Complexity:** $O(1)$ additional memory if indices are computed on the fly. $O(W \cdot H)$ if storing a precomputed index buffer.
- **Strengths:** Extremely fast index generation; no geometric intersection tests; simple implementation.
- **Limitations:** Only works for axis-aligned planes; cannot handle curved or arbitrary isosurfaces.
- **When to use:** When rendering slices of a volume along principal axes or generating structured grid surfaces.
- **When not to use:** When arbitrary isosurface extraction is required.

## Background
The method is derived from techniques used in voxel-based game engines and volume rendering systems where axis-aligned slicing is common. The specific implementation details, including the calculation of edge vectors and normals for plane generation, are documented in the context of a Minecraft-like voxel engine implementation [S1, unofficial].

## Sources
[S1] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
