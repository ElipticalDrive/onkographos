# Camera Matrix Ray Generation

Also called: Camera Ray Generation, Raster-to-World Ray Construction, Ray Generation from Camera Matrix, Ray Initialization & AABB Intersection, getBound, initialRays

## Overview
Camera Matrix Ray Generation computes the origin and direction of a primary ray for each pixel in an image based on camera intrinsic parameters (focal length, principal point) and extrinsic parameters (camera-to-world pose) [S10, S13, S16, unofficial]. It solves the problem of mapping 2D raster coordinates to 3D world-space rays, enabling subsequent volume rendering or ray tracing operations [S1, S4, S5, unofficial]. Unlike orthographic generation which uses parallel rays, this method constructs perspective rays that diverge from a single viewpoint [S10, S16, unofficial]. It differs from sample streaming payload methods by explicitly constructing the ray geometry from camera matrices rather than deriving it from pre-computed payloads or object-space offsets [S1509, S1642].

## How it works
**Inputs:**
*   Image dimensions: $H$ (height), $W$ (width) [S10, S16, unofficial].
*   Camera intrinsic matrix $K$ (3x3) or parameters: focal lengths $f_x, f_y$, principal point $(c_x, c_y)$ [S10, S13, S27, unofficial].
*   Camera-to-world transformation matrix $T_{c2w}$ (4x4), where the upper-left 3x3 block is rotation $\mathbf{R}_{c2w}$ and the rightmost 3-column vector is translation $\mathbf{t}_{c2w}$ (camera origin) [S10, S16, S27, unofficial].
*   Optional: Axis-aligned bounding box (AABB) of the scene for early ray termination [S13, unofficial].

**Outputs:**
*   Ray origins $\mathbf{o}$: Tensor of shape $(H, W, 3)$ in world space [S10, S16, unofficial].
*   Ray directions $\mathbf{d}$: Tensor of shape $(H, W, 3)$ in world space (often normalized) [S10, S16, unofficial].
*   Optional: Near and far t-values ($t_{min}, t_{max}$) for AABB intersection [S13, unofficial].

**Data Structures:**
*   Pixel coordinate grids: 2D arrays of shape $(H, W)$ containing integer or float pixel indices $i$ (column, width) and $j$ (row, height) [S10, S16, S20, unofficial].
*   Intrinsic parameters: Extracted from $K$ as scalars or vectors [S10, S27, unofficial].
*   Transformation matrices: Stored as 4x4 matrices or separate rotation/translation components [S10, S16, unofficial].

**Steps:**

1.  **Generate Pixel Coordinates:** Create 2D meshgrids for pixel indices $i \in [0, W)$ and $j \in [0, H)$ [S10, S16, S20, unofficial].
2.  **Center Pixel Coordinates (Optional):** Add 0.5 to $i$ and $j$ to target pixel centers instead of corners [S7, S13, S20, unofficial]. Note: Some implementations omit this if calibration accuracy is insufficient [S8, unofficial].
3.  **Compute Camera-Space Directions:** Calculate unnormalized direction vectors in the camera coordinate system. The camera typically looks along the negative Z-axis (OpenGL convention) or positive Z-axis (OpenCV convention). For the standard OpenGL convention (looking down -Z):
    $$
    \mathbf{d}_{cam} = \begin{bmatrix} \frac{i - c_x}{f_x} \\ -\frac{j - c_y}{f_y} \\ -1 \end{bmatrix}
    $$
    where the negative sign on the Y-component accounts for the image Y-axis being opposite to the camera V-axis (down vs up) [S10, S20, S25, S26, unofficial]. For OpenCV convention (looking down +Z), the Z component is $1$ and Y sign may vary depending on image orientation [S9, S27, unofficial].
4.  **Transform Directions to World Space:** Rotate the camera-space directions into world space using the rotation part of the camera-to-world matrix:
    $$
    \mathbf{d}_{world} = \mathbf{d}_{cam} \cdot \mathbf{R}_{c2w}^\top
    $$
    or equivalently $\mathbf{d}_{world} = \mathbf{R}_{c2w} \mathbf{d}_{cam}$ if $\mathbf{d}_{cam}$ is a column vector [S10, S16, S26, S27, unofficial].
5.  **Set Ray Origins:** The ray origin is the camera position in world space, extracted from the translation component of $T_{c2w}$:
    $$
    \mathbf{o}_{world} = \mathbf{t}_{c2w}
    $$
    This origin is broadcast to match the shape of the directions $(H, W, 3)$ [S10, S16, S20, unofficial].
6.  **Normalize Directions (Optional):** Normalize $\mathbf{d}_{world}$ to unit length if required by the renderer [S10, S15, unofficial].
7.  **Compute AABB Intersection (Optional):** If an AABB is provided, compute $t_{min}$ and $t_{max}$ for each ray to determine the valid marching range [S13, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_rays(H, W, K, c2w, use_pixel_centers=True, compute_aabb=False, aabb_min=None, aabb_max=None):
    """
    Generate ray origins and directions for all pixels.
    
    Args:
        H: Image height
        W: Image width
        K: 3x3 intrinsic matrix [fx, 0, cx; 0, fy, cy; 0, 0, 1]
        c2w: 4x4 camera-to-world matrix
        use_pixel_centers: Boolean, add 0.5 to pixel coordinates
        compute_aabb: Boolean, compute t_min/t_max for AABB
        aabb_min: 3-vector, AABB minimum bounds
        aabb_max: 3-vector, AABB maximum bounds
        
    Returns:
        rays_o: (H, W, 3) ray origins
        rays_d: (H, W, 3) ray directions (normalized)
        t_min: (H, W) optional near t-values
        t_max: (H, W) optional far t-values
    """
    # 1. Extract intrinsics
    fx = K[0, 0]
    fy = K[1, 1]
    cx = K[0, 2]
    cy = K[1, 2]
    
    # 2. Extract extrinsics
    R_c2w = c2w[:3, :3] # 3x3 rotation
    t_c2w = c2w[:3, 3]  # 3-vector translation (camera origin)
    
    # 3. Generate pixel coordinates
    # Create meshgrid with 'ij' indexing: i varies along rows (height), j along cols (width)
    # However, standard convention often uses i for x (width) and j for y (height)
    # Let's align with S10: j is rows (0..H-1), i is cols (0..W-1)
    j, i = torch.meshgrid(torch.arange(H, dtype=torch.float32), 
                          torch.arange(W, dtype=torch.float32), 
                          indexing='ij')
    
    # 4. Center pixel coordinates
    if use_pixel_centers:
        i = i + 0.5
        j = j + 0.5
        
    # 5. Compute camera-space directions (OpenGL convention: look down -Z)
    # d_cam = [(i - cx)/fx, -(j - cy)/fy, -1]
    # Note: Negative sign on Y because image Y is down, camera V is up
    d_cam_x = (i - cx) / fx
    d_cam_y = -(j - cy) / fy
    d_cam_z = -torch.ones_like(i)
    
    d_cam = torch.stack([d_cam_x, d_cam_y, d_cam_z], dim=-1) # (H, W, 3)
    
    # 6. Transform directions to world space
    # d_world = d_cam @ R_c2w.T
    d_world = torch.sum(d_cam[..., None, :] * R_c2w, dim=-1) # (H, W, 3)
    
    # 7. Normalize directions
    d_world = d_world / torch.norm(d_world, dim=-1, keepdim=True)
    
    # 8. Set ray origins
    # Broadcast t_c2w to (H, W, 3)
    rays_o = t_c2w.unsqueeze(0).unsqueeze(0).expand(H, W, 3)
    
    # 9. Compute AABB intersection if requested
    t_min = None
    t_max = None
    if compute_aabb and aabb_min is not None and aabb_max is not None:
        # Ray-AABB intersection for each pixel
        # t_min = max((bbox_min - o) / d) for each axis, handling division by zero
        # t_max = min((bbox_max - o) / d) for each axis
        
        # Avoid division by zero by adding small epsilon or handling inf
        eps = 1e-8
        d_safe = torch.where(torch.abs(d_world) < eps, eps, d_world)
        
        t1 = (aabb_min - rays_o) / d_safe
        t2 = (aabb_max - rays_o) / d_safe
        
        # t_min is max of mins, t_max is min of maxs
        t_min = torch.maximum(torch.minimum(t1, t2), dim=-1).values
        t_max = torch.minimum(torch.maximum(t1, t2), dim=-1).values
        
        # Filter out rays that don't intersect (t_min > t_max)
        # Or clamp to valid range
        valid = t_min < t_max
        t_min = torch.where(valid, t_min, torch.tensor(0.0))
        t_max = torch.where(valid, t_max, torch.tensor(0.0))
        
    return rays_o, d_world, t_min, t_max
```

**Parameters:**
*   $f_x, f_y$: Focal lengths in pixels [S10, S18, unofficial].
*   $c_x, c_y$: Principal point coordinates in pixels [S10, S18, unofficial].
*   $H, W$: Image height and width in pixels [S10, S16, unofficial].
*   $T_{c2w}$: 4x4 camera-to-world matrix [S10, S16, unofficial].
*   `use_pixel_centers`: Boolean, default True [S20, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $f_x$ or $f_y$ is zero, or if ray direction components are zero during AABB intersection, division by zero occurs. IEEE 754 infinities may result, requiring explicit checks or clamping [S13, S20, unofficial].
*   **Coordinate System Conventions:** OpenGL cameras look down -Z, OpenCV down +Z. Image Y-axis is typically down, while camera V-axis is up. Signs in direction calculation must match these conventions [S10, S25, S26, unofficial].
*   **Pixel Centering:** Omitting +0.5 offset can cause slight misalignment, but may be acceptable if calibration is inaccurate [S8, unofficial].
*   **AABB Intersection:** If $t_{min} > t_{max}$, the ray does not intersect the AABB. This case must be handled to avoid invalid marching ranges [S13, unofficial].

## Subtopics

### [Pinhole Camera Ray Generation](5058-pinhole-camera-ray-generation.md)
Generates rays for a pinhole camera model using field of view (FOV) and image size instead of explicit intrinsic matrices. It calculates ray directions based on normalized device coordinates (NDC) scaled by $\tan(FOV/2)$. Choose this when camera intrinsics are not available but FOV and aspect ratio are known. It is a simplified variant of Camera Matrix Ray Generation that assumes a square pixel aspect ratio and zero skew.

## Operating characteristics
*   **Time Complexity:** $O(H \times W)$ for generating rays for all pixels, as each pixel's ray is computed independently [S10, S16, unofficial].
*   **Memory Complexity:** $O(H \times W)$ to store ray origins and directions [S10, S16, unofficial].
*   **Performance:** Highly parallelizable on GPU using compute shaders or tensor operations [S1, S10, unofficial]. Performance is limited by memory bandwidth when storing large ray buffers [S16, unofficial].
*   **Strengths:** Accurate perspective projection; supports arbitrary camera poses and intrinsics; efficient batch computation [S10, S16, unofficial].
*   **Limitations:** Requires accurate camera calibration; does not handle lens distortion or non-pinhole models without extension [S10, S25, unofficial].
*   **When to Use:** Standard perspective rendering, NeRF, volume rendering with known camera poses [S10, S16, unofficial].
*   **When Not to Use:** Orthographic rendering (use Orthographic Camera Ray Generation), real-time interactive rendering with dynamic camera where overhead is critical (though still efficient), or when lens effects are needed [S1319, S25].

## Background
Originating from standard computer graphics and photogrammetry, the method maps pixel coordinates to world rays using camera matrices [S10, S24, unofficial]. It is fundamental to NeRF (Neural Radiance Fields) implementations, where accurate ray generation is critical for training and rendering [S10, S20, unofficial]. The approach is widely used in volume rendering, ray tracing, and 3D reconstruction pipelines [S1, S5, S16, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S4] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S5] kkhese/Volume-Rendering, https://github.com/kkhese/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S7] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
[S8] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S9] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S10] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S13] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S15] an1pse/voxel-traversal, https://github.com/an1pse/voxel-traversal, fetched 2026-10-27 (unofficial)
[S16] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S20] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S24] 7.6. Neural Radiance Fields for Drones, https://www.roboticsbook.org/S76_drone_learning.html, fetched 2026-10-29 (unofficial)
[S25] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S26] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S27] https://github.com/WangFeng18/3d-gaussian-splatting, https://github.com/WangFeng18/3d-gaussian-splatting, fetched 2026-10-30 (unofficial)
