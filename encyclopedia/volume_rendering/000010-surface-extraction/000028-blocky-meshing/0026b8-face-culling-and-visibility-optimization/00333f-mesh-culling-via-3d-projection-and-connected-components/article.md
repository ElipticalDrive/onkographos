# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method removes geometric artifacts and occluded regions from a 3D mesh by projecting its vertices into multiple camera views and comparing their depths against reference depth maps [S1, unofficial]. It classifies mesh faces as "seen" (visible in at least 20 views), "unseen," or "forecast" (distant but potentially valid) to filter out noise from unbounded reconstructions [S1, unofficial]. Unlike sibling methods that rely on voxel adjacency or backface normals, this approach uses multi-view consistency to determine visibility, making it suitable for cleaning meshes derived from Gaussian Splatting or similar radiance field representations [S1, unofficial].

## How it works
**Inputs:**
- A triangle mesh $M$ with vertices $V$ and faces $F$.
- A list of camera poses $C = \{c_2w_i\}$ and intrinsic parameters $K_i$ (focal lengths $f_x, f_y$, principal point $c_x, c_y$, resolution $H \times W$).
- Reference depth maps $D_i$ for each camera pose.
- Parameters: `points_batch_size` (default $5 \times 10^5$), `eps` (default $0.005$), `remove_small_geometry_threshold` (default $0.2$), `forecast_radius` (default $0$).

**Outputs:**
- A cleaned mesh $M'$ with unreferenced vertices removed.

**Data Structures:**
- `valid`: Boolean tensor of shape $(N_{vertices})$, initialized to False.
- `valid_num`: Integer tensor of shape $(N_{vertices})$, initialized to 0.
- `face_mask`: Boolean array of shape $(N_{faces})$.

**Steps:**

1. **Initialize Tracking Tensors:**
   Create `valid` and `valid_num` tensors on the target device (e.g., GPU) with size equal to the number of vertices in $M$. Initialize `valid_forecast` similarly if `forecast_radius > 0` [S1, unofficial].

2. **Iterate Over Cameras:**
   For each camera pose $c_2w$ and its corresponding depth map $D$:
   
   a. **Coordinate Transformation:**
      Adjust the camera-to-world matrix to match the expected coordinate system. Negate the y and z rows of the rotation part:
      $$ c_2w[:3, 1:3] \leftarrow -c_2w[:3, 1:3] $$
      Compute the world-to-camera matrix:
      $$ w2c = c_2w^{-1} $$
      [S1, unofficial]

   b. **Project Vertices to Camera Space:**
      For a batch of vertices $V_{batch}$:
      - Convert to homogeneous coordinates by appending 1s: $V_h = [V_{batch}; 1]$.
      - Transform to camera space: $P_{cam} = w2c \cdot V_h$.
      - Extract the first three rows to get $(x_c, y_c, z_c)$ [S1, unofficial].

   c. **Project to Image Plane:**
      Apply intrinsic matrix $K$:
      $$ u_{raw} = f_x \cdot \frac{x_c}{z_c} + c_x $$
      $$ v_{raw} = f_y \cdot \frac{y_c}{z_c} + c_y $$
      Extract depth $z = z_c + 10^{-8}$ to avoid division by zero.
      Compute normalized UV coordinates for grid sampling:
      $$ u = \frac{u_{raw}}{z}, \quad v = \frac{v_{raw}}{z} $$
      [S1, unofficial]

   d. **Frustum Culling:**
      Check if points are within the standard frustum:
      $$ \text{in\_frustum} = (u \ge 0) \land (u \le W-1) \land (v \ge 0) \land (v \le H-1) \land (z > 0) $$
      If `forecast_radius` $> 0$, also check an expanded frustum:
      $$ \text{forecast\_frustum} = (u \ge -r) \land (u \le W-1+r) \land (v \ge -r) \land (v \le H-1+r) \land (z > 0) $$
      [S1, unofficial]

   e. **Depth Sampling and Validation:**
      - Reshape reference depth $D$ to $(1, 1, H, W)$.
      - Normalize UV grid to $[-1, 1]$ for bilinear interpolation.
      - Sample depth at $(u, v)$ using `grid_sample` with border padding:
        $$ d_{sample} = \text{grid\_sample}(D, \text{vgrid}, \text{padding\_mode}='border', \text{align\_corners}=\text{True}) $$
      - Validate front-facing geometry:
        $$ \text{is\_front\_face} = (d_{sample} > 0.0) \land (z < d_{sample} + \text{eps}) $$
      [S1, unofficial]

   f. **Update Masks:**
      - For points in `in_frustum` and `is_front_face`, increment `valid_num` and set `valid` to True.
      - For points in `forecast_frustum` and `is_front_face`, set `valid_forecast` to True.
      [S1, unofficial]

3. **Finalize Vertex Visibility:**
   A vertex is considered "seen" if:
   $$ \text{final\_valid}_i = (\text{valid\_num}_i \ge 20) $$
   This requires the vertex to be visible in at least 20 distinct camera views [S1, unofficial].

4. **Create Face Mask:**
   A face is kept if all its vertices are valid:
   $$ \text{face\_mask}_j = \bigwedge_{v \in \text{face}_j} \text{final\_valid}_v $$
   [S1, unofficial]

5. **Optional Forecast Culling:**
   If `forecast_radius > 0`:
   - Create a forecast mesh from faces where vertices are in `valid_forecast`.
   - Compute the Oriented Bounding Box (OBB) of the cull mesh vertices.
   - Keep only forecast faces that lie within this OBB [S1, unofficial].

6. **Update Mesh:**
   - Apply `face_mask` to the mesh.
   - Remove unreferenced vertices [S1, unofficial].

7. **Post-Processing (Cluster Filtering):**
   - Filter mesh components by area.
   - Keep components where $\text{area} > \text{remove\_small\_geometry\_threshold} \times \text{global\_area}$.
   - Alternatively, keep only the largest connected component if `get_largest_components` is True [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def cull_mesh_by_projection(mesh, cameras, depth_maps, params):
    """
    mesh: trimesh.Trimesh object with vertices and faces
    cameras: list of dicts with keys 'c2w', 'fx', 'fy', 'cx', 'cy', 'H', 'W'
    depth_maps: list of torch tensors (H, W)
    params: dict with 'eps', 'batch_size', 'forecast_radius', 'remove_small_geometry_threshold'
    """
    vertices = torch.tensor(mesh.vertices, dtype=torch.float32, device='cuda')
    faces = torch.tensor(mesh.faces, dtype=torch.long, device='cuda')
    n_verts = vertices.shape[0]
    
    # Initialize masks
    valid = torch.zeros(n_verts, dtype=torch.bool, device='cuda')
    valid_num = torch.zeros(n_verts, dtype=torch.int32, device='cuda')
    
    if params['forecast_radius'] > 0:
        valid_forecast = torch.zeros(n_verts, dtype=torch.bool, device='cuda')
    
    # Process in batches to limit GPU memory
    batch_size = params.get('points_batch_size', 500000)
    n_batches = (n_verts + batch_size - 1) // batch_size
    
    for cam_idx, (cam, depth_map) in enumerate(zip(cameras, depth_maps)):
        # 1. Coordinate transformation
        c2w = torch.tensor(cam['c2w'], dtype=torch.float32, device='cuda')
        c2w[:3, 1:3] *= -1  # Negate y and z rows
        w2c = torch.inverse(c2w)
        
        # 2. Project vertices
        for i in range(n_batches):
            start = i * batch_size
            end = min(start + batch_size, n_verts)
            v_batch = vertices[start:end]
            
            # Homogeneous coordinates
            v_h = torch.cat([v_batch, torch.ones(end-start, 1, device='cuda')], dim=1)
            # World to camera
            v_cam = (w2c @ v_h.T).T
            x_c, y_c, z_c = v_cam[:, 0], v_cam[:, 1], v_cam[:, 2]
            
            # 3. Project to image plane
            eps_z = 1e-8
            u_raw = cam['fx'] * (x_c / (z_c + eps_z)) + cam['cx']
            v_raw = cam['fy'] * (y_c / (z_c + eps_z)) + cam['cy']
            
            # 4. Frustum check
            in_frustum = (u_raw >= 0) & (u_raw < cam['W']-1) & \
                         (v_raw >= 0) & (v_raw < cam['H']-1) & \
                         (z_c > 0)
            
            # 5. Depth sampling
            # Normalize UV for grid_sample
            u_norm = 2.0 * (u_raw / (z_c + eps_z)) / cam['W'] - 1.0
            v_norm = 2.0 * (v_raw / (z_c + eps_z)) / cam['H'] - 1.0
            vgrid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, N, 2)
            
            # Reshape depth map for sampling
            depth_4d = depth_map.unsqueeze(0).unsqueeze(0) # (1, 1, H, W)
            
            # Sample depth
            depth_sample = F.grid_sample(depth_4d, vgrid, padding_mode='border', align_corners=True)
            depth_sample = depth_sample.squeeze() # (N,)
            
            # Front-face validation
            is_front = (depth_sample > 0.0) & (z_c < depth_sample + params['eps'])
            
            # Update masks
            valid_mask = in_frustum & is_front
            valid_num[start:end] += valid_mask.int()
            valid[start:end] = valid[start:end] | valid_mask
            
            if params['forecast_radius'] > 0:
                r = params['forecast_radius']
                forecast_frustum = (u_raw >= -r) & (u_raw < cam['W']-1+r) & \
                                   (v_raw >= -r) & (v_raw < cam['H']-1+r) & \
                                   (z_c > 0)
                valid_forecast[start:end] = valid_forecast[start:end] | (forecast_frustum & is_front)
    
    # Finalize vertex visibility
    final_valid = valid_num >= 20
    
    # Create face mask
    # Check if all vertices of each face are valid
    face_verts_valid = final_valid[faces] # (N_faces, 3)
    face_mask = face_verts_valid.all(dim=1)
    
    # Update mesh
    mesh.update_faces(face_mask)
    mesh.remove_unreferenced_vertices()
    
    # Optional: Cluster filtering
    # This step is typically handled by a post-processing module like Mesh Post-Processing (Cluster Filtering)
    # Here we simulate the logic:
    if params.get('get_largest_components', False):
        # Find connected components and keep largest
        # Implementation depends on mesh library (e.g., trimesh.graph.connected_components)
        pass
    else:
        # Filter by area
        # Calculate area of each face
        areas = mesh.area_faces
        total_area = mesh.area
        threshold = params['remove_small_geometry_threshold'] * total_area
        # Keep faces with area > threshold? Or keep components?
        # The source says "keep components where area > ...", implying component-level filtering.
        # For simplicity, we assume the post-processing step handles this.
        pass
        
    return mesh
```

## Subtopics

### [Mesh Post-Processing (Cluster Filtering)](7002-mesh-post-processing-cluster-filtering.md)
Filters floating artifacts and disconnected components by clustering triangles and retaining only the largest connected clusters. It serves as a cleanup step following this method, specifically targeting small, noisy geometry that survives projection-based culling. Unlike this method which removes occluded or distant geometry, it removes topologically isolated fragments based on triangle count.

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$, where $N$ is the number of vertices and $C$ is the number of cameras. The depth sampling is $O(1)$ per vertex per camera using bilinear interpolation.
- **Memory Complexity:** $O(N + H \cdot W \cdot C)$ for storing depth maps and intermediate tensors. Batching limits vertex memory to `points_batch_size`.
- **Performance:** Efficient on GPU due to parallel projection and grid sampling. The requirement of 20 visible views makes it robust against noise but sensitive to sparse camera coverage.
- **Strengths:** Effective at removing background noise and artifacts in unbounded reconstructions. Does not rely on voxel grid structure.
- **Limitations:** Requires accurate camera poses and depth maps. Computationally expensive for very high-resolution meshes or large numbers of cameras. The "20 views" threshold may be too strict for scenes with limited camera overlap.
- **When to use:** Cleaning meshes from Gaussian Splatting or NeRF-based reconstructions where multi-view consistency is available.
- **When not to use:** Real-time applications where latency is critical, or scenes with very few camera views.

## Background
This method originates from the implementation of "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" by Huang et al. (SIGGRAPH 2024) [S1, unofficial]. It was developed to address the issue of artifacts in unbounded mesh extraction from 2D Gaussian Splatting, where the reconstruction may include noise from the background or unobserved regions. The method leverages the multi-view nature of the input data to filter out inconsistent geometry.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
