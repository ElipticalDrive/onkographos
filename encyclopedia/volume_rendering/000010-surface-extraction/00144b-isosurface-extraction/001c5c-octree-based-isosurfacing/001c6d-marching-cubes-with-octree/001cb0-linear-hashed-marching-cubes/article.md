# Linear Hashed Marching Cubes

Also called: Pointerless Octree Duals (loose implementation)

## Overview
Linear Hashed Marching Cubes is a surface extraction method that generates meshes from implicit volumes by applying the Marching Cubes algorithm over a linear hashed octree structure [S1, unofficial]. It serves as a high-performance alternative to standard grid-based Marching Cubes, aiming to reduce memory overhead and traversal time by using a pointerless, hash-based octree representation [S1, unofficial]. It builds on [Marching Cubes with Octree](1259-marching-cubes-with-octree.md) but replaces the standard octree traversal with a linear hashed variant for faster performance [S1, unofficial].

## How it works
**Inputs:**
- An implicit scalar field function $f(x, y, z)$ that returns the signed distance or density value at any point in 3D space.
- An isovalue threshold $T$ used to classify vertices as inside or outside the surface.
- `max_depth`: The maximum depth of the internal octree, defining the resolution limit. The tree spans a cubic grid of size $2^{\text{max\_depth}}$ in each dimension [S1, unofficial].

**Outputs:**
- A triangle mesh consisting of vertices and indices representing the isosurface.

**Data Structures:**
- **ImplicitOctree**: A pointerless octree structure that stores voxel data. Unlike traditional octrees, it uses a linear hash table for node storage, eliminating pointer overhead [S1, unofficial].
- **MortonKey**: A 64-bit integer representing the position of a voxel in the 3D grid using Morton (Z-order) curve encoding. This key is used to index into the linear hash table and to identify vertices in the mesh builder [S1, unofficial].
- **MeshTopologyBuilder**: A temporary structure that accumulates vertex positions and face indices during traversal. It uses `MortonKey` identifiers to track vertex uniqueness and ensure proper mesh connectivity [S1, unofficial].

**Algorithm Steps:**
1. **Initialization**: Create an `ImplicitOctree` instance with the specified `max_depth`. Initialize a `MeshTopologyBuilder` to store the resulting mesh data [S1, unofficial].
2. **Traversal**: Traverse the `ImplicitOctree`. For each visited cube (voxel) in the octree, retrieve its `MortonKey`, the 8 corner coordinates, and the scalar values at those corners [S1, unofficial].
3. **Classification**: Compute the sign of the scalar value at each of the 8 corners relative to the isovalue $T$. If $f(\text{corner}) < T$, the corner is classified as "inside" (0); otherwise, it is "outside" (1). This produces an 8-bit cube index (0–255) [S1, unofficial].
4. **Edge Intersection**: Based on the cube index, determine which of the 12 edges of the cube are intersected by the isosurface. For each intersected edge, calculate the exact intersection point using linear interpolation between the corner values [S1, unofficial].
   - Let $v_1$ and $v_2$ be the scalar values at the two endpoints of an edge, and $p_1$ and $p_2$ be their 3D positions. The intersection point $p_{int}$ is:
     $$ p_{int} = p_1 + \frac{T - v_1}{v_2 - v_1} (p_2 - p_1) $$
   - If $v_1 = v_2$, the edge is parallel to the isosurface; handle as a degenerate case (e.g., skip or use a small epsilon) [G].
5. **Mesh Construction**: Iterate through the triangles defined by the cube index (from the standard Marching Cubes lookup table). For each triangle with vertices $a, b, c$:
   - Assign a unique `MortonKey` to each vertex based on its position in the grid.
   - Add the vertex positions to the `MeshTopologyBuilder`.
   - Add the face (triangle) to the builder, referencing the vertex indices [S1, unofficial].
6. **Finalization**: After all cubes have been traversed, build the final mesh from the `MeshTopologyBuilder`. Extract the vertex positions and index buffer to produce the final triangle mesh [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_hashed_marching_cubes(iso_func, isovalue, max_depth):
    # Initialize structures
    octree = ImplicitOctree(max_depth=max_depth)
    mesh_builder = MeshTopologyBuilder()
    
    # Traverse the octree
    for cube in octree.traverse():
        # Get cube data
        key = cube.morton_key
        corners = cube.corners  # List of 8 (x, y, z) positions
        values = [iso_func(c) for c in corners]
        
        # Classify corners
        cube_index = 0
        for i in range(8):
            if values[i] < isovalue:
                cube_index |= (1 << i)
        
        # Find edge intersections
        edge_intersections = find_edge_intersections(corners, values, isovalue, cube_index)
        
        # Generate triangles based on cube_index
        triangles = get_triangles_for_cube(cube_index)
        
        for tri in triangles:
            # tri is a list of 3 edge indices
            verts = []
            for edge_idx in tri:
                # Get the vertex position for this edge intersection
                # The MortonKey for the vertex is derived from its grid position
                pos = edge_intersections[edge_idx]
                verts.append(pos)
            
            # Add to mesh builder
            mesh_builder.add_triangle(verts)
    
    # Build final mesh
    mesh = mesh_builder.build()
    return mesh

def find_edge_intersections(corners, values, isovalue, cube_index):
    intersections = {}
    # Define the 12 edges of a cube as pairs of corner indices
    edges = [
        (0, 1), (1, 2), (2, 3), (3, 0),  # Bottom face
        (4, 5), (5, 6), (6, 7), (7, 4),  # Top face
        (0, 4), (1, 5), (2, 6), (3, 7)   # Vertical edges
    ]
    
    for edge_idx, (i, j) in enumerate(edges):
        # Check if this edge is intersected
        if (cube_index & (1 << i)) != (cube_index & (1 << j)):
            v1 = values[i]
            v2 = values[j]
            p1 = corners[i]
            p2 = corners[j]
            
            # Linear interpolation
            if abs(v2 - v1) < 1e-9:
                # Degenerate case: edge is parallel to isosurface
                # Use midpoint as approximation [G]
                t = 0.5
            else:
                t = (isovalue - v1) / (v2 - v1)
            
            p_int = p1 + t * (p2 - p1)
            intersections[edge_idx] = p_int
            
    return intersections
```

**Parameters:**
- `max_depth`: Controls the maximum resolution. The grid size is $2^{\text{max\_depth}} \times 2^{\text{max\_depth}} \times 2^{\text{max\_depth}}$ [S1, unofficial]. Typical values depend on the desired detail and memory constraints.

**Edge Cases and Pitfalls:**
- **Degenerate Edges**: If the scalar values at both ends of an edge are equal, the interpolation factor is undefined. A small epsilon or midpoint approximation should be used [G].
- **Hash Collisions**: The linear hashed octree must handle hash collisions efficiently. The implementation should use a robust hashing strategy (e.g., double hashing or chaining) to ensure correct traversal [G].
- **Boundary Conditions**: The octree traversal must correctly handle the boundaries of the defined volume. Vertices on the boundary should be handled consistently to avoid gaps or artifacts [G].

## Subtopics

### [Adaptive Resolution Isosurface Rendering with Two-Pass Discontinuity Handling](1340-adaptive-resolution-isosurface-rendering-with-two-pass-disco.md)
Focuses on adaptive resolution and handling discontinuities in two passes. Choose when high-quality adaptive rendering is needed.

### [DMC Octree Contour Extraction](3661-dmc-octree-contour-extraction.md)
Uses Dual Marching Cubes on an octree for smoother surfaces. Choose when smoother surfaces are required and performance is less critical.

## Operating characteristics
- **Time Complexity**: Depends on the number of visited octree nodes. Linear hashed octrees can offer faster traversal than pointer-based octrees due to better cache locality [S1, unofficial].
- **Memory Complexity**: Significantly reduced compared to pointer-based octrees due to the lack of pointer overhead. The linear hash table stores only necessary nodes [S1, unofficial].
- **Performance**: Reported to be faster than standard Marching Cubes [S1, unofficial].
- **Strengths**: High performance, reduced memory usage, suitability for large volumes.
- **Limitations**: Potential for hash collisions, complexity in implementation compared to standard grid-based methods.
- **When to Use**: Large volumes where memory and performance are critical.
- **When Not to Use**: Small volumes where the overhead of octree construction is not justified, or when simpler implementations are preferred.

## Background
The method is described as a "loose implementation" of the paper "Fast Generation of Pointerless Octree Duals" [S1, unofficial]. It was developed to improve the performance of isosurface extraction by leveraging the efficiency of linear hashed octrees [S1, unofficial]. The source is an unofficial GitHub repository [S1, unofficial].

## Sources
[S1] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
