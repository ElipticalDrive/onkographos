# Mesh Bounding Box Calculation

Also called: axis-aligned bounding box, min-max vertex bounds

## Overview
Computes the axis-aligned bounding box (AABB) that encloses a mesh by determining the minimum and maximum coordinates across all its vertices [S1, unofficial]. It differs from [Character Bounding Box Initialization](4016-character-bounding-box-initialization.md) by operating on explicit vertex lists rather than parametric primitives, and from [Volume Bounds Computation](5114-volume-bounds-computation.md) by targeting mesh geometry rather than arbitrary point clouds or volumetric data [S1, unofficial].

## How it works
**Inputs:** A list of 3D vertices $V = \{v_0, v_1, \dots, v_{n-1}\}$, where each vertex $v_i$ has coordinates $(x_i, y_i, z_i)$.
**Outputs:** An AABB defined by a minimum corner $mn = (mn_x, mn_y, mn_z)$ and a maximum corner $mx = (mx_x, mx_y, mx_z)$.

**Data Structures:**
The AABB is typically stored as two 3D vectors or arrays of size 3.
- `mn`: Array of 3 values representing the lower bounds.
- `mx`: Array of 3 values representing the upper bounds.
- `v`: Array of vertices, each containing 3 floating-point values.

**Steps:**
1. **Initialization:** If the vertex list is empty, the method is undefined [S1, unofficial]. Otherwise, initialize `mn` and `mx` to the coordinates of the first vertex $v_0$:
   $$ mn = v_0, \quad mx = v_0 $$
2. **Iteration:** Loop through each remaining vertex $v_i$ for $i$ from 1 to $n-1$.
3. **Update Bounds:** For each dimension $d \in \{0, 1, 2\}$ (corresponding to X, Y, Z):
   $$ mn_d = \min(v_i[d], mn_d) $$
   $$ mx_d = \max(v_i[d], mx_d) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mesh_aabb(vertices):
    """
    Computes the AABB for a mesh given a list of 3D vertices.
    
    Args:
        vertices: List of tuples/arrays, each with 3 floats (x, y, z).
        
    Returns:
        Tuple (min_corner, max_corner), each a tuple/array of 3 floats.
        
    Raises:
        ValueError: If vertices list is empty.
    """
    if not vertices:
        raise ValueError("Cannot compute AABB for empty vertex list.")
    
    # Initialize min and max with the first vertex
    min_corner = [vertices[0][0], vertices[0][1], vertices[0][2]]
    max_corner = [vertices[0][0], vertices[0][1], vertices[0][2]]
    
    # Iterate through the rest of the vertices
    for i in range(1, len(vertices)):
        v = vertices[i]
        
        # Update X bounds
        if v[0] < min_corner[0]:
            min_corner[0] = v[0]
        if v[0] > max_corner[0]:
            max_corner[0] = v[0]
            
        # Update Y bounds
        if v[1] < min_corner[1]:
            min_corner[1] = v[1]
        if v[1] > max_corner[1]:
            max_corner[1] = v[1]
            
        # Update Z bounds
        if v[2] < min_corner[2]:
            min_corner[2] = v[2]
        if v[2] > max_corner[2]:
            max_corner[2] = v[2]
            
    return min_corner, max_corner
```

**Parameters:**
- Precision: Sources use `float` or `double` depending on configuration flags (e.g., `USE_DOUBLE`) [S1, unofficial].
- Epsilon: A small constant `EPSILON` ($10^{-9}$) is defined in some implementations but not explicitly used in the basic min/max loop [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Input:** Accessing `v[0]` on an empty vertex list causes undefined behavior [S1, unofficial]. The implementation must check for an empty list before initialization.
- **Degenerate Mesh:** If all vertices are identical, the resulting AABB will have zero volume (min equals max). This is a valid result but may cause issues in subsequent intersection tests if not handled (e.g., division by zero in ray-box intersection if box extents are used as divisors).

## Subtopics

### [Bounding Cube Creation](4007-bounding-cube-creation.md)
Adjusts an AABB to be a perfect cube by expanding the smaller dimensions to match the largest dimension, centered on the original bounding box center. Used in voxelization and sparse voxel octree (SVO) construction to ensure uniform grid resolution [4007-bounding-cube-creation.md].

### [Character Bounding Box Initialization](4016-character-bounding-box-initialization.md)
Computes the AABB enclosing specific geometric primitives (arcs, line segments) for characters. Optimizes sphere tracing by providing tight bounds for ray-primitive intersection [4016-character-bounding-box-initialization.md].

### [Volume Bounds Computation](5114-volume-bounds-computation.md)
Computes an AABB for a set of 3D points (point clouds or samples), often with optional padding. Operates on arbitrary sample sets rather than mesh vertex lists [5114-volume-bounds-computation.md].

### AABB Vertex Coordinate Encoding
Determines the coordinates of any of the 8 vertices of an AABB using a vertex index (0-7) and bit manipulation, avoiding floating-point arithmetic for coordinate selection [S2, unofficial].
- **Logic:** Bit 0 of the index corresponds to X, Bit 1 to Y, Bit 2 to Z. If the bit is 0, the coordinate is the min value; if 1, it is the max value.
- **Formula:** For vertex index $i$, the coordinates are:
  $$ x = \text{minmax}_x[i \& 1] $$
  $$ y = \text{minmax}_y[(i \& 2) >> 1] $$
  $$ z = \text{minmax}_z[(i \& 4) >> 2] $$
  where `minmax_x` is an array `[x_min, x_max]` [S2, unofficial].

### AABB Enclosure (Union)
Computes the AABB enclosing two input AABBs ($b_0$ and $b_1$). The new AABB's min is the component-wise minimum of the inputs' mins, and the max is the component-wise maximum of the inputs' maxs [S2, unofficial].
- **Code:**
  ```python
  def enclosing_aabb(b0, b1):
      return {
          'min': [min(b0['min'][d], b1['min'][d]) for d in range(3)],
          'max': [max(b0['max'][d], b1['max'][d]) for d in range(3)]
      }
  ```

### Bounding Box and Camera Center Calculation
Computes the AABB of a vertex set and calculates an orthographic/perspective camera center position based on the max edge length and a 45-degree field of view [S3, unofficial].
- **Definition:** Calculates the AABB and then positions the camera along the Z-axis.
- **Steps:**
  1. Compute AABB ($min_x, max_x, min_y, max_y, min_z, max_z$) from vertices.
  2. Compute size vector $S = (max - min)$.
  3. Compute center $C_{xy} = (\frac{min_x+max_x}{2}, \frac{min_y+max_y}{2})$.
  4. Compute max edge length $E_{max} = \max(max_x - min_x, max_y - min_y)$.
  5. Calculate camera center Z coordinate $C_z$.
- **Formula:**
  $$ C_z = Z_{max} + \frac{E_{max}}{\tan(22.5^\circ)} $$
  where $C_z$ is the camera center Z coordinate, $Z_{max}$ is the maximum Z bound, and $E_{max}$ is the maximum X or Y edge length [S3, unofficial].
- **Layout:** Result stored in a `CenteredView` struct containing a transform matrix and camera center vector [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(n)$, where $n$ is the number of vertices. Each vertex is visited exactly once.
- **Memory Complexity:** $O(1)$ auxiliary space, requiring only two 3D vectors for min/max storage regardless of input size.
- **Strengths:** Extremely fast, simple to implement, and robust for static meshes.
- **Limitations:** Does not account for mesh orientation (only axis-aligned). For rotated meshes, the AABB may be significantly larger than the mesh's actual bounding volume, leading to less efficient culling in ray-box intersection tests compared to oriented bounding boxes (OBBs).
- **When to use:** When axis-aligned culling is sufficient, or as a first-pass approximation before more expensive calculations.
- **When not to use:** When tight bounds are critical for performance and the mesh is arbitrarily rotated, or when the mesh is dynamic and frequent updates are required (though this method is fast, other structures like BVHs might be better for complex scenes).

## Background
The method of computing an AABB by iterating over vertex coordinates is a fundamental technique in computer graphics, originating from the need for efficient spatial partitioning and collision detection. It is widely used in ray tracing, voxelization, and scene graph management. The specific implementation details, such as the bit-manipulation trick for vertex encoding, have been popularized in various online graphics programming resources and unofficial repositories [S2, unofficial].

## Sources
[S1] kctess5/voxelizer, https://github.com/kctess5/voxelizer, fetched 2026-10-26 (unofficial)
[S2] A Couple 3D AABB Tricks, https://gpfault.net/posts/aabb-tricks.html, fetched 2026-10-29 (unofficial)
[S3] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
