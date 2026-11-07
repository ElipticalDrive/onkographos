# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method culls mesh geometry by comparing projected vertex depths against reference depth maps rendered from multiple camera poses, classifying faces as seen, unseen, or forecast to remove artifacts from unbounded reconstructions [S1, unofficial]. It sets itself apart from sibling methods like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by using explicit multi-view depth consistency checks rather than static coverage masks, and from [Block Visibility Culling](5334-block-visibility-culling.md) by operating on the full mesh topology rather than spatial blocks.

## How it works
**Inputs**: A triangle mesh $M$, a list of camera poses $C = \{c_2w_i\}$, intrinsic parameters $K = \{f_x, f_y, c_x, c_y\}$, image resolution $(H, W)$, and optional parameters `forecast_radius`, `remove_small_geometry_threshold`, `get_largest_components`, `cluster_to_keep`, `min_cluster_size`, and `points_batch_size`.
**Outputs**: A culled mesh with unreferenced vertices removed.

**Data Structures**:
- `mesh`: Triangle mesh with vertices $V \in \mathbb{R}^{N_v \times 3}$ and faces $F \in \mathbb{Z}^{N_f \times 3}$.
- `depth_list`: List of $N_c$ depth maps, each of shape $(H, W)$.
- `valid`, `valid_num`, `valid_forecast`: Boolean/intensity tensors of shape $(N_v)$ tracking vertex visibility.

**Steps**:

1.  **Extract Reference Depth Maps**: For each camera pose $c_2w_i$, render a forward depth map $D_i$ of the mesh using a rasterizer (e.g., PyRender with `DEPTH_ONLY` and `SKIP_CULL_FACES` flags) [S1, unofficial].

2.  **Initialize Masks**: Initialize `valid` (seen), `valid_num` (count), and `valid_forecast` (forecast) as zero tensors of shape $(N_v)$ on the target device [S1, unofficial].

3.  **Process Vertices in Batches**: For each batch of vertices (size `points_batch_size`, default $5 \times 10^5$) [S1, unofficial]:
    a.  **Coordinate Transformation**: For each camera $i$:
        -   Negate the $y$ and $z$ rows of the rotation matrix in $c_2w_i$: $c_2w_i[:3, 1:3] *= -1$ [S1, unofficial].
        -   Compute world-to-camera matrix: $W2C_i = c_2w_i^{-1}$ [S1, unofficial].
        -   Convert vertices to homogeneous coordinates: $P_h = [V_{batch}; 1]$ [S1, unofficial].
        -   Project to camera space: $P_c = W2C_i \cdot P_h$. Extract first three rows: $P_c = [x_c, y_c, z_c]^T$ [S1, unofficial].
        -   Apply intrinsic matrix $K$: $uv = K \cdot P_c$. Let $uv = [u_{raw}, v_{raw}, z]$.
        -   Compute normalized UV: $u = u_{raw} / z$, $v = v_{raw} / z$ [S1, unofficial].
        -   Add epsilon to depth: $z \leftarrow z + 10^{-8}$ [S1, unofficial].
    b.  **Frustum and Depth Validation**:
        -   Check standard frustum bounds: $in\_frustum = (0 \le u < W) \land (0 \le v < H) \land (z > 0)$ [S1, unofficial].
        -   Check forecast frustum with radius $r$: $forecast\_frustum = (-r \le u < W+r) \land (-r \le v < H+r) \land (z > 0)$ [S1, unofficial].
        -   Sample reference depth $D_i$ at $(u, v)$ using bilinear interpolation with border padding: $d_{sample} = \text{grid\_sample}(D_i, (u, v))$ [S1, unofficial].
        -   Validate front-facing geometry: $is\_front = (d_{sample} > 0) \land (z < d_{sample} + \epsilon)$, where $\epsilon = 0.005$ [S1, unofficial].
    c.  **Update Masks**:
        -   If $in\_frustum \land is\_front$: `valid` = True, `valid_num` += 1 [S1, unofficial].
        -   If $forecast\_frustum \land is\_front$: `valid_forecast` = True [S1, unofficial].

4.  **Finalize Vertex Masks**:
    -   A vertex is "seen" if `valid_num` $\ge 20$ (requires projection into at least 20 cameras) [S1, unofficial].
    -   A vertex is "forecast" if `valid_forecast` is True [S1, unofficial].

5.  **Create Face Mask**:
    -   A face is kept if all its vertices are "seen": $face\_mask = valid[faces].all(axis=1)$ [S1, unofficial].
    -   Update mesh: $M_{cull} = M.update\_faces(face\_mask)$ [S1, unofficial].

6.  **Optional Connected Component Culling (Forecast)**:
    -   If `forecast_radius` > 0:
        -   Create a forecast mesh from faces where vertices are "forecast" [S1, unofficial].
        -   Compute Oriented Bounding Box (OBB) of the cull mesh vertices [S1, unofficial].
        -   Keep only forecast faces within this OBB [S1, unofficial].

7.  **Post-Processing (Cluster Filtering)**:
    -   Identify connected triangle clusters using `cluster_connected_triangles()` [S1, unofficial].
    -   Sort clusters by size. Select threshold as the $N$-th largest cluster size, where $N = \text{cluster\_to\_keep}$ (default 1000) [S1, unofficial].
    -   Enforce minimum size: keep clusters with triangle count $\ge \text{min\_cluster\_size}$ (default 50) [S1, unofficial].
    -   Remove triangles from clusters below threshold using `remove_triangles_by_mask` [S1, unofficial].
    -   Remove unreferenced vertices and degenerate triangles [S1, unofficial].

8.  **Area-Based Filtering (Alternative/Additional)**:
    -   If `get_largest_components` is True, keep only the largest connected component [S1, unofficial].
    -   Otherwise, keep components where area $> \text{remove\_small\_geometry\_threshold} \times \text{global\_area}$ (default 0.2) [S1, unofficial].

**Pseudocode**:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def cull_mesh(mesh, cameras, intrinsics, H, W, forecast_radius=0.0, 
              remove_small_geometry_threshold=0.2, get_largest_components=False,
              cluster_to_keep=1000, min_cluster_size=50, points_batch_size=500000, eps=0.005):
    """
    Cull mesh geometry using multi-view depth projection and connected components.
    
    Args:
        mesh: Triangle mesh object with .vertices (N_v, 3) and .faces (N_f, 3).
        cameras: List of c2w matrices (4, 4).
        intrinsics: Tuple (fx, fy, cx, cy).
        H, W: Image height and width.
        forecast_radius: Radius for forecast frustum.
        remove_small_geometry_threshold: Min face area ratio.
        get_largest_components: If True, keep only largest component.
        cluster_to_keep: Max number of largest clusters to retain.
        min_cluster_size: Min triangles per cluster.
        points_batch_size: Batch size for vertex processing.
        eps: Depth comparison tolerance.
        
    Returns:
        Culled mesh.
    """
    N_v = mesh.vertices.shape[0]
    valid = torch.zeros(N_v, dtype=torch.bool, device=mesh.vertices.device)
    valid_num = torch.zeros(N_v, dtype=torch.int, device=mesh.vertices.device)
    valid_forecast = torch.zeros(N_v, dtype=torch.bool, device=mesh.vertices.device)
    
    fx, fy, cx, cy = intrinsics
    
    # Process vertices in batches
    for start in range(0, N_v, points_batch_size):
        end = min(start + points_batch_size, N_v)
        batch_verts = mesh.vertices[start:end]
        
        # Initialize batch masks
        batch_valid = torch.zeros(end - start, dtype=torch.bool, device=mesh.vertices.device)
        batch_valid_num = torch.zeros(end - start, dtype=torch.int, device=mesh.vertices.device)
        batch_valid_forecast = torch.zeros(end - start, dtype=torch.bool, device=mesh.vertices.device)
        
        for c2w in cameras:
            # Coordinate transformation
            c2w_t = c2w.clone()
            c2w_t[:3, 1:3] *= -1  # Negate y and z rows
            w2c = torch.inverse(c2w_t)
            
            # Project to camera space
            ones = torch.ones(end - start, 1, device=mesh.vertices.device)
            P_h = torch.cat([batch_verts, ones], dim=1)  # (B, 4)
            P_c = torch.matmul(w2c, P_h.t())  # (4, B)
            x_c, y_c, z_c = P_c[0], P_c[1], P_c[2]
            
            # Apply intrinsics
            u_raw = fx * x_c / z_c + cx
            v_raw = fy * y_c / z_c + cy
            z = z_c + 1e-8
            
            # Normalize UV
            u = u_raw / z
            v = v_raw / z
            
            # Frustum checks
            in_frustum = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z > 0)
            forecast_frustum = (u >= -forecast_radius) & (u < W + forecast_radius) & \
                               (v >= -forecast_radius) & (v < H + forecast_radius) & (z > 0)
            
            # Sample depth (pseudo-code for grid_sample)
            # In practice, use F.grid_sample with normalized UV grid
            # depth_sample = F.grid_sample(depth_map.unsqueeze(0).unsqueeze(0), 
            #                              torch.stack([u, v], dim=-1).unsqueeze(0), 
            #                              padding_mode='border', align_corners=True)
            # For pseudocode, assume depth_sample is available
            depth_sample = sample_depth(depth_map, u, v) 
            
            # Front-face validation
            is_front = (depth_sample > 0.0) & (z < depth_sample + eps)
            
            # Update masks
            seen = in_frustum & is_front
            forecast = forecast_frustum & is_front
            
            batch_valid |= seen
            batch_valid_num += seen.int()
            batch_valid_forecast |= forecast
        
        # Update global masks
        valid[start:end] |= batch_valid
        valid_num[start:end] += batch_valid_num
        valid_forecast[start:end] |= batch_valid_forecast
    
    # Finalize vertex masks
    seen_mask = valid_num >= 20
    
    # Create face mask
    face_mask = seen_mask[mesh.faces].all(dim=1)
    mesh.update_faces(face_mask)
    mesh.remove_unreferenced_vertices()
    
    # Optional connected component culling
    if forecast_radius > 0:
        forecast_mask = valid_forecast[mesh.faces].all(dim=1)
        forecast_mesh = mesh.copy()
        forecast_mesh.update_faces(forecast_mask)
        # Compute OBB of cull mesh vertices
        obb = compute_obb(mesh.vertices)
        # Keep forecast faces within OBB
        # ... (implementation specific)
        
    # Post-processing: Cluster Filtering
    clusters = cluster_connected_triangles(mesh)
    cluster_sizes = [len(c) for c in clusters]
    if len(cluster_sizes) > cluster_to_keep:
        threshold = sorted(cluster_sizes, reverse=True)[cluster_to_keep]
    else:
        threshold = 0
        
    keep_mask = torch.zeros(mesh.faces.shape[0], dtype=torch.bool)
    for i, cluster in enumerate(clusters):
        if len(cluster) >= min_cluster_size and len(cluster) >= threshold:
            keep_mask[cluster] = True
            
    mesh.remove_triangles_by_mask(~keep_mask)
    mesh.remove_unreferenced_vertices()
    mesh.remove_degenerate_triangles()
    
    # Area-based filtering
    if get_largest_components:
        components = get_connected_components(mesh)
        largest = max(components, key=len)
        mesh.update_faces(torch.tensor(largest, dtype=torch.bool))
    else:
        global_area = mesh.area
        for comp in get_connected_components(mesh):
            comp_area = mesh.component_area(comp)
            if comp_area < remove_small_geometry_threshold * global_area:
                # Remove this component
                pass
                
    return mesh
```

## Subtopics

### Mesh Post-Processing (Cluster Filtering)
A variant that focuses on removing small disconnected artifacts after the main projection culling. It identifies connected triangle clusters, sorts them by size, and removes clusters smaller than the $N$-th largest (where $N$ is `cluster_to_keep`) or smaller than `min_cluster_size`. This is useful for cleaning up noise in unbounded reconstructions [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N_v \cdot N_c \cdot B)$, where $N_v$ is the number of vertices, $N_c$ is the number of cameras, and $B$ is the batch size. Depth sampling is $O(1)$ per vertex per camera. Cluster filtering is $O(N_f \log N_f)$ or $O(N_f)$ depending on implementation.
- **Memory Complexity**: $O(N_v + N_f + N_c \cdot H \cdot W)$ for storing depth maps and masks.
- **Measured Performance**: On DTU dataset, this method achieves a mean Chamfer distance of 0.47 (reproduce) vs 0.80 (paper) [S1, unofficial].
- **Strengths**: Effective at removing unbounded artifacts and noise. Handles arbitrary mesh sizes.
- **Limitations**: Requires multiple camera poses. Sensitive to depth map accuracy. Batch processing needed for large meshes.
- **When to use**: When extracting meshes from unbounded Gaussian splatting or similar reconstructions where background artifacts are prevalent.
- **When not to use**: When only a single view is available or when real-time performance is critical and $N_c$ is large.

## Background
Originated in the context of 2D Gaussian Splatting for geometrically accurate radiance fields, specifically for unbounded mesh extraction [S1, unofficial]. The method was developed to address artifacts in meshes extracted from unbounded TSDF fusion with space contraction and adaptive truncation [S1, unofficial]. It is used in the 2D Gaussian Splatting repository [S1, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
