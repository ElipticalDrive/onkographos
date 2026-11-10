# Depth Estimation from Point Cloud
Also called: Closest Point Depth Estimation
## Overview
This method generates a depth image at a novel viewpoint by projecting a pre-existing 3D point cloud onto the camera's image plane [S1]. It solves the problem of converting sparse or dense geometric samples into a per-pixel depth map, handling occlusions by selecting the nearest surface point for each pixel [S1]. It differs from direct projection methods by explicitly resolving multiple points per pixel via distance minimization, and from volumetric methods by operating directly on discrete points rather than a voxel grid [S1].
## How it works
**Inputs:**
1. A 3D point cloud $P = \{p_1, p_2, ..., p_N\}$, where each $p_i \in \mathbb{R}^3$.
2. The camera intrinsic matrix $K$ and extrinsic parameters (rotation $R$ and translation $t$) defining the novel viewpoint.

**Outputs:**
A 2D depth image $D$ of resolution $W \times H$, where $D(u, v)$ is the depth value for pixel $(u, v)$.

**Data Structures:**
- `depth_map`: A 2D array of size $H \times W$, initialized to zero or infinity.
- `point_cloud`: List or array of 3D coordinates.

**Steps:**
1. **Transformation:** Transform each 3D point $p_i$ from world coordinates to camera coordinates $p'_i = R p_i + t$.
2. **Projection:** Project each transformed point $p'_i = (x'_i, y'_i, z'_i)$ to 2D pixel coordinates $(u_i, v_i)$ using the intrinsic matrix $K$.
   $$
   \begin{bmatrix} u_i \\ v_i \\ w_i \end{bmatrix} = K \begin{bmatrix} x'_i \\ y'_i \\ z'_i \end{bmatrix}
   $$
   $$
   u_i = \frac{x'_i}{w_i}, \quad v_i = \frac{y'_i}{w_i}
   $$
   [S1]
3. **Depth Calculation:** Calculate the Euclidean distance $d_i$ from the camera center to the point:
   $$
   d_i = \sqrt{(x'_i)^2 + (y'_i)^2 + (z'_i)^2} = z'_i \quad (\text{if } z'_i > 0 \text{ and aligned with optical axis})
   $$
   [S1]
4. **Occlusion Handling:** For each pixel $(u, v)$, if multiple points project to it, select the point with the minimum depth $d_{min}$.
   $$
   D(u, v) = \min \{ d_i \mid (u_i, v_i) = (u, v) \}
   $$
   [S1]
5. **Hole Handling:** Pixels with no projected points remain zero or undefined, representing background or missing surface data. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_depth_from_point_cloud(point_cloud, K, R, t, image_height, image_width):
    """
    point_cloud: list of (x, y, z) tuples
    K: 3x3 intrinsic matrix
    R: 3x3 rotation matrix
    t: 3x1 translation vector
    """
    depth_map = np.zeros((image_height, image_width))
    
    # Initialize with infinity to find minimums
    # Note: Using a separate map for valid depths to handle holes
    valid_depth_map = np.full((image_height, image_width), np.inf)
    
    for p in point_cloud:
        # 1. Transform to camera coordinates
        p_homogeneous = np.array([p[0], p[1], p[2], 1.0])
        p_cam_homogeneous = np.dot(np.hstack([R, t]), p_homogeneous)
        p_cam = p_cam_homogeneous[:3]
        
        x, y, z = p_cam
        
        # Check if point is in front of camera
        if z <= 0:
            continue
            
        # 2. Project to 2D
        # Assuming standard pinhole model: K * [x, y, z, 1]^T
        # But K is 3x3, so we use [x, y, z]
        proj = np.dot(K, np.array([x, y, z]))
        u = proj[0] / proj[2]
        v = proj[1] / proj[2]
        
        # 3. Check if pixel is within image bounds
        u_int = int(round(u))
        v_int = int(round(v))
        
        if 0 <= u_int < image_width and 0 <= v_int < image_height:
            # 4. Calculate depth (distance from camera)
            depth = np.sqrt(x**2 + y**2 + z**2)
            
            # 5. Update minimum depth for this pixel
            if depth < valid_depth_map[v_int, u_int]:
                valid_depth_map[v_int, u_int] = depth
                
    # 6. Fill holes: replace inf with 0 (or background indicator)
    depth_map = np.where(np.isfinite(valid_depth_map), valid_depth_map, 0.0)
    
    return depth_map
```

**Parameters:**
- $K$: Camera intrinsics. Not given in the sources.
- $R, t$: Camera extrinsics. Not given in the sources.

**Edge Cases:**
- **Holes:** Pixels with zero values may correspond to background areas or missing foreground information due to limited 3D points. [S1]
- **Multiple Points:** If multiple 3D points project to the same pixel, the closest one to the camera is used. [S1]
- **Precision:** Standard floating-point arithmetic is assumed. [G]

## Subtopics

### [Triangulation for Closest Point](9933-triangulation-for-closest-point.md)
Solves for the closest point to N camera rays originating from 2D projections by formulating it as a least-squares optimization problem. It is chosen when ray-based reconstruction is preferred over direct point projection. [Link: [Triangulation for Closest Point](9933-triangulation-for-closest-point.md)]

### Dense Depth Estimation via Plane Sweep
A live system for image capturing and dense depth estimation where the plane-sweep algorithm runs almost entirely on the GPU. It is limited to only four camera views but achieves up to 20 frames per second. [S2, unofficial]

## Operating characteristics
**Time Complexity:** Linear with respect to the number of points $N$ and constant with respect to image resolution for the projection step, though filling the depth map is $O(N)$. [S1]
**Memory Complexity:** $O(W \times H)$ for the depth map. [S1]
**Performance:** Not given in the sources for the basic point cloud projection method. [S1]
**Strengths:** Simple to implement; directly uses existing point cloud data. [S1]
**Limitations:** Produces holes where no points project; cannot distinguish between background and missing surface data without additional processing. [S1]
**When to use:** When a point cloud is already available and a quick depth map is needed. [S1]
**When not to use:** When high-density, hole-free depth maps are required without post-processing. [S1]

## Background
The method is described in the context of improving Neural Radiance Fields using near-surface sampling with point cloud generation. It is used in the test stage to obtain depth images at novel viewpoints. [S1]

## Sources
[S1] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
[S2] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
