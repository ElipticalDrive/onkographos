# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method culls mesh geometry by projecting vertices into multiple camera views and comparing their depths against reference depth maps to identify and remove unseen or artifact geometry [S1, unofficial]. It distinguishes itself from neighbor-based culling (e.g., [Face Culling](8735-face-culling.md)) by relying on explicit visibility testing against a set of camera poses rather than local voxel occupancy, making it suitable for unbounded or sparse reconstructions where topological connectivity is unreliable [S1, unofficial].

## How it works
**Inputs:**
*   `mesh`: A triangle mesh with vertices $V \in \mathbb{R}^{N \times 3}$ and faces $F \in \mathbb{R}^{M \times 3}$.
*   `cameras`: A list of $K$ camera poses (world-to-camera matrices $C_i$) and intrinsic parameters ($H, W, f_x, f_y, c_x, c_y$).
*   `depth_maps`: A list of $K$ reference depth maps $D_i \in \mathbb{R}^{H \times W}$ corresponding to the camera poses.
*   `forecast_radius` ($r$): Optional radius for extending the frustum check for "forecast" visibility.
*   `remove_small_geometry_threshold` ($\alpha$): Minimum area ratio to retain a component (default 0.2).

**Outputs:**
*   `culled_mesh`: The mesh with invisible faces removed and unreferenced vertices deleted.

**Data Structures:**
*   `valid`: Boolean tensor of shape $(N)$, true if a vertex is seen in $\ge 20$ cameras.
*   `valid_forecast`: Boolean tensor of shape $(N)$, true if a vertex is seen in the extended frustum.
*   `face_mask`: Boolean tensor of shape $(M)$, true if all three vertices of a face are visible.

**Parameters:**
*   `points_batch_size`: Max points processed per batch to limit GPU memory (default $5 \times 10^5$) [S1, unofficial].
*   `eps`: Depth comparison tolerance (default $0.005$) [S1, unofficial].
*   `far`: Far plane distance for depth extraction (default $20.0$) [S1, unofficial].

**Steps:**

1.  **Extract Reference Depth Maps:**
    Render forward depth maps from the input `mesh` using each camera pose $C_i$ and intrinsics. This provides the ground truth visibility for the current geometry state [S1, unofficial].

2.  **Initialize Masks:**
    Create `valid`, `valid_num`, and `valid_forecast` tensors of shape $(N)$ initialized to zero on the target device [S1, unofficial].

3.  **Project Vertices and Validate Visibility:**
    For each camera $i$ from $0$ to $K-1$:
    a.  **Coordinate Transformation:** Convert the camera pose. If $C_i$ is a world-to-camera matrix, ensure correct axis alignment (e.g., negate $y$ and $z$ rows of the rotation part if necessary for the specific engine) [S1, unofficial]. Compute $W2C = C_i^{-1}$ [S1, unofficial].
    b.  **Homogeneous Projection:** Append ones to $V$ to get $V_h \in \mathbb{R}^{N \times 4}$. Compute $P_{cam} = V_h \cdot W2C^T$. Extract the first three rows to get $P_{proj} \in \mathbb{R}^{N \times 3}$ (where the third component is depth $z$) [S1, unofficial].
    c.  **Intrinsic Projection:** Apply intrinsic matrix $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$. Compute $UV_{hom} = P_{proj} \cdot K^T$.
    d.  **Normalize Coordinates:**
        $$ z = UV_{hom}[:, 2] + 10^{-8} $$
        $$ u = UV_{hom}[:, 0] / z $$
        $$ v = UV_{hom}[:, 1] / z $$
    e.  **Frustum Check:**
        Determine if points are within the standard frustum:
        $$ \text{in\_frustum} = (u \ge 0) \land (u \le W-1) \land (v \ge 0) \land (v \le H-1) \land (z > 0) $$
        Determine if points are within the forecast frustum (extended by $r$):
        $$ \text{forecast\_frustum} = (u \ge -r) \land (u \le W-1+r) \land (v \ge -r) \land (v \le H-1+r) \land (z > 0) $$
    f.  **Depth Sampling:**
        Reshape $D_i$ to $(1, 1, H, W)$. Normalize $u, v$ to $[-1, 1]$ for grid sampling. Sample depth at $(u, v)$ using bilinear interpolation with border padding:
        $$ D_{sample} = \text{GridSample}(D_i, (u, v), \text{padding\_mode}=\text{'border'}) $$
    g.  **Front-Face Validation:**
        A point is considered "seen" if the sampled depth is positive and the projected point is in front of the sampled surface:
        $$ \text{is\_front} = (D_{sample} > 0.0) \land (z < D_{sample} + \epsilon) $$
    h.  **Update Masks:**
        For points satisfying `in_frustum` and `is_front`, increment `valid_num` and set `valid` to true.
        For points satisfying `forecast_frustum` and `is_front`, set `valid_forecast` to true [S1, unofficial].

4.  **Finalize Vertex Visibility:**
    A vertex is considered "seen" if it is visible in at least 20 cameras:
    $$ \text{final\_valid} = (\text{valid\_num} \ge 20) $$
    Combine with forecast visibility if `forecast_radius > 0`:
    $$ \text{visible\_vertices} = \text{final\_valid} \lor \text{valid\_forecast} $$

5.  **Face Culling:**
    Create a face mask by checking if all vertices of each face are visible:
    $$ \text{face\_mask} = \text{visible\_vertices}[F].\text{all}(\text{axis}=1) $$
    Update the mesh to remove faces where `face_mask` is false [S1, unofficial].

6.  **Connected Component Filtering (Optional):**
    If `forecast_radius > 0` or cleanup is required:
    a.  Compute the Oriented Bounding Box (OBB) of the currently valid mesh vertices.
    b.  Identify connected components in the remaining mesh.
    c.  Remove components with area $< \alpha \times \text{total\_area}$ or keep only the largest component if configured [S1, unofficial].

7.  **Cleanup:**
    Remove unreferenced vertices from the mesh [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def cull_mesh_by_projection(mesh, cameras, depth_maps, forecast_radius=0.0, 
                            remove_small_geometry_threshold=0.2, eps=0.005):
    """
    Cull mesh geometry based on multi-view depth projection.
    
    Args:
        mesh: Object with .vertices (N,3) and .faces (M,3)
        cameras: List of dicts with 'c2w' (4,4), 'H', 'W', 'fx', 'fy', 'cx', 'cy'
        depth_maps: List of tensors (H, W)
        forecast_radius: Float, radius for extended frustum
        remove_small_geometry_threshold: Float, min area ratio
        eps: Float, depth tolerance
        
    Returns:
        culled_mesh: Mesh object with updated faces/vertices
    """
    V = mesh.vertices.clone() # (N, 3)
    F = mesh.faces # (M, 3)
    N = V.shape[0]
    K = len(cameras)
    
    # Initialize masks
    valid_num = torch.zeros(N, device=V.device)
    valid = torch.zeros(N, device=V.device, dtype=torch.bool)
    valid_forecast = torch.zeros(N, device=V.device, dtype=torch.bool)
    
    # Batch processing for memory efficiency
    batch_size = 500000
    indices = torch.arange(N)
    
    for cam_idx, cam in enumerate(cameras):
        # 1. Coordinate Transformation
        c2w = cam['c2w']
        # Negate y and z rows of rotation part if necessary for specific engine convention
        # Assuming standard OpenGL/COLMAP convention where c2w is World-to-Camera
        # If c2w is Camera-to-World, invert it first. 
        # Note: Source implies c2w is used directly to compute w2c = inv(c2w)
        w2c = torch.inverse(c2w)
        
        # 2. Project Points
        # Append ones for homogeneous coordinates
        V_h = torch.cat([V, torch.ones(N, 1, device=V.device)], dim=1) # (N, 4)
        
        # Multiply by w2c. w2c is (4,4). V_h is (N,4).
        # Result P_cam is (N, 4). We need the first 3 rows of the projection.
        # Standard matrix mult: P_cam = V_h @ w2c.T
        P_cam = V_h @ w2c.T # (N, 4)
        
        # Extract x, y, z from the first 3 components of the transformed vector
        # Note: In some conventions, the 4th component is w. 
        # Source says "extract the first three rows" after multiplying by w2c.
        # This implies P_cam[:, :3] contains the projected coordinates in camera space.
        P_proj = P_cam[:, :3] # (N, 3)
        
        z = P_proj[:, 2]
        
        # 3. Intrinsic Projection
        fx = cam['fx']
        fy = cam['fy']
        cx = cam['cx']
        cy = cam['cy']
        
        u = (P_proj[:, 0] * fx + cx) / (z + 1e-8)
        v = (P_proj[:, 1] * fy + cy) / (z + 1e-8)
        
        # 4. Frustum Check
        H = cam['H']
        W = cam['W']
        
        in_frustum = (u >= 0) & (u <= W - 1) & (v >= 0) & (v <= H - 1) & (z > 0)
        
        forecast_frustum = (u >= -forecast_radius) & (u <= W - 1 + forecast_radius) & \
                           (v >= -forecast_radius) & (v <= H - 1 + forecast_radius) & (z > 0)
        
        # 5. Depth Sampling
        depth_map = depth_maps[cam_idx] # (H, W)
        # Reshape to (1, 1, H, W) for grid_sample
        depth_map_4d = depth_map.unsqueeze(0).unsqueeze(0)
        
        # Normalize u, v to [-1, 1] for grid_sample
        # grid_sample expects (N, H_out, W_out, 2) or (1, 1, H_out, W_out, 2)
        # We have N points. We need to create a grid of shape (1, 1, N, 2)
        u_norm = 2.0 * (u / (W - 1)) - 1.0
        v_norm = 2.0 * (v / (H - 1)) - 1.0
        
        # Clamp to avoid out of bounds sampling issues, though padding_mode='border' handles it
        # grid_sample expects coordinates in [-1, 1]
        grid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, N, 2)
        
        # Sample depth
        # padding_mode='border' pads with edge values
        depth_sample = F.grid_sample(depth_map_4d, grid, align_corners=True, padding_mode='border') # (1, 1, N, 1)
        depth_sample = depth_sample.squeeze() # (N,)
        
        # 6. Front-Face Validation
        is_front = (depth_sample > 0.0) & (z < depth_sample + eps)
        
        # 7. Update Masks
        # Only update if in frustum and front-facing
        mask_valid = in_frustum & is_front
        mask_forecast = forecast_frustum & is_front
        
        valid_num = valid_num + mask_valid.float()
        valid = valid | mask_valid
        valid_forecast = valid_forecast | mask_forecast
        
    # 8. Finalize Vertex Visibility
    # Require visibility in at least 20 cameras
    final_valid = valid_num >= 20
    
    # Combine with forecast if radius > 0
    if forecast_radius > 0:
        visible_vertices = final_valid | valid_forecast
    else:
        visible_vertices = final_valid
        
    # 9. Face Culling
    # Check if all vertices of each face are visible
    # F is (M, 3). indices are vertex indices.
    face_visibility = visible_vertices[F] # (M, 3) bool
    face_mask = face_visibility.all(dim=1) # (M,) bool
    
    # 10. Connected Component Filtering (Simplified)
    # Note: Full connected component analysis requires graph traversal.
    # Source mentions filtering by area or keeping largest component.
    # Here we implement a basic area-based filter if configured.
    if remove_small_geometry_threshold > 0:
        # Calculate area of each face (approximate)
        # This requires 3D geometry of the mesh. 
        # For simplicity, we assume the mesh object has a method to compute component areas.
        # If not, we fall back to keeping all faces that passed the visibility test.
        # [G] In a real implementation, one would build an adjacency graph of visible faces,
        # find connected components, compute their total area, and remove small ones.
        pass 
        
    # 11. Update Mesh
    mesh.faces = mesh.faces[face_mask]
    mesh.remove_unreferenced_vertices()
    
    return mesh
```

## Subtopics

### [Mesh Post-Processing (Cluster Filtering)](7002-mesh-post-processing-cluster-filtering.md)
Filters floating artifacts and disconnected components by clustering triangles and retaining only the largest connected clusters. It serves as a cleanup step following this method, specifically targeting small, noisy geometry that survives projection-based culling. Unlike this method which removes occluded or distant geometry, this method removes topologically isolated fragments based on triangle count.

## Operating characteristics
*   **Time Complexity:** $O(K \cdot N)$ for projection and depth sampling, where $K$ is the number of cameras and $N$ is the number of vertices. The depth sampling step involves bilinear interpolation which is $O(1)$ per point but requires memory access to the depth maps.
*   **Memory Complexity:** $O(N + K \cdot H \cdot W)$ to store the masks and depth maps. The `points_batch_size` parameter allows trading memory for speed by processing vertices in chunks.
*   **Strengths:** Effective for unbounded scenes or sparse reconstructions (e.g., Gaussian Splatting) where voxel-based neighbor checks fail due to lack of topological continuity. It explicitly uses viewing direction and occlusion.
*   **Limitations:** Requires a set of camera poses and depth maps. Performance degrades if the number of cameras $K$ is small, as the "seen in $\ge 20$ cameras" threshold may not be met. Precision depends on the accuracy of the reference depth maps and the intrinsic parameters.
*   **When to use:** When extracting meshes from unbounded radiance fields (like 2D Gaussian Splatting) or when the geometry is too sparse for voxel-based culling.
*   **When not to use:** For dense, closed-surface voxel grids where neighbor-based culling is faster and more robust.

## Background
This method was introduced in the context of **2D Gaussian Splatting for Geometrically Accurate Radiance Fields** (Huang et al., SIGGRAPH 2024) [S1, unofficial]. It addresses the challenge of extracting clean meshes from unbounded Gaussian splatting reconstructions, which often contain artifacts and noise in unseen regions. The method leverages the multi-view consistency of the splatting representation to filter out geometry that is not consistently visible across multiple camera poses.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
