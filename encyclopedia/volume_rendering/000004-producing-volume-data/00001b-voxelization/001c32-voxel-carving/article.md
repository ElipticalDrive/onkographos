# Voxel Carving

Also called: Carve Voxelization

## Overview
Voxel Carving is a technique that reconstructs a dense, filled 3D volumetric representation from a set of 2D depth maps or silhouettes captured from multiple viewpoints [S1]. Unlike surface-only voxelization methods that populate only the boundary voxels, Voxel Carving fills the interior volume consistent with the input views, creating a solid model of the given shape [S1]. It is distinguished from sibling methods like [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by its reliance on multi-view geometric consistency (silhouette intersection or depth truncation) rather than single-view rasterization or point cloud projection [S1].

## How it works
**Inputs:**
1.  A set of $N$ depth images or silhouette masks, each associated with a known camera pose (intrinsic and extrinsic parameters).
2.  A 3D bounding box defined by origin $(x_0, y_0, z_0)$ and dimensions $(W, H, D)$, or a voxel resolution $R$ and cubic size $S$.

**Outputs:**
A dense 3D voxel grid (boolean occupancy or density values) where a voxel is marked as occupied if it is consistent with all input views.

**Data Structures:**
*   **Voxel Grid:** A 3D array of size $N_x \times N_y \times N_z$.
    *   $N_x = \lceil W / v_{size} \rceil$, $N_y = \lceil H / v_{size} \rceil$, $N_z = \lceil D / v_{size} \rceil$.
    *   $v_{size}$ is the physical size of one voxel (e.g., $S / R$).
    *   Each element stores occupancy status (boolean) or density/color.
*   **Camera Parameters:**
    *   Intrinsic matrix $K$ (focal length, principal point).
    *   Extrinsic matrix $T$ (rotation $R_{cam}$ and translation $t_{cam}$) mapping world coordinates to camera coordinates.

**Steps:**

1.  **Initialize Grid:** Create a dense voxel grid with dimensions derived from the bounding box and voxel resolution. Initialize all voxels to "free" (unoccupied) or "unknown" [S1].
2.  **Iterate Views:** For each camera pose $i$ from $1$ to $N$:
    a.  **Project Voxels:** For each voxel $v$ in the grid, compute its 3D world position $P_v$.
    b.  **Transform to Camera Space:** Compute $P'_v = T_i \cdot P_v$.
    c.  **Project to Image Plane:** Compute pixel coordinates $(u, v)$ using intrinsic matrix $K$:
        $$
        \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} \propto K \cdot P'_v
        $$
    d.  **Check Consistency:**
        *   **Silhouette Method:** If $(u, v)$ falls within the image bounds and the pixel is part of the object silhouette in view $i$, the voxel is potentially occupied. If the pixel is background, the voxel is marked "free" [S1].
        *   **Depth Map Method:** If $(u, v)$ is valid, retrieve the depth $d_{img}$ from the depth map. Compute the distance $d_{vox}$ from the camera center to the voxel center. If $d_{vox} > d_{img}$ (voxel is behind the surface), mark the voxel as "free" [S1].
3.  **Finalize:** After processing all views, voxels remaining as "occupied" (or not marked "free") constitute the reconstructed volume [S1].

**Parameters:**
*   `voxel_resolution`: Controls grid density. $v_{size} = S / \text{voxel\_resolution}$ [S1].
*   `cubic_size`: Defines the bounding box dimension $S$. The grid is typically centered at the origin with bounds $[-S/2, S/2]$ [S1].
*   `dilated carving` (`-dc`): A binvox-specific option that stops carving 1 voxel before the intersection point to prevent over-thinning or artifacts [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_carving(depth_maps, camera_poses, voxel_resolution, cubic_size):
    """
    depth_maps: List of 2D arrays (H, W) containing depth values.
    camera_poses: List of tuples (K, T), where K is 3x3 intrinsic, T is 4x4 extrinsic.
    voxel_resolution: Integer, number of voxels along the largest dimension.
    cubic_size: Float, physical size of the bounding box.
    """
    # 1. Initialize Grid
    # Grid dimensions
    Nx = Ny = Nz = int(cubic_size / (cubic_size / voxel_resolution))
    voxel_size = cubic_size / voxel_resolution
    
    # Create grid, initialized to True (occupied)
    # In practice, start with all True and carve out free space, 
    # or start with all False and fill. Silhouette carving usually starts with 
    # the bounding box and removes voxels outside silhouettes.
    # Here we assume a "carve" approach: start with all occupied, remove if inconsistent.
    grid = np.ones((Nz, Ny, Nx), dtype=bool)
    
    # Origin of the grid in world space (centered at 0)
    origin = np.array([-cubic_size / 2.0, -cubic_size / 2.0, -cubic_size / 2.0])
    
    # Precompute voxel centers
    # This is expensive for large grids; often optimized by iterating only visible voxels
    # For clarity, we iterate all voxels.
    voxel_centers = []
    for z in range(Nz):
        for y in range(Ny):
            for x in range(Nx):
                # World coordinate of voxel center
                wx = origin[0] + (x + 0.5) * voxel_size
                wy = origin[1] + (y + 0.5) * voxel_size
                wz = origin[2] + (z + 0.5) * voxel_size
                voxel_centers.append(np.array([wx, wy, wz, 1.0]))
    
    voxel_centers = np.array(voxel_centers)
    
    # 2. Iterate through each camera view
    for depth_map, (K, T) in zip(depth_maps, camera_poses):
        H, W = depth_map.shape
        
        # Inverse extrinsic to transform camera to world? 
        # T is World -> Camera. We need Camera -> World to project voxel centers?
        # Actually, we project World -> Camera to check against depth map.
        # P_cam = T * P_world
        
        for idx, voxel_center_world in enumerate(voxel_centers):
            if not grid.flat[idx]:
                continue # Already marked free
            
            # Transform to camera space
            P_cam = T.dot(voxel_center_world)
            
            # Project to image plane
            # P_cam is [x, y, z, 1]. We use the first 3 components for projection
            x_cam, y_cam, z_cam = P_cam[:3]
            
            # Check if in front of camera
            if z_cam <= 0:
                grid.flat[idx] = False
                continue
                
            # u, v coordinates
            u = (K[0, 0] * x_cam / z_cam) + K[0, 2]
            v = (K[1, 1] * y_cam / z_cam) + K[1, 2]
            
            # Check if within image bounds
            if u < 0 or u >= W - 1 or v < 0 or v >= H - 1:
                grid.flat[idx] = False
                continue
            
            # Get depth from image
            # Note: depth_map values are typically in meters or pixels. 
            # Assuming meters to match voxel coordinates.
            d_img = depth_map[int(v), int(u)]
            
            # Check depth consistency
            # If voxel is further than the observed surface, it is free
            if z_cam > d_img:
                grid.flat[idx] = False
                
    return grid
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point errors in projection can cause voxels near the silhouette boundary to be incorrectly classified. Sub-pixel interpolation or anti-aliasing in depth maps helps [S1].
*   **Boundaries:** Voxels partially outside the camera frustum must be handled. If a voxel projects outside the image, it is typically considered free (background) [S1].
*   **Calibration Errors:** Inaccurate camera poses lead to "ghosting" or holes in the volume. Approximate Space Carving methods exist to handle calibration errors [S3, unofficial].
*   **Binvox Bugs:** Historical versions of binvox had rendering bugs in v0.48 and y-direction bugs in v1.38 [S2].

## Subtopics

### Streaming GPU-based Voxel Carving
A method that creates an explicit volume using only surface voxels in a GPU-optimal 2D data representation, avoiding 3D texture drawbacks and minimizing GPU-CPU data transfer through load balancing [S3, unofficial]. Unlike previous approaches (LiMS04, WoeKoch04, ZacKar04), it maintains interactivity and speed [S3, unofficial].

### Interactive Multi-resolution Voxel Carving
A CPU-based method using GVC-IB as its core, adding user-driven interactive refinement. Users assign perceptual significance to features via 2D image editing, allowing varying resolution and level of detail across the voxel structure [S3, unofficial].

### Cell Carving
An algorithm using correspondence between arbitrary image regions to reconstruct concavities that are difficult or impossible to reconstruct with standard silhouette intersection methods [S3, unofficial].

### Voxel Space Warping
A method that warps the voxel space to model large-scale scenes (e.g., outdoors) without requiring an excessive number of voxels, effectively adapting resolution to scene scale [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of views and $V$ is the number of voxels in the grid. This is computationally expensive for high-resolution grids [S3, unofficial].
*   **Memory Complexity:** $O(V)$ for the dense grid. Explicit volume representations can consume significant memory [S3, unofficial].
*   **Performance:** Typically requires tens of seconds to minutes for reconstruction, depending on resolution and view count [S3, unofficial]. GPU acceleration can improve this significantly [S3, unofficial].
*   **Strengths:** Produces dense, filled volumes; robust to noise if multiple views are available; handles concavities better than single-view methods [S1][S3, unofficial].
*   **Limitations:** Requires accurate camera calibration; sensitive to view coverage (holes if views are sparse); computationally intensive for high resolutions [S3, unofficial].
*   **When to use:** When a dense, filled volume is required and multiple calibrated views are available.
*   **When not to use:** When real-time performance is critical and resolution can be low; when camera calibration is unknown or inaccurate (use Approximate Space Carving) [S3, unofficial].

## Background
Voxel Carving, also known as Space Carving, was introduced by Seitz and Dyer in 1996 [S3, unofficial]. It is based on the principle of visual hulls, where the 3D shape is the intersection of visual hulls from multiple views. The method has been extended to handle uncalibrated views, multi-resolution representations, and GPU acceleration [S3, unofficial]. It is widely used in 3D reconstruction, computer vision, and graphics for creating volumetric models from image sequences [S1][S3, unofficial].

## Sources
[S1] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S2] binvox 3D mesh voxelizer, keywords: voxelization, voxelisation, 3D model, https://www.patrickmin.com/binvox/, fetched 2026-10-29
[S3] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
