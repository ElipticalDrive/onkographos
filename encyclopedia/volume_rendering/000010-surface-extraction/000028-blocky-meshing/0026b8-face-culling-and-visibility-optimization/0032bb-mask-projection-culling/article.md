# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes vertices and faces not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing their depth against rendered depth maps or binary masks [S1, S2, unofficial]. It differs from neighbor-based culling (e.g., [Face Culling](8735-face-culling.md)) by relying on external camera viewpoints rather than local voxel topology, and from simple back-face culling by using explicit depth testing against known geometry [S2, unofficial]. It is particularly useful for cleaning noisy reconstructions or removing geometry outside a known region of interest [S1, S2, unofficial].

## How it works
**Inputs:**
- A mesh (vertices $V$, faces $F$).
- A set of $V$ camera poses (world-to-camera matrices $W_i$) and intrinsic matrices $K_i$.
- Optional: Binary masks $M_i$ for each view indicating valid regions [S1, unofficial].
- Optional: Pre-rendered depth maps $D_i$ for each view [S2, unofficial].

**Outputs:**
- A filtered mesh containing only vertices and faces visible in at least $T$ views (typically $T=20$) [S2, unofficial].

**Data Structures:**
- `valid_counts`: Array of size $|V|$, initialized to 0. Stores the number of views in which each vertex is visible.
- `face_mask`: Boolean array of size $|F|$. A face is kept if all its vertices are valid.

**Parameters:**
- `eps`: Depth comparison tolerance. Typical value: 0.005 [S2, unofficial].
- `points_batch_size`: Max points processed per batch to limit GPU memory. Typical value: $5 \times 10^5$ [S2, unofficial].
- `mask_dilation_radius`: Radius for morphological dilation of binary masks. Typical value: 24 [S1, unofficial].
- `min_visibility_count`: Minimum number of views a vertex must be visible in. Typical value: 20 [S2, unofficial].

**Steps:**

1. **Preparation:**
   - Load camera parameters. For each view $i$, compute the projection matrix $P_i = W_i \times S_i$ (where $S_i$ is a scale matrix if present) [S1, unofficial].
   - Extract intrinsics $K_i$ and pose. Note coordinate system conventions (e.g., flip Y/Z axes if converting between OpenCV and OpenGL) [S2, unofficial].
   - If using binary masks, load $M_i$ and apply morphological dilation with a disk of radius `mask_dilation_radius` [S1, unofficial].
   - If using depth maps, render them from the mesh using an offscreen renderer with `DEPTH_ONLY` and `SKIP_CULL_FACES` flags [S2, unofficial].

2. **Projection and Visibility Check (per view $i$):**
   - Compute the inverse world-to-camera matrix $C_i = W_i^{-1}$ [S1, S2, unofficial].
   - For each vertex $v \in V$ (processed in batches of size `points_batch_size`):
     a. Transform vertex to camera space: $v_{cam} = C_i \times v_{homogeneous}$ [S1, S2, unofficial].
     b. Project to 2D: $p_{uv} = K_i \times v_{cam}$ [S1, S2, unofficial].
     c. Perform homogeneous division: $u = p_{uv}.x / (p_{uv}.z + \epsilon)$, $v_{coord} = p_{uv}.y / (p_{uv}.z + \epsilon)$, where $\epsilon = 10^{-6}$ [S1, S2, unofficial].
     d. Check frustum bounds:
        - If using pixel coordinates: $in\_frustum = (0 \le u < W) \land (0 \le v_{coord} < H) \land (z > 0)$ [S2, unofficial].
        - If using NDC: Normalize to $[-1, 1]$: $u_{ndc} = (u / (W-1) - 0.5) \times 2$, $v_{ndc} = (v_{coord} / (H-1) - 0.5) \times 2$. Valid if $-1 < u_{ndc} < 1$ and $-1 < v_{ndc} < 1$ [S1, unofficial].
     e. **Depth/Mask Test:**
        - If using **Depth Maps**: Sample depth $d_{sample}$ at $(u, v_{coord})$ using bilinear interpolation with `padding_mode='border'` to handle out-of-bounds [S2, unofficial].
          - Determine if vertex is in front: $is\_front = (z < d_{sample} + \epsilon)$ if $d_{sample} > 0$, else false [S2, unofficial].
        - If using **Binary Masks**: Sample the dilated mask $M_i$ at normalized coordinates using `grid_sample` with `mode='nearest'` and `padding_mode='zeros'` [S1, unofficial].
          - Valid if $sampled\_mask > 0$ [S1, unofficial].
     f. If the vertex passes the frustum and depth/mask test, increment `valid_counts[v]` [S1, S2, unofficial].

3. **Filtering:**
   - Create a vertex validity mask: $valid\_vertices = (valid\_counts \ge min\_visibility\_count)$ [S2, unofficial].
   - Create a face validity mask: $valid\_faces = valid\_vertices[F[:, 0]] \land valid\_vertices[F[:, 1]] \land valid\_vertices[F[:, 2]]$ [S1, unofficial].
   - Retain only vertices where $valid\_vertices$ is true and faces where $valid\_faces$ is true [S1, S2, unofficial].

4. **Post-processing (Optional):**
   - Transform remaining vertices back to world space using the inverse of the scale matrix [S1, unofficial].
   - Remove small disconnected components if their area ratio is below `remove_small_geometry_threshold` (0.2) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(mesh, cameras, masks=None, depth_maps=None, 
                            eps=0.005, min_visibility=20, batch_size=500000):
    """
    mesh: dict with 'vertices' (N, 3) and 'faces' (M, 3)
    cameras: list of dicts with 'w2c' (4, 4), 'K' (3, 3), 'W', 'H'
    masks: list of binary masks (H, W) or None
    depth_maps: list of depth maps (H, W) or None
    """
    num_views = len(cameras)
    num_verts = mesh['vertices'].shape[0]
    valid_counts = np.zeros(num_verts, dtype=np.int32)
    
    # Convert to tensors for GPU processing
    vertices = torch.tensor(mesh['vertices'], dtype=torch.float32, device='cuda')
    vertices_hom = torch.cat([vertices, torch.ones(num_verts, 1, device='cuda')], dim=1)
    
    for i, cam in enumerate(cameras):
        w2c = cam['w2c'].cuda()
        K = cam['K'].cuda()
        H, W = cam['H'], cam['W']
        
        # Handle coordinate system flip if necessary (e.g., OpenCV to OpenGL)
        # w2c[:3, 1:3] *= -1  # [G] Placeholder for specific coordinate adjustment
        
        # Prepare sampling grid
        # Process in batches to manage memory
        for batch_start in range(0, num_verts, batch_size):
            batch_end = min(batch_start + batch_size, num_verts)
            batch_verts = vertices_hom[batch_start:batch_end]
            
            # 1. Transform to camera space
            # v_cam = w2c @ v_hom
            v_cam = torch.matmul(w2c, batch_verts.t()).t() # (B, 4)
            
            # 2. Project to 2D
            # uv = K @ v_cam
            uv = torch.matmul(K, v_cam[:, :3].t()).t() # (B, 3)
            z = uv[:, 2:] + 1e-8
            uv_xy = uv[:, :2] / z # (B, 2)
            
            # 3. Check Frustum and Depth/Mask
            # Normalize to NDC for grid sampling if using masks, or pixel coords for depth
            # Using NDC [-1, 1] for grid_sample compatibility
            u_ndc = (uv_xy[:, 0] / (W - 1)) * 2.0 - 1.0
            v_ndc = (uv_xy[:, 1] / (H - 1)) * 2.0 - 1.0
            
            # Create grid for grid_sample: (1, B, 1, 2)
            grid = torch.stack([u_ndc, v_ndc], dim=-1).unsqueeze(0).unsqueeze(0)
            
            is_visible = torch.zeros(batch_end - batch_start, dtype=torch.bool, device='cuda')
            
            if masks is not None:
                # Use binary mask
                mask_tensor = torch.tensor(masks[i], dtype=torch.float32, device='cuda').unsqueeze(0).unsqueeze(0)
                # Dilate mask beforehand in preprocessing
                sampled = torch.nn.functional.grid_sample(mask_tensor, grid, 
                                                          mode='nearest', 
                                                          padding_mode='zeros', 
                                                          align_corners=True)
                # sampled shape: (1, 1, B, 1)
                valid_mask = sampled.squeeze() > 0.0
                
                # Also check if inside frustum for mask method (NDC bounds)
                in_frustum = (u_ndc > -1) & (u_ndc < 1) & (v_ndc > -1) & (v_ndc < 1) & (z.squeeze() > 0)
                is_visible = valid_mask & in_frustum
                
            elif depth_maps is not None:
                # Use depth map
                depth_tensor = torch.tensor(depth_maps[i], dtype=torch.float32, device='cuda').unsqueeze(0).unsqueeze(0)
                sampled_depth = torch.nn.functional.grid_sample(depth_tensor, grid, 
                                                                mode='bilinear', 
                                                                padding_mode='border', 
                                                                align_corners=True)
                sampled_depth = sampled_depth.squeeze()
                
                # Check if in front of surface
                # z is camera space depth (positive if in front)
                # sampled_depth is also camera space depth
                in_front = (z.squeeze() < sampled_depth + eps)
                in_frustum = (z.squeeze() > 0) & (sampled_depth > 0)
                is_visible = in_front & in_frustum
            
            valid_counts[batch_start:batch_end] += is_visible.int().cpu().numpy()
            
    # Filter vertices
    valid_verts_mask = valid_counts >= min_visibility
    valid_faces_mask = valid_verts_mask[mesh['faces']].all(dim=1)
    
    return {
        'vertices': mesh['vertices'][valid_verts_mask],
        'faces': mesh['faces'][valid_faces_mask]
    }
```

## Subtopics

### True Back-Face Culling via Group Masking
A technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5) and masks draw calls based on camera position relative to the chunk, preventing the draw call from being issued entirely [S3, unofficial]. Unlike standard back-face culling which happens in the fragment shader, this method avoids driver overhead associated with splitting chunks into 6 sub-meshes by reusing existing vertex pool bucketing [S3, unofficial]. It is chosen when rendering performance is critical and the scene is composed of blocky voxel chunks [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \times V \times B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size factor (related to processing overhead) [S2, unofficial].
- **Memory Complexity:** $O(N \times V)$ for intermediate masks and valid counts [S2, unofficial].
- **Performance:** Highly dependent on the number of views and resolution of depth maps/masks. Efficient on GPU due to parallel projection and grid sampling [S1, S2, unofficial].
- **Strengths:** Effective at removing noise and geometry outside the captured region; robust to local topology errors [S1, S2, unofficial].
- **Limitations:** Requires pre-existing camera poses and depth maps/masks; computationally expensive for very large meshes if many views are used [S1, S2, unofficial].
- **When to use:** Post-processing reconstructed meshes from NeRF or Gaussian Splatting; cleaning voxel meshes for specific viewpoints [S1, S2, unofficial].
- **When not to use:** Real-time rendering where per-frame culling is needed (use [Voxel Backface Culling](4569-voxel-backface-culling-and-internal-face-masking.md) instead); scenes without known camera poses [S2, unofficial].

## Background
Mask Projection Culling originated in the context of 3D Gaussian Splatting and NeRF-based reconstruction pipelines, where it is used to filter out spurious geometry or points that do not align with observed camera views [S1, S2, unofficial]. It leverages standard computer vision techniques (projections, depth sampling) to validate 3D structure against 2D observations [S1, S2, unofficial]. The "True Back-Face Culling via Group Masking" variant was developed for high-performance voxel engines to optimize draw calls by leveraging vertex pool bucketing [S3, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
