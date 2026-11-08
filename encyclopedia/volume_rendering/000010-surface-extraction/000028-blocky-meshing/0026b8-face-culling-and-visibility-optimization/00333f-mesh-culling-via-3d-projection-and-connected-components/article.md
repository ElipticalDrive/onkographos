# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method culls mesh geometry by projecting vertices into multiple camera views and comparing their depths against reference depth maps to identify and remove unseen or forecast geometry [S1, unofficial]. It sets itself apart from local adjacency-based culling (like [Face Culling](8735-face-culling.md)) by using global visibility information from arbitrary camera poses, making it suitable for unbounded reconstructions where local neighbor checks fail to detect distant occlusions or artifacts [S1, unofficial].

## How it works
The method classifies 3D points as "seen" (visible in multiple views), "unseen" (occluded or background), or "forecast" (potentially visible but requiring bounding box constraints) [S1, unofficial]. It operates on a mesh extracted from a radiance field (e.g., Gaussian Splatting) [S1, unofficial].

### Inputs and Outputs
- **Input**: A triangle mesh (vertices $V$, faces $F$), a list of camera poses (world-to-camera matrices $W_i$), camera intrinsics ($K_i$), and reference depth maps $D_i$ for each camera pose.
- **Output**: A culled mesh with faces and vertices removed based on visibility masks.

### Data Structures
- **Vertices**: Array of shape $(N, 3)$, where $N$ is the number of vertices.
- **Faces**: Array of shape $(M, 3)$, where $M$ is the number of triangles, storing vertex indices.
- **Depth Maps**: List of 2D arrays of shape $(H, W)$, representing the forward depth from the mesh for each camera.
- **Masks**: Boolean tensors of shape $(N)$ for vertices and $(M)$ for faces.

### Parameters
- `points_batch_size`: Maximum points processed per batch to limit GPU memory. Default: $5 \times 10^5$ [S1, unofficial].
- `remove_small_geometry_threshold`: Minimum face area ratio to keep a component. Default: $0.2$ [S1, unofficial].
- `eps`: Depth comparison tolerance. Default: $0.005$ [S1, unofficial].
- `forecast_radius`: Radius for expanding frustum bounds for forecast points. Default: Not explicitly given, but used in logic [S1, unofficial].
- `cluster_to_keep`: Max number of largest connected clusters to retain (in post-processing). Default: $1000$ [S1, unofficial].
- `min_cluster_size`: Minimum triangles per cluster to keep (in post-processing). Default: $50$ [S1, unofficial].

### Steps

1. **Extract Reference Depth Maps**: Render forward depth maps from the initial mesh using each camera pose. This involves setting up a renderer with `OFFSCREEN`, `DEPTH_ONLY`, and `SKIP_CULL_FACES` flags [S1, unofficial].

2. **Project Vertices to Camera Space**: For each camera $i$:
   - Convert world-to-camera matrix $W_i$. Note: Coordinate system adjustment may require negating y and z rows of the rotation matrix if converting from OpenGL to other conventions [S1, unofficial].
   - Project vertices $V$ to homogeneous camera coordinates: $P_{cam} = W_i \cdot [V | 1]^T$.
   - Apply intrinsic matrix $K_i$ to get UV coordinates: $uv = K_i \cdot P_{cam}$.
   - Extract depth $z = uv[:, 2]$ and normalize UV: $u = uv[:, 0]/z, v = uv[:, 1]/z$ [S1, unofficial].

3. **Frustum and Visibility Check**:
   - Check if point is in frustum: $0 \le u < W, 0 \le v < H, z > 0$ [S1, unofficial].
   - For "forecast" points, use expanded bounds: $-r \le u < W+r, -r \le v < H+r$ [S1, unofficial].
   - Sample the reference depth map $D_i$ at $(u, v)$ using bilinear interpolation with border padding [S1, unofficial].
   - Validate front-facing geometry: A point is "seen" if $z < D_i(u, v) + \epsilon$ [S1, unofficial].

4. **Aggregate Masks**:
   - Initialize `valid` (seen), `valid_num` (count), and `valid_forecast` (forecast) masks for all vertices [S1, unofficial].
   - For each camera, update masks. A vertex is marked as "seen" if it is front-facing in that view.
   - Finalize "seen" mask: A vertex is considered truly visible if it is seen in at least 20 cameras (`valid_num >= 20`) [S1, unofficial].

5. **Face Masking**:
   - Create a face mask: A face is kept if **all** its vertices are in the "seen" mask (`face_mask = mask[faces].all(axis=1)`) [S1, unofficial].
   - Update mesh with `face_mask` and remove unreferenced vertices [S1, unofficial].

6. **Forecast Culling (Optional)**:
   - If `forecast_radius > 0`, create a forecast mesh from vertices marked as "forecast" but not "seen".
   - Compute the Oriented Bounding Box (OBB) of the cull mesh's vertices.
   - Keep only forecast faces within this OBB [S1, unofficial].

7. **Connected Component Filtering**:
   - Identify connected triangle clusters using a graph traversal algorithm (e.g., Open3D's `cluster_connected_triangles`) [S1, unofficial].
   - Determine threshold: Sort clusters by size, select the N-th largest (where N=`cluster_to_keep`), and enforce `min_cluster_size` [S1, unofficial].
   - Remove triangles in clusters smaller than the threshold.
   - Remove unreferenced vertices and degenerate triangles [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cull_mesh_by_3d_projection(mesh, cameras, depth_maps, params):
    """
    Cull mesh geometry based on multi-view depth projection.
    
    Args:
        mesh: Triangle mesh object with vertices (N,3) and faces (M,3).
        cameras: List of camera objects with intrinsic K and extrinsic W2C.
        depth_maps: List of 2D arrays (H,W) representing reference depth for each camera.
        params: Dictionary of parameters (eps, batch_size, etc.).
    
    Returns:
        Culled mesh.
    """
    vertices = mesh.vertices
    faces = mesh.faces
    num_vertices = len(vertices)
    
    # Initialize masks
    valid = torch.zeros(num_vertices, dtype=torch.bool)
    valid_num = torch.zeros(num_vertices, dtype=torch.int)
    valid_forecast = torch.zeros(num_vertices, dtype=torch.bool)
    
    # Batch processing for memory efficiency
    batch_size = params.get('points_batch_size', 5e5)
    num_batches = (num_vertices + batch_size - 1) // batch_size
    
    for b in range(num_batches):
        start_idx = b * batch_size
        end_idx = min((b + 1) * batch_size, num_vertices)
        batch_vertices = vertices[start_idx:end_idx]
        
        # Initialize batch masks
        batch_valid = torch.zeros(len(batch_vertices), dtype=torch.bool)
        batch_valid_num = torch.zeros(len(batch_vertices), dtype=torch.int)
        batch_valid_forecast = torch.zeros(len(batch_vertices), dtype=torch.bool)
        
        for i, camera in enumerate(cameras):
            # 1. Project vertices to camera space
            # Convert world-to-camera matrix (handle coordinate system if needed)
            w2c = camera.w2c_matrix
            # Negate y and z rows if necessary for coordinate system match
            # w2c[:3, 1:3] *= -1  [G]
            
            # Homogeneous coordinates
            ones = torch.ones(len(batch_vertices), 1, device=vertices.device)
            homo_verts = torch.cat([batch_vertices, ones], dim=1)
            cam_coords = (w2c @ homo_verts.T).T
            
            # Extract depth and UV
            z = cam_coords[:, 2] + 1e-8
            uv = cam_coords[:, :2] / z.unsqueeze(1)
            u, v = uv[:, 0], uv[:, 1]
            
            # 2. Frustum Check
            H, W = depth_maps[i].shape
            in_frustum = (u >= 0) & (u <= W-1) & (v >= 0) & (v <= H-1) & (z > 0)
            
            # Forecast frustum (expanded)
            r = params.get('forecast_radius', 0)
            forecast_frustum = (u >= -r) & (u <= W-1+r) & (v >= -r) & (v <= H-1+r) & (z > 0)
            
            # 3. Depth Sampling and Validation
            # Normalize UV to [-1, 1] for grid sampling
            vgrid = torch.stack([u, v], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, B, 2)
            # Reshape depth map to (1, 1, H, W)
            depth_map = depth_maps[i].unsqueeze(0).unsqueeze(0).to(vertices.device)
            
            # Sample depth with bilinear interpolation
            depth_sample = F.grid_sample(depth_map, vgrid, padding_mode='border', align_corners=True)
            depth_sample = depth_sample.squeeze() # (B,)
            
            # Front-face validation
            is_front = (depth_sample > 0.0) & (z < depth_sample + params['eps'])
            
            # Update masks
            batch_valid_num[in_frustum] += 1
            batch_valid_forecast[forecast_frustum] = True
            batch_valid[in_frustum & is_front] = True
            
        # Aggregate batch results
        valid[start_idx:end_idx] = batch_valid
        valid_num[start_idx:end_idx] = batch_valid_num
        valid_forecast[start_idx:end_idx] = batch_valid_forecast
        
    # Finalize seen mask: requires visibility in at least 20 cameras
    seen_mask = valid_num >= 20
    
    # Create face mask: all vertices must be seen
    face_mask = seen_mask[faces].all(dim=1)
    
    # Update mesh
    mesh_with_hole = mesh.copy()
    mesh_with_hole.update_faces(face_mask)
    mesh_with_hole.remove_unreferenced_vertices()
    
    # Optional: Forecast Culling
    if params.get('forecast_radius', 0) > 0:
        forecast_mask = valid_forecast & ~seen_mask
        forecast_faces = forecast_mask[faces].all(dim=1)
        # Compute OBB of cull mesh vertices [G]
        # Keep only forecast faces within OBB [S1, unofficial]
        # ... implementation of OBB check ...
        
    # Connected Component Filtering
    clusters = mesh_with_hole.cluster_connected_triangles()
    # Sort clusters by size
    cluster_sizes = [len(c) for c in clusters]
    sorted_indices = np.argsort(cluster_sizes)[::-1]
    
    # Determine threshold
    n_clusters_to_keep = params.get('cluster_to_keep', 1000)
    min_cluster_size = params.get('min_cluster_size', 50)
    
    if len(sorted_indices) > n_clusters_to_keep:
        threshold = cluster_sizes[sorted_indices[n_clusters_to_keep]]
    else:
        threshold = 0
        
    # Remove small clusters
    keep_faces = torch.zeros(len(mesh_with_hole.faces), dtype=torch.bool)
    for idx in sorted_indices:
        if cluster_sizes[idx] < min_cluster_size:
            continue
        if len(sorted_indices) > n_clusters_to_keep and idx > sorted_indices[n_clusters_to_keep-1]:
            if cluster_sizes[idx] < threshold:
                continue
        # Mark faces in this cluster as kept
        cluster_face_indices = clusters[idx]
        keep_faces[cluster_face_indices] = True
        
    mesh_with_hole.remove_triangles_by_mask(~keep_faces)
    mesh_with_hole.remove_unreferenced_vertices()
    mesh_with_hole.remove_degenerate_triangles()
    
    return mesh_with_hole
```

## Subtopics

### Mesh Post-Processing (Cluster Filtering)
A variant that focuses on removing small, disconnected geometric artifacts after the main projection culling. It identifies connected triangle clusters and removes those smaller than a threshold determined by the N-th largest cluster or a minimum size count. It is chosen when the projection culling leaves behind noise or floating debris that is not easily filtered by area alone. It relies on graph connectivity of the mesh faces [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Dominated by the projection step, which is $O(N \cdot C)$ where $N$ is the number of vertices and $C$ is the number of cameras. Depth sampling is $O(1)$ per vertex per camera. Connected component analysis is roughly $O(M)$ where $M$ is the number of faces.
- **Memory Complexity**: $O(N \cdot C)$ for storing intermediate projection results if not batched, or $O(N)$ with batching. Depth maps are $O(H \cdot W \cdot C)$.
- **Measured Performance**: The method is used in the context of 2D Gaussian Splatting for geometrically accurate radiance fields, achieving improved Chamfer distance on DTU and TnT datasets compared to baseline methods [S1, unofficial].
- **Strengths**: Effective for unbounded scenes and removing artifacts from unbounded reconstructions. Global visibility check avoids local occlusion errors.
- **Limitations**: Requires multiple camera poses and reference depth maps. Computationally expensive for large meshes and many cameras. Sensitive to the number of cameras (threshold of 20) [S1, unofficial].
- **When to use**: When extracting meshes from radiance fields (e.g., Gaussian Splatting) where local culling is insufficient due to unbounded nature or noise.
- **When not to use**: For simple voxel-based blocky meshes with clear local adjacency, where [Face Culling](8735-face-culling.md) is faster and sufficient.

## Background
This method originates from the "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" paper by Huang et al. (SIGGRAPH 2024) [S1, unofficial]. It was developed to address the challenge of extracting clean, bounded meshes from unbounded 3D Gaussian Splatting reconstructions, which often contain artifacts and noise in the background. The method leverages the multi-view consistency of the original Gaussian Splatting training data to filter out unseen geometry.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
