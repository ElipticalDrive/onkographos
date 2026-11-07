# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method culls mesh geometry by projecting 3D vertices into multiple 2D camera views and comparing their depths against reference depth maps to classify faces as seen, unseen, or forecast [S1, unofficial]. It is specifically designed to remove artifacts from unbounded Gaussian splatting reconstruction where traditional voxel-based neighbor checks fail due to the lack of a bounded grid [S1, unofficial]. Unlike sibling methods that rely on local voxel occupancy (e.g., [Face Culling](8735-face-culling.md)), this approach uses global visibility information from multiple viewpoints to determine face retention.

## How it works
**Inputs:**
*   `mesh`: A triangle mesh object containing vertices $V$ and faces $F$.
*   `camera_poses`: A list of camera-to-world transformation matrices $C_{2W}$.
*   `intrinsics`: Camera intrinsic parameters $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$, image resolution $H \times W$.
*   `depth_maps`: Pre-rendered forward depth maps for each camera pose.
*   `forecast_radius`: Radius for expanding the frustum for "forecast" visibility.
*   `points_batch_size`: Maximum points processed per batch (default $5 \times 10^5$) [S1, unofficial].
*   `eps`: Depth comparison tolerance (default $0.005$) [S1, unofficial].
*   `remove_small_geometry_threshold`: Minimum face area ratio to keep a component (default $0.2$) [S1, unofficial].

**Outputs:**
*   `cull_mesh`: The filtered mesh with hidden faces and unreferenced vertices removed.

**Data Structures:**
*   `valid`: Boolean tensor of shape $(N_{pts})$, true if a point is seen in $\ge 20$ cameras.
*   `valid_forecast`: Boolean tensor of shape $(N_{pts})$, true if a point is in the forecast frustum and front-facing.
*   `face_mask`: Boolean array of shape $(N_{faces})$, true if all vertices of the face are visible.

**Steps:**

1.  **Extract Reference Depth Maps:**
    Render forward depth maps from the input mesh using the provided camera poses and intrinsics. Use flags `OFFSCREEN`, `DEPTH_ONLY`, and `SKIP_CULL_FACES` to ensure accurate depth sampling without self-occlusion artifacts from the mesh itself [S1, unofficial].

2.  **Initialize Masks:**
    Create tensors `valid`, `valid_num`, and `valid_forecast` of shape $(N_{vertices})$ initialized to zero on the target device [S1, unofficial].

3.  **Project Vertices to Camera Space:**
    For each camera pose $C_{2W}$:
    a.  Convert coordinate system by negating the y and z rows of the rotation matrix: $C_{2W}[:3, 1:3] *= -1$ [S1, unofficial].
    b.  Compute world-to-camera matrix: $W_{2C} = C_{2W}^{-1}$ [S1, unofficial].
    c.  Convert vertices $V$ to homogeneous coordinates by appending ones.
    d.  Transform to camera space: $P_{cam} = W_{2C} \cdot V_{homogeneous}$.
    e.  Extract the first three rows to get $(x, y, z)$ in camera space [S1, unofficial].

4.  **Project to UV and Check Frustum:**
    a.  Apply intrinsics: $uv = K \cdot P_{cam}$.
    b.  Extract depth $z = uv[:, -1] + 10^{-8}$ and compute normalized UV: $u = uv[:, 0] / z$, $v = uv[:, 1] / z$ [S1, unofficial].
    c.  Check standard frustum bounds:
        $$ \text{in\_frustum} = (u \ge 0) \land (u \le W-1) \land (v \ge 0) \land (v \le H-1) \land (z > 0) $$
    d.  Check forecast frustum with radius $r$:
        $$ \text{forecast\_frustum} = (u \ge -r) \land (u \le W-1+r) \land (v \ge -r) \land (v \le H-1+r) \land (z > 0) $$
        [S1, unofficial]

5.  **Sample Depth and Validate Front-Facing Geometry:**
    a.  Reshape reference depth map to $(1, 1, H, W)$ and normalize UV grid to $[-1, 1]$ for sampling [S1, unofficial].
    b.  Sample depth at $(u, v)$ using bilinear interpolation with border padding:
        $$ \text{depth\_sample} = \text{grid\_sample}(\text{depth}, \text{vgrid}, \text{padding\_mode='border'}, \text{align\_corners=True}) $$
    c.  Validate front-facing geometry:
        $$ \text{is\_front\_face} = (\text{depth\_sample} > 0.0) \land (z < \text{depth\_sample} + \epsilon) $$
        [S1, unofficial]

6.  **Update Masks:**
    a.  If `in_frustum` and `is_front_face` are true, increment `valid_num` and set `valid` to true for the corresponding vertices [S1, unofficial].
    b.  If `forecast_radius > 0` and `forecast_frustum` and `is_front_face` are true, set `valid_forecast` to true [S1, unofficial].

7.  **Finalize Visibility:**
    a.  A vertex is considered "seen" if `valid_num >= 20` (requires projection into at least 20 cameras) [S1, unofficial].
    b.  A vertex is considered "forecast" if `valid_forecast` is true [S1, unofficial].

8.  **Create Face Mask:**
    Determine visibility for each face by checking if all its vertices are visible (either seen or forecast):
    $$ \text{face\_mask} = \text{mask}[mesh.faces].\text{all}(axis=1) $$
    where `mask` combines `valid` and `valid_forecast` [S1, unofficial].

9.  **Update Mesh:**
    Apply `face_mask` to remove hidden faces, then remove unreferenced vertices [S1, unofficial].

10. **Connected Component Filtering (Optional):**
    If `forecast_radius > 0`:
    a.  Create a forecast mesh from `forecast_mask`.
    b.  Compute the Oriented Bounding Box (OBB) of the cull mesh's vertices.
    c.  Keep only forecast faces within this OBB [S1, unofficial].
    d.  Identify connected triangle clusters using `cluster_connected_triangles()`.
    e.  Determine threshold: sort `cluster_n_triangles`, select the N-th largest value (where N is `cluster_to_keep`, default 1000), and enforce `min_cluster_size` (default 50).
    f.  Remove triangles belonging to clusters smaller than the threshold, then remove unreferenced vertices and degenerate triangles [S1, unofficial].

11. **Area-Based Filtering:**
    Filter mesh components by area: keep components where area $> \text{remove\_small\_geometry\_threshold} \times \text{global\_area}$, or keep only the largest component if `get_largest_components` is True [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cull_mesh_via_3d_projection(mesh, camera_poses, intrinsics, depth_maps, forecast_radius=0.0, eps=0.005, points_batch_size=500000):
    """
    Cull mesh geometry based on multi-view depth projection.
    
    Args:
        mesh: Triangle mesh with vertices and faces.
        camera_poses: List of C2W matrices.
        intrinsics: Tuple (fx, fy, cx, cy, H, W).
        depth_maps: List of depth maps corresponding to camera_poses.
        forecast_radius: Radius for forecast frustum expansion.
        eps: Depth tolerance.
        points_batch_size: Batch size for point processing.
        
    Returns:
        Cull mesh.
    """
    vertices = mesh.vertices
    faces = mesh.faces
    n_verts = len(vertices)
    
    # Initialize masks
    valid = torch.zeros(n_verts, dtype=torch.bool)
    valid_num = torch.zeros(n_verts, dtype=torch.int)
    valid_forecast = torch.zeros(n_verts, dtype=torch.bool)
    
    fx, fy, cx, cy, H, W = intrinsics
    
    # Process vertices in batches to limit GPU memory
    for batch_start in range(0, n_verts, points_batch_size):
        batch_end = min(batch_start + points_batch_size, n_verts)
        batch_verts = vertices[batch_start:batch_end]
        
        batch_valid = torch.zeros(batch_end - batch_start, dtype=torch.bool)
        batch_valid_num = torch.zeros(batch_end - batch_start, dtype=torch.int)
        batch_valid_forecast = torch.zeros(batch_end - batch_start, dtype=torch.bool)
        
        for c2w, depth_map in zip(camera_poses, depth_maps):
            # Coordinate system conversion
            c2w_copy = c2w.clone()
            c2w_copy[:3, 1:3] *= -1
            
            # World-to-camera matrix
            w2c = torch.inverse(c2w_copy)
            
            # Project to camera space
            verts_homogeneous = torch.cat([batch_verts, torch.ones_like(batch_verts[:, :1])], dim=1)
            cam_space = (w2c @ verts_homogeneous.T).T[:, :3]
            
            # Apply intrinsics
            uv = cam_space @ torch.tensor([[fx, 0, cx], [0, fy, cy], [0, 0, 1]], dtype=cam_space.dtype).T
            z = uv[:, 2] + 1e-8
            u = uv[:, 0] / z
            v = uv[:, 1] / z
            
            # Frustum checks
            in_frustum = (u >= 0) & (u <= W-1) & (v >= 0) & (v <= H-1) & (z > 0)
            forecast_frustum = (u >= -forecast_radius) & (u <= W-1+forecast_radius) & (v >= -forecast_radius) & (v <= H-1+forecast_radius) & (z > 0)
            
            # Sample depth
            # Normalize UV to [-1, 1] for grid_sample
            vgrid = torch.stack([
                (u / (W-1) * 2 - 1),
                (v / (H-1) * 2 - 1)
            ], dim=-1).unsqueeze(0).unsqueeze(0)
            
            depth_sample = F.grid_sample(
                depth_map.unsqueeze(0).unsqueeze(0), 
                vgrid, 
                padding_mode='border', 
                align_corners=True
            ).squeeze()
            
            # Front-face validation
            is_front_face = (depth_sample > 0.0) & (z < depth_sample + eps)
            
            # Update masks
            front_and_in = in_frustum & is_front_face
            batch_valid_num += front_and_in.int()
            batch_valid_forecast |= (forecast_frustum & is_front_face)
            
        # Finalize seen mask
        batch_valid = batch_valid_num >= 20
        valid[batch_start:batch_end] = batch_valid
        valid_forecast[batch_start:batch_end] = batch_valid_forecast
        
    # Combine masks
    mask = valid | valid_forecast
    
    # Create face mask
    face_mask = mask[faces].all(dim=1)
    
    # Update mesh
    mesh_with_hole = mesh.copy()
    mesh_with_hole.update_faces(face_mask)
    mesh_with_hole.remove_unreferenced_vertices()
    
    # Connected component filtering (if forecast radius > 0)
    if forecast_radius > 0:
        forecast_mask = valid_forecast[faces].all(dim=1)
        forecast_mesh = mesh.copy()
        forecast_mesh.update_faces(forecast_mask)
        
        # Compute OBB of cull mesh vertices
        obb = compute_obb(mesh_with_hole.vertices)
        
        # Keep only forecast faces within OBB
        # (Implementation detail: check if face centroid is within OBB)
        # ...
        
        # Cluster filtering
        clusters = cluster_connected_triangles(mesh_with_hole)
        cluster_sizes = [len(c) for c in clusters]
        cluster_sizes.sort(reverse=True)
        
        if len(cluster_sizes) > 1000:
            threshold = cluster_sizes[1000]
        else:
            threshold = 0
            
        min_cluster_size = 50
        keep_mask = torch.zeros(len(mesh_with_hole.faces), dtype=torch.bool)
        for cluster in clusters:
            if len(cluster) >= max(threshold, min_cluster_size):
                keep_mask[cluster] = True
                
        mesh_with_hole.update_faces(keep_mask)
        mesh_with_hole.remove_unreferenced_vertices()
        mesh_with_hole.remove_degenerate_triangles()
        
    # Area-based filtering
    global_area = mesh_with_hole.area
    threshold_area = 0.2 * global_area
    components = mesh_with_hole.get_connected_components()
    keep_components = [c for c in components if c.area > threshold_area]
    
    if keep_components:
        final_mesh = merge_components(keep_components)
    else:
        final_mesh = mesh_with_hole
        
    return final_mesh
```

## Subtopics

### Mesh Post-Processing (Cluster Filtering)
A variant that focuses on removing small, disconnected noise clusters after the initial projection-based culling. It identifies connected triangle clusters, sorts them by size, and removes those smaller than the N-th largest cluster or a minimum triangle count (default 50 triangles, keep top 1000 clusters) [S1, unofficial]. This is useful for cleaning up floating artifacts that pass the depth test but are geometrically insignificant.

## Operating characteristics
*   **Time Complexity:** $O(N_{verts} \cdot N_{cams} + N_{faces})$, where $N_{verts}$ is the number of vertices, $N_{cams}$ is the number of cameras, and $N_{faces}$ is the number of faces. The depth sampling step is $O(N_{verts} \cdot N_{cams})$ due to bilinear interpolation.
*   **Memory Complexity:** $O(N_{verts} \cdot N_{cams})$ for storing intermediate projection masks if not batched, or $O(N_{verts} + N_{faces})$ with batching.
*   **Performance:** Depends heavily on the number of cameras and resolution. High-resolution depth maps increase sampling cost.
*   **Strengths:** Effective for unbounded scenes where voxel-based methods fail. Robust against self-occlusion artifacts by using multiple views.
*   **Limitations:** Requires pre-rendered depth maps, which can be memory-intensive for large meshes. The "seen" threshold (20 cameras) is arbitrary and may need tuning for sparse views.
*   **When to use:** When extracting meshes from unbounded Gaussian splatting or other implicit representations where traditional voxel culling is not applicable.
*   **When not to use:** For bounded voxel grids where neighbor checks are sufficient and faster.

## Background
This method originates from the "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" paper by Huang et al. (SIGGRAPH 2024) [S1, unofficial]. It was developed to address the challenge of extracting clean meshes from unbounded 2D Gaussian splatting reconstructions, which often contain floating artifacts and noise. The approach leverages the multi-view consistency of the splatting representation to filter out geometry that is not consistently visible across multiple camera poses.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
