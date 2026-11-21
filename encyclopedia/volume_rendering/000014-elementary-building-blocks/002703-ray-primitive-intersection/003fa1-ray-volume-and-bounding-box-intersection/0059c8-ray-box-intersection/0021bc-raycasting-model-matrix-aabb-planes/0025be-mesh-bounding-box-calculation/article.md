# Mesh Bounding Box Calculation

Also called: Bounding Box Volume Texture Mapping, axis-aligned bounding box, min-max vertex bounds

## Overview
Mesh Bounding Box Calculation computes the axis-aligned bounding box (AABB) that encloses a mesh by determining the minimum and maximum coordinates across all its vertices [S1, unofficial][S2, unofficial]. It solves the problem of establishing a tight spatial volume for ray-primitive intersection tests, voxelization, or volume texture generation. Unlike [Character Bounding Box Initialization](4016-character-bounding-box-initialization.md) which targets specific character primitives, or [Volume Bounds Computation](5114-volume-bounds-computation.md) which generalizes to arbitrary point clouds, this method operates specifically on mesh vertex data [S1, unofficial][S2, unofficial][G]. It differs from [Host-side Bounding Box Computation](13468-host-side-bounding-box-computation.md) by producing a single global AABB for the entire mesh rather than per-triangle bounds [S2, unofficial][G].

## How it works
**Inputs:**
- A list of vertices $V = \{v_0, v_1, \dots, v_{n-1}\}$, where each vertex $v_i$ is a 3D vector $(x, y, z)$.
- Optional: A model transformation matrix $M$ (if world-space bounds are required).

**Outputs:**
- A global AABB defined by two 3D vectors: $\mathbf{min}$ and $\mathbf{max}$, representing the lower-left-front and upper-right-back corners of the box in the mesh's local coordinate system.
- Optional: An offset vector $\mathbf{offset}$ from the mesh's geometric center to the AABB's center, used for volume texture alignment [S2, unofficial].

**Data Structures:**
- `Vec3`: A structure containing three floating-point components (x, y, z).
- `AABB`: A structure containing two `Vec3` fields: `min` and `max`.

**Steps:**
1. **Initialization**: If the vertex list is empty, the behavior is undefined [S1, unofficial]. Otherwise, initialize $\mathbf{min}$ and $\mathbf{max}$ to the coordinates of the first vertex $v_0$ [S1, unofficial].
2. **Iteration**: Loop through each vertex $v_i$ from index $1$ to $n-1$. For each dimension $d \in \{0, 1, 2\}$ (representing x, y, z):
   - Update $\mathbf{min}[d] = \min(v_i[d], \mathbf{min}[d])$.
   - Update $\mathbf{max}[d] = \max(v_i[d], \mathbf{max}[d])$.
3. **Offset Calculation (Optional)**: Compute the center of the AABB as $\mathbf{c}_{aabb} = \frac{\mathbf{min} + \mathbf{max}}{2}$. Compute the center of the mesh vertices as $\mathbf{c}_{mesh} = \frac{1}{n}\sum_{i=0}^{n-1} v_i$. The offset vector is $\mathbf{offset} = \mathbf{c}_{aabb} - \mathbf{c}_{mesh}$ [S2, unofficial]. This is used to correctly center sampling points if the mesh origin does not coincide with the bounding box center [S2, unofficial].
4. **World Transformation (Optional)**: If world-space bounds are needed, transform the 8 corner points of the AABB using the model matrix $M$. The 6 edge points (or 8 corners) are transformed to align the box with the mesh's rotation, size, and position in the world [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mesh_aabb(vertices):
    """
    Computes the axis-aligned bounding box for a list of 3D vertices.
    
    Args:
        vertices: List of Vec3 objects, each with x, y, z attributes.
        
    Returns:
        A tuple (min_vec, max_vec) representing the AABB bounds.
    """
    if not vertices:
        raise ValueError("Empty vertex list causes undefined behavior")
    
    # Step 1: Initialize min and max with the first vertex
    min_vec = Vec3(vertices[0].x, vertices[0].y, vertices[0].z)
    max_vec = Vec3(vertices[0].x, vertices[0].y, vertices[0].z)
    
    # Step 2: Iterate through remaining vertices
    for i in range(1, len(vertices)):
        v = vertices[i]
        
        # Update min for each dimension
        if v.x < min_vec.x: min_vec.x = v.x
        if v.y < min_vec.y: min_vec.y = v.y
        if v.z < min_vec.z: min_vec.z = v.z
        
        # Update max for each dimension
        if v.x > max_vec.x: max_vec.x = v.x
        if v.y > max_vec.y: max_vec.y = v.y
        if v.z > max_vec.z: max_vec.z = v.z
        
    return min_vec, max_vec

def compute_offset(vertices, min_vec, max_vec):
    """
    Computes the offset from the mesh center to the AABB center.
    
    Args:
        vertices: List of Vec3 objects.
        min_vec, max_vec: The computed AABB bounds.
        
    Returns:
        Vec3 offset vector.
    """
    if not vertices:
        return Vec3(0, 0, 0)
        
    # Calculate AABB center
    aabb_center_x = (min_vec.x + max_vec.x) / 2.0
    aabb_center_y = (min_vec.y + max_vec.y) / 2.0
    aabb_center_z = (min_vec.z + max_vec.z) / 2.0
    
    # Calculate mesh center (average of all vertices)
    sum_x = sum(v.x for v in vertices)
    sum_y = sum(v.y for v in vertices)
    sum_z = sum(v.z for v in vertices)
    
    mesh_center_x = sum_x / len(vertices)
    mesh_center_y = sum_y / len(vertices)
    mesh_center_z = sum_z / len(vertices)
    
    # Offset vector
    offset_x = aabb_center_x - mesh_center_x
    offset_y = aabb_center_y - mesh_center_y
    offset_z = aabb_center_z - mesh_center_z
    
    return Vec3(offset_x, offset_y, offset_z)
```

**Parameters:**
- **Padding**: The SDF size does not have to conform exactly to the bounding box; making it slightly larger introduces space with no mesh touching it, which can be useful for subsequent operations [S2, unofficial].
- **Precision**: The implementation may use `float` or `double` precision, defined by a compile-time flag or type alias (e.g., `precision_type`) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Input**: Accessing `v[0]` on an empty vertex list causes undefined behavior [S1, unofficial].
- **Origin Mismatch**: The mesh's origin is not necessarily the middle of the bounding box. Failing to account for this offset can lead to misaligned volume textures or sampling errors [S2, unofficial].
- **Precision**: Using single-precision floats may introduce small errors in tight bounds; double precision is available but incurs higher memory and compute costs [S1, unofficial].

## Subtopics

### [Bounding Cube Creation](4007-bounding-cube-creation.md)
Bounding Cube Creation adjusts an axis-aligned bounding box (AABB) to be a perfect cube by expanding the smaller dimensions to match the largest dimension, centered on the original bounding box center. This method is used in voxelization pipelines to ensure uniform grid resolution across all axes. It differs from sibling methods like [Character Bounding Box Initialization](4016-character-bounding-box-initialization.md) which may account for specific character proportions, and [Volume Bounds Computation](5114-volume-bounds-computation.md) which may define bounds for existing volumetric data rather than mesh primitives [G].

### [Character Bounding Box Initialization](4016-character-bounding-box-initialization.md)
Character Bounding Box Initialization computes the axis-aligned bounding box (AABB) enclosing a set of geometric primitives representing specific characters ('S', 'A', 'N', 'G') and arcs. It optimizes sphere tracing queries by providing a tight spatial bound for ray-primitive intersection tests.

### [Volume Bounds Computation](5114-volume-bounds-computation.md)
Volume Bounds Computation computes an Axis-Aligned Bounding Box (AABB) that encloses a set of 3D points, optionally applying padding to ensure the volume fully contains the geometry with a safety margin. It serves as the foundational step for initializing voxel grids, spatial indices, and rendering volumes in implicit and explicit volume rendering pipelines. Unlike [Mesh Bounding Box Calculation](3612-mesh-bounding-box-calculation.md), which operates on mesh vertices, this method generalizes to any point cloud or sampled data structure. It differs from [Silhouette-based Bounding Box Initialization](5448-silhouette-based-bounding-box-initialization.md) by requiring explicit 3D coordinates rather than inferring bounds from 2D projections.

### [Host-side Bounding Box Computation](13468-host-side-bounding-box-computation.md)
Host-side Bounding Box Computation computes the axis-aligned bounding box (AABB) for each triangle in a mesh on the CPU host, producing separate arrays for minimum and maximum coordinates. It serves as the prerequisite step for GPU-based voxelization by providing the spatial extent of each primitive before ray-primitive intersection or volume rasterization. It differs from [Mesh Bounding Box Calculation](3612-mesh-bounding-box-calculation.md), which computes a single global AABB for the entire mesh, by maintaining per-triangle bounds.

### AABB Vertex Coordinate Encoding
AABB Vertex Coordinate Encoding is a method to determine the coordinates of any of the 8 vertices of an AABB using a vertex index (0-7) and bit manipulation, avoiding floating-point arithmetic for coordinate selection [S3, unofficial]. The AABB has 8 vertices. For any vertex, each coordinate is either the min or max value for that axis. Bit 0 of the vertex index corresponds to the X axis, bit 1 to Y, bit 2 to Z. If the bit is 0, the coordinate is the min value; if 1, it is the max value [S3, unofficial]. This allows extracting vertex coordinates using bit masking and shifting to index the minmax arrays [S3, unofficial].

### AABB Enclosure (Union)
To compute the AABB enclosing two input AABBs ($b_0$ and $b_1$), create a new AABB where each axis's min is the minimum of the two inputs' mins, and each axis's max is the maximum of the two inputs' maxs [S3, unofficial]. This is implemented using component-wise min/max on the minmax arrays [S3, unofficial].

### Bounding Box and Camera Center Calculation
This variant computes the axis-aligned bounding box of a set of vertices and calculates an orthographic/perspective camera center position based on the max edge length and a 45-degree field of view [S4, unofficial]. It initializes min/max values, iterates through vertices, computes the size vector, and calculates the center as the midpoint of min/max [S4, unofficial]. The camera center Z coordinate is calculated using the formula:
$$C_z = Z_{max} + \frac{E_{max}}{\tan(22.5^\circ)}$$
where $C_z$ is the camera center Z coordinate, $Z_{max}$ is the maximum Z bound, and $E_{max}$ is the maximum X or Y edge length [S4, unofficial]. The result is stored in a `CenteredView` struct containing a transform matrix and camera center vector [S4, unofficial].

## Operating characteristics
- **Time Complexity**: $O(n)$, where $n$ is the number of vertices. Each vertex is visited once to update min/max values.
- **Memory Complexity**: $O(1)$ auxiliary space, storing only the current min and max vectors.
- **Performance**: The operation is extremely fast and trivial to implement, making it suitable for real-time applications [S3, unofficial].
- **Strengths**: Simple to implement, computationally inexpensive, provides a tight bound for the mesh geometry [S2, unofficial].
- **Limitations**: Does not account for mesh rotation (produces AABB in local space), requires all vertices to be available, and does not handle empty inputs gracefully without explicit checks [S1, unofficial][S2, unofficial].
- **When to use**: When a global spatial bound is needed for raycasting, voxelization, or culling.
- **When not to use**: When per-primitive bounds are required (use [Host-side Bounding Box Computation](13468-host-side-bounding-box-computation.md)) or when the mesh is not available but point cloud data is (use [Volume Bounds Computation](5114-volume-bounds-computation.md)).

## Background
The method of computing bounding boxes by finding min/max vertex coordinates is a fundamental technique in computer graphics. It is widely used in voxelization pipelines, ray-tracing acceleration structures, and volume rendering initialization. The specific implementation details, such as handling offsets for volume texture generation and bit-manipulation tricks for vertex encoding, have been documented in various unofficial sources and blogs [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial]. The AABB representation and its properties, such as the ability to easily compute enclosing boxes and encode vertices via bitmasks, are well-established in the literature [S3, unofficial].

## Sources
[S1] kctess5/voxelizer, https://github.com/kctess5/voxelizer, fetched 2026-10-26 (unofficial)
[S2] Signed Distance Field Rendering Journey pt.1 – kosmonaut's blog, https://kosmonautblog.wordpress.com/2017/05/01/signed-distance-field-rendering-journey-pt-1/, fetched 2026-10-29 (unofficial)
[S3] A Couple 3D AABB Tricks, https://gpfault.net/posts/aabb-tricks.html, fetched 2026-10-29 (unofficial)
[S4] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
