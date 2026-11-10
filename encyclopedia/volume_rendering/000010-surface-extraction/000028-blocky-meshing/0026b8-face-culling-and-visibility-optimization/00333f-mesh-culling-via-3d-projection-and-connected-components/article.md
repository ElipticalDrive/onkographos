# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method removes geometric artifacts and occluded geometry from a 3D mesh by projecting its vertices into multiple camera views and comparing their depths against reference depth maps [S1, unofficial]. It classifies mesh faces as "seen" (visible in sufficient views), "unseen" (occluded or background), or "forecast" (distant/unbounded) and culls those that do not meet visibility criteria [S1, unofficial]. It is distinct from neighbor-based culling (e.g., [Face Culling](8735-face-culling.md)) because it relies on multi-view consistency rather than local voxel adjacency, making it suitable for unbounded reconstructions like Gaussian Splatting where global visibility matters [S1, unofficial].

## How it works
**Inputs:**
- A triangle mesh $M = (V, F)$, where $V$ is the set of vertices and $F$ is the set of faces.
- A list of camera poses $C = \{c2w_i\}_{i=1}^N$ (world-to-camera transformation matrices).
- Camera intrinsic parameters $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$.
- Image resolution $(H, W)$.
- Reference depth maps $D_i \in \mathbb{R}^{H \times W}$ for each camera pose.
- Parameters: `points_batch_size` (max points per batch), `eps` (depth tolerance), `remove_small_geometry_threshold` (area ratio), `forecast_radius` (optional frustum expansion).

**Outputs:**
- A culled mesh $M'$ with unreferenced vertices removed.

**Data Structures:**
- `valid`: Boolean tensor of shape $(N_{pts})$, indicating if a point is "seen".
- `valid_num`: Integer tensor of shape $(N_{pts})$, counting how many cameras see the point.
- `face_mask`: Boolean array of shape $(N_{faces})$, true if all vertices of the face are valid.

**Steps:**

1. **Initialize Masks:**
   Create `valid`, `valid_num`, and `valid_forecast` tensors of zeros with length equal to the number of vertices $N_V$ on the target device (e.g., GPU) [S1, unofficial].

2. **Iterate Over Cameras:**
   For each camera pose $c2w_i$ and its corresponding depth map $D_i$:
   
   a. **Coordinate Transformation:**
      Adjust the camera-to-world matrix to match the expected coordinate system (e.g., OpenGL to OpenCV or vice versa). Typically, negate the Y and Z rows of the rotation part:
      $$ R' = \begin{bmatrix} R_{0,0} & -R_{0,1} & -R_{0,2} \\ R_{1,0} & -R_{1,1} & -R_{1,2} \\ R_{2,0} & -R_{2,1} & -R_{2,2} \end{bmatrix} $$
      $$ c2w' = [R' | t] $$
      Compute the world-to-camera matrix:
      $$ w2c = (c2w')^{-1} $$
      [S1, unofficial]

   b. **Project Vertices:**
      For a batch of vertices $V_{batch}$:
      - Convert to homogeneous coordinates: $V_h = [V_{batch}, \mathbf{1}]^T$.
      - Transform to camera space: $P_c = w2c \cdot V_h$.
      - Extract the first three rows to get $(x_c, y_c, z_c)$.
      [S1, unofficial]

   c. **Intrinsics Projection:**
      - Compute UV coordinates:
        $$ u = \frac{f_x \cdot x_c}{z_c} + c_x $$
        $$ v = \frac{f_y \cdot y_c}{z_c} + c_y $$
      - Compute depth $z = z_c + \epsilon$ (where $\epsilon = 10^{-8}$ to avoid division by zero).
      - Normalize UV for grid sampling:
        $$ u_{norm} = \frac{2u}{W} - 1 $$
        $$ v_{norm} = \frac{2v}{H} - 1 $$
      [S1, unofficial]

   d. **Frustum and Visibility Check:**
      - Check if points are within the standard frustum:
        $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z > 0) $$
      - If `forecast_radius` > 0, check an expanded frustum:
        $$ \text{forecast\_frustum} = (-r \le u < W+r) \land (-r \le v < H+r) \land (z > 0) $$
      [S1, unofficial]

   e. **Depth Sampling and Comparison:**
      - Reshape $D_i$ to $(1, 1, H, W)$.
      - Sample depth at $(u_{norm}, v_{norm})$ using bilinear interpolation with border padding:
        $$ d_{sample} = \text{GridSample}(D_i, [u_{norm}, v_{norm}]) $$
      - Validate front-facing geometry:
        $$ \text{is\_front} = (d_{sample} > 0) \land (z < d_{sample} + \text{eps}) $$
      [S1, unofficial]

   f. **Update Masks:**
      - For points in `in_frustum` and `is_front`, increment `valid_num` and set `valid` to true.
      - For points in `forecast_frustum`, update `valid_forecast`.
      [S1, unofficial]

3. **Finalize Point Visibility:**
   - A point is considered "seen" if `valid_num >= 20` (requires projection into at least 20 cameras) [S1, unofficial].
   - Combine `valid` and `valid_forecast` if forecasting is enabled.

4. **Face Masking:**
   - Create `face_mask` where a face is kept if all its vertices are valid:
     $$ \text{face\_mask}[f] = \bigwedge_{v \in f} \text{valid}[v] $$
   [S1, unofficial]

5. **Connected Components Filtering (Optional):**
   - If `forecast_radius` > 0 or for cleanup:
     - Identify connected components in the mesh using the `face_mask`.
     - Compute the area of each component.
     - Keep components where $\text{Area} > \text{remove\_small\_geometry\_threshold} \times \text{GlobalArea}$, or keep only the largest component if `get_largest_components` is true [S1, unofficial].
     - Alternatively, if forecasting is used, compute the Oriented Bounding Box (OBB) of the cull mesh vertices and keep only forecast faces within this OBB [S1, unofficial].

6. **Mesh Update:**
   - Apply `face_mask` to the mesh.
   - Remove unreferenced vertices.
   [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def cull_mesh_by_projection(mesh, cameras, depth_maps, H, W, fx, fy, cx, cy, 
                            eps=0.005, min_views=20, forecast_radius=0.0, 
                            remove_small_threshold=0.2, batch_size=500000):
    """
    Cull mesh faces based on multi-view depth projection.
    
    Args:
        mesh: Object with .vertices (N, 3) and .faces (M, 3)
        cameras: List of c2w matrices (N_cam, 4, 4)
        depth_maps: List of depth maps (N_cam, H, W)
        H, W: Image height and width
        fx, fy, cx, cy: Intrinsic parameters
        eps: Depth comparison tolerance
        min_views: Minimum number of cameras required to mark a point as 'seen'
        forecast_radius: Expansion for frustum check (0 to disable)
        remove_small_threshold: Minimum area ratio for component retention
        batch_size: Max vertices processed per batch
        
    Returns:
        Culled mesh object
    """
    V = mesh.vertices # (N, 3)
    F_idx = mesh.faces # (M, 3)
    N = V.shape[0]
    N_cam = len(cameras)
    
    # Initialize masks
    valid = torch.zeros(N, dtype=torch.bool, device=V.device)
    valid_num = torch.zeros(N, dtype=torch.int, device=V.device)
    
    # Process vertices in batches
    for start in range(0, N, batch_size):
        end = min(start + batch_size, N)
        V_batch = V[start:end]
        
        # Initialize batch masks
        batch_valid = torch.zeros(end - start, dtype=torch.bool, device=V.device)
        batch_valid_num = torch.zeros(end - start, dtype=torch.int, device=V.device)
        
        for i, (c2w, depth_map) in enumerate(zip(cameras, depth_maps)):
            # 1. Coordinate Transformation
            # Negate Y and Z rows of rotation part (example convention)
            c2w_adj = c2w.clone()
            c2w_adj[1:3, 1:3] *= -1
            
            # World-to-Camera
            w2c = torch.inverse(c2w_adj)
            
            # 2. Project Vertices
            # Homogeneous coordinates
            V_h = torch.cat([V_batch, torch.ones(end - start, 1, device=V.device)], dim=1) # (B, 4)
            # Camera space
            P_c = (w2c @ V_h.T).T # (B, 4) -> take first 3 rows
            x_c, y_c, z_c = P_c[:, 0], P_c[:, 1], P_c[:, 2]
            
            # 3. Intrinsics Projection
            z_safe = z_c + 1e-8
            u = (fx * x_c / z_safe) + cx
            v = (fy * y_c / z_safe) + cy
            
            # Normalize for grid sampling [-1, 1]
            u_norm = (2.0 * u / W) - 1.0
            v_norm = (2.0 * v / H) - 1.0
            
            # 4. Frustum Check
            in_frustum = (u >= 0) & (u <= W - 1) & (v >= 0) & (v <= H - 1) & (z_c > 0)
            
            # Forecast Frustum (if enabled)
            if forecast_radius > 0:
                forecast_frustum = (u >= -forecast_radius) & (u <= W - 1 + forecast_radius) & \
                                   (v >= -forecast_radius) & (v <= H - 1 + forecast_radius) & (z_c > 0)
            else:
                forecast_frustum = torch.zeros_like(in_frustum)
                
            # 5. Depth Sampling
            # Reshape depth map for grid_sample: (1, 1, H, W)
            depth_reshaped = depth_map.unsqueeze(0).unsqueeze(0)
            
            # Create grid for grid_sample: (1, H, W, 2) -> we need (1, B, 2) for batched sampling?
            # PyTorch grid_sample expects input (N, C, H, W) and grid (N, H_out, W_out, 2)
            # To sample N points, we can use F.grid_sample with a grid of shape (1, 1, N, 2)
            grid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(1) # (1, 1, B, 2)
            
            # Sample depth
            depth_sampled = F.grid_sample(depth_reshaped, grid, align_corners=True, padding_mode='border') # (1, 1, 1, B)
            depth_sampled = depth_sampled.squeeze().squeeze() # (B,)
            
            # 6. Front-Face Validation
            is_front = (depth_sampled > 0.0) & (z_c < depth_sampled + eps)
            
            # 7. Update Masks
            # Seen if in frustum and front-facing
            seen = in_frustum & is_front
            batch_valid_num += seen.int()
            batch_valid = batch_valid | seen
            
            # Forecast update (if needed)
            # Note: The source logic implies separate handling for forecast, 
            # but typically 'valid' is the final mask. We'll assume forecast is a subset or separate mask.
            # For simplicity, we merge forecast into valid if it meets criteria, 
            # or keep separate if the algorithm requires distinct OBB culling later.
            # Based on [S1, unofficial], forecast is used for OBB culling of "forecast faces".
            # Here we just accumulate 'seen' points.
            
        # Update global masks
        valid[start:end] = batch_valid
        valid_num[start:end] = batch_valid_num
        
    # Finalize point visibility
    final_valid = valid_num >= min_views
    
    # Face Mask: All vertices of a face must be valid
    # F_idx is (M, 3), final_valid is (N,)
    # face_valid = final_valid[F_idx] -> (M, 3)
    face_mask = final_valid[F_idx].all(dim=1)
    
    # Connected Components Filtering (Simplified)
    # In practice, this requires graph traversal on the mesh faces.
    # We assume a helper function exists or implement a basic area check.
    # For this pseudocode, we skip the complex graph traversal and assume 
    # the mesh object has a method to filter by component area.
    
    # Apply face mask
    mesh.faces = mesh.faces[face_mask]
    
    # Remove unreferenced vertices
    # This requires finding unique vertices in the remaining faces
    unique_verts = torch.unique(mesh.faces.flatten())
    # Create a mapping from old index to new index
    # And update mesh.vertices accordingly
    # [G] Standard mesh cleanup step not fully detailed in source, implemented generically.
    new_verts = mesh.vertices[unique_verts]
    face_indices = torch.searchsorted(unique_verts, mesh.faces.flatten()).reshape_as(mesh.faces)
    mesh.vertices = new_verts
    mesh.faces = face_indices
    
    return mesh
```

**Parameters:**
- `points_batch_size`: Maximum number of points processed per batch to limit GPU memory. Default: $5 \times 10^5$ [S1, unofficial].
- `remove_small_geometry_threshold`: Minimum face area ratio to keep a mesh component. Default: 0.2 [S1, unofficial].
- `eps`: Depth comparison tolerance for front-face validation. Default: 0.005 [S1, unofficial].
- `min_views`: Minimum number of cameras required to mark a point as 'seen'. Default: 20 [S1, unofficial].
- `forecast_radius`: Radius for expanding the frustum check for distant geometry. Default: 0 (disabled) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Depth comparison uses `eps` to handle floating-point inaccuracies. If `eps` is too small, valid faces may be culled; if too large, occluded geometry may remain [S1, unofficial].
- **Boundaries:** UV coordinates outside $[0, W-1]$ and $[0, H-1]$ are handled by border padding in `grid_sample`. Points outside the frustum are ignored [S1, unofficial].
- **Degenerate Input:** If the mesh has no vertices or faces, the method returns an empty mesh. If depth maps are missing or invalid, the visibility check fails, potentially culling all geometry [S1, unofficial].
- **Coordinate Systems:** The transformation of `c2w` (negating Y and Z) is specific to the coordinate system convention (e.g., OpenGL to OpenCV). Mismatched conventions lead to incorrect projections [S1, unofficial].

## Subtopics

### [Mesh Post-Processing (Cluster Filtering)](7002-mesh-post-processing-cluster-filtering.md)
Filters out floating artifacts and disconnected components by clustering triangles and retaining only the largest connected clusters. It serves as a cleanup step following this method, specifically targeting small, noisy geometry that survives projection-based culling. Unlike this method which removes occluded or distant geometry, this method removes topologically isolated fragments based on triangle count.

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$, where $N$ is the number of vertices and $C$ is the number of cameras. Each vertex is projected and sampled against each depth map.
- **Memory Complexity:** $O(N + C \cdot H \cdot W)$ for storing vertex masks and depth maps. Batch processing limits vertex memory usage to `points_batch_size` [S1, unofficial].
- **Measured Performance:** On the DTU dataset, this method contributes to geometric reconstruction with a mean Chamfer distance of 0.74 (Reproduce) compared to 0.80 (Paper) [S1, unofficial].
- **Strengths:** Effective for unbounded scenes and Gaussian Splatting reconstructions where local voxel adjacency is insufficient. Removes background noise and occluded geometry effectively.
- **Limitations:** Requires multiple camera views and depth maps. Computationally expensive for large meshes and many cameras. Sensitive to coordinate system conventions and depth map accuracy.
- **When to Use:** When reconstructing geometry from multi-view images (e.g., NeRF, Gaussian Splatting) and global visibility is important.
- **When Not to Use:** For single-view reconstruction or when computational resources are limited and local culling is sufficient.

## Background
This method originates from the "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" paper by Huang et al. (SIGGRAPH 2024) [S1, unofficial]. It was developed to address the challenge of extracting clean, geometrically accurate meshes from unbounded Gaussian Splatting reconstructions, where traditional voxel-based culling methods struggle with large, unbounded scenes. The method leverages multi-view consistency to distinguish between visible surface geometry and background artifacts.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
