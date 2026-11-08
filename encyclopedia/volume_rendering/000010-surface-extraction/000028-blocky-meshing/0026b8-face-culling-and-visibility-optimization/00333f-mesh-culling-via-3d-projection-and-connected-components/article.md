# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method culls mesh geometry by projecting vertices into multiple camera views and comparing their depths against reference depth maps to identify and remove unseen or artifact-prone faces [S1, unofficial]. It is designed to clean up unbounded Gaussian splatting reconstructions by classifying points as seen, unseen, or forecast, and removing geometry that does not correspond to front-facing surfaces visible from a sufficient number of viewpoints [S1, unofficial]. Unlike parent methods that rely on voxel neighbor checks, this approach uses explicit camera projection and depth validation [Parent].

## How it works
**Inputs:**
- A triangular mesh (vertices $V$, faces $F$).
- A list of camera poses (world-to-camera matrices $W_i$ or camera-to-world $C_i$).
- Camera intrinsics $K$ (focal lengths $f_x, f_y$, principal point $c_x, c_y$, image dimensions $H, W$).
- Reference depth maps $D_i$ for each camera pose.
- Parameters: `points_batch_size` (default $5 \times 10^5$), `eps` (default $0.005$), `remove_small_geometry_threshold` (default $0.2$), `forecast_radius` (default $0$, optional), `cluster_to_keep` (default $1000$), `min_cluster_size` (default $50$).

**Outputs:**
- A culled mesh with unreferenced vertices removed.

**Data Structures:**
- `valid`: Boolean tensor of shape $(N_{pts})$, true if a point is "seen" (visible in $\ge 20$ cameras).
- `valid_forecast`: Boolean tensor of shape $(N_{pts})$, true if a point is in the "forecast" zone (visible in $\ge 1$ camera but potentially outside strict bounds).
- `face_mask`: Boolean array of shape $(N_{faces})$, true if all vertices of the face are valid.

**Steps:**

1. **Extract Reference Depth Maps:**
   Render forward depth maps from the input mesh using the provided camera poses and intrinsics. Use a renderer (e.g., PyRender) with flags `OFFSCREEN`, `DEPTH_ONLY`, and `SKIP_CULL_FACES` to generate $D_i$ for each camera $i$ [S1, unofficial]. Set far plane to $20.0$ [S1, unofficial].

2. **Initialize Masks:**
   Create tensors `valid`, `valid_num` (count), and `valid_forecast` of shape $(N_{vertices})$ initialized to zero on the target device [S1, unofficial].

3. **Iterate Camera Poses:**
   For each camera pose $C_i$ (world-to-camera $W_i$):
   a. **Coordinate Transformation:**
      If $C_i$ is given as camera-to-world, negate the y and z rows of the rotation matrix to convert to world-to-camera orientation: $W_i[:3, 1:3] *= -1$ [S1, unofficial]. Compute $W_i = C_i^{-1}$ if not already provided [S1, unofficial].
   
   b. **Project Vertices:**
      For a batch of vertices $V_{batch}$:
      i. Convert to homogeneous coordinates: $V_h = [V_{batch}, 1]$ [S1, unofficial].
      ii. Transform to camera space: $V_c = W_i \cdot V_h$ [S1, unofficial].
      iii. Extract first three rows: $uvz = V_c[:3]$ [S1, unofficial].
      iv. Compute depth: $z = uvz[:, 2] + 10^{-8}$ [S1, unofficial].
      v. Normalize UV: $u = uvz[:, 0] / z$, $v = uvz[:, 1] / z$ [S1, unofficial].
   
   c. **Frustum and Depth Validation:**
      i. Check standard frustum bounds:
         $$ \text{in\_frustum} = (u \ge 0) \land (u \le W-1) \land (v \ge 0) \land (v \le H-1) \land (z > 0) $$
         [S1, unofficial]
      ii. Check forecast frustum (if `forecast_radius` $r > 0$):
         $$ \text{forecast\_frustum} = (u \ge -r) \land (u \le W-1+r) \land (v \ge -r) \land (v \le H-1+r) \land (z > 0) $$
         [S1, unofficial]
      iii. Sample reference depth:
          Reshape $D_i$ to $(1, 1, H, W)$. Normalize UV grid to $[-1, 1]$. Sample depth at $(u, v)$ using bilinear interpolation with border padding:
          $$ D_{sample} = \text{grid\_sample}(D_i, \text{vgrid}, \text{padding\_mode}='border', \text{align\_corners}=\text{True}) $$
          [S1, unofficial]
      iv. Validate front-facing geometry:
          $$ \text{is\_front\_face} = (D_{sample} > 0.0) \land (z < D_{sample} + \epsilon) $$
          [S1, unofficial]
      v. Update masks:
          - If `in_frustum` and `is_front_face`: increment `valid_num` and set `valid` to true for these vertices.
          - If `forecast_frustum` and `is_front_face`: set `valid_forecast` to true.
          [S1, unofficial]

4. **Finalize Point Masks:**
   - `valid = valid_num >= 20` [S1, unofficial].
   - Combine `valid` and `valid_forecast` if forecast culling is enabled.

5. **Cull Mesh Faces:**
   - Create `face_mask` where a face is kept if all its vertices are valid:
     $$ \text{face\_mask} = \text{valid}[F].\text{all}(\text{axis}=1) $$
     [S1, unofficial]
   - Update mesh faces with `face_mask`.
   - Remove unreferenced vertices [S1, unofficial].

6. **Optional: Connected Component Filtering (Cluster Filtering):**
   a. Identify connected triangle clusters in the culled mesh using `cluster_connected_triangles()` [S1, unofficial].
   b. Sort clusters by triangle count. Select the N-th largest value (where N is `cluster_to_keep`) as the threshold [S1, unofficial].
   c. Remove triangles belonging to clusters smaller than the threshold or smaller than `min_cluster_size` [S1, unofficial].
   d. Remove unreferenced vertices and degenerate triangles [S1, unofficial].

7. **Optional: Area-Based Filtering:**
   - Filter mesh components by area. Keep components where area $> \text{remove\_small\_geometry\_threshold} \times \text{global\_area}$, or keep only the largest component if `get_largest_components` is True [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
import numpy as np

def cull_mesh_by_projection(mesh, cameras, depth_maps, intrinsics, 
                            points_batch_size=5e5, eps=0.005, 
                            forecast_radius=0, remove_small_geometry_threshold=0.2,
                            cluster_to_keep=1000, min_cluster_size=50):
    """
    Cull mesh geometry using multi-view depth projection.
    
    Args:
        mesh: Open3D or trimesh mesh object with vertices and faces.
        cameras: List of camera poses (world-to-camera matrices or camera-to-world).
        depth_maps: List of depth maps corresponding to cameras.
        intrinsics: Camera intrinsic matrix K or parameters (fx, fy, cx, cy, H, W).
        points_batch_size: Max vertices processed per batch.
        eps: Depth comparison tolerance.
        forecast_radius: Radius for forecast frustum expansion.
        remove_small_geometry_threshold: Min area ratio to keep component.
        cluster_to_keep: Max number of largest clusters to retain.
        min_cluster_size: Min triangles per cluster to keep.
        
    Returns:
        Culled mesh.
    """
    vertices = mesh.vertices  # Shape (N, 3)
    faces = mesh.faces        # Shape (M, 3)
    N = vertices.shape[0]
    
    # Initialize masks
    valid = torch.zeros(N, dtype=torch.bool, device=vertices.device)
    valid_num = torch.zeros(N, dtype=torch.int, device=vertices.device)
    valid_forecast = torch.zeros(N, dtype=torch.bool, device=vertices.device)
    
    fx, fy, cx, cy = intrinsics['fx'], intrinsics['fy'], intrinsics['cx'], intrinsics['cy']
    H, W = intrinsics['H'], intrinsics['W']
    
    # Process each camera
    for i, (cam_pose, depth_map) in enumerate(zip(cameras, depth_maps)):
        # Convert to world-to-camera if needed
        if cam_pose.is_camera_to_world:
            w2c = cam_pose.inverse()
            # Negate y and z rows of rotation part if necessary for coordinate system
            # w2c[:3, 1:3] *= -1  # [G] Assuming standard OpenGL/COLMAP convention adjustment
        else:
            w2c = cam_pose
            
        # Prepare depth map for sampling
        # Reshape to (1, 1, H, W) and normalize UV grid
        depth_tensor = torch.from_numpy(depth_map).unsqueeze(0).unsqueeze(0).to(vertices.device)
        
        # Batch process vertices
        for start in range(0, N, points_batch_size):
            end = min(start + points_batch_size, N)
            v_batch = vertices[start:end]
            
            # Homogeneous coordinates
            v_h = torch.cat([v_batch, torch.ones(end - start, 1, device=vertices.device)], dim=1)
            
            # Project to camera space
            v_c = (w2c @ v_h.T).T  # Shape (batch, 3)
            z = v_c[:, 2] + 1e-8
            u = v_c[:, 0] / z
            v = v_c[:, 1] / z
            
            # Frustum checks
            in_frustum = (u >= 0) & (u <= W - 1) & (v >= 0) & (v <= H - 1) & (z > 0)
            
            forecast_frustum = torch.zeros_like(in_frustum)
            if forecast_radius > 0:
                r = forecast_radius
                forecast_frustum = (u >= -r) & (u <= W - 1 + r) & (v >= -r) & (v <= H - 1 + r) & (z > 0)
            
            # Sample depth
            # Normalize UV to [-1, 1] for grid_sample
            # grid_sample expects normalized coordinates in range [-1, 1]
            # u_norm = 2 * (u / (W - 1)) - 1
            # v_norm = 2 * (v / (H - 1)) - 1
            # vgrid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0)
            # depth_sample = F.grid_sample(depth_tensor, vgrid, padding_mode='border', align_corners=True).squeeze()
            
            # [G] Simplified sampling logic for pseudocode clarity
            # In practice, use F.grid_sample with proper normalization
            depth_sample = sample_depth(depth_tensor, u, v, H, W)
            
            # Front-face validation
            is_front_face = (depth_sample > 0.0) & (z < depth_sample + eps)
            
            # Update masks
            valid_mask = in_frustum & is_front_face
            valid_num[start:end] += valid_mask.int()
            valid[start:end] |= valid_mask
            
            forecast_mask = forecast_frustum & is_front_face
            valid_forecast[start:end] |= forecast_mask
            
    # Finalize valid mask
    valid = valid_num >= 20
    
    # Combine valid and forecast if needed
    # final_valid = valid | valid_forecast  # [G] Depends on specific usage
    
    # Cull faces
    # face_mask = valid[faces].all(dim=1)
    # mesh.update_faces(face_mask)
    # mesh.remove_unreferenced_vertices()
    
    # Optional: Cluster filtering
    # clusters = mesh.cluster_connected_triangles()
    # ... (cluster filtering logic)
    
    return mesh

def sample_depth(depth_tensor, u, v, H, W):
    """
    Sample depth at UV coordinates using bilinear interpolation.
    """
    # Normalize UV to [-1, 1]
    u_norm = 2.0 * (u / (W - 1)) - 1.0
    v_norm = 2.0 * (v / (H - 1)) - 1.0
    vgrid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0)
    
    depth_sample = F.grid_sample(depth_tensor, vgrid, padding_mode='border', align_corners=True).squeeze()
    return depth_sample
```

## Subtopics

### Mesh Post-Processing (Cluster Filtering)
A variant that filters the culled mesh by identifying connected triangle clusters and removing small ones. It sorts clusters by size, selects a threshold based on the N-th largest cluster (`cluster_to_keep`), and removes triangles below this threshold or below `min_cluster_size` [S1, unofficial]. Use this when the projection culling leaves behind small floating artifacts or noise that are not removed by area-based filtering alone.

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$, where $N$ is the number of vertices and $C$ is the number of cameras. Depth sampling is $O(1)$ per vertex per camera using bilinear interpolation.
- **Memory Complexity:** $O(N + C \cdot H \cdot W)$ for storing depth maps and intermediate masks.
- **Performance:** Depends heavily on the number of cameras and resolution. High-resolution depth maps and many cameras increase memory usage. Batch processing (`points_batch_size`) limits GPU memory usage [S1, unofficial].
- **Strengths:** Effective at removing unseen geometry and artifacts from unbounded reconstructions. Robust to noise if multiple views are used.
- **Limitations:** Requires accurate camera poses and depth maps. May remove valid geometry if cameras are sparse or depth maps are noisy. The threshold of 20 cameras is arbitrary and may need adjustment for different scenes [S1, unofficial].
- **When to use:** When extracting meshes from Gaussian splatting or other unbounded radiance fields where unseen geometry is common.
- **When not to use:** When camera coverage is sparse or depth maps are unreliable.

## Background
This method originates from the "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" paper by Huang et al. (SIGGRAPH 2024) [S1, unofficial]. It was developed to address the issue of unbounded Gaussian splatting reconstruction producing artifacts and unseen geometry. The method uses multi-view depth projection to classify points and cull mesh geometry, improving the geometric accuracy of the extracted meshes [S1, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
