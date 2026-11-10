# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method culls mesh geometry by projecting vertices into multiple camera views and comparing their depths against reference depth maps to identify and remove invisible or erroneous faces [S1, unofficial]. It is primarily used to clean up artifacts from unbounded Gaussian splatting reconstructions where standard adjacency culling is insufficient due to the lack of a strict voxel grid [S1, unofficial]. Unlike [Face Culling](8735-face-culling.md) which relies on local voxel occupancy, this method uses global visibility information from multiple viewpoints to distinguish between "seen" (visible), "unseen" (occluded/background), and "forecast" (distant/uncertain) geometry [S1, unofficial].

## How it works
The method operates in two main phases: visibility classification via multi-view projection and mesh filtering via connected components.

### Inputs and Outputs
*   **Input:** A triangular mesh $\mathcal{M} = (V, F)$ where $V$ is the set of vertices and $F$ is the set of faces; a list of camera poses $C = \{c2w_i\}$ and intrinsic parameters $K_i = \{f_x, f_y, c_x, c_y, H, W\}$; and configuration parameters (see below).
*   **Output:** A filtered mesh $\mathcal{M}'$ with removed faces and unreferenced vertices.

### Data Structures
*   **Depth Map:** A 2D tensor of shape $(H, W)$ storing the reference depth for each camera.
*   **Visibility Masks:**
    *   `valid`: Boolean mask of shape $(|V|)$ indicating if a vertex is "seen" (visible in $\ge 20$ cameras).
    *   `valid_forecast`: Boolean mask of shape $(|V|)$ indicating if a vertex is within the "forecast" radius.
    *   `face_mask`: Boolean mask of shape $(|F|)$ indicating if all vertices of a face are valid.

### Parameters
*   `points_batch_size`: Maximum number of points processed per batch to limit GPU memory. Default: $5 \times 10^5$ [S1, unofficial].
*   `remove_small_geometry_threshold`: Minimum face area ratio to keep a mesh component. Default: $0.2$ [S1, unofficial].
*   `eps`: Depth comparison tolerance. Default: $0.005$ [S1, unofficial].
*   `forecast_radius`: Radius $r$ for expanding the frustum bounds for "forecast" classification. Default: Not explicitly given, but used in logic [S1, unofficial].
*   `far`: Far plane distance for depth extraction. Default: $20.0$ [S1, unofficial].

### Steps

1.  **Extract Reference Depth Maps:**
    For each camera pose $c2w_i$, render a forward depth map $D_i$ of the current mesh using a renderer (e.g., PyRender) with flags `OFFSCREEN`, `DEPTH_ONLY`, and `SKIP_CULL_FACES` [S1, unofficial].

2.  **Initialize Masks:**
    Initialize `valid`, `valid_num`, and `valid_forecast` as zero tensors of shape $(|V|)$ on the target device [S1, unofficial].

3.  **Project Vertices and Classify Visibility:**
    For each camera $i$ in $C$:
    a.  **Coordinate Transformation:** Convert the camera-to-world matrix $c2w_i$ to world-to-camera $w2c_i$. Note: Negate the y and z rows of the rotation matrix in $c2w_i$ before inversion if the coordinate system requires it [S1, unofficial].
    $$ w2c_i = c2w_i^{-1} $$
    b.  **Homogeneous Projection:** Append ones to vertices $V$ to form homogeneous coordinates $V_h$. Project to camera space:
    $$ P_{cam} = w2c_i \cdot V_h $$
    c.  **Intrinsic Projection:** Apply intrinsic matrix $K_i$ to get UV coordinates:
    $$ uv = K_i \cdot P_{cam}[0:3] $$
    d.  **Depth and Normalization:** Extract depth $z = uv[:, 2] + 10^{-8}$ and normalize UV:
    $$ u = uv[:, 0] / z, \quad v = uv[:, 1] / z $$
    e.  **Frustum Bounds:** Check if points are within the standard frustum:
    $$ \text{in\_frustum} = (u \ge 0) \land (u \le W-1) \land (v \ge 0) \land (v \le H-1) \land (z > 0) $$
    Check if points are within the expanded "forecast" frustum with radius $r$:
    $$ \text{forecast\_frustum} = (u \ge -r) \land (u \le W-1+r) \land (v \ge -r) \land (v \le H-1+r) \land (z > 0) $$
    f.  **Depth Sampling:** Reshape reference depth $D_i$ to $(1, 1, H, W)$. Normalize UV grid to $[-1, 1]$ for bilinear interpolation. Sample depth at $(u, v)$:
    $$ d_{sample} = \text{grid\_sample}(D_i, (u, v), \text{padding\_mode}=\text{'border'}) $$
    g.  **Front-Face Validation:** A point is considered "seen" if it is in the frustum and the projected depth $z$ is closer than the sampled reference depth:
    $$ \text{is\_front\_face} = (d_{sample} > 0.0) \land (z < d_{sample} + \epsilon) $$
    h.  **Update Masks:**
    *   If `is_front_face` is true, increment `valid_num` and set `valid` to true for that vertex.
    *   If `forecast_frustum` is true, set `valid_forecast` to true for that vertex.

4.  **Finalize Vertex Masks:**
    A vertex is "valid" if it is seen in at least 20 cameras:
    $$ \text{valid} = \text{valid\_num} \ge 20 $$
    Combine `valid` and `valid_forecast` if forecast culling is enabled [S1, unofficial].

5.  **Face Masking:**
    Create a face mask by checking if all vertices of each face are valid:
    $$ \text{face\_mask} = \text{valid}[\mathcal{M}.faces].\text{all}(\text{axis}=1) $$
    Update the mesh with this mask and remove unreferenced vertices [S1, unofficial].

6.  **Connected Components Filtering (Optional):**
    If `forecast_radius > 0`:
    a.  Create a "forecast mesh" from faces where vertices are in `valid_forecast` but not `valid`.
    b.  Compute the Oriented Bounding Box (OBB) of the cull mesh vertices.
    c.  Keep only forecast faces within this OBB.
    d.  Filter mesh components by area: Keep components where area $> \text{remove\_small\_geometry\_threshold} \times \text{global\_area}$, or keep only the largest component if `get_largest_components` is True [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def cull_mesh_by_projection(mesh, cameras, params):
    """
    Cull mesh geometry based on multi-view depth projection.
    
    Args:
        mesh: Mesh object with vertices (V, 3) and faces (F, 3).
        cameras: List of dicts with keys 'c2w' (4,4), 'fx', 'fy', 'cx', 'cy', 'H', 'W'.
        params: Dict with 'eps', 'forecast_radius', 'remove_small_geometry_threshold', 'batch_size'.
        
    Returns:
        Filtered mesh.
    """
    V = mesh.vertices  # (N, 3)
    F_idx = mesh.faces # (M, 3)
    N = V.shape[0]
    
    # Initialize masks
    valid = torch.zeros(N, dtype=torch.bool, device=V.device)
    valid_num = torch.zeros(N, dtype=torch.int, device=V.device)
    valid_forecast = torch.zeros(N, dtype=torch.bool, device=V.device)
    
    eps = params.get('eps', 0.005)
    r = params.get('forecast_radius', 0.0) # Default 0 if not specified
    batch_size = params.get('points_batch_size', 500000)
    
    # Process vertices in batches to limit memory
    for start in range(0, N, batch_size):
        end = min(start + batch_size, N)
        V_batch = V[start:end]
        
        for cam in cameras:
            c2w = cam['c2w']
            fx, fy = cam['fx'], cam['fy']
            cx, cy = cam['cx'], cam['cy']
            H, W = cam['H'], cam['W']
            
            # Coordinate transformation: negate y and z rows of rotation part
            c2w_adj = c2w.clone()
            c2w_adj[:3, 1] *= -1
            c2w_adj[:3, 2] *= -1
            
            w2c = torch.inverse(c2w_adj)
            
            # Homogeneous coordinates
            V_h = torch.cat([V_batch, torch.ones(V_batch.shape[0], 1, device=V_batch.device)], dim=1)
            
            # Project to camera space
            P_cam = torch.matmul(w2c, V_h.T).T[:, :3] # (B, 3)
            
            # Intrinsic projection
            u = P_cam[:, 0] * fx / P_cam[:, 2] + cx
            v = P_cam[:, 1] * fy / P_cam[:, 2] + cy
            z = P_cam[:, 2] + 1e-8
            
            # Frustum checks
            in_frustum = (u >= 0) & (u <= W - 1) & (v >= 0) & (v <= H - 1) & (z > 0)
            forecast_frustum = (u >= -r) & (u <= W - 1 + r) & (v >= -r) & (v <= H - 1 + r) & (z > 0)
            
            # Sample reference depth
            # Assuming reference depth maps are pre-rendered and stored in a list/dict
            # depth_map shape: (H, W)
            depth_map = get_reference_depth(cam) # Helper to fetch pre-rendered depth
            
            # Normalize UV to [-1, 1] for grid_sample
            u_norm = 2.0 * u / (W - 1) - 1.0
            v_norm = 2.0 * v / (H - 1) - 1.0
            grid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, B, 2)
            
            # Reshape depth for grid_sample: (1, 1, H, W)
            depth_4d = depth_map.unsqueeze(0).unsqueeze(0)
            
            # Bilinear interpolation with border padding
            depth_sample = F.grid_sample(depth_4d, grid, align_corners=True, padding_mode='border')
            depth_sample = depth_sample.squeeze().squeeze() # (B,)
            
            # Front-face validation
            is_front = (depth_sample > 0.0) & (z < depth_sample + eps)
            
            # Update masks
            valid_mask = in_frustum & is_front
            valid_num[start:end] += valid_mask.int()
            valid[start:end] = valid[start:end] | valid_mask
            
            valid_forecast[start:end] = valid_forecast[start:end] | forecast_frustum
            
    # Finalize valid mask: seen in >= 20 cameras
    final_valid = valid_num >= 20
    
    # Combine with forecast if needed (logic depends on specific implementation details)
    # For this summary, we assume final_valid is the primary culling mask
    # If forecast culling is active, valid_forecast might be used to keep distant geometry
    
    # Face masking
    face_valid = final_valid[F_idx].all(dim=1)
    
    # Update mesh
    mesh.update_faces(face_valid)
    mesh.remove_unreferenced_vertices()
    
    # Connected components filtering (simplified)
    # This step typically requires a graph traversal library or custom implementation
    # to find connected components and filter by area
    if params.get('forecast_radius', 0) > 0:
        mesh = filter_connected_components(mesh, params)
        
    return mesh

def filter_connected_components(mesh, params):
    """
    Filter mesh components by area or size.
    """
    # Placeholder for actual connected components logic
    # In practice, this involves building an adjacency graph from faces/vertices
    # and computing the area of each component.
    pass
```

## Subtopics

### [Mesh Post-Processing (Cluster Filtering)](7002-mesh-post-processing-cluster-filtering.md)
Filters out floating artifacts and disconnected components by clustering triangles and retaining only the largest connected clusters. It serves as a cleanup step following this method, specifically targeting small, noisy geometry that survives projection-based culling. Unlike this method which removes occluded or distant geometry, this method removes topologically isolated fragments based on triangle count.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot C)$ where $N$ is the number of vertices and $C$ is the number of cameras. The depth sampling step is $O(1)$ per vertex per camera using bilinear interpolation.
*   **Memory Complexity:** $O(N + C \cdot H \cdot W)$ for storing the depth maps and masks.
*   **Performance:** The method is efficient on GPU due to batched tensor operations. The default batch size of $5 \times 10^5$ points helps manage memory.
*   **Strengths:** Effective at removing background noise and artifacts from unbounded reconstructions. Robust to partial occlusions by requiring visibility in multiple views.
*   **Limitations:** Requires pre-rendered depth maps for all camera poses. Sensitive to the number of cameras (requires $\ge 20$ for reliable "seen" classification). May remove valid geometry if the camera coverage is insufficient or if the depth maps are inaccurate.
*   **When to use:** When dealing with unbounded Gaussian splatting meshes or scenes where voxel-based culling is not applicable.
*   **When not to use:** When camera coverage is sparse or when real-time performance is critical and pre-rendering depth maps is not feasible.

## Background
This method was introduced in the context of "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" (2DGS) by Huang et al. (SIGGRAPH 2024) [S1, unofficial]. It addresses the challenge of extracting clean meshes from 2D Gaussian splats, which do not have an inherent voxel structure. The method leverages the multi-view consistency of the splats to distinguish between visible surface geometry and background noise.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
