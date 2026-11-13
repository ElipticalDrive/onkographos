# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps [S1, S2, unofficial]. It solves the problem of retaining only the geometry visible from a specific set of viewpoints, effectively pruning occluded or background geometry. Unlike static neighbor-based culling in [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), this method is view-dependent and relies on multi-view consistency rather than local voxel occupancy [S1, S2, unofficial]. It differs from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by explicitly using depth comparison to determine front-facing visibility, whereas coverage masks may rely solely on 2D projection overlap [S1, S2, unofficial].

## How it works
The method filters a mesh by projecting its vertices into multiple camera views, checking if they fall within dilated binary masks, and verifying if they are in front of the surface depth in those views [S1, S2, unofficial].

**Inputs:**
- A mesh defined by vertices $V \in \mathbb{R}^{N \times 3}$ and faces $F \in \mathbb{R}^{M \times 3}$ (indices into $V$).
- A set of $V$ camera poses, each defined by a world-to-camera matrix $W2C$ and an intrinsic matrix $K$.
- Binary masks $M_i \in \{0, 1\}^{H \times W}$ for each camera view $i$, indicating valid regions.
- Depth maps $D_i \in \mathbb{R}^{H \times W}$ for each camera view $i$, representing the distance to the nearest surface.
- Parameters:
  - `eps`: Depth comparison tolerance. Typical value: $0.005$ [S2, unofficial].
  - `points_batch_size`: Maximum points processed per batch. Typical value: $5 \times 10^5$ [S2, unofficial].
  - `remove_small_geometry_threshold`: Minimum area ratio for a connected component to be retained. Typical value: $0.2$ [S2, unofficial].
  - Image dimensions: $W=1600, H=1200$ (hard-coded in some implementations) [S1, unofficial].

**Data Structures:**
- `valid_counts`: Array of size $N$, initialized to 0, storing the number of views in which each vertex is visible.
- `face_mask`: Boolean array of size $M$, indicating if a face should be retained.

**Steps:**

1. **Load Camera Parameters:**
   Extract scale matrices and world matrices from camera data. Compute the projection matrix $P = W \times S$ (where $W$ is world mat, $S$ is scale mat) and decompose it to obtain intrinsic $K$ and pose matrices [S1, unofficial].
   *Note: Handle coordinate system mismatches (e.g., OpenCV vs OpenGL) by flipping Y and Z axes of the camera pose if necessary: $c2w[:3, 1:3] *= -1$ [S2, unofficial].*

2. **Render/Load Reference Data:**
   Render forward depth maps from the mesh for each estimated camera pose using an offscreen renderer with flags `OFFSCREEN | DEPTH_ONLY | SKIP_CULL_FACES` [S2, unofficial]. Load binary masks from image files [S1, unofficial].

3. **Project Vertices to Camera Space:**
   For each camera view $i$:
   a. Compute the inverse world-to-camera matrix: $W2C_i = (C2W_i)^{-1}$ [S1, S2, unofficial].
   b. Transform vertices to camera space: $P_{cam} = W2C_i \times V^T$ (where $V^T$ is $3 \times N$) [S1, unofficial].
   c. Project to 2D homogeneous coordinates: $P_{proj} = K \times P_{cam}$ [S1, S2, unofficial].
   d. Perform perspective division to get pixel coordinates:
      $$ z = P_{proj}[2, :] + 10^{-6} $$
      $$ uv = \frac{P_{proj}[0:2, :]}{z} $$
      [S1, S2, unofficial]

4. **Check Frustum Bounds:**
   Determine if points lie within the image frustum:
   $$ u \in [0, W-1], \quad v \in [0, H-1], \quad z > 0 $$
   [S2, unofficial]
   Alternatively, normalize to NDC space $[-1, 1]$:
   $$ pix\_coords_{ndc} = \left( \frac{uv}{[W-1, H-1]} - 0.5 \right) \times 2 $$
   Validity mask: $valid = (pix\_coords_{ndc} > -1) \land (pix\_coords_{ndc} < 1)$ [S1, unofficial].

5. **Sample Mask and Depth:**
   a. **Mask Sampling:** Dilate the binary mask $M_i$ using morphology (e.g., disk radius 24) to account for projection errors [S1, unofficial]. Sample the dilated mask at the projected UV coordinates using nearest-neighbor interpolation with `padding_mode='zeros'` and `align_corners=True` [S1, unofficial].
   b. **Depth Sampling:** Sample the depth map $D_i$ at the UV coordinates using bilinear interpolation with `padding_mode='border'` to handle out-of-bounds queries (preventing NaNs) [S2, unofficial].

6. **Determine Visibility:**
   A point is considered visible in view $i$ if:
   a. It is within the mask: $sampled\_mask > 0$ [S1, unofficial].
   b. It is in front of the surface: $z < sampled\_depth + eps$ [S2, unofficial].
   Accumulate visibility counts:
   $$ valid\_counts[j] += 1 \quad \text{if point } j \text{ is visible in view } i $$
   [S2, unofficial]

7. **Filter Mesh:**
   a. Select vertices visible in at least $T$ frames (e.g., $T=20$) [S2, unofficial].
   b. A face is retained if all its vertices are valid: $face\_mask = valid\_vertices[faces].all(axis=1)$ [S1, unofficial].
   c. Optionally, remove small connected components based on area ratio [S2, unofficial].

8. **Export:**
   Update mesh vertices and faces. Transform remaining vertices back to world space using the scale matrix [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(vertices, faces, cameras, masks, depths, eps=0.005, min_visibility=20):
    """
    vertices: Tensor [N, 3]
    faces: Tensor [M, 3]
    cameras: List of dicts with 'K' (3x3), 'c2w' (4x4)
    masks: List of Tensors [H, W] (binary)
    depths: List of Tensors [H, W] (float)
    """
    N = vertices.shape[0]
    valid_counts = torch.zeros(N, dtype=torch.int32)
    
    # Handle coordinate system mismatch if necessary
    # c2w[:, 1:3] *= -1 
    
    for cam, mask, depth in zip(cameras, masks, depths):
        K = cam['K']
        c2w = cam['c2w']
        
        # Compute world-to-camera
        w2c = torch.inverse(c2w)
        
        # Project vertices to camera space
        # vertices: [N, 3] -> [3, N]
        verts_hom = torch.cat([vertices.T, torch.ones(1, N, device=vertices.device)], dim=0)
        cam_verts = w2c @ verts_hom # [4, N]
        
        # Project to 2D
        proj = K @ cam_verts[:3, :] # [3, N]
        z = proj[2, :] + 1e-6
        uv = proj[:2, :] / z # [2, N]
        
        # Check frustum bounds
        W, H = mask.shape
        in_frustum = (uv[0] >= 0) & (uv[0] < W-1) & (uv[1] >= 0) & (uv[1] < H-1) & (z > 0)
        
        if not torch.any(in_frustum):
            continue
            
        # Sample depth and mask
        # Normalize UV to [-1, 1] for grid_sample
        vgrid = torch.stack([uv[0], uv[1]], dim=-1) # [N, 2]
        vgrid = vgrid / torch.tensor([W-1, H-1], device=vertices.device) * 2.0 - 1.0
        vgrid = vgrid.unsqueeze(0).unsqueeze(0) # [1, 1, N, 2]
        
        # Sample depth (bilinear, border padding)
        depth_tensor = depth.unsqueeze(0).unsqueeze(0).to(vertices.device) # [1, 1, H, W]
        depth_sample = torch.nn.functional.grid_sample(depth_tensor, vgrid, padding_mode='border', align_corners=True)
        depth_sample = depth_sample.squeeze() # [N]
        
        # Sample mask (nearest, zero padding)
        # Dilate mask beforehand in preprocessing
        mask_tensor = mask.unsqueeze(0).unsqueeze(0).float().to(vertices.device) # [1, 1, H, W]
        mask_sample = torch.nn.functional.grid_sample(mask_tensor, vgrid, mode='nearest', padding_mode='zeros', align_corners=True)
        mask_sample = mask_sample.squeeze() # [N]
        
        # Determine visibility
        is_front = z < (depth_sample + eps)
        is_in_mask = mask_sample > 0.5
        visible = in_frustum & is_front & is_in_mask
        
        valid_counts += visible.int()
        
    # Filter vertices
    valid_vertices_mask = valid_counts >= min_visibility
    valid_vertex_indices = torch.where(valid_vertices_mask)[0]
    
    # Filter faces
    # faces: [M, 3]
    face_valid = valid_vertices_mask[faces].all(dim=1)
    valid_faces_mask = face_valid
    
    return valid_vertex_indices, valid_faces_mask
```

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5) [S3, unofficial]. Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. It is chosen when driver overhead must be minimized compared to splitting chunks into 6 sub-meshes [S3, unofficial]. Implementation involves clearing a `groups` set, inserting group IDs based on camera position signs (e.g., if `cam::pos.x < 0` insert 0), and calling `vertpool.mask` with a lambda checking `groups.contains(cmd.group)` [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size [S2, unofficial].
- **Memory Complexity:** $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
- **Strengths:** Effectively removes occluded and background geometry based on multi-view consistency; handles transparency and depth discontinuities better than simple 2D projection [S1, S2, unofficial].
- **Limitations:** Requires pre-rendered depth maps and masks; sensitive to coordinate system mismatches; performance degrades with a large number of views or high-resolution depth maps [S2, unofficial].
- **When to use:** When a static set of camera views is available and geometry needs to be pruned for those specific viewpoints (e.g., NeRF mesh extraction) [S1, S2, unofficial].
- **When not to use:** For dynamic scenes with unknown camera paths or when real-time culling without pre-computed depth maps is required [S2, unofficial].

## Background
Mask Projection Culling is used in the context of 3D Gaussian Splatting and NeRF-based mesh extraction, specifically in tools like `2d-gaussian-splatting` and `svraster` [S1, S2, unofficial]. It addresses the need to convert volumetric or point-based representations into clean, visible meshes by leveraging the multi-view consistency inherent in the capture process [S1, S2, unofficial]. The technique relies on standard computer graphics operations (projection, rasterization/depth sampling) and is often implemented using PyTorch for GPU acceleration [S1, S2, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
