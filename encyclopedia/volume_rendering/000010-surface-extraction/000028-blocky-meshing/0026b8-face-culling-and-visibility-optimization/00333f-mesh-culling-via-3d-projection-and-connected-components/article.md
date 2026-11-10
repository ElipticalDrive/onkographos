# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method removes geometric artifacts and occluded geometry from a 3D mesh by projecting its vertices into multiple camera views and comparing their depths against reference depth maps [S1, unofficial]. It classifies mesh faces as "seen" (visible in sufficient views), "unseen" (occluded or background noise), or "forecast" (distant/unbounded) and culls those that do not meet visibility criteria [S1, unofficial]. Unlike sibling methods that rely on voxel occupancy or face coverage masks, this approach uses multi-view consistency to filter out reconstruction noise, particularly in unbounded Gaussian splatting scenarios [S1, unofficial].

## How it works
**Inputs:**
- A triangle mesh $M$ with vertices $V$ and faces $F$.
- A set of $N$ camera poses (extrinsics $C2W_i$) and intrinsic parameters ($H, W, f_x, f_y, c_x, c_y$).
- Reference depth maps $D_i$ for each camera pose, typically extracted from the mesh or a prior reconstruction step.
- Parameters: `points_batch_size` (default $5 \times 10^5$), `eps` (default $0.005$), `remove_small_geometry_threshold` (default $0.2$), `forecast_radius` (default $0$, optional).

**Outputs:**
- A culled mesh $M'$ with unreferenced vertices removed.

**Data Structures:**
- `valid`: Boolean tensor of shape $(N_{vertices})$, initialized to zero. Marks vertices seen in $\ge 20$ cameras.
- `valid_num`: Integer tensor of shape $(N_{vertices})$, initialized to zero. Counts how many cameras see each vertex.
- `face_mask`: Boolean array of shape $(N_{faces})$. True if all three vertices of a face are valid.

**Steps:**

1. **Initialize Masks:**
   Create `valid`, `valid_num`, and `valid_forecast` tensors on the target device, initialized to zeros with length equal to the number of vertices $N_v$.

2. **Iterate Camera Poses:**
   For each camera pose $i$ with extrinsic matrix $C2W_i$ and depth map $D_i$:
   
   a. **Coordinate Transformation:**
      Adjust the camera coordinate system. Negate the y and z rows of the rotation part of $C2W_i$:
      $$ C2W_i[:3, 1] *= -1 $$
      $$ C2W_i[:3, 2] *= -1 $$
      Compute the world-to-camera matrix:
      $$ W2C_i = C2W_i^{-1} $$

   b. **Project Vertices:**
      For a batch of vertices $V_{batch}$ (size up to `points_batch_size`):
      - Convert to homogeneous coordinates by appending 1s: $V_h = [V_{batch}, 1]$.
      - Transform to camera space: $V_c = W2C_i \cdot V_h^T$. Extract the first three rows to get $(x_c, y_c, z_c)$.
      - Apply intrinsic matrix $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$:
        $$ uv = K \cdot \begin{bmatrix} x_c \\ y_c \\ z_c \end{bmatrix} $$
      - Extract depth $z = uv[:, 2]$ and compute normalized UV coordinates:
        $$ u = uv[:, 0] / z $$
        $$ v = uv[:, 1] / z $$
      - Add small epsilon to depth to avoid division by zero: $z \leftarrow z + 10^{-8}$.

   c. **Frustum Culling:**
      Determine if points are within the standard frustum:
      $$ \text{in\_frustum} = (u \ge 0) \land (u \le W-1) \land (v \ge 0) \land (v \le H-1) \land (z > 0) $$
      If `forecast_radius` $> 0$, also compute forecast frustum bounds:
      $$ \text{forecast\_frustum} = (u \ge -r) \land (u \le W-1+r) \land (v \ge -r) \land (v \le H-1+r) \land (z > 0) $$

   d. **Depth Sampling:**
      - Reshape reference depth $D_i$ to $(1, 1, H, W)$.
      - Normalize UV grid to $[-1, 1]$ for bilinear interpolation.
      - Sample depth at $(u, v)$ using `grid_sample` with `padding_mode='border'` and `align_corners=True`:
        $$ d_{sample} = \text{grid\_sample}(D_i, \text{vgrid}) $$

   e. **Front-Face Validation:**
      Check if the vertex is in front of the sampled depth:
      $$ \text{is\_front\_face} = (d_{sample} > 0.0) \land (z < d_{sample} + \text{eps}) $$

   f. **Update Masks:**
      For vertices satisfying `in_frustum` and `is_front_face`:
      - Increment `valid_num` for those vertices.
      - If `forecast_radius` $> 0$ and `forecast_frustum` is true, mark `valid_forecast`.

3. **Finalize Visibility:**
   A vertex is considered "seen" if it is visible in at least 20 cameras:
   $$ \text{valid} = (\text{valid\_num} \ge 20) $$

4. **Face Masking:**
   Create a face mask by checking if all vertices of each face are valid:
   $$ \text{face\_mask} = \text{valid}[M.\text{faces}].\text{all}(\text{axis}=1) $$
   Apply the mask to the mesh and remove unreferenced vertices.

5. **Connected Components Filtering (Optional):**
   If `forecast_radius` $> 0$:
   - Create a forecast mask from `valid_forecast`.
   - Compute the Oriented Bounding Box (OBB) of the cull mesh vertices.
   - Keep only forecast faces within this OBB.
   - Filter mesh components by area: Keep components where area $> \text{remove\_small\_geometry\_threshold} \times \text{global\_area}$, or keep only the largest component if `get_largest_components` is True.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def cull_mesh_by_projection(mesh, cameras, depth_maps, eps=0.005, min_views=20, forecast_radius=0, remove_threshold=0.2):
    """
    Cull mesh geometry based on multi-view depth projection.
    
    Args:
        mesh: Triangle mesh object with vertices (N, 3) and faces (M, 3).
        cameras: List of dicts with keys 'c2w' (4,4), 'H', 'W', 'fx', 'fy', 'cx', 'cy'.
        depth_maps: List of depth tensors (H, W) corresponding to cameras.
        eps: Depth comparison tolerance.
        min_views: Minimum number of cameras a vertex must be seen in.
        forecast_radius: Radius for extended frustum check.
        remove_threshold: Minimum area ratio for component retention.
        
    Returns:
        Culled mesh.
    """
    device = mesh.vertices.device
    n_verts = mesh.vertices.shape[0]
    
    # Initialize masks
    valid_num = torch.zeros(n_verts, dtype=torch.int32, device=device)
    valid_forecast = torch.zeros(n_verts, dtype=torch.bool, device=device)
    
    batch_size = 500000  # points_batch_size
    
    for i, cam in enumerate(cameras):
        c2w = cam['c2w'].clone()
        # Negate y and z rows for coordinate system adjustment
        c2w[:3, 1] *= -1
        c2w[:3, 2] *= -1
        
        w2c = torch.inverse(c2w)
        
        H, W = cam['H'], cam['W']
        fx, fy = cam['fx'], cam['fy']
        cx, cy = cam['cx'], cam['cy']
        
        depth_map = depth_maps[i].to(device)
        
        # Process vertices in batches
        for start in range(0, n_verts, batch_size):
            end = min(start + batch_size, n_verts)
            verts = mesh.vertices[start:end]
            
            # Homogeneous coordinates
            verts_h = torch.cat([verts, torch.ones(verts.shape[0], 1, device=device)], dim=1)
            
            # World to camera
            verts_c = (w2c @ verts_h.T).T[:, :3]
            
            # Project to UV
            u = (verts_c[:, 0] * fx + cx) / (verts_c[:, 2] + 1e-8)
            v = (verts_c[:, 1] * fy + cy) / (verts_c[:, 2] + 1e-8)
            z = verts_c[:, 2]
            
            # Frustum check
            in_frustum = (u >= 0) & (u <= W - 1) & (v >= 0) & (v <= H - 1) & (z > 0)
            
            forecast_frustum = torch.zeros_like(in_frustum)
            if forecast_radius > 0:
                r = forecast_radius
                forecast_frustum = (u >= -r) & (u <= W - 1 + r) & (v >= -r) & (v <= H - 1 + r) & (z > 0)
            
            # Sample depth
            # Normalize UV to [-1, 1]
            u_norm = 2.0 * u / (W - 1) - 1.0
            v_norm = 2.0 * v / (H - 1) - 1.0
            vgrid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0)
            
            # Grid sample expects (N, H, W, 2) or similar, adjust for batch dim
            # depth_map shape: (H, W). Reshape to (1, 1, H, W)
            depth_reshaped = depth_map.unsqueeze(0).unsqueeze(0)
            depth_sample = F.grid_sample(depth_reshaped, vgrid, padding_mode='border', align_corners=True)
            depth_sample = depth_sample.squeeze()
            
            # Front face validation
            is_front = (depth_sample > 0.0) & (z < depth_sample + eps)
            
            # Update valid_num
            valid_mask = in_frustum & is_front
            valid_num[start:end] += valid_mask.int()
            
            # Update forecast
            if forecast_radius > 0:
                valid_forecast[start:end] |= (forecast_frustum & is_front)
                
    # Finalize seen mask
    valid = valid_num >= min_views
    
    # Face mask: all vertices of a face must be valid
    face_mask = valid[mesh.faces].all(dim=1)
    
    # Apply mask
    new_faces = mesh.faces[face_mask]
    new_verts = mesh.vertices[valid]
    
    # Re-index vertices
    # Create mapping from old valid indices to new indices
    valid_indices = torch.where(valid)[0]
    index_map = torch.full((n_verts,), -1, dtype=torch.long, device=device)
    index_map[valid_indices] = torch.arange(len(valid_indices), device=device)
    
    new_faces = index_map[new_faces]
    
    # Remove unreferenced vertices (already handled by new_verts)
    # Note: In a real implementation, you might want to run connected components
    # on the new mesh to remove small floating artifacts.
    
    return Mesh(new_verts, new_faces)
```

## Subtopics

### [Mesh Post-Processing (Cluster Filtering)](7002-mesh-post-processing-cluster-filtering.md)
Filters out floating artifacts and disconnected components by clustering triangles and retaining only the largest connected clusters. It serves as a cleanup step following this method, specifically targeting small, noisy geometry that survives projection-based culling. Unlike this method which removes occluded or distant geometry, this method removes topologically isolated fragments based on triangle count.

## Operating characteristics
- **Time Complexity:** $O(N_v \cdot N_c)$, where $N_v$ is the number of vertices and $N_c$ is the number of cameras. The depth sampling step is $O(1)$ per vertex using bilinear interpolation.
- **Memory Complexity:** $O(N_v + N_c \cdot H \cdot W)$ for storing depth maps and masks.
- **Performance:** The method is efficient for scenes with a moderate number of cameras. The batch processing of vertices (`points_batch_size`) limits GPU memory usage.
- **Strengths:** Effective at removing background noise and unbounded artifacts in Gaussian splatting reconstructions. Robust to occlusions by requiring multi-view consistency.
- **Limitations:** Requires pre-computed depth maps or a rendering step to generate them. Performance degrades with a very large number of cameras or high-resolution depth maps. The threshold `min_views` (default 20) may need tuning for scenes with sparse camera coverage.
- **When to use:** When extracting meshes from unbounded Gaussian splatting or other radiance field representations where background noise is prevalent.
- **When not to use:** For bounded scenes with dense camera coverage where simple voxel-based culling might be faster.

## Background
This method was introduced in the context of "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" (SIGGRAPH 2024) by Huang et al. [S1, unofficial]. It addresses the challenge of extracting clean, watertight meshes from 2D Gaussian splats, which often contain artifacts and unbounded geometry. The approach leverages multi-view depth consistency to distinguish between valid surface geometry and noise.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
