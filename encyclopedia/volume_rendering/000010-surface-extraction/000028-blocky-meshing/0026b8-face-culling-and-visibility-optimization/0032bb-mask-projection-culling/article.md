# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing their depth against rendered depth maps or binary masks [S1, unofficial][S2, unofficial]. It differs from neighbor-based culling (which relies on voxel topology) and simple back-face culling (which relies on surface normals) by using actual camera visibility data to prune geometry, making it effective for removing floating artifacts and background noise in reconstructed meshes [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- A mesh defined by vertices $V \in \mathbb{R}^{N \times 3}$ and faces $F \in \mathbb{R}^{M \times 3}$.
- A set of $V$ camera poses, each defined by a world-to-camera matrix $W_i$ and intrinsic matrix $K_i$.
- Optional binary masks $M_i \in \{0, 1\}^{H \times W}$ for each view, representing valid regions (e.g., foreground objects).
- Image dimensions $W, H$ (typically $1600 \times 1200$) [S1, unofficial].
- Depth tolerance $\epsilon$ (typically $0.005$) [S2, unofficial].
- Batch size $B$ (typically $5 \times 10^5$) to limit GPU memory [S2, unofficial].
- Minimum visibility count $T$ (typically $20$ frames) [S2, unofficial].

**Data Structures:**
- `valid_counts`: Array of size $N$, initialized to 0, storing the number of views in which each vertex is visible.
- `depth_maps`: Array of size $V$, storing rendered depth maps for each camera pose.
- `masks`: Array of size $V$, storing binary or dilated masks for each camera pose.

**Steps:**

1. **Load Camera Parameters:**
   Extract world-to-camera matrices ($W_i$) and intrinsic matrices ($K_i$) for all $V$ views. If using OpenCV/OpenGL mixed conventions, flip the Y and Z axes of the camera pose to ensure correct depth comparison [S2, unofficial].
   $$ W_i = \text{world\_mat}_i \times \text{scale\_mat}_i $$
   $$ K_i, W_i = \text{load\_K\_Rt\_from\_P}(P_i) $$

2. **Render Reference Depth Maps:**
   For each camera pose $i$, render the input mesh to obtain a depth map $D_i$ of size $H \times W$. Use an offscreen renderer with `DEPTH_ONLY` and `SKIP_CULL_FACES` flags to ensure all geometry is rendered regardless of orientation [S2, unofficial].

3. **Prepare Masks (Optional):**
   If binary masks $M_i$ are provided, dilate them using a morphological operation (e.g., disk of radius 24) to account for projection inaccuracies and mesh discretization errors [S1, unofficial]. Convert to float tensors.

4. **Project and Validate Vertices:**
   For each vertex $v_j \in V$:
   a. Transform $v_j$ to camera space for view $i$:
      $$ p_{cam} = W_i \times \begin{bmatrix} v_j \\ 1 \end{bmatrix} $$
      $$ z_j = p_{cam}[2] $$
   b. Project to 2D pixel coordinates $(u, v)$:
      $$ \begin{bmatrix} u' \\ v' \\ z' \end{bmatrix} = K_i \times \begin{bmatrix} p_{cam}[0] \\ p_{cam}[1] \\ p_{cam}[2] \end{bmatrix} $$
      $$ u = \frac{u'}{z' + 10^{-8}}, \quad v = \frac{v'}{z' + 10^{-8}} $$
   c. Check if the point is within the image frustum:
      $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z_j > 0) $$
   d. If in frustum, sample the reference depth $d_{sample}$ at $(u, v)$ using bilinear interpolation with border padding to handle out-of-bounds queries [S2, unofficial].
   e. Determine if the vertex is "in front" of the surface:
      $$ \text{is\_front} = (z_j < d_{sample} + \epsilon) $$
   f. If masks are used, sample the dilated mask $m_{sample}$ at $(u, v)$ using nearest-neighbor interpolation. The vertex is valid in this view if $\text{in\_frustum} \land \text{is\_front} \land (m_{sample} > 0)$ [S1, unofficial].
   g. If the vertex is valid in view $i$, increment `valid_counts[j]`.

5. **Filter Mesh:**
   After processing all views, retain vertex $v_j$ if `valid_counts[j]` $\ge T$ [S2, unofficial].
   Retain face $f_k$ only if all three of its vertices are retained.
   $$ \text{keep}(v_j) = (\text{valid\_counts}[j] \ge T) $$
   $$ \text{keep}(f_k) = \text{keep}(v_{k,0}) \land \text{keep}(v_{k,1}) \land \text{keep}(v_{k,2}) $$

6. **Post-Processing:**
   Remove isolated small geometry components based on area ratio threshold (e.g., 0.2) [S2, unofficial]. Transform remaining vertices back to world space if necessary.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(vertices, faces, cameras, depth_maps, masks=None, 
                            epsilon=0.005, min_visibility=20, batch_size=500000):
    """
    vertices: (N, 3) float tensor, world space
    faces: (M, 3) int tensor
    cameras: list of dicts with 'W' (4x4 world-to-cam), 'K' (3x3 intrinsic)
    depth_maps: list of (H, W) float tensors, rendered depth
    masks: list of (H, W) float tensors, optional dilated binary masks
    """
    N = vertices.shape[0]
    valid_counts = torch.zeros(N, dtype=torch.int32, device=vertices.device)
    
    # Process in batches to limit memory
    for batch_start in range(0, N, batch_size):
        batch_end = min(batch_start + batch_size, N)
        batch_verts = vertices[batch_start:batch_end]
        batch_valid = torch.zeros(batch_end - batch_start, dtype=torch.bool, device=vertices.device)
        
        for i, cam in enumerate(cameras):
            W = cam['W'] # (4, 4)
            K = cam['K'] # (3, 3)
            depth = depth_maps[i]
            
            # Transform vertices to camera space
            # Add homogeneous coordinate
            verts_hom = torch.cat([batch_verts, torch.ones(batch_verts.shape[0], 1, device=vertices.device)], dim=1)
            cam_verts = (W @ verts_hom.T).T # (B, 4)
            z = cam_verts[:, 2]
            
            # Project to 2D
            uv_hom = (K @ cam_verts[:, :3].T).T # (B, 3)
            u = uv_hom[:, 0] / (uv_hom[:, 2] + 1e-8)
            v = uv_hom[:, 1] / (uv_hom[:, 2] + 1e-8)
            
            # Check frustum
            in_frustum = (u >= 0) & (u < depth.shape[1]) & (v >= 0) & (v < depth.shape[0]) & (z > 0)
            
            if not in_frustum.any():
                continue
                
            # Sample depth
            # Normalize to [-1, 1] for grid_sample
            u_norm = (u / (depth.shape[1] - 1) * 2.0 - 1.0)
            v_norm = (v / (depth.shape[0] - 1) * 2.0 - 1.0)
            grid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, B, 2)
            
            # Depth map shape: (1, 1, H, W)
            depth_tensor = depth.unsqueeze(0).unsqueeze(0).float()
            depth_sample = torch.nn.functional.grid_sample(depth_tensor, grid, 
                                                         padding_mode='border', 
                                                         align_corners=True)
            depth_sample = depth_sample.squeeze().squeeze() # (B,)
            
            # Check if in front
            is_front = z < (depth_sample + epsilon)
            
            view_valid = in_frustum & is_front
            
            # Check mask if provided
            if masks is not None:
                mask = masks[i].unsqueeze(0).unsqueeze(0).float()
                mask_sample = torch.nn.functional.grid_sample(mask, grid, 
                                                              mode='nearest', 
                                                              padding_mode='zeros', 
                                                              align_corners=True)
                mask_sample = mask_sample.squeeze().squeeze()
                view_valid = view_valid & (mask_sample > 0.5)
            
            # Update counts for this batch
            batch_indices = torch.arange(batch_start, batch_end, device=vertices.device)
            # We need to map back to global indices, but here we just update the batch mask
            # To avoid complex indexing, we accumulate in a global tensor or use scatter
            # For simplicity in this pseudocode, we assume we can update directly if memory allows
            # Or use scatter_add
            valid_counts.scatter_add_(0, batch_indices[view_valid], torch.ones_like(batch_indices[view_valid]))
            
        # After all views, determine which vertices in this batch are valid
        # Note: The above loop updates valid_counts globally. 
        # We need to re-check valid_counts for the current batch vertices
        batch_valid = valid_counts[batch_start:batch_end] >= min_visibility
        
        # Store valid indices for this batch
        # In a real implementation, we would collect these indices
        # Here we just return the mask for the batch
        # To make this self-contained, we'll assume we collect valid indices globally
        pass

    # Final filtering
    final_valid_mask = valid_counts >= min_visibility
    valid_vertices = vertices[final_valid_mask]
    
    # Re-index faces
    # Create a map from old index to new index
    new_indices = torch.full((N,), -1, dtype=torch.int64, device=vertices.device)
    new_indices[final_valid_mask] = torch.arange(final_valid_mask.sum(), device=vertices.device)
    
    # Map faces
    new_faces = new_indices[faces]
    # Remove faces with any -1 (invalid vertex)
    valid_faces_mask = (new_faces != -1).all(dim=1)
    final_faces = new_faces[valid_faces_mask]
    
    return valid_vertices, final_faces
```

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5). Draw calls are masked based on camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. It is chosen when driver overhead must be minimized compared to splitting chunks into 6 sub-meshes [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size [S2, unofficial].
- **Memory Complexity:** $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
- **Strengths:** Effectively removes floating artifacts and background noise that neighbor-based culling cannot detect. Works well with multi-view reconstruction pipelines [S1, unofficial][S2, unofficial].
- **Limitations:** Requires pre-rendered depth maps or masks for all views, increasing preprocessing time. Sensitive to coordinate system mismatches (e.g., OpenCV vs. OpenGL) [S2, unofficial].
- **When to use:** When high-quality mesh extraction from multi-view data is required, and computational resources allow for preprocessing depth maps.
- **When not to use:** For real-time applications where preprocessing is not feasible, or when the mesh is already clean and only internal faces need removal.

## Background
Mask Projection Culling is commonly used in 3D reconstruction pipelines, particularly those involving Gaussian Splatting or NeRF-based methods, to extract clean meshes from implicit or point-based representations [S1, unofficial][S2, unofficial]. It leverages the geometric consistency across multiple views to filter out noise.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
