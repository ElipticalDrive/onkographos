# Mesh Bounding Box Calculation

Also called: Bounding Box Volume Texture Mapping, axis-aligned bounding box, min-max vertex bounds

## Overview
Computes the axis-aligned bounding box (AABB) that encloses a mesh by determining the minimum and maximum coordinates across all its vertices [S1, unofficial]. It provides a tight spatial bound for ray-primitive intersection tests and volume texture generation [S2, unofficial].

## How it works
**Inputs:** A list of 3D vertices $V = \{v_0, v_1, \dots, v_{n-1}\}$, where each vertex $v_i$ has coordinates $(x_i, y_i, z_i)$ [S1, unofficial].
**Outputs:** An AABB defined by a minimum corner $mn = (mn_x, mn_y, mn_z)$ and a maximum corner $mx = (mx_x, mx_y, mx_z)$ [S1, unofficial].

**Data Structures:**
The AABB is stored as two 3D vectors or arrays of size 3 [S1, unofficial].
- `mn`: Array of 3 values representing the lower bounds [S1, unofficial].
- `mx`: Array of 3 values representing the upper bounds [S1, unofficial].
- `v`: Array of vertices, each containing 3 floating-point values [S1, unofficial].

**Steps:**
1. **Initialization:** Initialize `mn` and `mx` to the coordinates of the first vertex $v_0$ [S1, unofficial].
2. **Iteration:** Loop through each remaining vertex $v_i$ for $i$ from 1 to $n-1$ [S1, unofficial].
3. **Update Bounds:** For each dimension $d \in \{0, 1, 2\}$ (corresponding to X, Y, Z), update $mn_d = \min(v_i[d], mn_d)$ and $mx_d = \max(v_i[d], mx_d)$ [S1, unofficial].
4. **Transformation:** Transform the 6 edge points of the bounding box with the model's world matrix to align it to the mesh's rotation, size, and position [S2, unofficial].
5. **Texture Fitting:** Order sampling points inside the untransformed bounding box, then fit the volume texture to the bounding box at runtime [S2, unofficial].

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
- Offset Vector: Stores the vector from the mesh's middle to the bounding box middle to correctly center the sampling [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Input:** Accessing `v[0]` on an empty vertex list causes undefined behavior [S1, unofficial]. The implementation must check for an empty list before initialization [G].
- **Degenerate Mesh:** If all vertices are identical, the resulting AABB will have zero volume (min equals max) [G]. This is a valid result but may cause issues in subsequent intersection tests if not handled (e.g., division by zero in ray-box intersection if box extents are used as divisors) [G].
- **Origin Mismatch:** The mesh's origin is not necessarily the middle of the bounding box, requiring an offset calculation [S2, unofficial].
- **SDF Padding:** The SDF size does not have to conform exactly to the bounding box; making it slightly larger introduces space with no mesh touching it [S2, unofficial].

## Subtopics

### [Bounding Cube Creation](4007-bounding-cube-creation.md)
Adjusts an AABB to be a perfect cube by expanding the smaller dimensions to match the largest dimension, centered on the original bounding box center. It is used in voxelization pipelines to ensure uniform grid resolution across all axes. Choose it when uniform grid spacing is required for downstream processing [G].

### [Character Bounding Box Initialization](4016-character-bounding-box-initialization.md)
Computes the AABB that encloses specific geometric primitives (arcs and line segments) representing characters. It optimizes sphere tracing queries by providing a tight spatial bound for ray-primitive intersection tests. Choose it when working with explicit parametric primitives rather than vertex lists [G].

### [Volume Bounds Computation](5114-volume-bounds-computation.md)
Computes an AABB that encloses a set of 3D points with optional padding. It operates on arbitrary point clouds or sample sets rather than mesh vertex coordinates. Choose it when defining bounds for existing volumetric data or point clouds [G].

### AABB Vertex Coordinate Encoding
Determines the coordinates of any of the 8 vertices of an AABB using a vertex index (0-7) and bit manipulation, avoiding floating-point arithmetic for coordinate selection [S3, unofficial]. Bit 0 of the index corresponds to X, Bit 1 to Y, Bit 2 to Z [S3, unofficial]. If the bit is 0, the coordinate is the min value; if 1, it is the max value [S3, unofficial]. For vertex index $i$, the coordinates are $x = \text{minmax}_x[i \& 1]$, $y = \text{minmax}_y[(i \& 2) >> 1]$, $z = \text{minmax}_z[(i \& 4) >> 2]$ [S3, unofficial].

### AABB Enclosure (Union)
Computes the AABB enclosing two input AABBs ($b_0$ and $b_1$) [S3, unofficial]. The new AABB's min is the component-wise minimum of the inputs' mins, and the max is the component-wise maximum of the inputs' maxs [S3, unofficial].
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def enclosing_aabb(b0, b1):
    return {
        'min': [min(b0['min'][d], b1['min'][d]) for d in range(3)],
        'max': [max(b0['max'][d], b1['max'][d]) for d in range(3)]
    }
```

### Bounding Box and Camera Center Calculation
Computes the AABB of a vertex set and calculates an orthographic/perspective camera center position based on the max edge length and a 45-degree field of view [S4, unofficial]. It initializes min/max to the first vertex, iterates through all vertices, computes the size vector and center midpoint, and calculates the maximum X or Y edge length [S4, unofficial]. The camera center Z coordinate is calculated as $C_z = Z_{max} + \frac{E_{max}}{\tan(22.5^\circ)}$ [S4, unofficial]. The result is stored in a `CenteredView` struct containing a transform matrix and camera center vector [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(n)$, where $n$ is the number of vertices, as each vertex is visited exactly once [G].
- **Memory Complexity:** $O(1)$ auxiliary space, requiring only two 3D vectors for min/max storage regardless of input size [G].
- **Strengths:** Extremely fast, simple to implement, and robust for static meshes [G].
- **Limitations:** Does not account for mesh orientation, so the AABB may be significantly larger than the mesh's actual bounding volume for rotated meshes [G]. This leads to less efficient culling in ray-box intersection tests compared to oriented bounding boxes (OBBs) [G].
- **When to use:** When axis-aligned culling is sufficient, or as a first-pass approximation before more expensive calculations [G].
- **When not to use:** When tight bounds are critical for performance and the mesh is arbitrarily rotated, or when the mesh is dynamic and frequent updates are required [G].

## Background
The method of computing an AABB by iterating over vertex coordinates is a fundamental technique in computer graphics, originating from the need for efficient spatial partitioning and collision detection [G]. It is widely used in ray tracing, voxelization, and scene graph management [G]. The bit-twiddling trick for vertex coordinate selection was independently discovered by graphics programmers and documented in informal online resources [S3, unofficial].

## Sources
[S1] kctess5/voxelizer, https://github.com/kctess5/voxelizer, fetched 2026-10-26 (unofficial)
[S2] Signed Distance Field Rendering Journey pt.1 – kosmonaut's blog, https://kosmonautblog.wordpress.com/2017/05/01/signed-distance-field-rendering-journey-pt-1/, fetched 2026-10-29 (unofficial)
[S3] A Couple 3D AABB Tricks, https://gpfault.net/posts/aabb-tricks.html, fetched 2026-10-29 (unofficial)
[S4] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
