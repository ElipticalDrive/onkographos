# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps or binary occupancy masks [S1, S2, unofficial]. It solves the problem of retaining only the geometry visible from a specific set of viewpoints, effectively discarding occluded or background noise. It differs from [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) by relying on explicit camera projections and depth/mask sampling rather than local voxel neighbor checks, and from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by utilizing multiple views to establish consensus visibility rather than a single view's coverage.

## How it works
The method filters a mesh by projecting its vertices into the coordinate systems of multiple validation cameras, checking if they fall within valid image bounds and in front of the scene surface (or within a dilated mask), and retaining only those vertices and faces that satisfy these conditions across a minimum number of views.

### Inputs and Outputs
*   **Input:** A mesh defined by vertices $V \in \mathbb{R}^{N \times 3}$ and faces $F \in \mathbb{R}^{M \times 3}$ (indices into $V$); a set of $V$ camera poses (world-to-camera or camera-to-world matrices) and intrinsic matrices; optional binary masks for each view; image dimensions $W, H$.
*   **Output:** A filtered mesh with reduced vertices and faces, retaining only those visible in at least $K$ views (typically $K=20$ in source implementations, though the general logic is "at least 2 views" or "all views" depending on configuration) [S1, S2, unofficial].

### Data Structures
*   **Camera Parameters:**
    *   `world_mat`: $4 \times 4$ transformation matrix from world to camera (or vice versa, depending on convention).
    *   `scale_mat`: $4 \times 4$ matrix containing scaling factors, often applied to normalize scene coordinates.
    *   `intrinsic`: $3 \times 3$ camera intrinsic matrix $K$.
    *   Combined projection matrix $P = \text{world\_mat} \times \text{scale\_mat}$ [S1, unofficial].
*   **Visibility Count:** An array `valid_counts` of size $N$ (number of vertices), initialized to zero, storing how many views each vertex is visible in [S2, unofficial].
*   **Depth Maps / Masks:** Rendered depth maps $D_i \in \mathbb{R}^{H \times W}$ and binary masks $M_i \in \{0, 1\}^{H \times W}$ for each view $i$ [S1, S2, unofficial].

### Steps

1.  **Load and Prepare Camera Data:**
    *   Extract `scale_mats` and `world_mats` from camera data (e.g., `cameras.npz`) [S1, unofficial].
    *   Compute the projection matrix $P = \text{world\_mat} \times \text{scale\_mat}$ [S1, unofficial].
    *   Decompose $P$ to obtain intrinsic matrix $K$ and pose matrix (camera-to-world or world-to-camera) using a utility function (e.g., `rend_util.load_K_Rt_from_P`) [S1, unofficial].
    *   Handle coordinate system mismatches (e.g., OpenCV vs. OpenGL) by flipping Y and Z axes of the camera pose if necessary: $c2w[:3, 1:3] *= -1$ [S2, unofficial].

2.  **Render Reference Data (Optional but Recommended for Depth):**
    *   Render forward depth maps from the input mesh for each estimated camera pose using an offscreen renderer with flags `OFFSCREEN | DEPTH_ONLY | SKIP_CULL_FACES` [S2, unofficial].
    *   Load pre-computed binary masks from files (e.g., `mask_dir/*.png`) with fixed dimensions $W=1600, H=1200$ (or as specified) [S1, unofficial].

3.  **Initialize Visibility Tracking:**
    *   Initialize `valid_counts` array of size $N$ to zeros [S2, unofficial].
    *   Define batch size `points_batch_size` (e.g., $5 \times 10^5$) to limit GPU memory usage [S2, unofficial].

4.  **Iterate Over Views and Vertices:**
    *   For each view $i$ from $0$ to $V-1$:
        *   Get camera pose $c2w_i$ (camera-to-world) and invert it to get $w2c_i = c2w_i^{-1}$ [S2, unofficial].
        *   Process vertices in batches of size `points_batch_size`:
            *   Let $V_{batch}$ be the current batch of vertices.
            *   Transform vertices to camera space: $P_{cam} = w2c_i \times V_{batch}$ (assuming homogeneous coordinates, typically $4 \times N$) [S1, S2, unofficial].
            *   Project to 2D using intrinsic matrix $K$:
                $$ uv_{hom} = K \times P_{cam} $$
                $$ z = uv_{hom}[2, :] + \epsilon $$ (where $\epsilon = 10^{-6}$ or $10^{-8}$) [S1, S2, unofficial].
                $$ uv = \frac{uv_{hom}[0:2, :]}{z} $$
            *   Normalize UV coordinates to NDC space $[-1, 1]$ for grid sampling:
                $$ uv_{ndc}[0] = \frac{uv[0]}{W - 1} \times 2.0 - 1.0 $$
                $$ uv_{ndc}[1] = \frac{uv[1]}{H - 1} \times 2.0 - 1.0 $$
            *   Check if points are within the image frustum bounds:
                $$ \text{in\_frustum} = (0 \le uv[0] < W) \land (0 \le uv[1] < H) \land (z > 0) $$ [S2, unofficial].
                Alternatively, using NDC:
                $$ \text{valid} = (-1 < uv_{ndc}[0] < 1) \land (-1 < uv_{ndc}[1] < 1) $$ [S1, unofficial].

5.  **Sample Visibility Masks/Depth:**
    *   **Mask-based Culling (S1):**
        *   Dilate the binary mask $M_i$ using morphology (e.g., `binary_dilation` with a disk of radius 24) to account for projection errors and aliasing [S1, unofficial].
        *   Sample the dilated mask at $uv_{ndc}$ using `F.grid_sample` with `mode='nearest'`, `padding_mode='zeros'`, `align_corners=True` [S1, unofficial].
        *   A vertex is considered "in mask" if the sampled value $> 0$ [S1, unofficial].
    *   **Depth-based Culling (S2):**
        *   Sample the depth map $D_i$ at $uv_{ndc}$ using `F.grid_sample` with `mode='bilinear'`, `padding_mode='border'`, `align_corners=True` [S2, unofficial].
        *   Compare the vertex's camera-space depth $z$ with the sampled depth $d_{sample}$:
            $$ \text{is\_front} = (d_{sample} > 0) \land (z < d_{sample} + \text{eps}) $$
            where $\text{eps} = 0.005$ [S2, unofficial].
        *   If $d_{sample} \le 0$, the point is not in front of any surface (or out of bounds), so it is not visible [S2, unofficial].

6.  **Update Visibility Counts:**
    *   For each vertex in the batch, if it is valid (in frustum, in mask, and in front of surface), increment `valid_counts` for that vertex index [S2, unofficial].

7.  **Filter Mesh:**
    *   After processing all views, determine valid vertices:
        $$ \text{valid\_vertices} = \text{valid\_counts} \ge K $$
        where $K$ is the minimum number of views required (e.g., 20 in S2, or "all" in some configurations) [S2, unofficial].
    *   Filter faces: A face is valid if all its three vertices are valid:
        $$ \text{valid\_faces} = \text{valid\_vertices}[F[:, 0]] \land \text{valid\_vertices}[F[:, 1]] \land \text{valid\_vertices}[F[:, 2]] $$ [S1, unofficial].
    *   Remove invalid vertices and faces from the mesh.
    *   Transform remaining vertices back to world space using the inverse of the scale/pose transformations (e.g., multiply by `scale_mat`) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(mesh, cameras, masks=None, depth_maps=None, 
                            min_views=20, eps=0.005, batch_size=500000):
    """
    Filter mesh vertices and faces based on visibility across multiple camera views.
    
    Args:
        mesh: Object with .vertices (N,3) and .faces (M,3)
        cameras: List of dicts with 'c2w' (4,4), 'K' (3,3), 'W', 'H'
        masks: List of binary masks (H, W) or None
        depth_maps: List of depth maps (H, W) or None
        min_views: Minimum number of views a vertex must be visible in
        eps: Depth comparison tolerance
        batch_size: Number of vertices to process per batch
        
    Returns:
        Filtered mesh with .vertices and .faces
    """
    N = mesh.vertices.shape[0]
    valid_counts = np.zeros(N, dtype=np.int32)
    
    # Pre-process cameras: ensure c2w is camera-to-world
    # If input is world-to-camera, invert it. 
    # Also handle coordinate system flips if needed (e.g., OpenGL to OpenCV)
    processed_cameras = []
    for cam in cameras:
        c2w = cam['c2w'].copy()
        # Flip Y and Z if necessary for coordinate system consistency
        c2w[:3, 1:3] *= -1
        processed_cameras.append({
            'c2w': c2w,
            'K': cam['K'],
            'W': cam['W'],
            'H': cam['H']
        })
        
    # Convert mesh vertices to tensor (N, 3) -> (4, N) for homogeneous
    vertices = torch.tensor(mesh.vertices, dtype=torch.float32).T # (3, N)
    vertices_h = torch.cat([vertices, torch.ones(1, N, dtype=torch.float32)], dim=0) # (4, N)
    
    for i, cam in enumerate(processed_cameras):
        c2w = torch.tensor(cam['c2w'], dtype=torch.float32)
        w2c = torch.inverse(c2w)
        K = torch.tensor(cam['K'], dtype=torch.float32)
        W, H = cam['W'], cam['H']
        
        # Get depth map or mask for this view
        if depth_maps is not None:
            depth_map = torch.tensor(depth_maps[i], dtype=torch.float32) # (H, W)
        if masks is not None:
            mask = torch.tensor(masks[i], dtype=torch.float32) # (H, W)
            
        # Process in batches
        for start in range(0, N, batch_size):
            end = min(start + batch_size, N)
            v_batch = vertices_h[:, start:end] # (4, B)
            
            # Transform to camera space
            cam_points = w2c @ v_batch # (4, B)
            
            # Project to 2D
            uv_hom = K @ cam_points[:3, :] # (3, B)
            z = uv_hom[2, :] + 1e-8
            uv = uv_hom[:2, :] / z # (2, B)
            
            # Normalize to NDC [-1, 1] for grid_sample
            uv_ndc = torch.zeros(2, uv.shape[1], dtype=torch.float32)
            uv_ndc[0, :] = (uv[0, :] / (W - 1)) * 2.0 - 1.0
            uv_ndc[1, :] = (uv[1, :] / (H - 1)) * 2.0 - 1.0
            
            # Check frustum bounds (pixel coordinates)
            u_px = uv[0, :]
            v_px = uv[1, :]
            in_frustum = (u_px >= 0) & (u_px < W) & (v_px >= 0) & (v_px < H) & (z > 0)
            
            # Sample visibility
            visible_in_view = torch.zeros(end - start, dtype=torch.bool)
            
            if depth_maps is not None:
                # Reshape uv_ndc for grid_sample: (1, B, 1, 2)
                vgrid = uv_ndc.permute(1, 0).unsqueeze(0).unsqueeze(0) # (1, B, 1, 2)
                # Sample depth
                depth_sample = torch.nn.functional.grid_sample(
                    depth_map.unsqueeze(0).unsqueeze(0), # (1, 1, H, W)
                    vgrid,
                    mode='bilinear',
                    padding_mode='border',
                    align_corners=True
                ).squeeze() # (B,)
                
                # Check if in front of surface
                is_front = (depth_sample > 0) & (z < depth_sample + eps)
                visible_in_view = in_frustum & is_front
                
            elif masks is not None:
                # Dilate mask (simplified: assume pre-dilated or use simple threshold)
                # In practice, use morphology.binary_dilation on numpy array before converting to tensor
                # For this pseudocode, assume mask is already dilated
                vgrid = uv_ndc.permute(1, 0).unsqueeze(0).unsqueeze(0)
                mask_sample = torch.nn.functional.grid_sample(
                    mask.unsqueeze(0).unsqueeze(0),
                    vgrid,
                    mode='nearest',
                    padding_mode='zeros',
                    align_corners=True
                ).squeeze()
                
                visible_in_view = in_frustum & (mask_sample > 0)
                
            valid_counts[start:end] += visible_in_view.int().numpy()
            
    # Filter vertices
    valid_vertex_mask = valid_counts >= min_views
    valid_vertex_indices = np.where(valid_vertex_mask)[0]
    
    # Create mapping from old index to new index
    old_to_new = {old: new for new, old in enumerate(valid_vertex_indices)}
    
    # Filter faces
    valid_faces = []
    for face in mesh.faces:
        if all(old_to_new.get(v, -1) != -1 for v in face):
            new_face = [old_to_new[v] for v in face]
            valid_faces.append(new_face)
            
    new_vertices = mesh.vertices[valid_vertex_mask]
    new_faces = np.array(valid_faces)
    
    return Mesh(vertices=new_vertices, faces=new_faces)
```

### Parameters
*   `eps`: Depth comparison tolerance to account for discretization and projection errors. Typical value: 0.005 [S2, unofficial].
*   `points_batch_size`: Maximum number of points processed per batch to limit GPU memory usage. Typical value: $5 \times 10^5$ [S2, unofficial].
*   `min_views`: Minimum number of views a vertex must be visible in to be retained. Typical value: 20 (in S2), but can be 2 or all views depending on application [S2, unofficial].
*   `mask_dilation_radius`: Radius for dilating binary masks to handle projection inaccuracies. Typical value: 24 pixels [S1, unofficial].

### Edge Cases and Pitfalls
*   **Coordinate System Mismatch:** OpenCV and OpenGL conventions differ in Y and Z axes. Failure to flip axes in the camera pose can lead to incorrect depth comparisons and culling errors. Handled by flipping Y and Z: `c2w[:3, 1:3] *= -1` [S2, unofficial].
*   **Out-of-Bounds UV Coordinates:** Sampling depth or masks at UV coordinates outside the image bounds can cause NaNs or crashes. Handled by setting `padding_mode='border'` in `grid_sample` [S2, unofficial].
*   **Precision Errors:** Small projection errors can cause vertices to fall just outside the mask or behind the depth surface. Mitigated by dilating masks and using a small `eps` in depth comparison [S1, S2, unofficial].
*   **Degenerate Input:** If a mesh has no vertices or faces, or if all cameras are invalid, the output will be an empty mesh. Guard against division by zero in normalization if $W=1$ or $H=1$ [G].

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5). Draw calls are masked based on camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader. It is chosen when driver overhead must be minimized and chunk meshing is used. It differs from standard back-face culling by avoiding the 6-fold increase in driver overhead that would result from splitting chunks into 6 sub-meshes [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \times V \times B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size (effectively $O(N \times V)$ if batched processing is linear) [S2, unofficial].
*   **Memory Complexity:** $O(N \times V)$ for intermediate masks and valid counts, though this can be reduced to $O(N)$ by processing views sequentially and accumulating counts [S2, unofficial].
*   **Strengths:** Effective at removing occluded geometry and background noise; robust to small projection errors via mask dilation; works well with multiple views to establish consensus.
*   **Limitations:** Requires pre-computed depth maps or masks; sensitive to camera pose accuracy; may remove valid geometry if the scene is not well-covered by views; computationally expensive for large meshes and many views.
*   **When to Use:** When a set of reference views is available and the goal is to extract only the visible surface; for cleaning up noisy reconstructions.
*   **When Not to Use:** When real-time culling is required (this is a preprocessing step); when views are sparse or poorly aligned.

## Background
Mask Projection Culling is derived from techniques used in 3D reconstruction and novel view synthesis, particularly in the context of Gaussian Splatting and NeRF-based methods. It is used to filter out geometry that is not consistent with the observed views, improving the quality of the reconstructed mesh. The method is commonly implemented in frameworks like 2D Gaussian Splatting and SVRaster for post-processing reconstructed models [S1, S2, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
