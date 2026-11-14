# Depth Estimation from Point Cloud
Also called: Closest Point Depth Estimation
## Overview
This method generates a depth image at a novel viewpoint by projecting a pre-existing 3D point cloud onto the camera's image plane [S1]. It solves the problem of converting geometric samples into a per-pixel depth map, handling occlusions by selecting the nearest surface point for each pixel [S1]. The method builds on [3D Point Projection to 2D](3148-3d-point-projection-to-2d.md) and is described in the context of improving Neural Radiance Fields using near-surface sampling with point cloud generation [S1].
## How it works
**Inputs:**
1. A 3D point cloud $P = \{p_1, p_2, \dots, p_N\}$, where each $p_i \in \mathbb{R}^3$ [S1].
2. The camera intrinsic matrix $K$ and extrinsic parameters (rotation $R$ and translation $t$) defining the novel viewpoint [S1].

**Outputs:**
A 2D depth image $D$ of resolution $W \times H$, where $D(u, v)$ is the depth value for pixel $(u, v)$ [S1].

**Data Structures:**
- `depth_map`: A 2D array of size $H \times W$, initialized to infinity to track minimum depths [G].
- `point_cloud`: List or array of 3D coordinates [G].

**Steps:**
1. **Transformation:** Transform each 3D point $p_i = (x, y, z)$ from world coordinates to camera coordinates $p'_i = (x', y', z') = R p_i + t$ [S1].
2. **Projection:** Project each transformed point to 2D pixel coordinates $(u_i, v_i)$ using the intrinsic matrix $K$:
   $$
   u_i = \frac{K_{00} x' + K_{01} y' + K_{02} z'}{z'}, \quad v_i = \frac{K_{10} x' + K_{11} y' + K_{12} z'}{z'}
   $$
   [S1, G]
3. **Depth Calculation:** Calculate the Euclidean distance $d_i$ from the camera center to the point:
   $$
   d_i = \sqrt{(x')^2 + (y')^2 + (z')^2}
   $$
   [S1]
4. **Occlusion Handling:** For each pixel $(u, v)$, if multiple points project to it, select the point with the minimum depth:
   $$
   D(u, v) = \min \{ d_i \mid (u_i, v_i) = (u, v) \}
   $$
   [S1]
5. **Hole Handling:** Pixels with no projected points remain at the initialization value, representing background areas or missing surface data due to limited 3D points [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_depth_from_point_cloud(point_cloud, K, R, t, image_height, image_width):
    # Initialize depth map with infinity to track minimums
    valid_depth_map = [[float('inf')] * image_width for _ in range(image_height)]
    
    for p in point_cloud:
        # 1. Transform to camera coordinates
        x, y, z = p
        rx = R[0][0]*x + R[0][1]*y + R[0][2]*z + t[0]
        ry = R[1][0]*x + R[1][1]*y + R[1][2]*z + t[1]
        rz = R[2][0]*x + R[2][1]*y + R[2][2]*z + t[2]
        
        # Check if point is in front of camera
        if rz <= 0:
            continue
            
        # 2. Project to 2D
        proj_u = K[0][0]*rx + K[0][1]*ry + K[0][2]*rz
        proj_v = K[1][0]*rx + K[1][1]*ry + K[1][2]*rz
        proj_w = K[2][0]*rx + K[2][1]*ry + K[2][2]*rz
        
        if proj_w == 0:
            continue
            
        u = proj_u / proj_w
        v = proj_v / proj_w
        
        # 3. Check if pixel is within image bounds
        u_int = int(round(u))
        v_int = int(round(v))
        
        if 0 <= u_int < image_width and 0 <= v_int < image_height:
            # 4. Calculate depth (Euclidean distance from camera)
            depth = (rx**2 + ry**2 + rz**2)**0.5
            
            # 5. Update minimum depth for this pixel
            if depth < valid_depth_map[v_int][u_int]:
                valid_depth_map[v_int][u_int] = depth
                    
    # 6. Fill holes: replace inf with 0 (background indicator)
    depth_map = [[0.0 if valid_depth_map[v][u] == float('inf') else valid_depth_map[v][u] 
                  for u in range(image_width)] for v in range(image_height)]
    
    return depth_map
```

**Parameters:**
- $K$: Camera intrinsics. Not given in the sources. [G]
- $R, t$: Camera extrinsics. Not given in the sources. [G]

**Edge Cases:**
- **Holes:** Pixels with zero values may correspond to background areas or missing foreground information due to limited 3D points [S1].
- **Multiple Points:** If multiple 3D points project to the same pixel, the closest one to the camera is used [S1].
- **Precision:** Standard floating-point arithmetic is assumed [G].

## Subtopics

### [Triangulation for Closest Point](9933-triangulation-for-closest-point.md)
Solves for the closest point to $N$ camera rays originating from 2D projections by formulating it as a least-squares optimization problem. It is distinguished from simple back-projection by incorporating a visibility mask $\gamma_i$ derived from 2D tracking occlusion checks and depth-based verification against rendered Gaussian splats, ensuring robustness in dynamic scenes. It is chosen when ray-based reconstruction is preferred over direct point projection.

### Dense Depth Estimation via Plane Sweep
A live system for image capturing and dense depth estimation where the plane-sweep algorithm runs almost entirely on the GPU [S2, unofficial]. It is limited to only four camera views but achieves up to 20 frames per second [S2, unofficial].

## Operating characteristics
**Time Complexity:** Not given in the sources. [G]
**Memory Complexity:** Not given in the sources. [G]
**Performance:** The method achieved 31.44 dB PSNR on the Pavillon dataset with $\alpha = 1/2$ and $N = 8$ [S1].
**Strengths:** Directly converts existing point cloud data into depth maps without iterative optimization [G].
**Limitations:** Produces holes where no points project; cannot distinguish between background areas and missing foreground surface data without additional processing [S1].
**When to use:** When a point cloud is already available and a fast, direct depth map is required [G].
**When not to use:** Not given in the sources. [G]

## Background
The method is described in the context of improving Neural Radiance Fields using near-surface sampling with point cloud generation [S1]. It is used in the test stage to obtain depth images at novel viewpoints [S1].

## Sources
[S1] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
[S2] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
