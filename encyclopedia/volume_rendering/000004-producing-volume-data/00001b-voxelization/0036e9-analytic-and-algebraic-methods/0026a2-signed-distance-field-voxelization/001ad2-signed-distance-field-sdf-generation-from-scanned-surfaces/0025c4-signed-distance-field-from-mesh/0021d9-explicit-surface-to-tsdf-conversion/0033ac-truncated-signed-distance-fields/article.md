# Truncated Signed Distance Fields

Also called: Truncated SDF

## Overview
Truncated Signed Distance Fields (TSDF) represent a 3D implicit surface as the zero crossing of a function interpolating truncated signed distances on a uniform grid [S1, S2]. Unlike full Signed Distance Fields (SDFs), TSDFs limit the distance magnitude to a specific interval, handling incomplete information and reducing memory usage for robotics and reconstruction applications [S2]. It differs from Projective TSDF Fusion by focusing on the static representation definition rather than the multi-view integration process, and from Mesh-to-TSDF Sequence Encoding by addressing the fundamental voxel value definition rather than temporal compression [S2658](2658-projective-tsdf-fusion.md), [S3649](3649-mesh-to-tsdf-sequence-encoding-pipeline.md).

## How it works
**Inputs:** A closed surface $Q$ (mesh or point cloud), truncation limits $d_{min}$ and $d_{max}$, and a uniform 3D grid resolution $W \times H \times D$.
**Outputs:** A 3D grid where each voxel stores a scalar TSDF value.

**Data Structures:**
The output is a uniform $W \times H \times D$ 3D grid. Each voxel stores a single floating-point value representing the truncated signed distance. The grid coordinates correspond to spatial positions in the world frame.

**Steps:**
1. For each point $p$ in the grid (or voxel center):
2. Find the closest point $q$ on the surface $Q$:
   $$ q = \arg\min_{q' \in Q} \|p - q'\|_2 $$
3. Determine the sign using the surface normal $n(q)$ at $q$:
   $$ I_{\pm}(p) = \text{sign}(n(q)^T \cdot (p - q)) $$
   [G] If $n(q)$ is not available (e.g., point cloud), use the vector $(p-q)$ normalized and dot with a consistent outward normal estimate, or default to positive if outside the convex hull.
4. Compute the signed distance:
   $$ d_{signed}(p) = I_{\pm}(p) \cdot \|p - q\|_2 $$
5. Truncate the value to the interval $[d_{min}, d_{max}]$:
   $$ d(p) = \min(d_{max}, \max(d_{min}, d_{signed}(p))) $$

**Formulas:**
The TSDF value $d(p)$ for a point $p$ is defined as [S2]:
$$ d(p) = \min(d_{max}, \max(d_{min}, I_{\pm}(p) \min_{q \in Q} \|p - q\|_2)) $$
where:
- $d_{min}$ and $d_{max}$ are the lower and upper truncation limits.
- $Q$ is the surface.
- $I_{\pm}(p)$ assigns the sign based on the surface normal $n(q)$ at the closest point $q$ via $I_{\pm}(p) = \text{sign}(n(q)^T \cdot (p - q))$.
- $\| \cdot \|_2$ denotes the Euclidean distance.

**Parameters:**
- Truncation limits: Typical values are $d_{min} = -0.04$ and $d_{max} = 0.1$ (in meters, assuming 2cm voxel size context) [S2].
- Sign convention: Distances outside the surface are positive; distances inside are negative [S1].
- $\tau$: A general truncation threshold value that limits the magnitude of signed distances stored in the grid [S1].

**Edge Cases:**
- If $p$ is exactly on the surface, $d(p) = 0$.
- If the closest point $q$ is ambiguous (e.g., equidistant to multiple surface parts), the sign may be unstable.
- Precision issues may arise when $p$ is very close to $q$; use epsilon guards for zero-crossing detection.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_tsdf(p, surface, n_min, n_max):
    """
    Compute TSDF value for a single point p.
    p: 3D vector (x, y, z)
    surface: List of surface points or mesh data
    n_min: Lower truncation limit (float)
    n_max: Upper truncation limit (float)
    Returns: Truncated signed distance (float)
    """
    # Find closest point q on surface
    min_dist = infinity
    closest_q = None
    normal_at_q = None
    
    for q in surface.points:
        dist = euclidean_distance(p, q)
        if dist < min_dist:
            min_dist = dist
            closest_q = q
            normal_at_q = q.normal # Assume surface provides normals
            
    # If no normal available, estimate from geometry [G]
    if normal_at_q is None:
        # Placeholder for normal estimation if not provided by surface
        normal_at_q = normalize(p - closest_q) # Fallback, may be inaccurate
        
    # Compute signed distance
    # Vector from surface to point
    vec = p - closest_q
    # Dot product with normal to determine side
    dot_product = dot(normal_at_q, vec)
    
    if dot_product > 0:
        sign = 1.0
    elif dot_product < 0:
        sign = -1.0
    else:
        sign = 0.0
        
    signed_dist = sign * min_dist
    
    # Truncate
    truncated = max(n_min, min(n_max, signed_dist))
    
    return truncated
```

## Subtopics

### [Projective TSDF Fusion](2658-projective-tsdf-fusion.md)
Integrates multiple depth images into a TSDF volume. Choose when fusing sensor data.

### [Mesh-to-TSDF Sequence Encoding Pipeline](3649-mesh-to-tsdf-sequence-encoding-pipeline.md)
Encodes TSDF sequences for compression. Choose when storing temporal TSDF data.

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of voxels and $M$ is the number of surface points/vertices, unless accelerated with spatial indexing (e.g., KD-tree, BVH).
- **Memory Complexity:** $O(W \times H \times D)$ for the grid. Truncation reduces the effective dynamic range but not the storage size per voxel.
- **Strengths:** Robust to noise and incomplete data; compact representation for surface proximity; compatible with standard image processing filters [S2].
- **Limitations:** Memory intensive for high-resolution volumes; truncation loses information about distant geometry; sign ambiguity near complex surfaces [S2].
- **Use Cases:** Robotics scene reconstruction, pose estimation, collision detection [S2].

## Background
TSDFs were introduced to address difficulties in computing and updating distance fields based on incomplete information, proving useful in robotics research for accurate scene reconstruction and rigid-body pose estimation [S2]. They are applied to speeding up image-alignment, raycasting, collision detection, and motion planning [S2]. The concept is often used in conjunction with marching cubes for surface extraction [S1].

## Sources
[S1] Deep Implicit Volume Compression, https://doi.org/10.1109/cvpr42600.2020.00137, fetched 2026-10-26
[S2] Compressed Voxel-Based Mapping Using Unsupervised Learning, https://doi.org/10.3390/robotics6030015, fetched 2026-10-26
