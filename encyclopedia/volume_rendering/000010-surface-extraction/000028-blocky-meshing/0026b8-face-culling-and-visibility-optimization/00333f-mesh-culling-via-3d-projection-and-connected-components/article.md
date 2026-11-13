# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method removes geometric artifacts and occluded geometry from unbounded meshes (typically derived from Gaussian Splatting) by projecting vertices into multiple camera views and validating visibility against reference depth maps. It distinguishes itself from topological culling by using view-dependent depth consistency rather than voxel neighbor occupancy, and from simple back-face culling by requiring multi-view agreement. It serves as a robust post-processing step for mesh extraction from radiance fields, specifically targeting the "halo" artifacts common in unbounded reconstructions.

## How it works
The method operates in two main phases: **Visibility Masking** via multi-view projection and **Connected Component Filtering**. It classifies mesh faces as "seen" (visible in sufficient cameras), "unseen" (occluded or background), or "forecast" (potentially visible within a radius).

### Inputs and Outputs
*   **Input**: A triangle mesh (vertices $V$, faces $F$), a list of camera poses ($C_{2W}$), camera intrinsics ($K$), and reference depth maps ($D_{ref}$) for each camera.
*   **Output**: A filtered mesh containing only faces classified as "seen" or valid "forecast" components, with unreferenced vertices removed.

### Data Structures
*   **Mesh**: Standard triangle mesh representation.
    *   `vertices`: Tensor of shape $(N_v, 3)$, coordinates in world space.
    *   `faces`: Tensor of shape $(N_f, 3)$, indices into `vertices`.
*   **Camera Data**:
    *   `c2w_list`: List of $4 \times 4$ world-to-camera transformation matrices (or camera-to-world, see step 2).
    *   `depth_list`: List of depth maps, shape $(H, W)$, representing the distance from the camera center to the surface.
    *   `intrinsics`: $3 \times 3$ matrix $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$.
*   **Masks**:
    *   `valid`: Boolean tensor of shape $(N_v)$, true if vertex is visible in $\ge 20$ cameras.
    *   `face_mask`: Boolean tensor of shape $(N_f)$, true if all 3 vertices of the face are in `valid`.

### Parameters
*   `points_batch_size`: Maximum number of vertices processed per batch to limit GPU memory. Default: $5 \times 10^5$ [S1, unofficial].
*   `eps`: Depth comparison tolerance for front-face validation. Default: $0.005$ [S1, unofficial].
*   `remove_small_geometry_threshold`: Minimum area ratio to keep a mesh component. Default: $0.2$ [S1, unofficial].
*   `forecast_radius`: Radius for expanding the frustum for "forecast" visibility. Default: Not explicitly given, but used in logic [S1, unofficial].
*   `far`: Far plane distance for depth extraction. Default: $20.0$ [S1, unofficial].

### Algorithm Steps

1.  **Initialize Masks**: Create boolean tensors `valid`, `valid_forecast` of size $N_v$ initialized to false, and `valid_num` (counter) of size $N_v$ initialized to 0.

2.  **Coordinate System Adjustment**: For each camera pose $C_{2W}$:
    *   The source code negates the y and z rows of the rotation matrix to convert coordinate systems (likely from OpenGL/COLMAP convention to the internal representation).
    *   Compute the World-to-Camera matrix $W_{2C} = C_{2W}^{-1}$.

3.  **Batch Processing Vertices**: Iterate over vertices in batches of `points_batch_size`:
    *   Let $P$ be the batch of vertices, shape $(B, 3)$.
    *   **Homogeneous Projection**: Append ones to $P$ to get $P_h$ of shape $(B, 4)$. Compute camera-space coordinates: $P_c = P_h \cdot W_{2C}^T$. Extract the first three rows ($x_c, y_c, z_c$).
    *   **Intrinsic Projection**: Compute normalized device coordinates (NDC) or UV:
        $$ u = \frac{f_x \cdot x_c + c_x \cdot z_c}{z_c} $$
        $$ v = \frac{f_y \cdot y_c + c_y \cdot z_c}{z_c} $$
        $$ z = z_c + 10^{-8} \quad \text{(epsilon for stability)} $$
    *   **Frustum Culling**:
        *   Define image dimensions $H, W$.
        *   Check if point is in standard frustum:
            $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z > 0) $$
        *   Check if point is in forecast frustum (expanded by `forecast_radius` $r$):
            $$ \text{forecast\_frustum} = (-r \le u < W+r) \land (-r \le v < H+r) \land (z > 0) $$

4.  **Depth Validation**:
    *   For points in `in_frustum`, sample the reference depth map $D_{ref}$ at $(u, v)$.
    *   Normalize UV coordinates to $[-1, 1]$ for bilinear interpolation:
        $$ u_{norm} = \frac{2u}{W} - 1, \quad v_{norm} = \frac{2v}{H} - 1 $$
    *   Sample depth: $d_{sample} = \text{GridSample}(D_{ref}, u_{norm}, v_{norm})$.
    *   Validate front-facing geometry:
        $$ \text{is\_front} = (d_{sample} > 0) \land (z < d_{sample} + \epsilon) $$
    *   Update masks:
        *   If `is_front` is true, increment `valid_num` and set `valid` to true for that vertex.
        *   If `forecast_frustum` is true and `is_front` is true, set `valid_forecast` to true.

5.  **Finalize Vertex Masks**:
    *   A vertex is considered "seen" if `valid_num` $\ge 20$ (requires visibility in at least 20 cameras) [S1, unofficial].

6.  **Face Masking**:
    *   Create `face_mask` for all faces:
        $$ \text{face\_mask}[i] = \text{valid}[F[i, 0]] \land \text{valid}[F[i, 1]] \land \text{valid}[F[i, 2]] $$
    *   Apply mask to mesh: keep only faces where `face_mask` is true.
    *   Remove unreferenced vertices.

7.  **Connected Component Filtering (Optional)**:
    *   If `forecast_radius > 0`:
        *   Create a temporary mesh from `valid_forecast` vertices.
        *   Compute the Oriented Bounding Box (OBB) of the cull mesh vertices.
        *   Keep only forecast faces within this OBB.
    *   Filter by area:
        *   Compute area of each connected component in the remaining mesh.
        *   Remove components where $\text{Area} < \text{remove\_small\_geometry\_threshold} \times \text{GlobalArea}$.
        *   Alternatively, keep only the largest connected component if `get_largest_components` is True [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def cull_mesh_by_projection(mesh, cameras, depth_maps, intrinsics, 
                            eps=0.005, min_views=20, batch_size=500000):
    """
    Cull mesh faces based on multi-view depth projection consistency.
    
    Args:
        mesh: Object with .vertices (N,3) and .faces (M,3)
        cameras: List of 4x4 camera-to-world matrices (C2W)
        depth_maps: List of 2D arrays (H, W) representing reference depth
        intrinsics: 3x3 matrix K
        eps: Depth tolerance
        min_views: Minimum number of cameras a vertex must be visible in
        batch_size: Batch size for vertex processing
        
    Returns:
        Filtered mesh
    """
    vertices = mesh.vertices
    faces = mesh.faces
    n_verts = vertices.shape[0]
    
    # Initialize masks
    valid = torch.zeros(n_verts, dtype=torch.bool, device=vertices.device)
    valid_num = torch.zeros(n_verts, dtype=torch.int, device=vertices.device)
    
    # Extract camera parameters
    c2w_list = [cam.to_tensor() for cam in cameras]
    depth_list = [torch.from_numpy(d).to(vertices.device) for d in depth_maps]
    
    H, W = depth_list[0].shape
    fx, fy = intrinsics[0,0], intrinsics[1,1]
    cx, cy = intrinsics[0,2], intrinsics[1,2]
    
    # Process vertices in batches
    for start in range(0, n_verts, batch_size):
        end = min(start + batch_size, n_verts)
        batch_verts = vertices[start:end]
        
        # Initialize batch masks
        batch_valid = torch.zeros(end - start, dtype=torch.bool, device=vertices.device)
        batch_valid_num = torch.zeros(end - start, dtype=torch.int, device=vertices.device)
        
        for c2w, depth_map in zip(c2w_list, depth_list):
            # Adjust coordinate system (negate y, z rows of rotation part)
            # Note: This assumes c2w is 4x4. 
            # In practice, this converts from OpenGL/COLMAP to internal convention.
            c2w_adj = c2w.clone()
            c2w_adj[1:3, 1:3] *= -1
            
            # World to Camera
            w2c = torch.inverse(c2w_adj)
            
            # Project vertices to camera space
            # Homogeneous coordinates
            verts_h = torch.cat([batch_verts, torch.ones(end-start, 1, device=vertices.device)], dim=1)
            verts_cam = (verts_h @ w2c.T)[:, :3] # x, y, z in camera space
            
            z = verts_cam[:, 2] + 1e-8
            
            # Intrinsic projection
            u = (fx * verts_cam[:, 0] + cx * z) / z
            v = (fy * verts_cam[:, 1] + cy * z) / z
            
            # Frustum check
            in_frustum = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z > 0)
            
            if not torch.any(in_frustum):
                continue
                
            # Sample depth
            # Normalize UV to [-1, 1] for grid_sample
            u_norm = 2.0 * u / W - 1.0
            v_norm = 2.0 * v / H - 1.0
            
            # Create grid for sampling
            # Note: grid_sample expects (N, H, W, 2) or similar. 
            # Here we sample specific points.
            # Using F.grid_sample with padding_mode='border'
            depth_map_4d = depth_map.unsqueeze(0).unsqueeze(0) # (1, 1, H, W)
            grid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, 1, 2)
            
            # Sample depth at UV coordinates
            # Reshape for grid_sample: (1, 1, 1, 2) -> (1, 1, 1, 2)
            # Actually, grid_sample takes (N, C, H, W) and grid (N, H_out, W_out, 2)
            # We need to reshape batch_verts to match.
            
            # Simplified sampling logic for pseudocode:
            # depth_sample = bilinear_interpolate(depth_map, u, v)
            depth_sample = sample_depth_bilinear(depth_map_4d, u_norm, v_norm)
            
            # Front-face validation
            is_front = (depth_sample > 0) & (z < depth_sample + eps)
            
            # Update masks for points in frustum
            mask = in_frustum & is_front
            batch_valid_num[mask] += 1
            batch_valid[mask] = True
            
        # Finalize batch validity
        final_valid = batch_valid_num >= min_views
        valid[start:end] = final_valid
        
    # Face masking
    # face_mask[i] is true if all 3 vertices of face i are valid
    face_mask = valid[faces].all(dim=1)
    
    # Update mesh
    filtered_faces = faces[face_mask]
    filtered_mesh = create_mesh(filtered_faces, vertices)
    
    # Remove unreferenced vertices
    filtered_mesh = remove_unreferenced_vertices(filtered_mesh)
    
    # Optional: Connected Component Filtering
    # filtered_mesh = filter_connected_components(filtered_mesh)
    
    return filtered_mesh

def sample_depth_bilinear(depth_map, u, v):
    """
    Sample depth map at (u, v) using bilinear interpolation.
    Assumes u, v are in [0, W-1], [0, H-1].
    """
    # Normalize to [-1, 1]
    u_norm = 2.0 * u / (depth_map.shape[1] - 1) - 1.0
    v_norm = 2.0 * v / (depth_map.shape[0] - 1) - 1.0
    
    # Create grid
    grid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0)
    
    # Sample
    depth_4d = depth_map.unsqueeze(0).unsqueeze(0)
    sampled = torch.nn.functional.grid_sample(depth_4d, grid, align_corners=True, padding_mode='border')
    return sampled.squeeze()
```

## Subtopics

### [Mesh Post-Processing (Cluster Filtering)](7002-mesh-post-processing-cluster-filtering.md)
Filters out floating artifacts and disconnected components from a 3D mesh by clustering triangles and retaining only the largest connected clusters. It serves as a cleanup step following this method, specifically targeting small, noisy geometry that survives projection-based culling. Unlike this method which removes occluded or distant geometry, this method removes topologically isolated fragments based on triangle count.

## Operating characteristics
*   **Time Complexity**: $O(N_v \cdot N_c)$, where $N_v$ is the number of vertices and $N_c$ is the number of cameras. Depth sampling is $O(1)$ per vertex per camera.
*   **Memory Complexity**: $O(N_v \cdot N_c)$ for intermediate masks if not batched, or $O(N_v)$ with batching. Depth maps are stored in memory.
*   **Performance**: The method is computationally intensive due to the large number of camera projections. Batching (`points_batch_size`) is critical for GPU memory management. The requirement of $\ge 20$ views makes it suitable for scenes with dense camera captures (e.g., COLMAP datasets).
*   **Strengths**: Effectively removes "halo" artifacts and background noise in unbounded reconstructions. Robust to occlusions by requiring multi-view consistency.
*   **Limitations**: Requires pre-existing depth maps from multiple views. Computationally expensive for large meshes and many cameras. Fails if the reference depth maps are inaccurate or if the mesh is severely distorted.
*   **When to use**: Post-processing mesh extraction from Gaussian Splatting or NeRF models where unbounded reconstruction leads to artifacts.
*   **When not to use**: Real-time applications due to high computational cost. Scenes with sparse camera coverage (fewer than 20 views).

## Background
This method originates from the **2D Gaussian Splatting for Geometrically Accurate Radiance Fields** paper (Huang et al., SIGGRAPH 2024) [S1, unofficial]. It was developed to address the issue of geometric artifacts in unbounded mesh extraction from 2D Gaussian Splatting, which often produces "halo" effects and noisy geometry in regions not well-covered by the training views. The method leverages the multi-view nature of the input data to validate surface consistency.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
