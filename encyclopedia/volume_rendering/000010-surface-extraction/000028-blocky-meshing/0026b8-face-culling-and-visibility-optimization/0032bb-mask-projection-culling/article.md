# Mask Projection Culling

Also called: Mesh Culling via Mask Projection

## Overview
Mask Projection Culling filters mesh vertices and faces by projecting them into multiple camera views and checking if they fall within dilated binary masks, removing geometry not visible in any view [S1, unofficial]. It differs from sibling methods like [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by operating on explicit mesh geometry (vertices/faces) rather than voxel occupancy or block placement, and from [Block Visibility Culling](5334-block-visibility-culling.md) by using per-pixel mask sampling instead of frustum or bounding-box tests. It is chosen when a set of reference images with associated segmentation or depth masks is available to define the visible region of interest.

## How it works
**Inputs:**
- A mesh defined by vertices $V \in \mathbb{R}^{N \times 3}$ and faces $F \in \mathbb{Z}^{M \times 3}$.
- Camera parameters: `world_mats` (4x4), `scale_mats` (4x4), and intrinsics.
- Binary masks: $M_i \in \{0, 1\}^{H \times W}$ for each camera view $i$.
- Image dimensions: $W=1600$, $H=1200$ [S1, unofficial].
- Morphology dilation radius: $r=24$ [S1, unofficial].

**Outputs:**
- A filtered mesh with vertices and faces retained only if they are visible in at least one camera view within the dilated mask.

**Data Structures:**
- `vertices`: Array of shape $(N, 3)$.
- `faces`: Array of shape $(M, 3)$, indices into `vertices`.
- `masks`: List of binary tensors of shape $(H, W)$.

**Steps:**

1.  **Load and Transform Camera Parameters:**
    For each camera $i$:
    - Compute projection matrix $P_i = \text{world\_mat}_i \times \text{scale\_mat}_i$.
    - Extract intrinsic matrix $K_i$ and pose $T_{w2c, i}$ from $P_i$ using `rend_util.load_K_Rt_from_P(None, P_i)` [S1, unofficial].
    - Compute inverse pose $W2C_i = T_{w2c, i}^{-1}$ [S1, unofficial].

2.  **Project Vertices to Image Space:**
    For each vertex $v \in V$:
    - Transform to camera space: $v_{cam} = W2C_i \times [v_x, v_y, v_z, 1]^T$.
    - Project to pixel coordinates:
      $$
      p_{uv} = \frac{K_i \times v_{cam}[0:2]}{v_{cam}[2] + \epsilon}
      $$
      where $\epsilon = 10^{-6}$ prevents division by zero [S1, unofficial].
    - Normalize to NDC space $[-1, 1]$:
      $$
      p_{ndc} = \left( \frac{p_{uv}}{[W-1, H-1]} - 0.5 \right) \times 2
      $$
      where division is element-wise [S1, unofficial].

3.  **Compute Validity Mask:**
    Determine if projected points lie within the NDC bounds:
    $$
    \text{valid}_i = \left( p_{ndc, x} > -1 \right) \land \left( p_{ndc, x} < 1 \right) \land \left( p_{ndc, y} > -1 \right) \land \left( p_{ndc, y} < 1 \right)
    $$
    [S1, unofficial].

4.  **Dilate Masks:**
    Apply morphological dilation to each binary mask $M_i$ using a disk structuring element of radius $r=24$ [S1, unofficial]. Convert to float tensor.

5.  **Sample Masks at Projected Coordinates:**
    For each vertex $v$, sample the dilated mask $M_i$ at $p_{ndc}$ using `F.grid_sample` with `mode='nearest'`, `padding_mode='zeros'`, and `align_corners=True` [S1, unofficial]. Let $s_{i,v}$ be the sampled value.

6.  **Filter Vertices:**
    A vertex $v$ is valid if it is within NDC bounds for at least one view $i$ AND the sampled mask value is positive for that view:
    $$
    \text{vertex\_mask}_v = \bigvee_{i} \left( \text{valid}_{i,v} \land (s_{i,v} > 0) \right)
    $$
    [S1, unofficial].

7.  **Filter Faces:**
    A face $f = (v_a, v_b, v_c)$ is retained if all its vertices are valid:
    $$
    \text{face\_mask}_f = \text{vertex\_mask}_{v_a} \land \text{vertex\_mask}_{v_b} \land \text{vertex\_mask}_{v_c}
    $$
    [S1, unofficial].

8.  **Update and Export:**
    - Remove faces where `face_mask` is false.
    - Remove vertices that are no longer part of any retained face.
    - Transform remaining vertices back to world space using `scale_mat`:
      $$
      v_{world} = \text{scale\_mat} \times [v_x, v_y, v_z, 1]^T
      $$
      Specifically, scale by `scale_mat[0,0]` and translate by `scale_mat[:3,3]` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
from scipy import ndimage
import numpy as np

def mask_projection_culling(vertices, faces, cameras, masks, W=1600, H=1200, dilation_radius=24):
    """
    vertices: (N, 3) numpy array
    faces: (M, 3) numpy array
    cameras: list of dicts with 'world_mat', 'scale_mat', 'K', 'pose'
    masks: list of (H, W) numpy arrays (binary)
    """
    N = vertices.shape[0]
    vertex_valid = np.zeros(N, dtype=bool)
    
    # Precompute dilated masks
    dilated_masks = []
    for mask in masks:
        # Morphological dilation with disk structuring element
        # Note: scipy.ndimage.binary_dilation requires a structuring element
        # A disk can be approximated or created via meshgrid
        y, x = np.ogrid[:H, :W]
        center_y, center_x = H // 2, W // 2
        disk = (x - center_x)**2 + (y - center_y)**2 <= dilation_radius**2
        dilated = ndimage.binary_dilation(mask.astype(bool), structure=disk).astype(np.float32)
        dilated_masks.append(torch.tensor(dilated).unsqueeze(0).unsqueeze(0)) # (1, 1, H, W)
    
    # Process each camera view
    for i, cam in enumerate(cameras):
        # Extract parameters
        # P = world_mat @ scale_mat
        P = cam['world_mat'] @ cam['scale_mat']
        K, pose = load_K_Rt_from_P(None, P) # Helper function from rend_util
        W2C = torch.inverse(torch.tensor(pose, dtype=torch.float32)).cuda()
        K_tensor = torch.tensor(K, dtype=torch.float32).cuda()
        
        # Transform vertices to camera space
        # Add homogeneous coordinate
        vertices_h = torch.cat([torch.tensor(vertices, dtype=torch.float32).cuda(), torch.ones(N, 1).cuda()], dim=1)
        vertices_cam = (W2C @ vertices_h.T).T # (N, 4)
        
        # Project to pixel coordinates
        cam_points = K_tensor @ vertices_cam[:, :3].T # (3, N)
        pix_coords = cam_points[:2, :] / (cam_points[2, :].unsqueeze(0) + 1e-6) # (2, N)
        
        # Normalize to NDC
        pix_coords[0, :] /= (W - 1)
        pix_coords[1, :] /= (H - 1)
        pix_coords = (pix_coords - 0.5) * 2 # (2, N)
        
        # Check NDC bounds
        valid = ((pix_coords[0, :] > -1.0) & (pix_coords[0, :] < 1.0) &
                 (pix_coords[1, :] > -1.0) & (pix_coords[1, :] < 1.0))
        
        # Sample dilated mask
        # Grid sample expects (N, H, W, 2) or similar, but here we sample at specific points
        # F.grid_sample expects input (N, C, H, W) and grid (N, H_out, W_out, 2)
        # We need to reshape pix_coords to (1, 1, N, 2) for grid_sample
        grid = pix_coords.permute(1, 0).unsqueeze(0).unsqueeze(0) # (1, 1, N, 2)
        
        # Sample mask
        sampled = F.grid_sample(dilated_masks[i], grid, mode='nearest', padding_mode='zeros', align_corners=True) # (1, 1, N, 1)
        sampled_values = sampled.squeeze() # (N,)
        
        # Update vertex validity
        view_valid = valid.cpu().numpy() & (sampled_values.cpu().numpy() > 0.0)
        vertex_valid = vertex_valid | view_valid
        
    # Filter faces
    face_mask = vertex_valid[faces].all(axis=1)
    
    # Update mesh
    new_faces = faces[face_mask]
    # Get unique vertices in new faces
    unique_verts = np.unique(new_faces)
    # Create mapping from old index to new index
    vert_map = np.zeros(N, dtype=int)
    vert_map[unique_verts] = np.arange(len(unique_verts))
    new_vertices = vertices[unique_verts]
    new_faces_mapped = vert_map[new_faces]
    
    # Transform back to world space
    # scale_mat[0,0] is scale, scale_mat[:3,3] is translation
    scale = cam['scale_mat'][0, 0]
    translation = cam['scale_mat'][:3, 3]
    new_vertices_world = new_vertices * scale + translation
    
    return new_vertices_world, new_faces_mapped

def load_K_Rt_from_P(None, P):
    # Placeholder for rend_util.load_K_Rt_from_P
    # This function decomposes P into K and R,t
    # Implementation details depend on the specific library
    # For the purpose of this article, we assume it returns K and pose
    pass
```

**Parameters:**
- $W=1600, H=1200$: Image resolution [S1, unofficial].
- $r=24$: Dilation radius in pixels [S1, unofficial].
- $\epsilon=10^{-6}$: Epsilon for numerical stability [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Division by zero in projection is handled by $\epsilon$ [S1, unofficial].
- **Boundaries:** Points outside NDC bounds are explicitly excluded [S1, unofficial].
- **Degenerate Input:** If no vertices are visible in any mask, the output mesh is empty.
- **Mask Alignment:** Masks must be correctly aligned with the camera views. Misalignment leads to incorrect culling.

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$, where $N$ is the number of vertices and $C$ is the number of camera views. Each vertex is projected and sampled for each view.
- **Memory Complexity:** $O(N \cdot C)$ for storing intermediate projection results if processed in parallel, or $O(N + C \cdot H \cdot W)$ if masks are processed sequentially.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Effective at removing geometry outside the region of interest defined by reference images. Simple to implement with standard computer vision libraries.
- **Limitations:** Requires pre-existing camera parameters and masks. Computationally expensive for large meshes and many views. Does not account for occlusion between mesh parts, only visibility relative to the mask.
- **When to Use:** When a set of reference images with segmentation masks is available and the goal is to extract the visible surface.
- **When Not to Use:** When camera parameters are unknown or masks are not available. When occlusion handling is critical (consider ray casting instead).

## Background
Mask Projection Culling is used in the context of 3D Gaussian Splatting and mesh extraction from 2D Gaussian Splatting models. It was introduced in the `2d-gaussian-splatting` repository by hbb1 [S1, unofficial]. The method leverages the availability of camera parameters and masks generated during the training or evaluation phase to filter the resulting mesh.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
