# Depth Estimation from Point Cloud
Also called: Closest Point Depth Estimation

## Overview
This method generates a depth map for a novel camera viewpoint by projecting an existing 3D point cloud onto the 2D image plane and resolving occlusions [S1]. It converts 3D geometry into an explicit per-pixel depth buffer by relying on discrete point proximity rather than volumetric ray-marching [G].

## How it works
**Inputs:**
- A 3D point cloud $P = \{p_1, p_2, \dots, p_N\}$, where each $p_i \in \mathbb{R}^3$ is a point in world coordinates [G].
- A target camera defined by its intrinsic matrix $K$ and extrinsic matrix $[R|t]$ (rotation and translation from world to camera coordinates) [G].

**Outputs:**
- A depth image $D$ of resolution $W \times H$, where $D(u,v)$ is the distance from the camera center to the closest surface point at pixel $(u,v)$ [G].

**Data Structures:**
- A 2D array `depth_map` of size $H \times W$, initialized to infinity to track minimum depths [G].

**Steps:**
1. **Coordinate Transformation**: Transform each 3D point $p_i$ from world coordinates to camera coordinates $p'_i$: $p'_i = R p_i + t$ [G]. Let $p'_i = (X_i, Y_i, Z_i)$.
2. **Projection**: Project each transformed point $p'_i$ onto the image plane using the intrinsic matrix $K$ [G]. For a pinhole camera model: $u_i = \frac{f_x X_i}{Z_i} + c_x$, $v_i = \frac{f_y Y_i}{Z_i} + c_y$, where $f_x, f_y$ are focal lengths and $c_x, c_y$ are principal point coordinates [G].
3. **Validation**: Check if the projected pixel $(u_i, v_i)$ lies within the image boundaries $[0, W-1] \times [0, H-1]$ and if the depth $Z_i > 0$ (point is in front of the camera) [G]. Discard points failing these checks.
4. **Occlusion Resolution**: For each valid pixel $(u, v)$, collect all points $p_i$ that project to this pixel [S1]. Select the point $p_{closest}$ with the minimum depth value $Z_{min} = \min \{Z_i\}$ [S1].
5. **Depth Assignment**: Set the depth value for pixel $(u, v)$ to the depth of the closest point: $D(u, v) = Z_{min}$ [S1].
6. **Hole Handling**: Pixels with no projected points remain at their initial value, representing background or missing surface information [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_depth_from_point_cloud(point_cloud, K, R, t, width, height):
    # Initialize depth map with infinity
    depth_map = [[float('inf')] * width for _ in range(height)]
    
    for p in point_cloud:
        # 1. Coordinate Transformation
        X = R[0][0]*p[0] + R[0][1]*p[1] + R[0][2]*p[2] + t[0]
        Y = R[1][0]*p[0] + R[1][1]*p[1] + R[1][2]*p[2] + t[1]
        Z = R[2][0]*p[0] + R[2][1]*p[1] + R[2][2]*p[2] + t[2]
        
        # 3. Validation: point must be in front of camera
        if Z <= 0:
            continue
            
        # 2. Projection
        u = K[0][0] * X / Z + K[0][2]
        v = K[1][1] * Y / Z + K[1][2]
        
        # Check image boundaries
        if 0 <= u < width and 0 <= v < height:
            # 4 & 5. Occlusion Resolution & Depth Assignment
            u_int = int(u)
            v_int = int(v)
            if Z < depth_map[v_int][u_int]:
                depth_map[v_int][u_int] = Z
                
    # 6. Hole Handling: replace infinity with 0
    for y in range(height):
        for x in range(width):
            if depth_map[y][x] == float('inf'):
                depth_map[y][x] = 0
                
    return depth_map
```

**Parameters:**
- $N$: Number of points in the cloud. Typical values depend on the generation method (e.g., $N=8$ views used in source [S1] for generation, but $N$ points in cloud vary) [S1].

**Edge Cases:**
- **Holes**: Pixels with no projected points result in zero depth. The source notes that these holes cannot be automatically distinguished from background areas without additional context [S1].
- **Precision**: Floating-point errors in projection may cause slight misalignments. Integer rounding of $(u,v)$ is required for array indexing [G].

## Subtopics

### [Triangulation for Closest Point](9933-triangulation-for-closest-point.md)
Computes 3D position by finding the closest intersection to multiple camera rays, using least-squares optimization and visibility masks. Choose when multi-view consistency is required.

### [Dense Depth Estimation via Plane Sweep](10850-dense-depth-estimation-via-plane-sweep.md)
Reconstructs dense depth by evaluating photo-consistency across depth planes. Choose for textureless regions or when explicit point clouds are unavailable.

## Operating characteristics
- **Time Complexity**: $O(N)$ for projection and $O(W \cdot H)$ for depth map initialization and hole filling [G]. The bottleneck is typically the projection of $N$ points [G].
- **Memory Complexity**: $O(N)$ to store the point cloud and $O(W \cdot H)$ for the depth map [G].
- **Performance**: The source reports a PSNR of 31.44 dB on the Pavillon dataset using this method, compared to 30.27 dB for standard MVS [S1].
- **Strengths**: Simple, fast, and directly utilizes existing 3D points [G].
- **Limitations**: Produces sparse depth maps with holes if the point cloud is incomplete. Cannot infer depth in regions without 3D points [S1].

## Background
This method is used in the test stage of neural radiance field improvements that incorporate point cloud generation [S1]. It serves as a bridge between implicit/explicit 3D representations and explicit 2D depth buffers for novel view synthesis [G].

## Sources
[S1] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
