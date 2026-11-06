# Dense Volume SDF Generation

Also called: Dense Grid SDF, SDF Voxelization, meshToDistanceVolume

## Overview
Dense Volume SDF Generation computes a scalar field $D(x)$ for every voxel center $x$ in a defined 3D grid, representing the signed Euclidean distance to the input surface [S1, unofficial][S2]. It solves the problem of creating a fully populated 3D array where every voxel contains a distance value, which is ideal for full data access, simulation, or marching cubes [S2]. It differs from its siblings [Signed Distance Field (SDF) Generation from Scanned Surfaces](854-signed-distance-field-sdf-generation-from-scanned-surfaces.md) and [SDF-based Binary Voxelization](2402-sdf-based-binary-voxelization.md) by explicitly storing the continuous distance value for every voxel in the grid rather than just binary occupancy or processed scan data [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by a list of vertices $V$ and faces $F$. Vertices are 3D points; faces are indices into $V$.
*   **Output:** A 3D array (tensor) of signed distances, typically stored in row-major order with dimensions `height`, `width`, `depth` [S1, unofficial].

**Data Structures:**
*   **Grid Parameters:**
    *   `voxelSize`: A scalar or vector defining the physical size of a voxel edge.
    *   `origin`: The 3D coordinate of the grid's minimum corner $(x_{min}, y_{min}, z_{min})$.
    *   `dimensions`: The integer resolution $(N_x, N_y, N_z)$ of the grid.
*   **SDF Array:** `Eigen::Tensor<float, 3, Eigen::RowMajor> sdf` with shape `(height, width, depth)` [S1, unofficial].

**Algorithm Steps:**

1.  **Grid Initialization:**
    Calculate the voxel size, origin, and dimensions based on the mesh bounding box. A common heuristic sets the voxel size to achieve a target total voxel count (e.g., $10^7$) [S2].
    $$ \text{voxelSize} = \text{suggestVoxelSize}(\text{mesh}, 10^7) $$
    $$ (\text{origin}, \text{dimensions}) = \text{calcOriginAndDimensions}(\text{mesh.computeBoundingBox}(), \text{voxelSize}) $$
    Define the bounds for each voxel index $(w, h, d)$ where $0 \le w < N_x$, $0 \le h < N_y$, $0 \le d < N_z$. The voxel center is calculated as:
    $$ \text{center} = \text{origin} + (w + 0.5, h + 0.5, d + 0.5) \times \text{voxelSize} $$
    *Note: If `VoxelizationMode::CORNER` is used, the center is $(w, h, d) \times \text{voxelSize}$ [S1, unofficial].*

2.  **Distance Evaluation (Parallel Loop):**
    Iterate over all voxels in parallel. For each voxel index $i$ from $0$ to $N_x \times N_y \times N_z - 1$:
    1.  Decode indices:
        $$ d = i \% N_z $$
        $$ w = (i / N_z) \% N_x $$
        $$ h = (i / N_z) / N_x $$
    2.  Initialize the SDF value for this voxel to infinity:
        $$ \text{sdf}(h, w, d) = \infty $$
    3.  Iterate over all faces $f$ in the mesh. For each face with vertices $v_1, v_2, v_3$:
        a. Calculate the closest point on the triangle to the voxel center.
        b. Compute the Euclidean distance:
        $$ \text{dist} = \| \text{center} - \text{closest\_point} \| $$
        c. Update the minimum distance if closer:
        $$ \text{if } (\text{dist} < \text{sdf}(h, w, d)) \text{ then } \text{sdf}(h, w, d) = \text{dist} $$

3.  **Sign Determination:**
    Determine if the voxel center is inside or outside the mesh.
    *   **Method A (Ray Casting):** Cast a ray from the voxel center to the origin $(0,0,0)$ (or a point far outside). Count intersections with the mesh triangles.
        $$ \text{num\_intersect} = \sum_{f} \text{intersect}(\text{center}, \text{origin}, v_1, v_2, v_3) $$
        If `num_intersect` is odd, the point is inside.
        $$ \text{if } (\text{num\_intersect} \% 2 == 1) \text{ then } \text{sdf}(h, w, d) *= -1 $$
    *   **Method B (Winding Number):** Use a robust winding number algorithm (e.g., `HoleWindingRule`) which is preferred for meshes with holes or self-intersections [S2]. If a GPU is available, a `FastWindingNumber` object can be used for acceleration [S2].

4.  **Clamping (Optional):**
    If `nullOutsideMinMax` is true, voxels with distances greater than a threshold (e.g., $3 \times \text{voxelSize}$) may be set to null or infinity to save memory/computation [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from typing import Tuple, List

def dense_volume_sdf_generation(
    vertices: List[List[float]], 
    faces: List[List[int]], 
    target_voxel_count: int = 10_000_000
) -> Tuple[np.ndarray, dict]:
    """
    Computes a dense Signed Distance Field for a mesh.
    
    Args:
        vertices: List of [x, y, z] coordinates.
        faces: List of [i, j, k] vertex indices.
        target_voxel_count: Target resolution for grid size.
        
    Returns:
        sdf: 3D numpy array of shape (depth, height, width) containing signed distances.
        params: Dictionary with origin, dimensions, and voxel_size.
    """
    vertices = np.array(vertices)
    faces = np.array(faces)
    
    # 1. Grid Initialization
    # Calculate bounding box
    min_coords = vertices.min(axis=0)
    max_coords = vertices.max(axis=0)
    bbox_size = max_coords - min_coords
    bbox_volume = np.prod(bbox_size)
    
    # Estimate voxel size to hit target count
    # Assuming cubic voxels for simplicity in estimation
    estimated_voxel_size = (bbox_volume / target_voxel_count) ** (1/3)
    
    # Calculate dimensions
    dimensions = np.ceil(bbox_size / estimated_voxel_size).astype(int)
    # Ensure at least 1 voxel per dimension
    dimensions = np.maximum(dimensions, 1)
    
    # Recalculate exact voxel size based on integer dimensions
    voxel_size = bbox_size / dimensions
    
    origin = min_coords
    
    # Initialize SDF array (Row-major: depth, height, width)
    # Shape: (depth, height, width)
    sdf = np.full((dimensions[2], dimensions[1], dimensions[0]), np.inf)
    
    # Pre-calculate face vertices for speed
    face_vertices = vertices[faces] # Shape: (num_faces, 3, 3)
    
    # 2. Distance Evaluation
    # Iterate over all voxels
    # Note: In practice, this loop is parallelized on CPU/GPU.
    # Here we show the sequential logic for clarity.
    
    for d in range(dimensions[2]):
        for h in range(dimensions[1]):
            for w in range(dimensions[0]):
                # Calculate voxel center
                center = origin + np.array([w, h, d]) * voxel_size + (voxel_size / 2.0)
                
                # Find minimum distance to any triangle
                min_dist = np.inf
                
                # Vectorized distance calculation for all faces
                # For each face, compute distance from center to triangle
                # This is a simplified placeholder for the actual triangle-point distance function
                # which involves projecting the point onto the triangle plane and clamping.
                
                # Pseudocode for triangle distance:
                # v1, v2, v3 = face_vertices[f]
                # closest_point = closest_point_on_triangle(center, v1, v2, v3)
                # dist = np.linalg.norm(center - closest_point)
                
                # Since we cannot implement full triangle distance in pure python without loops
                # efficiently, we assume a helper function exists.
                # In a real implementation, use a library or CUDA kernel.
                
                # Placeholder for the core loop over faces
                for f in range(len(faces)):
                    v1, v2, v3 = face_vertices[f]
                    closest_point = _closest_point_on_triangle(center, v1, v2, v3)
                    dist = np.linalg.norm(center - closest_point)
                    if dist < min_dist:
                        min_dist = dist
                
                sdf[d, h, w] = min_dist

    # 3. Sign Determination
    # Using Ray Casting Parity Test
    # Ray from center to origin (0,0,0)
    ray_direction = -center # Vector from center to origin
    
    for d in range(dimensions[2]):
        for h in range(dimensions[1]):
            for w in range(dimensions[0]):
                center = origin + np.array([w, h, d]) * voxel_size + (voxel_size / 2.0)
                ray_direction = -center
                
                num_intersect = 0
                for f in range(len(faces)):
                    v1, v2, v3 = face_vertices[f]
                    if _triangle_ray_intersection(center, ray_direction, v1, v2, v3):
                        num_intersect += 1
                
                if num_intersect % 2 == 1:
                    sdf[d, h, w] *= -1

    params = {
        "origin": origin,
        "dimensions": dimensions,
        "voxel_size": voxel_size
    }
    
    return sdf, params

def _closest_point_on_triangle(p, v1, v2, v3):
    """
    Computes the closest point on triangle v1-v2-v3 to point p.
    Returns the closest point coordinates.
    """
    # Vector from v1 to v2, v1 to v3, v1 to p
    edge1 = v2 - v1
    edge2 = v3 - v1
    v1p = p - v1
    
    # Dot products
    dot1 = np.dot(v1p, edge1)
    dot2 = np.dot(v1p, edge2)
    dot3 = np.dot(edge1, edge2)
    dot4 = np.dot(edge1, edge1)
    dot5 = np.dot(edge2, edge2)
    
    # Barycentric coordinates
    denom = dot1 * dot5 - dot2 * dot3
    if denom == 0:
        return v1 # Degenerate triangle or parallel
    
    u = (dot2 * dot4 - dot1 * dot5) / denom
    v = (dot1 * dot3 - dot2 * dot4) / denom
    
    # Check if point is inside triangle
    if u >= 0 and v >= 0 and (u + v) <= 1:
        return v1 + u * edge1 + v * edge2
    
    # If outside, find closest point on edges
    # Edge 1: v1-v2
    t = np.dot(v1p, edge1) / dot4
    t = max(0, min(1, t))
    closest_edge1 = v1 + t * edge1
    dist1 = np.linalg.norm(p - closest_edge1)
    
    # Edge 2: v1-v3
    t = np.dot(v1p, edge2) / dot5
    t = max(0, min(1, t))
    closest_edge2 = v1 + t * edge2
    dist2 = np.linalg.norm(p - closest_edge2)
    
    # Edge 3: v2-v3
    edge3 = v3 - v2
    v2p = p - v2
    dot6 = np.dot(v2p, edge3)
    dot7 = np.dot(edge3, edge3)
    t = dot6 / dot7
    t = max(0, min(1, t))
    closest_edge3 = v2 + t * edge3
    dist3 = np.linalg.norm(p - closest_edge3)
    
    # Return closest of the three
    if dist1 <= dist2 and dist1 <= dist3:
        return closest_edge1
    elif dist2 <= dist1 and dist2 <= dist3:
        return closest_edge2
    else:
        return closest_edge3

def _triangle_ray_intersection(origin, direction, v1, v2, v3):
    """
    Checks if ray (origin, direction) intersects triangle v1-v2-v3.
    Returns True if intersection occurs at distance >= 0.
    """
    # Möller–Trumbore algorithm
    epsilon = 1e-8
    edge1 = v2 - v1
    edge2 = v3 - v1
    h = np.cross(direction, edge2)
    a = np.dot(edge1, h)
    
    if a > -epsilon and a < epsilon:
        return False # Ray parallel to triangle
    
    f = 1.0 / a
    s = origin - v1
    u = f * np.dot(s, h)
    
    if u < 0.0 or u > 1.0:
        return False
    
    q = np.cross(s, edge1)
    v = f * np.dot(direction, q)
    
    if v < 0.0 or u + v > 1.0:
        return False
    
    t = f * np.dot(edge2, q)
    
    if t > epsilon:
        return True # Intersection found at positive distance
    
    return False
```

## Subtopics

### [Cube SDF Grid Construction](4883-cube-sdf-grid-construction.md)
Constructs a discrete Signed Distance Field (SDF) grid for a cube centered in the grid using analytical distance formulas for interior and exterior voxels. It differs from its sibling [Sphere SDF Grid Construction](4895-sphere-sdf-grid-construction.md) by using box-based min/max operations rather than spherical symmetry and Euclidean distance from the center.

### [Naive Jumping Farthest Away (JFA)](7621-naive-jumping-farthest-away-jfa.md)
Naive Jumping Farthest Away (JFA) is a GPU-based algorithm for computing the Signed Distance Field (SDF) and the position of the closest surface point for a dense voxel grid. It solves the problem of efficiently propagating distance information from known surface voxels to empty space using a multi-pass approach where the search radius is halved in each iteration. It differs from its sibling, [Cube SDF Grid Construction](4883-cube-sdf-grid-construction.md), by using a parallel, multi-scale neighborhood search rather than a fixed local stencil, and from its child, [Sequential Jumping Farthest Away (JFA)](7637-sequential-jumping-farthest-away-jfa.md), by executing the propagation loop in parallel on the GPU rather than sequentially on the CPU.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot F)$, where $N$ is the number of voxels and $F$ is the number of triangles in the mesh. This is computationally expensive for high-resolution grids and complex meshes [S1, unofficial].
*   **Memory Complexity:** $O(N)$, storing one float per voxel [S1, unofficial].
*   **Performance:** Suitable for small to medium-sized meshes where performance and memory use are not bottlenecks [S2]. Low resolution (e.g., $64^3$) results in poor reconstruction quality; higher resolutions significantly improve results but increase computation time linearly [S1, unofficial].
*   **Strengths:** Provides exact voxel-by-voxel control; ideal for simulation, remeshing, or exporting to machine learning models [S2].
*   **Limitations:** Slow for large meshes due to brute-force triangle iteration. Does not use spatial acceleration structures in its naive form [S1, unofficial].
*   **When to use:** When full data access is required, or when using marching cubes directly on the volume [S2].
*   **When not to use:** For very large meshes or real-time applications where performance is critical; consider [Naive Jumping Farthest Away (JFA)](7621-naive-jumping-farthest-away-jfa.md) or spatial acceleration techniques instead.

## Background
The method is implemented in libraries such as `meshlib` (MRMesh) and is often referred to as `meshToDistanceVolume` [S2]. It is a fundamental technique in computational geometry for converting polygonal meshes into volumetric representations. The approach of iterating over all voxels and all triangles is a direct application of the definition of the signed distance function, ensuring exactness at the cost of performance [S1, unofficial].

## Sources
[S1] Mesh Voxelization - GitHub, https://github.com/davidstutz/mesh-voxelization, fetched 2026-10-26 (unofficial)
[S2] Mesh to SDF (Signed Distance Field) Library for Python & C++, https://meshlib.io/feature/mesh-to-sdf/, fetched 2026-10-30
