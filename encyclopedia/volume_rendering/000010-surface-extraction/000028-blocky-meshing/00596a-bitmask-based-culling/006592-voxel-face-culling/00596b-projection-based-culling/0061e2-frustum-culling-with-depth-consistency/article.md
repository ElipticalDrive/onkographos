# Frustum Culling with Depth Consistency


## Overview
Frustum Culling with Depth Consistency is a visibility filtering method for triangle meshes that retains only faces observed by multiple camera views and consistent with reference depth maps. It solves the problem of noisy or incomplete mesh reconstruction by discarding triangles that lie outside the camera frustums or are occluded by closer geometry in the input depth data. Unlike simple frustum culling, it enforces depth consistency against ground-truth or rendered depth maps, and unlike backface culling, it relies on multi-view observation counts rather than surface normals.

## How it works
**Inputs:**
- A triangle mesh (vertices $V$, faces $F$).
- A set of $N$ camera poses, each defined by a world-to-camera matrix $C_i$ and intrinsics $K_i = \text{diag}(f_{x,i}, f_{y,i}, 1)$ with principal point $(c_{x,i}, c_{y,i})$ and image dimensions $H_i \times W_i$.
- Reference depth maps $D_i$ for each camera pose (optional, if `remove_missing_depth` is enabled).
- Boolean flags: `subdivide`, `remove_occlusion`, `remove_missing_depth`.
- Parameters: `max_edge` (float), `th1` (int, default 3), `eps` (float, default 0.02), `inv_ratio` (float, default 0.7).

**Outputs:**
- A filtered mesh containing only triangles where at least one vertex is observed in $\ge th1$ views and the triangle is not marked invalid by depth consistency checks.

**Data Structures:**
- `obs_mask`: Integer array of shape $(|V|, N)$, storing the count of views in which each vertex is inside the frustum and not occluded.
- `invalid_mask`: Integer array of shape $(|V|, N)$, storing the count of views in which each vertex projects to a pixel with invalid (missing) depth.
- `triangle_valid`: Boolean array of shape $(|F|, 1)$, indicating final validity of each triangle.

**Steps:**

1. **Subdivision (Optional):** If `subdivide` is true, subdivide the mesh such that no edge exceeds `max_edge`. This is typically done using a subdivision algorithm with a maximum iteration limit (e.g., 10) to ensure sufficient sampling density for depth consistency checks [S1, unofficial].

2. **Coordinate System Conversion:** For each camera pose $i$, convert the world-to-camera matrix $C_i$ to align with the rendering convention (e.g., OpenCV/pyrender). This involves flipping the Y and Z axes:
   $$ C'_i = C_i \cdot \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & -1 & 0 & 0 \\ 0 & 0 & -1 & 0 \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
   Extract rotation $R_i$ and translation $t_i$ from $C'_i$ [S1, unofficial].

3. **Depth Rendering (Optional):** If `remove_occlusion` is true, render a depth map $D^{render}_i$ for the current mesh state from camera $i$ using a rasterizer with depth-only output [S1, unofficial].

4. **Per-Vertex Visibility Aggregation:** Initialize `obs_mask` and `invalid_mask` to zeros. For each vertex $v \in V$ and each camera $i$:
   a. **Transform to Camera Space:**
      $$ p_{cam} = R_i \cdot v + t_i $$
   b. **Project to NDC:**
      $$ \begin{bmatrix} u' \\ v' \\ w' \end{bmatrix} = K_i \cdot p_{cam} $$
   c. **Compute Pixel Coordinates and Depth:**
      $$ z_c = w' + 10^{-8} $$
      $$ u = \frac{u'}{z_c}, \quad v = \frac{v'}{z_c} $$
   d. **Frustum Test:** Check if the point is within the image bounds and in front of the camera:
      $$ \text{in\_frustum} = (0 \le u < W_i) \land (0 \le v < H_i) \land (z_c > 0) $$
   e. **Occlusion Test (if `remove_occlusion`):** If `in_frustum` is true, sample the rendered depth $D^{render}_i[v, u]$. The point is considered visible (not occluded) if:
      $$ z_c < D^{render}_i[v, u] + \text{eps} $$
      If visible, increment `obs_mask[v, i]`.
   f. **Missing Depth Test (if `remove_missing_depth`):** If `in_frustum` is true, sample the reference depth $D_i[v, u]$. If $D_i[v, u] \le 0.0$, the pixel is invalid. Increment `invalid_mask[v, i]` [S1, unofficial].

5. **Triangle Validity Determination:** For each triangle $f$ with vertices $v_1, v_2, v_3$:
   a. **Observation Count:** Calculate the maximum observation count among its vertices:
      $$ \text{obs}_{max} = \max_{j \in \{1,2,3\}} \left( \sum_{i=1}^{N} \text{obs\_mask}[v_j, i] \right) $$
      The triangle is considered observed if $\text{obs}_{max} \ge th1$.
   b. **Invalidity Check:** Calculate the invalid ratio for each vertex:
      $$ \text{inv\_ratio}_j = \frac{\sum_{i=1}^{N} \text{invalid\_mask}[v_j, i]}{\sum_{i=1}^{N} \text{obs\_mask}[v_j, i] + 10^{-8}} $$
      The triangle is marked invalid if all three vertices have an invalid ratio greater than `inv_ratio`:
      $$ \text{inv\_ratio}_1 > \text{inv\_ratio} \land \text{inv\_ratio}_2 > \text{inv\_ratio} \land \text{inv\_ratio}_3 > \text{inv\_ratio} $$
   c. **Final Mask:** A triangle is retained if it is observed AND NOT invalid [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def frustum_culling_depth_consistency(mesh, cameras, depth_maps=None, 
                                      subdivide=True, max_edge=0.015, 
                                      remove_occlusion=True, remove_missing_depth=False,
                                      th1=3, eps=0.02, inv_ratio=0.7):
    """
    Filter mesh triangles based on multi-view frustum visibility and depth consistency.
    
    Args:
        mesh: Object with vertices (N, 3) and faces (M, 3).
        cameras: List of dicts with keys 'c2w', 'K', 'H', 'W'.
        depth_maps: List of depth maps (H, W) corresponding to cameras.
        subdivide: Whether to subdivide mesh before culling.
        max_edge: Max edge length for subdivision.
        remove_occlusion: Use rendered depth to remove occluded faces.
        remove_missing_depth: Use ground-truth depth to remove faces in missing regions.
        th1: Min number of views a vertex must be observed in.
        eps: Tolerance for depth consistency check.
        inv_ratio: Threshold for invalid depth ratio.
        
    Returns:
        Filtered mesh.
    """
    # Step 1: Subdivision
    if subdivide:
        mesh = subdivide_mesh(mesh, max_edge=max_edge, max_iter=10)
        
    V = mesh.vertices  # Shape (N, 3)
    F = mesh.faces     # Shape (M, 3)
    N_views = len(cameras)
    
    # Initialize masks
    obs_mask = np.zeros((len(V), N_views), dtype=np.int32)
    invalid_mask = np.zeros((len(V), N_views), dtype=np.int32)
    
    # Coordinate system conversion matrix (OpenCV/pyrender convention)
    coord_convert = np.array([
        [1, 0, 0, 0],
        [0, -1, 0, 0],
        [0, 0, -1, 0],
        [0, 0, 0, 1]
    ])
    
    # Pre-render depth maps if occlusion removal is enabled
    rendered_depths = None
    if remove_occlusion:
        rendered_depths = render_depths(mesh, cameras)
        
    # Step 4: Per-Vertex Visibility Aggregation
    for i, cam in enumerate(cameras):
        # Extract rotation and translation from converted c2w
        c2w = cam['c2w'] @ coord_convert
        R = c2w[:3, :3]
        t = c2w[:3, 3]
        K = cam['K']
        H, W = cam['H'], cam['W']
        
        # Transform vertices to camera space
        # V is (N, 3), R is (3, 3), t is (3,)
        V_cam = (R @ V.T).T + t  # Shape (N, 3)
        
        # Project to NDC
        # K is (3, 3), V_cam is (N, 3)
        uvz = (K @ V_cam.T).T  # Shape (N, 3)
        
        # Compute pixel coordinates and depth
        pz = uvz[:, 2] + 1e-8
        px = uvz[:, 0] / pz
        py = uvz[:, 1] / pz
        
        # Frustum test
        in_frustum = (px >= 0) & (px < W) & (py >= 0) & (py < H) & (pz > 0)
        
        # Occlusion test
        if remove_occlusion:
            # Sample rendered depth
            # Note: In practice, this requires efficient sampling or rasterization
            # Here we assume a function to sample rendered depth at pixel coordinates
            rendered_d = sample_depth(rendered_depths[i], px, py, H, W)
            not_occluded = pz < (rendered_d + eps)
            visible = in_frustum & not_occluded
            obs_mask[visible, i] += 1
            
        # Missing depth test
        if remove_missing_depth:
            gt_d = depth_maps[i]
            # Sample ground truth depth
            gt_d_sampled = sample_depth(gt_d, px, py, H, W)
            invalid_depth = in_frustum & (gt_d_sampled <= 0.0)
            invalid_mask[invalid_depth, i] += 1
            
    # Step 5: Triangle Validity Determination
    triangle_obs_counts = np.zeros(len(F), dtype=np.int32)
    triangle_invalid_flags = np.zeros(len(F), dtype=bool)
    
    for j, face in enumerate(F):
        v_indices = face
        v_obs_counts = np.sum(obs_mask[v_indices], axis=1)
        v_inv_counts = np.sum(invalid_mask[v_indices], axis=1)
        
        # Check observation count
        max_obs = np.max(v_obs_counts)
        is_observed = max_obs >= th1
        
        # Check invalidity
        # Avoid division by zero
        v_inv_ratios = v_inv_counts / (v_obs_counts + 1e-8)
        is_invalid = np.all(v_inv_ratios > inv_ratio)
        
        if is_observed and not is_invalid:
            triangle_obs_counts[j] = 1
        else:
            triangle_invalid_flags[j] = True
            
    # Filter faces
    valid_faces = np.where(~triangle_invalid_flags)[0]
    return filter_mesh(mesh, valid_faces)

def sample_depth(depth_map, px, py, H, W):
    """Sample depth map at pixel coordinates."""
    # Clamp coordinates
    px = np.clip(px, 0, W - 1)
    py = np.clip(py, 0, H - 1)
    # Convert to integer indices
    u = px.astype(np.int32)
    v = py.astype(np.int32)
    # Sample depth
    return depth_map[v, u]

def render_depths(mesh, cameras):
    """Render depth maps for the mesh from all camera poses."""
    # Implementation depends on the renderer (e.g., pyrender)
    # Returns list of depth maps
    pass

def subdivide_mesh(mesh, max_edge, max_iter):
    """Subdivide mesh to meet max_edge constraint."""
    # Implementation depends on the subdivision library (e.g., trimesh)
    pass
```

## Operating characteristics
- **Time Complexity:** $O(N \cdot V \cdot C)$, where $N$ is the number of views, $V$ is the number of vertices, and $C$ is the cost of depth sampling/rendering. Subdivision increases $V$.
- **Memory Complexity:** $O(N \cdot V)$ for storing observation and invalid masks.
- **Strengths:** Effectively removes noise and floating artifacts by requiring multi-view consistency. Handles occlusions explicitly via depth maps.
- **Limitations:** Requires accurate camera poses and depth maps. Performance degrades with large numbers of views or high-resolution depth maps. Subdivision can be computationally expensive for dense meshes.
- **When to Use:** When cleaning up meshes reconstructed from multi-view RGB-D data (e.g., NeRF, SLAM) where depth maps are available.
- **When Not to Use:** When depth maps are unavailable or unreliable, or when real-time performance is critical and $N$ is large.

## Background
This method is implemented in the `dn-splatter` project [S1, unofficial]. It is designed for post-processing meshes generated from neural implicit representations or multi-view stereo, particularly in the context of datasets like MuSHRoom, Replica, and ScanNet++ [S1, unofficial]. The approach combines standard frustum culling with depth consistency checks to ensure that retained geometry is both visible and geometrically consistent with input sensor data.

## Sources
[S1] maturk/dn-splatter, https://github.com/maturk/dn-splatter, fetched 2026-10-27 (unofficial)
