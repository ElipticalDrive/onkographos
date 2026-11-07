# Mask Projection Culling

Also called: Mesh Culling via Mask Projection

## Overview
Mask Projection Culling is a visibility optimization technique that filters mesh vertices and faces by projecting them into multiple camera views and checking if they fall within dilated binary masks, removing geometry not visible in any view [S1, unofficial]. It differs from neighbor-based face culling (e.g., [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md)) by relying on camera-space projection rather than spatial adjacency, making it suitable for complex scenes where occlusion is view-dependent rather than purely topological.

## How it works
**Inputs:**
- A mesh defined by vertices $V$ and faces $F$.
- Camera parameters: `world_mats` and `scale_mats` from `cameras.npz`.
- Intrinsic matrix $K$ and pose $P$ derived from world/scale matrices.
- Binary masks $M_i$ for each camera view $i$, stored as images (e.g., `mask_dir/*.png`).
- Image dimensions $W=1600$, $H=1200$ [S1, unofficial].
- Morphological dilation radius $r=24$ [S1, unofficial].

**Outputs:**
- A filtered mesh with vertices and faces retained only if visible in at least one camera view.

**Data Structures:**
- **Vertices:** Array of 3D coordinates.
- **Faces:** Array of vertex indices.
- **Masks:** Binary tensors or images, one per camera view.

**Steps:**

1.  **Load Camera Parameters:**
    Extract `scale_mats` and `world_mats` from `cameras.npz`. Compute the projection matrix $P = \text{world\_mat} \times \text{scale\_mat}$. Extract intrinsics $K$ and pose $C$ using `rend_util.load_K_Rt_from_P(None, P)` [S1, unofficial].

2.  **Load Masks:**
    Load binary masks $M_i$ from `mask_dir/*.png`. Ensure dimensions are $W \times H$ [S1, unofficial].

3.  **Project Vertices:**
    For each camera view $i$:
    a. Compute inverse pose $W2C = C^{-1}$ [S1, unofficial].
    b. Transform vertices to camera space: $V_{cam} = K \times W2C \times V$.
    c. Compute pixel coordinates:
       $$
       \text{pix\_coords} = \frac{V_{cam}[0:2, :]}{V_{cam}[2, :] + \epsilon}
       $$
       where $\epsilon = 10^{-6}$ prevents division by zero [S1, unofficial].

4.  **Normalize to NDC:**
    Convert pixel coordinates to Normalized Device Coordinates (NDC) space $[-1, 1]$:
    $$
    \text{pix\_coords}_{ndc}[0] = \frac{\text{pix\_coords}[0]}{W - 1} \times 2 - 1
    $$
    $$
    \text{pix\_coords}_{ndc}[1] = \frac{\text{pix\_coords}[1]}{H - 1} \times 2 - 1
    $$
    [S1, unofficial]

5.  **Compute Validity Mask:**
    Determine which projected points lie within the NDC bounds:
    $$
    \text{valid} = (\text{pix\_coords}_{ndc}[0] > -1) \land (\text{pix\_coords}_{ndc}[0] < 1) \land (\text{pix\_coords}_{ndc}[1] > -1) \land (\text{pix\_coords}_{ndc}[1] < 1)
    $$
    [S1, unofficial]

6.  **Dilate Masks:**
    Apply morphological dilation to each binary mask $M_i$ using a disk structuring element of radius $r=24$. Convert the result to a float tensor [S1, unofficial].

7.  **Sample Masks at Projected Coordinates:**
    Sample the dilated masks at the NDC coordinates using bilinear or nearest-neighbor interpolation (typically `F.grid_sample` with `mode='nearest'`, `padding_mode='zeros'`, `align_corners=True`) [S1, unofficial]. Let $S_i$ be the sampled values for view $i$.

8.  **Filter Vertices:**
    A vertex is valid if it is inside the NDC bounds AND sampled as visible in all views where it is projected (or specifically, if it falls within the dilated mask of at least one view, depending on implementation intent; the source checks `all` views for validity in the context of filtering, but typically culling removes geometry *not* visible in *any* view. The source code snippet `mask = (sampled_masks > 0.).all(dim=-1)` suggests a strict intersection, but standard culling usually implies union. However, following the source strictly: a vertex is kept if it is valid in NDC and sampled as >0 in all considered masks? No, the source says "filter mesh vertices and faces by projecting them into multiple camera views and checking if they fall within dilated binary masks, removing geometry not visible in any view." The code `mask = (sampled_masks > 0.).all(dim=-1)` implies the vertex must be visible in *all* masks to be kept? This contradicts "removing geometry not visible in any view". Let's re-read carefully. "removing geometry not visible in any view" means keep if visible in $\ge 1$. The code `all(dim=-1)` might be checking if the vertex is valid across all *processed* views if the masks are combined. Given the ambiguity, we follow the explicit code logic: `mask` is true if `sampled_masks > 0` for all views. However, standard practice is union. We will note the source's specific implementation [S1, unofficial].
    
    *Correction based on standard culling logic vs source code:* The source code `mask = (sampled_masks > 0.).all(dim=-1)` is likely checking if the vertex is within the mask for *all* views if the masks are concatenated. But if the goal is to remove geometry *not* visible in *any* view, the condition should be `any`. The source text says "removing geometry not visible in any view". The code `all` might be a specific implementation detail for a specific dataset where masks are complementary or the logic is inverted. We will describe the source's code exactly [S1, unofficial].

9.  **Filter Faces:**
    A face is kept if all its constituent vertices are valid:
    $$
    \text{face\_mask} = \text{mask}[F].\text{all}(\text{axis}=1)
    $$
    [S1, unofficial]

10. **Update Mesh:**
    Retain vertices and faces where `face_mask` is true. Transform vertices back to world space using `scale_mat[0,0]` and `scale_mat[:3,3]` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
import numpy as np
from scipy.ndimage import binary_dilation
from scipy.ndimage import generate_binary_structure

def mask_projection_culling(mesh_vertices, mesh_faces, cameras, mask_dir, W=1600, H=1200, dilation_radius=24):
    """
    Filters mesh vertices and faces based on visibility in multiple camera views.
    """
    num_views = len(cameras)
    num_vertices = mesh_vertices.shape[0]
    
    # Initialize mask for vertices: True if visible in at least one view? 
    # Source code uses 'all', but logic suggests 'any' for culling. 
    # We follow source code logic: vertex must be valid in all views? 
    # Actually, let's look at the source: `mask = (sampled_masks > 0.).all(dim=-1)`
    # This implies the vertex is kept only if it is inside the mask for ALL views.
    # This is a very strict culling. We will implement as per source.
    
    vertex_validity = torch.ones(num_vertices, dtype=torch.float32)
    
    for i in range(num_views):
        cam = cameras[i]
        world_mat = cam['world_mat']
        scale_mat = cam['scale_mat']
        
        # Compute projection matrix P
        P = world_mat @ scale_mat
        
        # Extract intrinsics and pose
        # Assuming rend_util.load_K_Rt_from_P is available
        K, Rt = load_K_Rt_from_P(None, P)
        pose = Rt # 4x4 matrix
        
        # Load mask
        mask_img = load_mask(mask_dir, i) # Returns HxW binary image
        mask_tensor = torch.from_numpy(mask_img).float().cuda()
        
        # Dilate mask
        # Using scipy for dilation then converting back
        struct = generate_binary_structure(2, 1) # 8-connectivity
        dilated_mask_np = binary_dilation(mask_img, structure=struct, iterations=1) # Note: source uses disk radius 24, 
        # binary_dilation with iterations=1 is not radius 24. 
        # Source says "disk of radius 24". 
        # We will approximate with a larger structuring element or multiple iterations if needed.
        # For pseudocode, we assume a function `dilate_mask(mask, radius)` exists.
        dilated_mask_np = dilate_mask(mask_img, dilation_radius)
        dilated_mask_tensor = torch.from_numpy(dilated_mask_np).float().cuda()
        
        # Project vertices
        # vertices: Nx3
        # Add homogeneous coordinate
        vertices_h = torch.cat([mesh_vertices, torch.ones(num_vertices, 1)], dim=1).cuda() # Nx4
        
        # Transform to camera space
        # w2c = torch.inverse(pose)
        w2c = torch.inverse(pose)
        vertices_cam = vertices_h @ w2c.T # Nx4
        
        # Project to 2D
        # cam_points = intrinsic @ w2c @ vertices
        # K is 3x3. vertices_cam is Nx4.
        # We need to handle the homogeneous division.
        # cam_points = vertices_cam @ K.T ? No.
        # Standard: p = K * [R|t] * V
        # vertices_cam is already in camera space (after w2c).
        # So p = K * vertices_cam[0:3]
        
        vertices_cam_3d = vertices_cam[:, :3] # Nx3
        cam_points = vertices_cam_3d @ K.T # Nx3
        
        # Pixel coordinates
        pix_coords = cam_points[:, :2] / (cam_points[:, 2:3] + 1e-6) # Nx2
        
        # Normalize to NDC
        pix_coords_ndc = pix_coords.clone()
        pix_coords_ndc[:, 0] = (pix_coords[:, 0] / (W - 1)) * 2 - 1
        pix_coords_ndc[:, 1] = (pix_coords[:, 1] / (H - 1)) * 2 - 1
        
        # Validity in NDC
        valid_ndc = (pix_coords_ndc[:, 0] > -1) & (pix_coords_ndc[:, 0] < 1) & \
                    (pix_coords_ndc[:, 1] > -1) & (pix_coords_ndc[:, 1] < 1)
        
        # Sample mask at projected coordinates
        # grid_sample expects (N, C, H, W) or (N, C, H, W) for 2D?
        # pix_coords_ndc is Nx2. Needs to be (1, N, 1, 2) for grid_sample?
        # Actually, grid_sample takes input (N, C, H, W) and grid (N, H_out, W_out, 2).
        # We want to sample at specific points.
        # Reshape pix_coords_ndc to (1, 1, N, 2)
        grid = pix_coords_ndc.unsqueeze(0).unsqueeze(0) # 1, 1, N, 2
        
        # Sample
        # dilated_mask_tensor is 1, 1, H, W
        sampled = F.grid_sample(dilated_mask_tensor.unsqueeze(0), grid, mode='nearest', 
                                padding_mode='zeros', align_corners=True)
        
        # sampled is 1, 1, N, 1
        sampled_values = sampled.squeeze() # N
        
        # Update vertex validity
        # Source: mask = (sampled_masks > 0.).all(dim=-1)
        # This implies we accumulate validity across views.
        # If a vertex is not in mask for ANY view, it becomes invalid?
        # Or if it is not in mask for ALL views?
        # The source code `all` suggests it must be in ALL masks.
        # We follow source.
        is_in_mask = sampled_values > 0.
        vertex_validity = vertex_validity & is_in_mask.float()
        
    # Filter faces
    # face_mask = mask[mesh.faces].all(axis=1)
    vertex_validity_np = vertex_validity.cpu().numpy()
    face_mask = vertex_validity_np[mesh_faces].all(axis=1)
    
    # Update mesh
    new_vertices = mesh_vertices[face_mask.any(axis=1)] # Keep vertices used by valid faces
    # Re-index faces
    # This is a simplified update. 
    # In practice, you filter faces first, then vertices.
    valid_faces = mesh_faces[face_mask]
    unique_vertices = np.unique(valid_faces)
    vertex_map = {v: i for i, v in enumerate(unique_vertices)}
    new_faces = np.array([[vertex_map[v] for v in f] for f in valid_faces])
    
    return new_vertices, new_faces

def load_K_Rt_from_P(None, P):
    # Placeholder for the utility function described in source
    # Decomposes P into K and Rt
    pass

def dilate_mask(mask, radius):
    # Placeholder for morphological dilation with disk radius
    pass
```

**Edge Cases and Pitfalls:**
- **Precision:** Division by zero in projection is handled by $\epsilon = 10^{-6}$ [S1, unofficial].
- **Boundaries:** NDC bounds are strictly $(-1, 1)$. Points on the boundary are excluded [S1, unofficial].
- **Dilation:** The choice of dilation radius ($r=24$) is critical for handling aliasing and camera jitter. Too small a radius may cull visible geometry; too large may include occluded geometry [S1, unofficial].
- **Mask Alignment:** Masks must be perfectly aligned with the camera views. Misalignment leads to incorrect culling [S1, unofficial].

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5). Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S2, unofficial]. It is chosen when driver overhead is a concern, as it avoids splitting chunks into 6 sub-meshes (which would increase driver overhead 6-fold) by using existing vertex pool bucketing [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(V \cdot N)$ where $V$ is the number of vertices and $N$ is the number of camera views. Projection and sampling are linear in vertices.
- **Memory Complexity:** $O(V + N \cdot W \cdot H)$ for storing masks and intermediate projections.
- **Performance:** Reduces rendering load by eliminating invisible geometry early. Performance gain depends on the sparsity of visible geometry across views.
- **Strengths:** Effective for complex scenes with significant occlusion. Simple to implement with standard graphics libraries.
- **Limitations:** Requires pre-computed masks. Strict visibility criteria (e.g., `all` vs `any`) can lead to over-culling or under-culling. Sensitive to mask alignment and dilation parameters.
- **When to Use:** When camera positions are known and fixed, and masks are available. Suitable for pre-processing static meshes.
- **When Not to Use:** For dynamic scenes where camera positions change frequently and masks cannot be pre-computed. Not suitable for real-time culling without significant optimization.

## Background
Mask Projection Culling is used in 3D Gaussian Splatting and related rendering techniques to optimize view-dependent rendering. It was popularized by the `2d-gaussian-splatting` repository, which uses it to filter Gaussians (and by extension, mesh representations) based on visibility in training views [S1, unofficial]. The technique leverages the fact that geometry not visible in any training view contributes little to the final rendered image and can be safely removed.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
