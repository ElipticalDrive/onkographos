# Mesh Culling via 3D Projection and Connected Components

Also called: Mesh Culling via Multi-View Depth Projection, cull with 3d projection

## Overview
This method removes geometric artifacts and occluded surfaces from a 3D mesh by projecting its vertices into multiple camera views and comparing rendered depths against reference depth maps [S1, unofficial]. It distinguishes itself from topological culling by using explicit visibility testing against known camera poses, making it effective for cleaning meshes derived from unbounded reconstructions like Gaussian Splatting where neighbor checks are insufficient [S1, unofficial]. It serves as a visibility optimization step that precedes connected component filtering [S1, unofficial].

## How it works
The method classifies mesh faces as "seen" (visible in multiple views), "unseen" (occluded or background), or "forecast" (potentially visible but requiring bounding box validation) [S1, unofficial]. It operates on a mesh $M$ with vertices $V$ and faces $F$, given a set of camera poses $C = \{c_i\}$ and intrinsic parameters $K$.

### Inputs and Outputs
**Inputs:**
- Mesh $M = (V, F)$, where $V \in \mathbb{R}^{N_v \times 3}$ and $F \in \mathbb{R}^{N_f \times 3}$ (triangle indices).
- Camera poses $c_i \in \mathbb{R}^{4 \times 4}$ (world-to-camera or camera-to-world, normalized as per implementation).
- Intrinsic matrix $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$.
- Image dimensions $H, W$.
- Reference depth maps $D_i \in \mathbb{R}^{H \times W}$ for each camera $i$.
- Parameters: `points_batch_size` (default $5 \times 10^5$), `eps` (default $0.005$), `remove_small_geometry_threshold` (default $0.2$), `forecast_radius` (default $0$).

**Outputs:**
- Culled mesh $M'$ with reduced face count and unreferenced vertices removed.

### Data Structures
- `valid`: Boolean tensor of shape $(N_v)$, true if vertex is seen in $\ge 20$ cameras.
- `valid_forecast`: Boolean tensor of shape $(N_v)$, true if vertex is in forecast frustum and within OBB.
- `face_mask`: Boolean tensor of shape $(N_f)$, true if all three vertices of the face are valid.
- `depth_sample`: Tensor of shape $(N_v)$ or batched, containing interpolated depth from reference maps.

### Steps

1. **Initialize Masks**: Create `valid`, `valid_num`, and `valid_forecast` tensors of zeros with length $N_v$ on the target device.

2. **Iterate Cameras**: For each camera pose $c_i$ in $C$:
   a. **Coordinate Transformation**: If $c_i$ is camera-to-world, convert to world-to-camera by negating the y and z rows of the rotation matrix:
      $$ R_{w2c} = \begin{bmatrix} r_{00} & r_{01} & r_{02} \\ -r_{10} & -r_{11} & -r_{12} \\ -r_{20} & -r_{21} & -r_{22} \end{bmatrix} $$
      $$ t_{w2c} = -R_{w2c} \cdot c_{world} $$
      Compute $W2C = \begin{bmatrix} R_{w2c} & t_{w2c} \\ 0 & 1 \end{bmatrix}$.
   b. **Project Vertices**: For a batch of vertices $V_{batch}$:
      - Append ones: $V_h = [V_{batch}, \mathbf{1}]^T$.
      - Transform to camera space: $P_c = W2C \cdot V_h$.
      - Extract first three rows: $P_{cam} = [x, y, z]^T$.
   c. **Intrinsic Projection**:
      - $u = f_x \frac{x}{z} + c_x$
      - $v = f_y \frac{y}{z} + c_y$
      - $z_{proj} = z + 10^{-8}$
   d. **Frustum Check**:
      - Standard frustum: $u \in [0, W-1]$, $v \in [0, H-1]$, $z > 0$.
      - Forecast frustum (if `forecast_radius` $r > 0$): $u \in [-r, W-1+r]$, $v \in [-r, H-1+r]$, $z > 0$.
   e. **Depth Sampling**:
      - Normalize UV to $[-1, 1]$: $u_{norm} = \frac{2u}{W-1} - 1$, $v_{norm} = \frac{2v}{H-1} - 1$.
      - Sample reference depth $D_i$ using bilinear interpolation with border padding:
        $$ d_{sample} = \text{grid\_sample}(D_i, [u_{norm}, v_{norm}], \text{padding\_mode='border'}) $$
   f. **Visibility Validation**:
      - Front-face check: $d_{sample} > 0$ AND $z < d_{sample} + \text{eps}$.
      - If valid, increment `valid_num` for these vertices.
      - If in forecast frustum, mark `valid_forecast`.

3. **Finalize Vertex Masks**:
   - `valid = valid_num >= 20` (requires projection into at least 20 cameras).
   - Combine with forecast if enabled: `final_valid = valid | valid_forecast`.

4. **Face Culling**:
   - Create `face_mask`: For each face $f \in F$, `face_mask[f] = final_valid[v_0] & final_valid[v_1] & final_valid[v_2]`.
   - Apply mask: $M' = M[\text{face\_mask}]$.
   - Remove unreferenced vertices from $M'$.

5. **Connected Component Filtering (Optional)**:
   - If `forecast_radius > 0` or cleanup is required:
     - Compute Oriented Bounding Box (OBB) of the cull mesh vertices.
     - Keep only forecast faces within this OBB.
     - Filter components by area: Keep components where $\text{area} > \text{remove\_small\_geometry\_threshold} \times \text{global\_area}$, or keep only the largest component if `get_largest_components` is True.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def cull_mesh_by_projection(mesh, cameras, intrinsics, depth_maps, 
                            H, W, eps=0.005, batch_size=500000, 
                            forecast_radius=0, remove_threshold=0.2):
    """
    Cull mesh faces based on multi-view depth projection.
    
    Args:
        mesh: Triangle mesh with vertices V (N, 3) and faces F (M, 3).
        cameras: List of camera poses (4, 4) in world-to-camera or camera-to-world.
        intrinsics: (3, 3) intrinsic matrix K.
        depth_maps: List of depth maps (H, W) for each camera.
        H, W: Image height and width.
        eps: Depth comparison tolerance.
        batch_size: Max vertices processed per batch.
        forecast_radius: Radius for extended frustum check.
        remove_threshold: Minimum area ratio for component retention.
        
    Returns:
        Culled mesh.
    """
    V = mesh.vertices  # (N, 3)
    F = mesh.faces     # (M, 3)
    N = V.shape[0]
    
    # Initialize masks
    valid = torch.zeros(N, dtype=torch.bool, device=V.device)
    valid_num = torch.zeros(N, dtype=torch.int, device=V.device)
    valid_forecast = torch.zeros(N, dtype=torch.bool, device=V.device)
    
    # Process each camera
    for i, (c2w, depth_map) in enumerate(zip(cameras, depth_maps)):
        # Convert c2w to w2c if necessary (negate y, z rows of rotation)
        # Assuming c2w is provided as camera-to-world, convert to world-to-camera
        R_c2w = c2w[:3, :3]
        t_c2w = c2w[:3, 3]
        
        # Negate y and z rows for w2c rotation
        R_w2c = R_c2w.clone()
        R_w2c[1, :] *= -1
        R_w2c[2, :] *= -1
        
        # Compute translation: t_w2c = -R_w2c * t_c2w
        t_w2c = -torch.matmul(R_w2c, t_c2w)
        
        # Construct W2C matrix
        W2C = torch.eye(4, device=V.device)
        W2C[:3, :3] = R_w2c
        W2C[:3, 3] = t_w2c
        
        # Project vertices in batches
        for start in range(0, N, batch_size):
            end = min(start + batch_size, N)
            V_batch = V[start:end]
            
            # Homogeneous coordinates
            V_h = torch.cat([V_batch, torch.ones(end - start, 1, device=V.device)], dim=1)
            
            # World to camera
            P_cam = torch.matmul(W2C, V_h.T).T  # (batch, 4)
            x, y, z = P_cam[:, 0], P_cam[:, 1], P_cam[:, 2]
            
            # Intrinsic projection
            u = intrinsics[0, 0] * (x / z) + intrinsics[0, 2]
            v = intrinsics[1, 1] * (y / z) + intrinsics[1, 2]
            z_proj = z + 1e-8
            
            # Frustum checks
            in_frustum = (u >= 0) & (u <= W - 1) & (v >= 0) & (v <= H - 1) & (z > 0)
            
            in_forecast_frustum = False
            if forecast_radius > 0:
                in_forecast_frustum = (u >= -forecast_radius) & (u <= W - 1 + forecast_radius) & \
                                      (v >= -forecast_radius) & (v <= H - 1 + forecast_radius) & (z > 0)
            
            # Depth sampling
            # Normalize UV to [-1, 1]
            u_norm = 2.0 * u / (W - 1) - 1.0
            v_norm = 2.0 * v / (H - 1) - 1.0
            
            # Reshape for grid_sample: (1, 1, H, W)
            depth_tensor = depth_map.unsqueeze(0).unsqueeze(0)
            grid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0)
            
            # Sample depth with border padding
            depth_sample = F.grid_sample(depth_tensor, grid, align_corners=True, padding_mode='border')
            depth_sample = depth_sample.squeeze()  # (batch,)
            
            # Front-face validation
            is_front = (depth_sample > 0.0) & (z_proj < depth_sample + eps)
            
            # Update masks
            valid_mask = in_frustum & is_front
            valid_num[start:end] += valid_mask.int()
            
            if forecast_radius > 0:
                valid_forecast[start:end] |= (in_forecast_frustum & is_front)
    
    # Finalize vertex validity
    valid = valid_num >= 20
    final_valid = valid | valid_forecast
    
    # Face culling
    face_mask = final_valid[F].all(dim=1)
    
    # Create new mesh with culled faces
    new_mesh = mesh.copy()
    new_mesh.faces = F[face_mask]
    new_mesh.remove_unreferenced_vertices()
    
    # Connected component filtering (simplified)
    if forecast_radius > 0 or remove_threshold > 0:
        # Compute OBB of cull mesh vertices
        # Keep only forecast faces within OBB
        # Filter by area
        pass
        
    return new_mesh
```

### Parameters
- `points_batch_size`: Maximum number of vertices processed per batch to limit GPU memory. Default: $5 \times 10^5$ [S1, unofficial].
- `eps`: Depth comparison tolerance for front-face validation. Default: $0.005$ [S1, unofficial].
- `remove_small_geometry_threshold`: Minimum face area ratio to keep a mesh component. Default: $0.2$ [S1, unofficial].
- `forecast_radius`: Radius for extended frustum check. Default: $0$ [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Depth comparison uses `eps` to handle floating-point inaccuracies. Small `eps` may reject valid faces due to rasterization artifacts [S1, unofficial].
- **Boundaries**: UV coordinates outside $[0, W-1]$ are handled by border padding in `grid_sample`, preventing NaNs [S1, unofficial].
- **Degenerate Input**: Vertices with $z \le 0$ are excluded. Faces with all vertices invalid are removed [S1, unofficial].
- **Memory**: Large meshes require batching to avoid OOM. `points_batch_size` should be tuned based on GPU memory [S1, unofficial].

## Subtopics

### [Mesh Post-Processing (Cluster Filtering)](7002-mesh-post-processing-cluster-filtering.md)
Filters floating artifacts and disconnected components by clustering triangles and retaining only the largest connected clusters. Used after this method to remove small, noisy geometry that survives projection-based culling.

## Operating characteristics
- **Time Complexity**: $O(N_v \cdot N_c \cdot B)$, where $N_v$ is vertex count, $N_c$ is camera count, and $B$ is batch size. Depth sampling is $O(1)$ per vertex per camera.
- **Memory Complexity**: $O(N_v + N_f + N_c \cdot H \cdot W)$ for storing masks and depth maps.
- **Performance**: Effective for unbounded reconstructions where topological culling fails. Requires pre-computed depth maps and camera poses [S1, unofficial].
- **Strengths**: Removes occluded geometry and background artifacts. Robust to non-manifold meshes.
- **Limitations**: Requires accurate camera poses and depth maps. Computationally expensive for large $N_c$. Does not fix topological errors [S1, unofficial].
- **When to Use**: Cleaning meshes from Gaussian Splatting or unbounded TSDF fusion. When topological culling is insufficient [S1, unofficial].
- **When Not to Use**: Real-time rendering where pre-processing is not feasible. When camera poses are inaccurate [S1, unofficial].

## Background
Originated in the context of 2D Gaussian Splatting for geometrically accurate radiance fields, specifically for unbounded mesh extraction [S1, unofficial]. The method was developed to address artifacts in meshes derived from Gaussian Splatting, where traditional voxel-based culling is not applicable [S1, unofficial]. It is used in the official implementation of 2D Gaussian Splatting for geometry reconstruction on datasets like DTU and TnT [S1, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
