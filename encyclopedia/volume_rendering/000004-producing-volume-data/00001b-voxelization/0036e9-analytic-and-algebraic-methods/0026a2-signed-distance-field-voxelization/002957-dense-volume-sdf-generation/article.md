# Dense Volume SDF Generation

Also called: Dense Grid SDF, Mesh-to-Volume Conversion, SDF Voxelization, mesh2volume, meshToDistanceVolume

## Overview
Dense Volume SDF Generation converts a continuous triangle mesh into a dense 3D grid where every voxel stores the exact signed Euclidean distance to the mesh surface [S3]. Unlike sparse representations that only store narrow-band data, this method produces a full volumetric occupancy grid suitable for simulation, machine learning inputs, or direct marching cubes extraction [S3]. It distinguishes itself from [Signed Distance Field Voxelization](3849-signed-distance-field-voxelization.md) by explicitly targeting dense, contiguous memory layouts rather than sparse tree structures, and from [SDF-based Binary Voxelization](2402-sdf-based-binary-voxelization.md) by preserving continuous distance values rather than quantizing to binary occupancy.

## How it works
**Inputs and Outputs**
*   **Input**: A triangle mesh defined by a vertex array $V$ (size $N_v \times 3$) and a face array $F$ (size $N_f \times 3$) containing vertex indices [S2, unofficial].
*   **Output**: A dense 3D tensor $D$ of shape $(H, W, D)$ storing signed distance values, typically in `float32` format [S2, unofficial].

**Data Structures**
The output grid is a 3D array indexed by $(h, w, d)$ corresponding to voxel coordinates. The spatial layout is determined by an origin vector $O$ and a voxel size $s$. The grid dimensions $(H, W, D)$ are calculated to encompass the mesh bounding box expanded by a search radius [S3].

**Parameters**
*   `voxelSize` ($s$): The edge length of a cubic voxel. Often derived from the mesh bounding box length $L$ and target resolution $R$ via $s = L / (R - 1)$ [S1, unofficial].
*   `maxDistSq`: The squared maximum distance to compute. A common heuristic is $(3 \cdot s)^2$, covering 3 voxel layers outside the mesh [S3].
*   `signMode`: Enum controlling inside/outside determination. `HoleWindingRule` is recommended for robustness against holes and self-intersections [S3].
*   `nullOutsideMinMax`: Boolean. If true, voxels with distance $> \sqrt{\text{maxDistSq}}$ are set to a null/empty value [S3].

**Algorithm Steps**
1.  **Grid Setup**: Calculate the bounding box of the mesh. Determine the voxel size $s$ and grid dimensions $(H, W, D)$ such that the grid fully contains the mesh plus the search radius [S3]. Define the grid origin $O$ [S3].
2.  **Initialization**: Initialize the output tensor $D$ with infinity (or null) [S2, unofficial].
3.  **Parallel Voxel Processing**: Iterate over every voxel index $i$ from $0$ to $H \times W \times D - 1$ [S2, unofficial].
    a.  **Coordinate Mapping**: Convert linear index $i$ to voxel coordinates $(h, w, d)$:
        $$ d = i \pmod D $$
        $$ w = \lfloor i / D \rfloor \pmod W $$
        $$ h = \lfloor i / (W \cdot D) \rfloor $$
        Calculate the voxel center $C$ in world space:
        $$ C = O + (w + 0.5, h + 0.5, d + 0.5) \cdot s $$
        *(Note: If `VoxelizationMode::CORNER` is used, use $(w, h, d) \cdot s$ [S2, unofficial]).*
    b.  **Distance Calculation**: Initialize $dist = \infty$. Iterate over all faces $f$ in the mesh. For each face with vertices $v_1, v_2, v_3$:
        i.  Compute the closest point $P_{close}$ on the triangle $(v_1, v_2, v_3)$ to $C$.
        ii. Compute $d_{tri} = \| C - P_{close} \|$.
        iii. If $d_{tri} < dist$, set $dist = d_{tri}$.
    c.  **Sign Determination**: Determine if $C$ is inside or outside the mesh.
        i.  Cast a ray from $C$ to a point far outside (e.g., origin or infinity).
        ii. Count intersections with mesh triangles. Let $N_{int}$ be the number of valid intersections where the intersection distance is positive.
        iii. If $N_{int}$ is odd, the voxel is inside. Set $D[h, w, d] = -dist$.
        iv. If $N_{int}$ is even, the voxel is outside. Set $D[h, w, d] = dist$.
        *(Alternative: Use Fast Winding Number for robust sign calculation [S3]).*
    d.  **Clamping**: If `nullOutsideMinMax` is true and $dist^2 > \text{maxDistSq}$, set $D[h, w, d]$ to null/empty [S3].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from typing import Tuple, List

def dense_volume_sdf_generation(
    vertices: np.ndarray,  # Shape (N_v, 3)
    faces: np.ndarray,     # Shape (N_f, 3)
    target_resolution: int = 128,
    max_dist_layers: int = 3
) -> Tuple[np.ndarray, np.ndarray, float]:
    """
    Generates a dense Signed Distance Field from a triangle mesh.
    
    Args:
        vertices: Array of mesh vertices.
        faces: Array of face indices.
        target_resolution: Target grid dimension (assumes cubic grid).
        max_dist_layers: Number of voxel layers outside mesh to compute.
        
    Returns:
        sdf_grid: 3D numpy array of shape (D, W, H) with signed distances.
        origin: World space origin of the grid.
        voxel_size: Size of one voxel edge.
    """
    # 1. Grid Setup
    bbox_min = np.min(vertices, axis=0)
    bbox_max = np.max(vertices, axis=0)
    bbox_center = (bbox_min + bbox_max) / 2.0
    bbox_extent = bbox_max - bbox_min
    bbox_length = np.max(bbox_extent)
    
    # Calculate voxel size based on bounding box and resolution
    # Formula: voxel_size = length / (dim - 1) [S1, unofficial]
    voxel_size = bbox_length / (target_resolution - 1)
    
    # Expand bounds by max distance layers to ensure full coverage
    max_dist = max_dist_layers * voxel_size
    expanded_min = bbox_min - max_dist
    expanded_max = bbox_max + max_dist
    
    # Calculate grid dimensions to cover expanded bounds
    # Dimensions must be integers
    dims = np.ceil((expanded_max - expanded_min) / voxel_size).astype(int)
    origin = expanded_min
    
    # Initialize SDF grid with infinity
    sdf_grid = np.full((dims[2], dims[1], dims[0]), np.inf, dtype=np.float32)
    
    # Pre-compute face vertices for faster access
    face_vertices = vertices[faces] # Shape (N_f, 3, 3)
    
    # 2. Parallel Voxel Processing
    # Iterate over all voxels
    for d in range(dims[2]):
        for w in range(dims[1]):
            for h in range(dims[0]):
                # Calculate voxel center in world space
                # Center is at (w+0.5, h+0.5, d+0.5) relative to origin [S2, unofficial]
                center = origin + np.array([w + 0.5, h + 0.5, d + 0.5]) * voxel_size
                
                # 3. Distance Calculation
                min_dist = np.inf
                
                # Iterate over all triangles
                for tri in face_vertices:
                    v1, v2, v3 = tri
                    
                    # Compute closest point on triangle to center
                    # Vector math for point-to-triangle distance
                    edge1 = v2 - v1
                    edge2 = v3 - v1
                    edge3 = v3 - v2
                    
                    v0 = center - v1
                    
                    dot00 = np.dot(v0, v1 - v1) # 0
                    dot01 = np.dot(v0, edge1)
                    dot02 = np.dot(v0, edge2)
                    dot11 = np.dot(edge1, edge1)
                    dot12 = np.dot(edge1, edge2)
                    dot22 = np.dot(edge2, edge2)
                    
                    inv_denom = 1.0 / (dot11 * dot22 - dot12 * dot12)
                    u = (dot22 * dot01 - dot12 * dot02) * inv_denom
                    v = (dot11 * dot02 - dot12 * dot01) * inv_denom
                    
                    # Determine closest point based on barycentric coordinates
                    if u + v <= 1.0:
                        if u < 0.0:
                            if v < 0.0:
                                # Closest to v1
                                closest = v1
                            else:
                                # Closest to edge v1-v3
                                if v > 1.0:
                                    closest = v3
                                else:
                                    t = v
                                    closest = v1 + (v3 - v1) * t
                        elif v < 0.0:
                            # Closest to edge v1-v2
                            closest = v1 + edge1 * u
                        else:
                            # Inside triangle
                            closest = v1 + edge1 * u + edge2 * v
                    else:
                        if u > 1.0:
                            closest = v2
                        else:
                            # Closest to edge v2-v3
                            t = (u + v - 1.0) / (1.0 - dot12 * inv_denom * (dot11 * dot22 - dot12 * dot12) + dot12 * dot12 * inv_denom * (dot11 * dot22 - dot12 * dot12)) # Simplified: t = (u+v-1)/(1-v) logic approx
                            # Correct logic for edge v2-v3:
                            # Project onto edge, clamp to [0,1]
                            # Vector from v2 to center
                            v2_center = center - v2
                            edge23 = v3 - v2
                            len_sq = np.dot(edge23, edge23)
                            if len_sq == 0:
                                closest = v2
                            else:
                                t = np.dot(v2_center, edge23) / len_sq
                                t = np.clip(t, 0.0, 1.0)
                                closest = v2 + edge23 * t
                    
                    dist_sq = np.sum((center - closest) ** 2)
                    if dist_sq < min_dist:
                        min_dist = dist_sq
                
                # 4. Sign Determination
                # Ray casting from center to origin (or far away)
                # Count intersections with triangles
                num_intersects = 0
                ray_dir = -center # Ray towards origin
                ray_len = np.linalg.norm(ray_dir)
                if ray_len > 0:
                    ray_dir = ray_dir / ray_len
                
                for tri in face_vertices:
                    v1, v2, v3 = tri
                    
                    # Möller–Trumbore intersection algorithm
                    # Edge vectors
                    e1 = v2 - v1
                    e2 = v3 - v1
                    
                    # P = cross(ray_dir, e2)
                    P = np.cross(ray_dir, e2)
                    
                    # det = dot(e1, P)
                    det = np.dot(e1, P)
                    
                    if det > -1e-8: # Not parallel, assuming front-face culling not strictly required for parity
                        # T = -dot(v1, P) / det ? No, standard MT:
                        # T = v1 - center (vector from ray origin to triangle vertex)
                        T = v1 - center
                        u = np.dot(T, P)
                        
                        if u >= 0.0 and u <= 1.0:
                            Q = np.cross(T, e1)
                            v = np.dot(ray_dir, Q)
                            
                            if v >= 0.0 and u + v <= 1.0:
                                t = np.dot(e2, Q) / det
                                if t > 1e-8: # Intersection is in front of ray origin
                                    num_intersects += 1
                
                # 5. Finalize Value
                dist = np.sqrt(min_dist)
                
                # Check max distance
                if min_dist > (max_dist_layers * voxel_size) ** 2:
                    sdf_grid[d, w, h] = np.nan # Null outside
                else:
                    if num_intersects % 2 == 1:
                        sdf_grid[d, w, h] = -dist # Inside
                    else:
                        sdf_grid[d, w, h] = dist # Outside

    return sdf_grid, origin, voxel_size
```

## Subtopics

### [Cube SDF Grid Construction](4883-cube-sdf-grid-construction.md)
Constructs a discrete Signed Distance Field (SDF) grid for a cube centered in the grid using analytical distance formulas for interior and exterior voxels. It differs from its sibling [Sphere SDF Grid Construction](4895-sphere-sdf-grid-construction.md) by using box-based min/max operations rather than spherical symmetry and Euclidean distance from the center.

### [Naive Jumping Farthest Away (JFA)](7621-naive-jumping-farthest-away-jfa.md)
Naive Jumping Farthest Away (JFA) is a GPU-based algorithm for computing the Signed Distance Field (SDF) and the position of the closest surface point for a dense 3D voxel grid. It solves the problem of efficiently propagating distance information from known surface voxels to empty space using a multi-pass approach where the search radius $K$ is halved in each iteration. It differs from its sibling, Cube SDF Grid Construction, by using a logarithmic halving stride strategy rather than a fixed local neighborhood expansion, and from its child, Sequential JFA, by executing the propagation loop in parallel on the GPU rather than sequentially on the CPU.

### [General Mesh SDF and Occupancy Voxelization](10658-general-mesh-sdf-and-occupancy-voxelization.md)
This method converts a solid boundary representation (B-rep) into a regular volumetric occupancy grid and optionally a signed-distance field (SDF). It solves the problem of discretizing continuous CAD geometry into a dense 3D grid suitable for downstream applications such as 3D-print lattice generation, finite element (FE) meshing, and collision queries. Unlike core B-rep operations, this is an application-level capability that relies on tessellated mesh data rather than exact analytical surfaces.

### Volumetric Distance Function Computation
A scalar field defined on a voxel grid where empty voxels are populated with the distance from their center to the nearest point on the surface, derived from the initial voxelization [S4, unofficial]. It operates by several 'sweepings' over the volume to efficiently compute distances for all empty voxels based on the surface-intersecting voxels [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N_v \cdot N_f \cdot H \cdot W \cdot D)$ for the naive triangle intersection approach, where $N_v, N_f$ are vertices/faces and $H,W,D$ are grid dimensions. This is computationally expensive for high resolutions. Optimizations like spatial hashing or GPU parallelization (e.g., Fast Winding Number) are often required [S3].
*   **Memory Complexity**: $O(H \cdot W \cdot D)$ for the dense grid. This is significantly higher than sparse methods (e.g., OpenVDB narrow-band) [S1, unofficial].
*   **Performance**: Suitable for small to medium-sized meshes where performance and memory are not bottlenecks [S3]. Low resolutions (e.g., $64^3$) result in poor reconstruction quality, while higher resolutions (e.g., $128^3$ or more) improve accuracy but increase memory linearly [S2, unofficial].
*   **Strengths**: Provides exact voxel-by-voxel control, ideal for simulation, remeshing, and exporting to ML models [S3].
*   **Limitations**: High memory usage for large volumes; slow for very high-resolution grids without GPU acceleration. Numerical instability can occur if `voxel_size` is too small ($< 1e-5$) [S1, unofficial].

## Background
The method originates from the need to convert continuous CAD geometry (triangle meshes) into discrete volumetric representations for downstream processing. Early implementations used naive ray-casting for every voxel [S2, unofficial]. Modern libraries like MeshLib provide optimized C++/Python bindings for this conversion, supporting features like narrow-band computation and GPU acceleration via CUDA [S3]. The technique is widely used in 3D printing, finite element analysis, and computer graphics for generating signed distance fields from scanned or modeled surfaces [S1, unofficial][S3].

## Sources
[S1] zzilch/Mesh2Volume, https://github.com/zzilch/Mesh2Volume, fetched 2026-10-26 (unofficial)
[S2] Mesh Voxelization - GitHub, https://github.com/davidstutz/mesh-voxelization, fetched 2026-10-26 (unofficial)
[S3] Mesh to SDF (Signed Distance Field) Library for Python & C++, https://meshlib.io/feature/mesh-to-sdf/, fetched 2026-10-30
[S4] How to project a 0 genus mesh model on a sphere?, https://scicomp.stackexchange.com/questions/21048, fetched 2026-10-30 (unofficial)
