# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method removes geometric artifacts and occluded geometry from a 3D mesh by projecting its vertices into multiple camera views and comparing their depths against reference depth maps [S1, unofficial]. It classifies mesh faces as "seen" (visible in sufficient views), "unseen" (occluded or background), or "forecast" (distant potential geometry) to cull invalid triangles [S1, unofficial]. Unlike sibling methods that rely on voxel occupancy or local face coverage, this approach uses global visibility constraints derived from the capture setup, making it particularly effective for cleaning meshes extracted from unbounded radiance fields like Gaussian Splatting [S1, unofficial].

## How it works
**Inputs:**
- A triangle mesh $M = (V, F)$ where $V$ is the set of vertices and $F$ is the set of faces.
- A list of camera poses $C = \{c_2w_i\}_{i=1}^N$ (world-to-camera or camera-to-world matrices).
- Camera intrinsics $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$.
- Image dimensions $H \times W$.
- Reference depth maps $D_i$ for each camera pose $i$.
- Parameters: `points_batch_size` (default $5 \times 10^5$), `eps` (default $0.005$), `remove_small_geometry_threshold` (default $0.2$), `forecast_radius` (default $0$).

**Outputs:**
- A culled mesh $M'$ with reduced face count and removed unreferenced vertices.

**Data Structures:**
- `valid`: Boolean tensor of shape $(N_{vertices})$, initialized to zero. Marks vertices seen in $\ge 20$ cameras.
- `valid_num`: Integer tensor of shape $(N_{vertices})$, counts visible cameras per vertex.
- `face_mask`: Boolean array of shape $(N_{faces})$, true if all three vertices of a face are valid.

**Steps:**

1. **Initialize Masks:**
   Create `valid`, `valid_num`, and `valid_forecast` tensors of size $N_{vertices}$ on the target device, initialized to zero.

2. **Iterate Camera Poses:**
   For each camera pose $c_2w_i$ in $C$:
   
   a. **Coordinate Transformation:**
   Adjust the camera-to-world matrix for the specific coordinate system convention. Negate the y and z rows of the rotation part:
   $$ c_2w' = c_2w $$
   $$ c_2w'[:3, 1] = -c_2w[:3, 1] $$
   $$ c_2w'[:3, 2] = -c_2w[:3, 2] $$
   Compute the world-to-camera matrix:
   $$ w2c = (c_2w')^{-1} $$

   b. **Project Vertices:**
   For a batch of vertices $V_{batch}$:
   - Convert to homogeneous coordinates by appending 1s: $V_h = [V_{batch}, \mathbf{1}]^T$.
   - Transform to camera space: $P_{cam} = w2c \cdot V_h$.
   - Extract the first three rows to get $(x, y, z)$.
   - Apply intrinsic matrix $K$ to get UV coordinates:
     $$ u_{raw} = f_x \cdot \frac{x}{z} + c_x $$
     $$ v_{raw} = f_y \cdot \frac{y}{z} + c_y $$
     $$ z_{depth} = z + 10^{-8} $$
   - Normalize UV for grid sampling:
     $$ u = \frac{u_{raw}}{z_{depth}} $$
     $$ v = \frac{v_{raw}}{z_{depth}} $$
     *(Note: The source code normalizes UV to [-1, 1] for `grid_sample`, but the logical check uses pixel coordinates. The implementation below follows the source's logic of checking frustum bounds on pixel coordinates before sampling.)*

   c. **Frustum and Visibility Check:**
   - Check if point is in standard frustum:
     $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z > 0) $$
   - Check if point is in forecast frustum (if `forecast_radius` > 0):
     $$ \text{forecast\_frustum} = (-r \le u < W+r) \land (-r \le v < H+r) \land (z > 0) $$
   - Reshape reference depth $D_i$ to $(1, 1, H, W)$.
   - Normalize UV grid to $[-1, 1]$ for bilinear interpolation.
   - Sample depth at $(u, v)$ using `grid_sample` with `padding_mode='border'` and `align_corners=True`:
     $$ d_{sample} = \text{grid\_sample}(D_i, \text{vgrid}) $$
   - Validate front-facing geometry:
     $$ \text{is\_front} = (d_{sample} > 0) \land (z < d_{sample} + \text{eps}) $$
   - Update masks:
     $$ \text{valid\_num}[\text{in\_frustum} \land \text{is\_front}] += 1 $$
     $$ \text{valid\_forecast}[\text{forecast\_frustum} \land \text{is\_front}] = 1 $$

3. **Finalize Vertex Masks:**
   - A vertex is "seen" if it is visible in at least 20 cameras:
     $$ \text{valid} = (\text{valid\_num} \ge 20) $$

4. **Generate Face Mask:**
   - For each face $f \in F$ with vertices $v_1, v_2, v_3$:
     $$ \text{face\_mask}[f] = \text{valid}[v_1] \land \text{valid}[v_2] \land \text{valid}[v_3] $$

5. **Update Mesh:**
   - Apply `face_mask` to keep only valid faces.
   - Remove unreferenced vertices.

6. **Optional Connected Component Filtering (Forecast):**
   If `forecast_radius` > 0:
   - Create a "forecast mesh" from faces where vertices are marked in `valid_forecast`.
   - Compute the Oriented Bounding Box (OBB) of the cull mesh vertices.
   - Keep only forecast faces that lie within this OBB.
   - Merge these faces back into the mesh.

7. **Post-Processing (Cluster Filtering):**
   - Filter mesh components by area.
   - Keep components where $\text{Area}(C) > \text{remove\_small\_geometry\_threshold} \times \text{GlobalArea}$.
   - Alternatively, keep only the largest connected component if `get_largest_components` is True.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def cull_mesh_by_projection(mesh, cameras, intrinsics, depth_maps, 
                            points_batch_size=5e5, eps=0.005, 
                            forecast_radius=0, remove_small_threshold=0.2):
    """
    Cull mesh geometry based on multi-view depth projection.
    
    Args:
        mesh: Triangle mesh object with .vertices and .faces attributes.
        cameras: List of camera-to-world matrices (4x4).
        intrinsics: Tuple (fx, fy, cx, cy, H, W).
        depth_maps: List of depth tensors (H, W) for each camera.
        points_batch_size: Max vertices processed per batch.
        eps: Depth comparison tolerance.
        forecast_radius: Radius for extended frustum check.
        remove_small_threshold: Minimum area ratio for component retention.
        
    Returns:
        Culled mesh.
    """
    V = mesh.vertices  # Shape: (N, 3)
    F_faces = mesh.faces  # Shape: (M, 3)
    N = V.shape[0]
    
    fx, fy, cx, cy, H, W = intrinsics
    
    # Initialize masks
    valid_num = torch.zeros(N, device=V.device, dtype=torch.int)
    valid_forecast = torch.zeros(N, device=V.device, dtype=torch.int)
    
    # Process vertices in batches
    for start in range(0, N, points_batch_size):
        end = min(start + points_batch_size, N)
        V_batch = V[start:end]
        
        for i, c2w in enumerate(cameras):
            depth_map = depth_maps[i]
            
            # Coordinate transformation
            c2w_adj = c2w.clone()
            c2w_adj[:3, 1] *= -1
            c2w_adj[:3, 2] *= -1
            w2c = torch.inverse(c2w_adj)
            
            # Project to camera space
            V_h = torch.cat([V_batch, torch.ones(V_batch.shape[0], 1, device=V.device)], dim=1)
            P_cam = (w2c @ V_h.T).T  # Shape: (batch, 4)
            x, y, z = P_cam[:, 0], P_cam[:, 1], P_cam[:, 2]
            
            # Intrinsic projection
            u_raw = fx * (x / (z + 1e-8)) + cx
            v_raw = fy * (y / (z + 1e-8)) + cy
            z_depth = z + 1e-8
            
            # Frustum check
            in_frustum = (u_raw >= 0) & (u_raw < W) & (v_raw >= 0) & (v_raw < H) & (z > 0)
            
            # Forecast frustum check
            if forecast_radius > 0:
                forecast_frustum = (u_raw >= -forecast_radius) & (u_raw < W + forecast_radius) & \
                                   (v_raw >= -forecast_radius) & (v_raw < H + forecast_radius) & (z > 0)
            else:
                forecast_frustum = torch.zeros_like(in_frustum)
            
            # Sample depth
            # Normalize UV to [-1, 1] for grid_sample
            u_norm = 2.0 * (u_raw / (z_depth)) / W - 1.0
            v_norm = 2.0 * (v_raw / (z_depth)) / H - 1.0
            vgrid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, batch, 2)
            
            # Reshape depth to (1, 1, H, W)
            depth_reshaped = depth_map.unsqueeze(0).unsqueeze(0)
            
            # Bilinear interpolation
            depth_sample = F.grid_sample(depth_reshaped, vgrid, align_corners=True, padding_mode='border')
            depth_sample = depth_sample.squeeze().squeeze() # (batch,)
            
            # Front-face validation
            is_front = (depth_sample > 0.0) & (z < depth_sample + eps)
            
            # Update counts
            valid_mask = in_frustum & is_front
            valid_num[start:end][valid_mask] += 1
            
            if forecast_radius > 0:
                forecast_mask = forecast_frustum & is_front
                valid_forecast[start:end][forecast_mask] = 1

    # Finalize seen mask: requires visibility in at least 20 cameras
    valid = valid_num >= 20
    
    # Create face mask: all vertices of a face must be valid
    face_mask = valid[F_faces].all(dim=1)
    
    # Update mesh
    mesh.update_faces(face_mask)
    mesh.remove_unreferenced_vertices()
    
    # Optional: Connected component filtering for forecast
    if forecast_radius > 0:
        forecast_vertices = valid_forecast.bool()
        forecast_faces = F_faces[forecast_vertices[F_faces].all(dim=1)]
        if len(forecast_faces) > 0:
            # Compute OBB of current mesh vertices
            obb = compute_obb(mesh.vertices)
            # Keep forecast faces within OBB
            # (Implementation detail: check if face centroid is inside OBB)
            # Merge back into mesh
            pass
            
    # Post-processing: Cluster filtering
    mesh = filter_components_by_area(mesh, threshold=remove_small_threshold)
    
    return mesh

def compute_obb(vertices):
    """Compute Oriented Bounding Box. Not given in sources. [G]"""
    # Placeholder for OBB computation logic
    pass

def filter_components_by_area(mesh, threshold):
    """Filter mesh components by area. Not given in sources. [G]"""
    # Placeholder for connected component analysis
    pass
```

**Edge Cases and Pitfalls:**
- **Precision:** The depth comparison `z < depth_sample + eps` is sensitive to `eps`. A value of $0.005$ is used in the source [S1, unofficial].
- **Boundary Conditions:** The frustum check uses strict inequalities for pixel bounds ($0 \le u < W$). Points exactly on the boundary may be excluded.
- **Coordinate Systems:** The source explicitly negates y and z rows of the rotation matrix. Failure to do so will result in incorrect projections [S1, unofficial].
- **Batch Size:** `points_batch_size` limits GPU memory usage. If set too high, OOM errors may occur; if too low, performance degrades due to overhead [S1, unofficial].

## Subtopics

### [Mesh Post-Processing (Cluster Filtering)](7002-mesh-post-processing-cluster-filtering.md)
Filters floating artifacts and disconnected components by clustering triangles and retaining the largest connected clusters. Used after this method to remove small, noisy geometry that survives projection-based culling.

## Operating characteristics
- **Time Complexity:** $O(N \cdot V \cdot C)$, where $N$ is the number of vertices, $V$ is the number of vertices per batch, and $C$ is the number of cameras. The depth sampling is $O(1)$ per vertex per camera using bilinear interpolation.
- **Memory Complexity:** $O(N + C \cdot H \cdot W)$ for storing vertex masks and depth maps.
- **Performance:** Effective for removing background noise and unbounded artifacts in Gaussian Splatting reconstructions. Performance scales linearly with the number of cameras and vertices.
- **Strengths:** Globally consistent visibility; removes geometry that is not supported by multiple views.
- **Limitations:** Requires accurate camera poses and depth maps; sensitive to `eps` and batch size; may remove valid geometry if views are sparse.

## Background
This method originates from the "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" paper by Huang et al. (SIGGRAPH 2024) [S1, unofficial]. It was developed to address the challenge of extracting clean, bounded meshes from unbounded 3D Gaussian Splatting representations, which often contain artifacts and background noise. The method leverages the multi-view nature of the capture process to validate geometry.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
