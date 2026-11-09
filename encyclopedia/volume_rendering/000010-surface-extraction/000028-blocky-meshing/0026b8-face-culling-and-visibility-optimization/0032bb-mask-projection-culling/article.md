# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps or binary masks [S1, S2, unofficial]. It differs from standard face culling methods like [Face Culling](8735-face-culling.md) or [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by relying on external camera views and depth/mask validation rather than local voxel neighbor occupancy, making it suitable for complex, non-voxelized, or occluded geometry where local checks fail [S1, S2, unofficial].

## How it works
**Inputs:**
- A mesh (vertices $V$, faces $F$).
- A set of $V$ camera poses (world-to-camera matrices $W_i$ or camera-to-world $C_i$) and intrinsic matrices $K_i$ for $i \in \{1, \dots, V\}$.
- Optional: Binary masks $M_i$ (images) corresponding to each camera view, indicating valid regions [S1, unofficial].
- Optional: Rendered depth maps $D_i$ for each camera view [S2, unofficial].

**Outputs:**
- A filtered mesh containing only vertices and faces deemed visible in at least $T$ views (typically $T=20$) [S2, unofficial].

**Data Structures:**
- `valid_counts`: Array of size $N$ (number of vertices), initialized to 0. Stores the number of views in which each vertex is valid [S2, unofficial].
- `face_mask`: Boolean array of size $|F|$. True if all vertices of the face are valid [S1, unofficial].

**Parameters:**
- `eps`: Depth comparison tolerance. Typical value: $0.005$ [S2, unofficial].
- `points_batch_size`: Maximum points processed per batch to limit GPU memory. Typical value: $5 \times 10^5$ [S2, unofficial].
- `remove_small_geometry_threshold`: Minimum area ratio for a connected component to be retained. Typical value: $0.2$ [S2, unofficial].
- `mask_dilation_radius`: Radius for morphological dilation of binary masks. Typical value: $24$ [S1, unofficial].

**Steps:**

1. **Load Camera Parameters:**
   Extract world matrices and scale matrices from camera data (e.g., `cameras.npz`). Compute projection matrix $P = \text{world\_mat} \times \text{scale\_mat}$. Extract intrinsics $K$ and pose $W$ (world-to-camera) using helper functions like `rend_util.load_K_Rt_from_P` [S1, unofficial].
   *Note:* Handle coordinate system mismatches (e.g., OpenCV vs OpenGL) by flipping Y and Z axes of the camera pose if necessary: $C_i[:3, 1:3] *= -1$ [S2, unofficial].

2. **Render/Load Reference Data:**
   - If using masks: Load binary masks $M_i$ from files (e.g., `mask_dir/*.png`). Hard-code or detect image dimensions $W, H$ (e.g., $W=1600, H=1200$) [S1, unofficial].
   - If using depth: Render forward depth maps $D_i$ from the mesh for each camera pose using an offscreen renderer with flags `OFFSCREEN | DEPTH_ONLY | SKIP_CULL_FACES` [S2, unofficial].

3. **Project Vertices to Camera Space:**
   For each camera $i$:
   a. Compute inverse pose $W_i = C_i^{-1}$ (world-to-camera) [S1, S2, unofficial].
   b. Transform vertices $v \in V$ to camera space: $v_{cam} = W_i \times v_{hom}$, where $v_{hom}$ is the vertex in homogeneous coordinates [S1, S2, unofficial].
   c. Project to 2D:
      $$
      uv = K_i \times v_{cam}
      $$
      $$
      z = uv[:, -1] + \epsilon \quad (\text{where } \epsilon = 10^{-6} \text{ or } 10^{-8})
      $$
      $$
      uv_{xy} = uv[:, :2] / z
      $$
   d. Normalize to NDC space $[-1, 1]$ for grid sampling:
      $$
      uv_{ndc} = \left( \frac{uv_{xy}}{[W-1, H-1]} \times 2.0 \right) - 1.0
      $$
      [S1, S2, unofficial].

4. **Check Frustum and Validity:**
   Determine if points are within the image bounds and in front of the camera:
   $$
   \text{in\_frustum} = (u \ge 0) \land (u \le W-1) \land (v \ge 0) \land (v \le H-1) \land (z > 0)
   $$
   [S2, unofficial]. Alternatively, check NDC bounds:
   $$
   \text{valid} = ((uv_{ndc} > -1) \land (uv_{ndc} < 1)).\text{all}(\text{dim}=-1)
   $$
   [S1, unofficial].

5. **Sample Mask or Depth:**
   - **Mask Mode:** Dilate binary mask $M_i$ using morphology (e.g., `binary_dilation` with disk radius 24) [S1, unofficial]. Sample the dilated mask at $uv_{ndc}$ using `F.grid_sample` with `mode='nearest'`, `padding_mode='zeros'`, `align_corners=True` [S1, unofficial].
   - **Depth Mode:** Sample depth $D_i$ at $uv_{ndc}$ using `F.grid_sample` with `mode='bilinear'`, `padding_mode='border'`, `align_corners=True` to handle out-of-bounds queries without NaNs [S2, unofficial].

6. **Determine Visibility:**
   - **Mask Mode:** A vertex is valid in view $i$ if the sampled mask value $> 0$ [S1, unofficial].
   - **Depth Mode:** A vertex is valid if it is in front of the surface:
     $$
     \text{is\_front} = (z < D_{sampled} + \text{eps}) \land (D_{sampled} > 0)
     $$
     [S2, unofficial].

7. **Accumulate Counts:**
   Increment `valid_counts` for vertices that are valid in view $i$ [S2, unofficial].

8. **Filter Mesh:**
   - Select vertices where `valid_counts >= T` (e.g., $T=20$) [S2, unofficial].
   - Select faces where all three vertices are in the selected vertex set [S1, unofficial].
   - (Optional) Remove small connected components based on `remove_small_geometry_threshold` [S2, unofficial].

9. **Transform and Export:**
   Transform remaining vertices back to world space using scale matrices and export the filtered mesh [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
import torch.nn.functional as F

def mask_projection_culling(mesh, cameras, masks=None, depth_maps=None, 
                            eps=0.005, threshold=20, batch_size=500000):
    """
    mesh: dict with 'vertices' (N, 3) and 'faces' (F, 3)
    cameras: list of dicts with 'K' (3,3), 'C2W' (4,4)
    masks: list of tensors (V, H, W) or None
    depth_maps: list of tensors (V, H, W) or None
    """
    vertices = mesh['vertices']
    faces = mesh['faces']
    N = vertices.shape[0]
    V = len(cameras)
    
    valid_counts = torch.zeros(N, dtype=torch.int32, device=vertices.device)
    
    # Process in batches to limit memory
    for batch_start in range(0, N, batch_size):
        batch_end = min(batch_start + batch_size, N)
        batch_verts = vertices[batch_start:batch_end]
        batch_valid = torch.zeros(batch_verts.shape[0], dtype=torch.bool, device=vertices.device)
        
        for i in range(V):
            K = cameras[i]['K'].to(vertices.device)
            C2W = cameras[i]['C2W'].to(vertices.device)
            
            # Handle coordinate system flip if needed (e.g., OpenCV to OpenGL)
            # C2W_copy = C2W.clone()
            # C2W_copy[:3, 1:3] *= -1
            
            W2C = torch.inverse(C2W)
            
            # Homogeneous coordinates
            verts_hom = torch.cat([batch_verts, torch.ones(batch_verts.shape[0], 1, device=vertices.device)], dim=1)
            
            # Transform to camera space
            cam_verts = (W2C @ verts_hom.T).T[:, :3]
            cam_z = cam_verts[:, 2]
            
            # Project
            uv = (K @ cam_verts.T).T
            z = uv[:, 2] + 1e-8
            uv_xy = uv[:, :2] / z
            
            # Normalize to NDC [-1, 1]
            H, W = 1200, 1600 # Hardcoded or from mask/depth shape
            uv_ndc = (uv_xy / torch.tensor([W-1, H-1], device=vertices.device) * 2.0) - 1.0
            
            # Check frustum bounds
            in_frustum = (uv_xy[:, 0] >= 0) & (uv_xy[:, 0] <= W-1) & \
                         (uv_xy[:, 1] >= 0) & (uv_xy[:, 1] <= H-1) & \
                         (z > 0)
            
            if masks is not None:
                # Mask mode
                mask = masks[i].to(vertices.device)
                # Sample mask
                # grid_sample expects (N, C, H, W) or (1, C, H, W)
                # uv_ndc is (B, 2), need (B, 1, 1, 2)
                grid = uv_ndc.unsqueeze(1).unsqueeze(1)
                # mask should be (1, 1, H, W)
                sampled = F.grid_sample(mask.unsqueeze(0).unsqueeze(0), grid, 
                                        mode='nearest', padding_mode='zeros', 
                                        align_corners=True)
                sampled = sampled.squeeze()
                is_valid = sampled > 0.0
            elif depth_maps is not None:
                # Depth mode
                depth = depth_maps[i].to(vertices.device)
                grid = uv_ndc.unsqueeze(1).unsqueeze(1)
                sampled_depth = F.grid_sample(depth.unsqueeze(0).unsqueeze(0), grid, 
                                              mode='bilinear', padding_mode='border', 
                                              align_corners=True)
                sampled_depth = sampled_depth.squeeze()
                is_valid = (sampled_depth > 0.0) & (cam_z < sampled_depth + eps)
            else:
                is_valid = torch.zeros(batch_verts.shape[0], dtype=torch.bool, device=vertices.device)
            
            # Combine with frustum check
            is_valid = is_valid & in_frustum
            
            # Update counts
            valid_counts[batch_start:batch_end] += is_valid.int()
            
        # Keep vertices valid in at least 'threshold' views
        batch_valid = valid_counts[batch_start:batch_end] >= threshold
        # Note: In a full implementation, we would accumulate a global valid mask
        # For simplicity, assuming we filter at the end based on final counts
    
    # Final filtering
    final_valid_verts = valid_counts >= threshold
    valid_vertex_indices = torch.where(final_valid_verts)[0]
    
    # Filter faces
    # Create a map from old index to new index or just boolean mask
    face_mask = final_valid_verts[faces].all(dim=1)
    
    filtered_faces = faces[face_mask]
    # Re-index vertices if necessary
    # ...
    
    return {'vertices': vertices[final_valid_verts], 'faces': filtered_faces}
```

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5). Draw calls are masked based on camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. It is chosen when driver overhead must be minimized compared to splitting chunks into 6 sub-meshes [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size [S2, unofficial].
- **Memory Complexity:** $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
- **Strengths:** Effective for complex occlusions and non-voxelized geometry; robust against local neighbor ambiguities [S1, S2, unofficial].
- **Limitations:** Requires pre-existing camera poses and masks/depth maps; computationally expensive for large meshes due to multiple projections [S2, unofficial].
- **When to use:** When local voxel culling fails due to complex topology or when external view information is available [S1, S2, unofficial].
- **When not to use:** For real-time rendering where pre-processing is not feasible, or when camera data is unavailable [S2, unofficial].

## Background
Mask Projection Culling originates from techniques used in 3D reconstruction and Gaussian Splatting pipelines, where visibility is determined by projecting points into multiple camera views and validating against depth or mask data [S1, S2, unofficial]. It is commonly used in post-processing steps for NeRF and 3DGS models to clean up noisy geometry [S1, S2, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
