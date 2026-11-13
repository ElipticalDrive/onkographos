# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method removes geometric artifacts and occluded regions from a 3D mesh by projecting its vertices into multiple camera views and comparing against reference depth maps. It classifies faces as "seen" (visible in at least 20 views), "unseen" (occluded or background), or "forecast" (distant potential geometry) [S1, unofficial]. It is distinct from neighbor-based culling (e.g., [Face Culling](8735-face-culling.md)) because it relies on multi-view consistency rather than local voxel occupancy, making it suitable for unbounded or non-volumetric meshes like those from Gaussian Splatting [S1, unofficial].

## How it works
**Inputs:**
- A triangle mesh $M$ with vertices $V \in \mathbb{R}^{N_v \times 3}$ and faces $F \in \mathbb{R}^{N_f \times 3}$.
- A set of $K$ camera poses $C = \{c_2w_k\}_{k=1}^K$ (world-to-camera transformation matrices).
- Camera intrinsics $K_{intr} = \text{diag}(f_x, f_y, 1)$ and principal point $(c_x, c_y)$.
- Reference depth maps $D_k \in \mathbb{R}^{H \times W}$ for each camera pose.
- Parameters: `points_batch_size` (default $5 \times 10^5$), `eps` (default $0.005$), `remove_small_geometry_threshold` (default $0.2$), `forecast_radius` (default $0$, optional).

**Outputs:**
- A culled mesh $M'$ containing only faces classified as "seen" or valid "forecast" components.

**Data Structures:**
- `valid`: Boolean tensor of shape $(N_v)$, true if vertex is seen.
- `valid_forecast`: Boolean tensor of shape $(N_v)$, true if vertex is forecast.
- `face_mask`: Boolean tensor of shape $(N_f)$, true if all 3 vertices of the face are valid.

**Steps:**

1. **Initialize Masks:**
   Create zero tensors `valid`, `valid_forecast`, and counter `valid_num` of shape $(N_v)$ on the target device.

2. **Iterate Camera Poses:**
   For each camera pose $c_2w_k$ and depth map $D_k$:
   
   a. **Coordinate Transformation:**
      Adjust for coordinate system differences (e.g., OpenGL to PyRender). Negate the $y$ and $z$ rows of the rotation matrix:
      $$ R' = \begin{bmatrix} R_{00} & R_{01} & R_{02} \\ R_{10} & -R_{11} & -R_{12} \\ R_{20} & -R_{21} & -R_{22} \end{bmatrix} $$
      Construct $c_2w'$ with $R'$ and the same translation. Compute world-to-camera matrix:
      $$ W2C = (c_2w')^{-1} $$

   b. **Projection:**
      For a batch of vertices $V_{batch}$:
      - Convert to homogeneous coordinates: $V_h = [V_{batch}, \mathbf{1}]^T \in \mathbb{R}^{4 \times B}$.
      - Transform to camera space: $P_{cam} = W2C \cdot V_h$.
      - Extract first three rows: $P_{uvz} = P_{cam}[0:3, :]$.
      - Apply intrinsics:
        $$ u = f_x \frac{P_{uvz}[0]}{P_{uvz}[2]} + c_x $$
        $$ v = f_y \frac{P_{uvz}[1]}{P_{uvz}[2]} + c_y $$
        $$ z = P_{uvz}[2] $$
      - Add epsilon to depth to avoid division by zero or log issues if needed: $z \leftarrow z + 10^{-8}$.

   c. **Frustum and Visibility Check:**
      - **Standard Frustum:**
        $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z > 0) $$
      - **Forecast Frustum (if `forecast_radius` $r > 0$):**
        $$ \text{forecast\_frustum} = (-r \le u < W+r) \land (-r \le v < H+r) \land (z > 0) $$
      - **Depth Sampling:**
        Normalize UV to $[-1, 1]$ for grid sampling:
        $$ u_{norm} = \frac{2u}{W} - 1, \quad v_{norm} = \frac{2v}{H} - 1 $$
        Sample reference depth $D_k$ at $(u_{norm}, v_{norm})$ using bilinear interpolation with border padding:
        $$ d_{sample} = \text{grid\_sample}(D_k, [u_{norm}, v_{norm}]) $$
      - **Front-Face Validation:**
        A point is considered "seen" if it is in the standard frustum and closer than the reference depth:
        $$ \text{is\_front} = (d_{sample} > 0) \land (z < d_{sample} + \epsilon) $$
        Update masks:
        $$ \text{valid}[V_{batch}] \leftarrow \text{valid}[V_{batch}] \lor \text{is\_front} $$
        $$ \text{valid\_num}[V_{batch}] \leftarrow \text{valid\_num}[V_{batch}] + \text{is\_front} $$
        
        If `forecast_radius` $> 0$, points in the forecast frustum but not in the standard frustum (or potentially further away) are marked as forecast candidates. Specifically, if `forecast_frustum` is true and `is_front` is false, mark as forecast.

3. **Finalize Vertex Masks:**
   - A vertex is "seen" if it is valid in at least 20 cameras:
     $$ \text{final\_valid} = \text{valid\_num} \ge 20 $$
   - Combine with forecast mask if applicable.

4. **Face Culling:**
   - Create face mask:
     $$ \text{face\_mask} = \text{final\_valid}[F[:, 0]] \land \text{final\_valid}[F[:, 1]] \land \text{final\_valid}[F[:, 2]] $$
   - Update mesh:
     $$ M' = M[\text{face\_mask}] $$
   - Remove unreferenced vertices from $M'$.

5. **Connected Component Filtering (Optional):**
   If `forecast_radius` $> 0$ or to clean up artifacts:
   - Compute the Oriented Bounding Box (OBB) of the cull mesh vertices.
   - Keep only forecast faces within this OBB.
   - Filter components by area:
     - Compute area of each connected component in $M'$.
     - Keep components where $\text{Area}_i > \text{remove\_small\_geometry\_threshold} \times \text{TotalArea}$.
     - Alternatively, keep only the largest component if `get_largest_components` is true.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def cull_mesh(mesh, cameras, depth_maps, params):
    """
    mesh: trimesh.Mesh object with vertices (N,3) and faces (M,3)
    cameras: list of (c2w, K_intr, H, W) tuples
    depth_maps: list of (H, W) tensors
    params: dict with eps, forecast_radius, remove_small_geometry_threshold
    """
    V = mesh.vertices  # (N, 3)
    F = mesh.faces     # (M, 3)
    N = V.shape[0]
    
    # Initialize masks
    valid = torch.zeros(N, dtype=torch.bool, device=V.device)
    valid_num = torch.zeros(N, dtype=torch.int, device=V.device)
    valid_forecast = torch.zeros(N, dtype=torch.bool, device=V.device)
    
    batch_size = params.get('points_batch_size', 500000)
    eps = params.get('eps', 0.005)
    forecast_radius = params.get('forecast_radius', 0)
    
    # Process vertices in batches
    for start in range(0, N, batch_size):
        end = min(start + batch_size, N)
        V_batch = V[start:end]
        
        for k, (c2w, K_intr, H, W) in enumerate(cameras):
            depth_map = depth_maps[k]
            
            # Coordinate transformation
            R = c2w[:3, :3]
            t = c2w[:3, 3]
            # Negate y and z rows for OpenGL/PyRender compatibility
            R_adj = R.clone()
            R_adj[1, :] *= -1
            R_adj[2, :] *= -1
            c2w_adj = torch.eye(4, device=V.device)
            c2w_adj[:3, :3] = R_adj
            c2w_adj[:3, 3] = t
            
            # World to Camera
            W2C = torch.inverse(c2w_adj)
            
            # Project vertices
            V_h = torch.cat([V_batch, torch.ones(end-start, 1, device=V.device)], dim=1) # (B, 4)
            P_cam = (W2C @ V_h.T).T # (B, 4)
            z = P_cam[:, 2] + 1e-8
            u = (K_intr[0, 0] * P_cam[:, 0] / z) + K_intr[0, 2]
            v = (K_intr[1, 1] * P_cam[:, 1] / z) + K_intr[1, 2]
            
            # Frustum checks
            in_frustum = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z > 0)
            
            # Forecast frustum
            if forecast_radius > 0:
                r = forecast_radius
                forecast_frustum = (u >= -r) & (u < W + r) & (v >= -r) & (v < H + r) & (z > 0)
            else:
                forecast_frustum = torch.zeros(end-start, dtype=torch.bool, device=V.device)
            
            # Depth sampling
            if in_frustum.any():
                # Normalize to [-1, 1]
                u_norm = 2.0 * u / W - 1.0
                v_norm = 2.0 * v / H - 1.0
                vgrid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, B, 2)
                
                # Reshape depth map for grid_sample: (1, 1, H, W)
                depth_reshaped = depth_map.unsqueeze(0).unsqueeze(0)
                
                # Sample depth
                depth_sample = F.grid_sample(depth_reshaped, vgrid, padding_mode='border', align_corners=True)
                depth_sample = depth_sample.squeeze().squeeze() # (B,)
                
                # Front face check
                is_front = (depth_sample > 0.0) & (z < depth_sample + eps)
                
                # Update valid mask
                valid_mask = in_frustum & is_front
                valid[start:end] = valid[start:end] | valid_mask
                valid_num[start:end] = valid_num[start:end] + valid_mask.int()
                
                # Update forecast mask
                if forecast_radius > 0:
                    # Forecast if in forecast frustum but not valid (or potentially further)
                    # Note: The source implies forecast is a separate category for distant geometry
                    # We mark as forecast if in forecast frustum and NOT front-facing
                    is_forecast = forecast_frustum & ~is_front
                    valid_forecast[start:end] = valid_forecast[start:end] | is_forecast

    # Finalize seen mask: requires at least 20 views
    final_valid = valid_num >= 20
    
    # Combine with forecast if needed
    # If forecast_radius > 0, we might keep forecast vertices for OBB calculation
    # For simplicity, we assume the main culling is based on 'final_valid'
    # and 'valid_forecast' is used for secondary filtering or OBB
    
    # Face mask: all vertices must be valid
    face_mask = final_valid[F[:, 0]] & final_valid[F[:, 1]] & final_valid[F[:, 2]]
    
    # Update mesh
    new_faces = F[face_mask]
    new_vertices = V[final_valid]
    
    # Re-index vertices
    # Create a mapping from old vertex index to new vertex index
    vertex_map = torch.full((N,), -1, dtype=torch.long, device=V.device)
    vertex_map[final_valid] = torch.arange(final_valid.sum(), device=V.device)
    new_faces = vertex_map[new_faces]
    
    # Remove faces with any -1 (should not happen if logic is correct)
    valid_face_mask = (new_faces >= 0).all(dim=1)
    new_faces = new_faces[valid_face_mask]
    
    # Connected Component Filtering (Optional)
    if forecast_radius > 0 or params.get('remove_small_geometry', True):
        # Compute OBB of new_vertices
        # Keep only faces within OBB if forecast_radius > 0
        # Filter by area
        # This step is complex to implement from scratch without a library like trimesh
        # Assuming trimesh is available for component analysis
        import trimesh
        temp_mesh = trimesh.Trimesh(vertices=new_vertices, faces=new_faces, process=False)
        
        # Get connected components
        components = temp_mesh.split(only_watertight=False)
        
        # Filter components
        total_area = sum(c.area for c in components)
        threshold = params.get('remove_small_geometry_threshold', 0.2) * total_area
        
        kept_faces = []
        for comp in components:
            if comp.area > threshold:
                # Map component faces back to global indices
                # This requires tracking face indices
                pass
        
        # Simplified: Keep largest component if get_largest_components is True
        if params.get('get_largest_components', False):
            largest_comp = max(components, key=lambda c: c.area)
            new_faces = largest_comp.faces
            new_vertices = largest_comp.vertices
        else:
            # Keep components above threshold
            # Reconstruct mesh from kept components
            pass
            
    return trimesh.Trimesh(vertices=new_vertices, faces=new_faces)
```

## Subtopics

### [Mesh Post-Processing (Cluster Filtering)](7002-mesh-post-processing-cluster-filtering.md)
Filters floating artifacts and disconnected components by clustering triangles and retaining only the largest connected clusters. It serves as a cleanup step following this method, specifically targeting small, noisy geometry that survives projection-based culling. Unlike this method which removes occluded or distant geometry, this method removes topologically isolated fragments based on triangle count.

## Operating characteristics
- **Time Complexity:** $O(N_v \cdot K)$, where $N_v$ is the number of vertices and $K$ is the number of camera poses. Depth sampling is $O(1)$ per vertex per camera.
- **Memory Complexity:** $O(N_v \cdot K)$ for storing intermediate masks if not batched, or $O(N_v + N_f)$ if processed in batches.
- **Performance:** Effective for unbounded meshes from Gaussian Splatting where local voxel occupancy is not defined. Requires a sufficient number of camera poses (at least 20 for the threshold) to ensure robust visibility estimation.
- **Strengths:** Handles complex, non-volumetric geometry; removes background noise and occluded artifacts; works with arbitrary mesh topology.
- **Limitations:** Requires pre-existing camera poses and depth maps; sensitive to depth map accuracy; may remove valid geometry if camera coverage is sparse or depth maps are noisy.
- **When to use:** When extracting meshes from 3D Gaussian Splatting or other unbounded representations where neighbor-based culling is not applicable.
- **When not to use:** For voxel-based meshes where neighbor occupancy is available and sufficient; when camera poses are not available or inaccurate.

## Background
This method originates from the "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" paper by Huang et al. (SIGGRAPH 2024) [S1, unofficial]. It was developed to address the challenge of extracting clean, geometrically accurate meshes from 2D Gaussian Splatting reconstructions, which often contain artifacts and background noise. The method leverages multi-view consistency to distinguish between visible surface geometry and occluded or spurious elements.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
