# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps or binary masks [S1, S2, unofficial]. It differs from simple adjacency culling by using actual camera views to determine visibility, allowing it to cull geometry that is occluded by other objects in the scene rather than just adjacent voxels. It differs from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by operating on the final mesh geometry (vertices/faces) rather than pre-computing coverage masks during voxel traversal.

## How it works
**Inputs:**
- A 3D mesh (vertices $V$, faces $F$).
- A set of $V$ camera poses (world-to-camera matrices $W_i$ and intrinsic matrices $K_i$).
- Optional: Binary masks $M_i$ for each camera view (e.g., from segmentation or background removal).
- Optional: Rendered depth maps $D_i$ for each camera view.

**Outputs:**
- A filtered mesh containing only vertices and faces visible in at least $N_{min}$ views.

**Data Structures:**
- `vertices`: Array of shape $(N, 3)$, float32.
- `faces`: Array of shape $(M, 3)$, int32, indices into `vertices`.
- `valid_counts`: Array of shape $(N,)$, int32, initialized to 0.
- `camera_poses`: List of $V$ matrices, each $4 \times 4$.
- `intrinsics`: List of $V$ matrices, each $3 \times 3$.
- `masks`: List of $V$ binary images, shape $(H, W)$, uint8.
- `depth_maps`: List of $V$ depth images, shape $(H, W)$, float32.

**Parameters:**
- `eps`: Depth comparison tolerance. Typical value: $0.005$ [S2, unofficial].
- `points_batch_size`: Max points per batch. Typical value: $5 \times 10^5$ [S2, unofficial].
- `min_views`: Minimum number of views a point must be visible in. Typical value: $20$ (if $V=20$) or a fraction of $V$ [S2, unofficial].
- `mask_dilation_radius`: Radius for morphological dilation of binary masks. Typical value: $24$ [S1, unofficial].

**Steps:**

1. **Load Camera Parameters:**
   Extract world-to-camera matrices ($W_i$) and intrinsic matrices ($K_i$) for each view $i \in [0, V-1]$. If using COLMAP or similar, ensure coordinate system consistency (e.g., flip Y/Z axes if converting from OpenCV to OpenGL conventions) [S2, unofficial].

2. **Render/Load Reference Data:**
   - If binary masks are available, load them. Dilate each mask $M_i$ using a disk kernel of radius `mask_dilation_radius` to account for projection inaccuracies and aliasing [S1, unofficial].
   - If depth maps are available, load them. If not, render forward depth maps from the mesh for each camera pose using an offscreen renderer with flags `OFFSCREEN | DEPTH_ONLY | SKIP_CULL_FACES` [S2, unofficial].

3. **Project and Validate Points:**
   For each batch of vertices (size `points_batch_size`):
   a. Initialize `valid_counts` for the batch to 0.
   b. For each view $i$:
      i. Transform vertices to camera space:
         $$ P_{cam} = W_i \times V_{world} $$
         where $V_{world}$ is the homogeneous vertex vector $(x, y, z, 1)^T$.
      ii. Project to 2D:
          $$ uv_{hom} = K_i \times P_{cam} $$
          $$ z = uv_{hom}[2] + \epsilon $$
          $$ u = uv_{hom}[0] / z $$
          $$ v = uv_{hom}[1] / z $$
      iii. Check frustum bounds:
           $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z > 0) $$
      iv. Sample visibility:
           - **If using masks:** Normalize $(u, v)$ to NDC $[-1, 1]$ for grid sampling:
             $$ u_{ndc} = \frac{2u}{W-1} - 1 $$
             $$ v_{ndc} = \frac{2v}{H-1} - 1 $$
             Sample the dilated mask $M_i$ at $(u_{ndc}, v_{ndc})$ using nearest-neighbor interpolation with zero padding.
             $$ \text{visible}_i = (M_{sampled} > 0) $$
           - **If using depth maps:** Sample depth $D_{sampled}$ at $(u, v)$ using bilinear interpolation with border padding.
             $$ \text{is\_front} = (z < D_{sampled} + \epsilon) $$
             $$ \text{visible}_i = \text{is\_front} \land \text{in\_frustum} $$
      v. Update counts:
         $$ \text{valid\_counts} += \text{visible}_i $$

4. **Filter Mesh:**
   a. Identify valid vertices:
      $$ V_{valid} = \{ v \mid \text{valid\_counts}[v] \ge \text{min\_views} \} $$
   b. Identify valid faces:
      $$ F_{valid} = \{ f \in F \mid \text{all vertices of } f \in V_{valid} \} $$
   c. Construct new mesh with $V_{valid}$ and $F_{valid}$.

5. **Post-Processing (Optional):**
   Remove small disconnected components if their area is less than `remove_small_geometry_threshold` of the total mesh area [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(vertices, faces, cameras, masks=None, depth_maps=None, 
                            eps=0.005, min_views=20, mask_dilation_radius=24, 
                            points_batch_size=500000):
    """
    vertices: (N, 3) numpy array
    faces: (M, 3) numpy array
    cameras: list of dicts with 'w2c' (4,4), 'K' (3,3), 'W', 'H'
    masks: list of (H, W) numpy arrays, optional
    depth_maps: list of (H, W) numpy arrays, optional
    """
    N = vertices.shape[0]
    valid_counts = np.zeros(N, dtype=np.int32)
    
    # Pre-process masks if provided
    if masks is not None:
        processed_masks = []
        for m in masks:
            # Dilate mask
            m_dilated = binary_dilation(m, disk(mask_dilation_radius)).astype(np.float32)
            processed_masks.append(m_dilated)
        masks = processed_masks
        
    # Convert vertices to homogeneous coordinates
    vertices_h = np.hstack([vertices, np.ones((N, 1))])
    
    # Process in batches to limit memory
    for batch_start in range(0, N, points_batch_size):
        batch_end = min(batch_start + points_batch_size, N)
        batch_verts_h = vertices_h[batch_start:batch_end]
        batch_size = batch_end - batch_start
        
        # Initialize batch valid counts
        batch_valid = np.zeros(batch_size, dtype=np.int32)
        
        for i, cam in enumerate(cameras):
            w2c = cam['w2c']
            K = cam['K']
            W = cam['W']
            H = cam['H']
            
            # Transform to camera space
            # P_cam = w2c @ V_world
            # Note: w2c is 4x4, V_world is (batch_size, 4)
            # Result is (batch_size, 4)
            P_cam = (w2c @ batch_verts_h.T).T
            
            # Project to 2D
            uv_hom = (K @ P_cam[:, :3].T).T # (batch_size, 3)
            z = uv_hom[:, 2] + eps
            u = uv_hom[:, 0] / z
            v = uv_hom[:, 1] / z
            
            # Check frustum
            in_frustum = (u >= 0) & (u <= W - 1) & (v >= 0) & (v <= H - 1) & (z > 0)
            
            visible = np.zeros(batch_size, dtype=bool)
            
            if masks is not None:
                # Sample mask
                # Normalize to NDC [-1, 1]
                u_ndc = 2.0 * u / (W - 1) - 1.0
                v_ndc = 2.0 * v / (H - 1) - 1.0
                
                # Create grid for grid_sample
                # grid_sample expects (N, H_out, W_out, 2) with xy coordinates
                # We have batch_size points, so we create a grid of shape (1, 1, batch_size, 2)
                # But grid_sample is usually for images. For points, we can use F.grid_sample
                # or manual interpolation. Assuming PyTorch grid_sample usage:
                
                # Convert to tensor
                u_ndc_t = torch.tensor(u_ndc, dtype=torch.float32).unsqueeze(0).unsqueeze(0).unsqueeze(0) # (1, 1, 1, batch_size)
                v_ndc_t = torch.tensor(v_ndc, dtype=torch.float32).unsqueeze(0).unsqueeze(0).unsqueeze(0)
                grid = torch.cat([u_ndc_t, v_ndc_t], dim=-1) # (1, 1, batch_size, 2)
                
                # Mask tensor: (1, 1, H, W)
                mask_t = torch.tensor(masks[i], dtype=torch.float32).unsqueeze(0).unsqueeze(0)
                
                # Sample
                # grid_sample output: (1, 1, batch_size, 1)
                sampled = torch.nn.functional.grid_sample(mask_t, grid, mode='nearest', 
                                                          padding_mode='zeros', align_corners=True)
                sampled = sampled.squeeze().cpu().numpy()
                
                visible = sampled > 0.0
            else:
                # Sample depth
                # Normalize to NDC [-1, 1] for grid_sample if using PyTorch
                u_ndc = 2.0 * u / (W - 1) - 1.0
                v_ndc = 2.0 * v / (H - 1) - 1.0
                
                u_ndc_t = torch.tensor(u_ndc, dtype=torch.float32).unsqueeze(0).unsqueeze(0).unsqueeze(0)
                v_ndc_t = torch.tensor(v_ndc, dtype=torch.float32).unsqueeze(0).unsqueeze(0).unsqueeze(0)
                grid = torch.cat([u_ndc_t, v_ndc_t], dim=-1)
                
                depth_t = torch.tensor(depth_maps[i], dtype=torch.float32).unsqueeze(0).unsqueeze(0)
                
                sampled_depth = torch.nn.functional.grid_sample(depth_t, grid, mode='bilinear', 
                                                                padding_mode='border', align_corners=True)
                sampled_depth = sampled_depth.squeeze().cpu().numpy()
                
                # Check if point is in front of surface
                is_front = z < (sampled_depth + eps)
                visible = is_front & in_frustum
            
            batch_valid += visible.astype(np.int32)
        
        valid_counts[batch_start:batch_end] = batch_valid
        
    # Filter vertices
    valid_vertex_indices = np.where(valid_counts >= min_views)[0]
    valid_vertex_set = set(valid_vertex_indices)
    
    # Filter faces
    valid_faces = []
    for f in faces:
        if f[0] in valid_vertex_set and f[1] in valid_vertex_set and f[2] in valid_vertex_set:
            valid_faces.append(f)
            
    valid_faces = np.array(valid_faces)
    
    # Remap vertex indices
    vertex_map = {old: new for new, old in enumerate(valid_vertex_indices)}
    valid_faces = np.vectorize(vertex_map.get)(valid_faces)
    
    valid_vertices = vertices[valid_vertex_indices]
    
    return valid_vertices, valid_faces
```

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5) [S3, unofficial]. Draw calls are masked based on camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. Unlike regular chunk meshing which would require splitting a chunk into 6 sub-meshes (increasing driver overhead 6-fold), this method uses existing vertex pool bucketing [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size factor (effectively $N/B$ batches) [S2, unofficial].
- **Memory Complexity:** $O(N \cdot V)$ for intermediate masks and valid counts if not processed in batches, or $O(N \cdot B)$ per batch [S2, unofficial].
- **Strengths:** Effective at removing occluded geometry that adjacency culling misses. Works well with multi-view stereo data.
- **Limitations:** Requires camera poses and either masks or depth maps. Sensitive to coordinate system mismatches. Performance depends on the number of views.
- **When to use:** When high-quality mesh extraction from multi-view images is required and occlusion is significant.
- **When not to use:** When camera poses are unavailable or inaccurate. When real-time performance is critical and $V$ is large.

## Background
Mask Projection Culling is commonly used in 3D reconstruction pipelines, particularly those involving Gaussian Splatting or NeRF-based methods, to extract clean meshes from implicit or point-based representations [S1, S2, unofficial]. It leverages the geometric consistency across multiple views to filter out noise and artifacts.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
