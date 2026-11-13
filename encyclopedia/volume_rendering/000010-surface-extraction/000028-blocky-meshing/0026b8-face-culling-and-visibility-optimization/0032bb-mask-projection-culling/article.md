# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes vertices and faces not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing their depth against rendered depth maps or binary masks [S1, S2, unofficial]. It differs from neighbor-based culling methods (like [Face Culling](8735-face-culling.md)) by relying on external camera views rather than local voxel topology, making it suitable for cleaning meshes derived from multi-view reconstruction or NeRF-based methods where global visibility is known [S1, S2, unofficial].

## How it works
The method filters a mesh by validating each vertex against a set of camera views. A vertex is retained only if it is visible (in front of the surface) and within the valid region (mask) of a sufficient number of views.

### Inputs and Outputs
- **Input**:
  - Mesh: Vertices $V \in \mathbb{R}^{N \times 3}$, Faces $F \in \mathbb{R}^{M \times 3}$.
  - Cameras: A set of $V_{cam}$ camera poses. Each pose includes a world-to-camera transformation matrix and intrinsic parameters.
  - Masks/Depth: For each camera, a binary mask image $M_i$ (or depth map $D_i$) of resolution $W \times H$.
- **Output**:
  - Filtered Mesh: Vertices $V'$ and Faces $F'$ where all vertices of a face are valid.

### Data Structures
- **Visibility Counter**: An array `valid_counts` of size $N$ (number of vertices), initialized to 0. Stores the number of views in which each vertex was deemed visible.
- **Camera Parameters**:
  - $P_i$: Projection matrix for view $i$, computed as $P_i = \text{world\_mat}_i \times \text{scale\_mat}_i$ [S1, unofficial].
  - $K_i$: Intrinsic matrix extracted from $P_i$ [S1, unofficial].
  - $W_{2C,i}$: World-to-camera transformation matrix, derived from $P_i$ using `rend_util.load_K_Rt_from_P` [S1, unofficial] or explicitly as the inverse of the camera-to-world matrix [S2, unofficial].

### Steps
1. **Initialize**: Set `valid_counts` to zeros. Define batch size $B$ (e.g., $5 \times 10^5$) to limit GPU memory [S2, unofficial].
2. **Iterate Views**: For each camera view $i$ from $0$ to $V_{cam}-1$:
   a. **Load View Data**: Load mask $M_i$ (or depth $D_i$) and camera matrices [S1, S2, unofficial].
   b. **Process Batches**: Split vertices into batches of size $B$. For each batch:
      i. **Transform to Camera Space**:
         Compute $W_{2C,i}$ [S1, S2, unofficial].
         Transform vertices $v \in V_{batch}$ to camera space:
         $$ v_{cam} = W_{2C,i} \cdot v $$
         [S1, S2, unofficial]
      ii. **Project to 2D**:
         Compute homogeneous coordinates:
         $$ p_{hom} = K_i \cdot v_{cam} $$
         [S1, S2, unofficial]
         Perform perspective division to get pixel coordinates $(u, v)$:
         $$ z = p_{hom, z} + \epsilon $$
         $$ u = \frac{p_{hom, x}}{z}, \quad v = \frac{p_{hom, y}}{z} $$
         [S1, S2, unofficial]
         where $\epsilon$ is a small value (e.g., $10^{-6}$ or $10^{-8}$) to prevent division by zero [S1, S2, unofficial].
      iii. **Frustum Check**:
         Determine if points are within the image bounds:
         $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z > 0) $$
         [S2, unofficial]
         Alternatively, normalize to NDC space $[-1, 1]$:
         $$ u_{ndc} = \frac{u}{W-1} \times 2 - 1 $$
         $$ v_{ndc} = \frac{v}{H-1} \times 2 - 1 $$
         $$ \text{valid\_ndc} = (-1 < u_{ndc} < 1) \land (-1 < v_{ndc} < 1) $$
         [S1, S2, unofficial]
      iv. **Visibility Test**:
         - **Mask-Based (S1)**:
           Dilate the binary mask $M_i$ using a disk of radius 24 [S1, unofficial].
           Sample the dilated mask at $(u, v)$ using nearest-neighbor interpolation with zero padding:
           $$ \text{sampled\_mask} = \text{grid\_sample}(M_{dilated}, (u_{ndc}, v_{ndc}), \text{mode='nearest'}, \text{padding\_mode='zeros'}) $$
           [S1, unofficial]
           A point is valid in this view if $\text{sampled\_mask} > 0$ [S1, unofficial].
         - **Depth-Based (S2)**:
           Sample the depth map $D_i$ at $(u, v)$ using bilinear interpolation with border padding:
           $$ d_{sampled} = \text{grid\_sample}(D_i, (u_{ndc}, v_{ndc}), \text{mode='bilinear'}, \text{padding\_mode='border'}) $$
           [S2, unofficial]
           Compare the point's camera-space depth $z$ against the sampled depth:
           $$ \text{is\_front} = (d_{sampled} > 0) \land (z < d_{sampled} + \text{eps}) $$
           [S2, unofficial]
           where $\text{eps} = 0.005$ [S2, unofficial].
      v. **Update Counts**:
         For points satisfying both frustum and visibility tests, increment `valid_counts` for the corresponding global vertex indices.
3. **Filter Mesh**:
   - Define a threshold $T$ (e.g., 20 views) [S2, unofficial].
   - Identify valid vertices: $V_{valid} = \{ v_j \mid \text{valid\_counts}[j] \ge T \}$ [S2, unofficial].
   - Identify valid faces: A face $f_k = (v_a, v_b, v_c)$ is valid if all three vertices are in $V_{valid}$ [S1, unofficial].
   - Reconstruct the mesh using only valid vertices and faces.
4. **Post-Processing (Optional)**:
   - Transform vertices back to world space if they were stored in camera space (though typically vertices are kept in world space and only projected for testing) [S1, unofficial].
   - Remove small connected components based on area ratio threshold (e.g., 0.2) [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mask_projection_culling(mesh, cameras, masks, depth_maps, threshold=20, eps=0.005, batch_size=500000):
    """
    mesh: dict with 'vertices' (N,3) and 'faces' (M,3)
    cameras: list of dicts with 'K' (3,3), 'W2C' (4,4), 'scale_mat' (4,4)
    masks: list of binary images (H, W)
    depth_maps: list of depth images (H, W)
    """
    N = mesh['vertices'].shape[0]
    valid_counts = np.zeros(N, dtype=np.int32)
    
    # Precompute NDC scaling factors
    W, H = masks[0].shape[1], masks[0].shape[0]
    ndc_scale_x = 2.0 / (W - 1)
    ndc_scale_y = 2.0 / (H - 1)
    
    for i, cam in enumerate(cameras):
        K = cam['K']
        W2C = cam['W2C']
        
        # Load mask and depth for this view
        mask = masks[i]
        depth = depth_maps[i]
        
        # Dilate mask (S1)
        # Note: Implementation of binary_dilation depends on library (e.g., scipy)
        mask_dilated = binary_dilation(mask, structure=disk(24))
        
        # Process vertices in batches
        for start in range(0, N, batch_size):
            end = min(start + batch_size, N)
            batch_vertices = mesh['vertices'][start:end]
            batch_indices = np.arange(start, end)
            
            # Transform to camera space
            # Homogeneous coordinates: (N, 4)
            batch_homo = np.hstack([batch_vertices, np.ones((end - start, 1))])
            cam_vertices = (W2C @ batch_homo.T).T[:, :3]
            
            # Project to 2D
            cam_points_homo = (K @ cam_vertices.T).T
            z = cam_points_homo[:, 2] + 1e-8
            
            u = cam_points_homo[:, 0] / z
            v = cam_points_homo[:, 1] / z
            
            # Check frustum bounds (S2)
            in_frustum = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z > 0)
            
            # Normalize to NDC for grid sampling (S1, S2)
            u_ndc = u * ndc_scale_x - 1.0
            v_ndc = v * ndc_scale_y - 1.0
            
            # Prepare grid for sampling
            # grid_sample expects (N, 2) in NDC [-1, 1]
            grid = np.stack([u_ndc, v_ndc], axis=-1)
            
            # Sample Mask (S1)
            # Using nearest neighbor, zero padding
            sampled_mask = grid_sample_nearest_zero_padded(mask_dilated, grid)
            mask_valid = sampled_mask > 0
            
            # Sample Depth (S2)
            # Using bilinear, border padding
            sampled_depth = grid_sample_bilinear_border_padded(depth, grid)
            depth_valid = (sampled_depth > 0) & (z < sampled_depth + eps)
            
            # Combine validity
            # Note: S1 uses mask only, S2 uses depth. 
            # Assuming a combined approach or choosing one based on config.
            # Here we assume both must be valid if both are provided, 
            # or follow the specific variant. 
            # Based on S1 and S2 notes, they are distinct implementations.
            # We'll assume the mask-based check is the primary "Mask Projection" definition,
            # but depth is used for "in front of surface" check in S2.
            # Let's assume S2's logic is the robust one: in frustum AND in mask AND in front of depth.
            
            view_valid = in_frustum & mask_valid & depth_valid
            
            # Update counts
            valid_counts[batch_indices[view_valid]] += 1
            
    # Filter mesh
    valid_vertices_mask = valid_counts >= threshold
    valid_vertex_indices = np.where(valid_vertices_mask)[0]
    
    # Create mapping from old index to new index
    index_map = np.full(N, -1, dtype=np.int32)
    index_map[valid_vertex_indices] = np.arange(len(valid_vertex_indices))
    
    # Filter faces
    old_faces = mesh['faces']
    new_vertex_indices = index_map[old_faces]
    valid_faces_mask = np.all(new_vertex_indices != -1, axis=1)
    new_faces = new_vertex_indices[valid_faces_mask]
    
    # Filter vertices
    new_vertices = mesh['vertices'][valid_vertices_mask]
    
    return {'vertices': new_vertices, 'faces': new_faces}
```

### Parameters
- `threshold`: Minimum number of views a vertex must be visible in to be retained. Typical value: 20 [S2, unofficial].
- `eps`: Depth comparison tolerance. Typical value: 0.005 [S2, unofficial].
- `batch_size`: Max points per batch. Typical value: $5 \times 10^5$ [S2, unofficial].
- `mask_dilation_radius`: Radius for morphological dilation. Typical value: 24 [S1, unofficial].

### Pitfalls
- **Coordinate System Mismatch**: OpenCV and OpenGL conventions differ in Y and Z axes. Ensure camera poses are flipped appropriately (`c2w[:3, 1:3] *= -1`) if using standard OpenGL depth buffers [S2, unofficial].
- **Out-of-Bounds Sampling**: UV coordinates outside $[0, W] \times [0, H]$ can cause NaNs. Use `padding_mode='border'` for depth and `padding_mode='zeros'` for masks [S1, S2, unofficial].
- **Precision Errors**: Projection and division by zero must be handled with small epsilon values [S1, S2, unofficial].

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5) [S3, unofficial]. Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. It uses the existing vertex pool bucketing, avoiding the 6-fold driver overhead increase associated with splitting chunks into 6 sub-meshes [S3, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size factor (related to grid sampling cost) [S2, unofficial].
- **Memory Complexity**: $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
- **Strengths**: Effective for removing noise and artifacts from multi-view reconstructions; leverages existing camera data; robust to local topology ambiguities.
- **Limitations**: Requires pre-existing camera poses and masks/depth maps; computationally expensive for large meshes due to per-vertex projection across multiple views; sensitive to camera calibration errors.
- **When to Use**: Post-processing meshes from NeRF, 3D Gaussian Splatting, or multi-view stereo where global visibility is known.
- **When Not to Use**: Real-time rendering where per-frame culling is needed (use [Voxel Backface Culling](4569-voxel-backface-culling.md) or [True Back-Face Culling](#true-back-face-culling-via-group-masking) instead); scenes without known camera poses.

## Background
Mask Projection Culling is commonly used in the context of 3D Gaussian Splatting and NeRF-based mesh extraction pipelines to clean up noisy geometry [S1, S2, unofficial]. It was popularized by implementations in libraries such as `2d-gaussian-splatting` and `svraster-win10` for refining meshes extracted from implicit or point-based representations [S1, S2, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
