# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing their depth against rendered depth maps [S2, unofficial]. It differs from sibling methods by explicitly combining binary mask constraints to exclude background regions with depth-buffer comparisons to exclude back-facing or occluded geometry across a set of validation views [S1, S2, unofficial]. Unlike [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling-and-internal-face-masking.md) which operates on voxel primitives, or [Mesh Culling via 3D Projection and Connected Components](6963-mesh-culling-via-3d-projection-and-connected-components.md) which relies on connectivity, this method relies on per-vertex visibility counts derived from 2D projections [S1, S2, unofficial].

## How it works
**Inputs:**
*   A mesh $M$ with vertices $V$ and faces $F$ [S1, S2, unofficial].
*   A set of $V_{cam}$ camera poses (world-to-camera matrices) and intrinsic parameters $K$ [S1, S2, unofficial].
*   Binary masks $M_i$ for each view $i$, indicating valid regions [S1, unofficial].
*   Depth maps $D_i$ for each view $i$, rendered from the input mesh [S2, unofficial].
*   Image dimensions $W, H$ [S1, unofficial].

**Outputs:**
*   A filtered mesh containing only vertices and faces visible in at least $N_{min}$ views [S1, S2, unofficial].

**Data Structures:**
*   `valid_counts`: Array of size $|V|$, initialized to 0, storing the number of views in which each vertex is visible [S2, unofficial].
*   `depth_maps`: Tensor of shape $(V_{cam}, H, W)$, storing depth values for each view [S2, unofficial].
*   `masks`: Tensor of shape $(V_{cam}, H, W)$, storing binary mask values for each view [S1, unofficial].

**Steps:**

1.  **Preprocessing Cameras:**
    For each camera $i$, compute the world-to-camera matrix $W2C_i$ [S2, unofficial]. Extract intrinsics $K$ and pose from the combined projection matrix $P = W_{world} \cdot S_{scale}$ [S1, unofficial]. Handle coordinate system mismatches by flipping the Y and Z axes of the camera pose if necessary [S2, unofficial].

2.  **Render Reference Depth Maps:**
    Render a depth map $D_i$ from the input mesh for each camera pose using an offscreen renderer configured for `DEPTH_ONLY` mode [S2, unofficial].

3.  **Load and Dilate Masks:**
    Load binary masks $M_i$ from image files [S1, unofficial]. Dilate the masks using a morphological operation with a disk of radius $r=24$ to account for projection inaccuracies [S1, unofficial]. Convert masks to float tensors [S1, unofficial].

4.  **Project and Test Visibility (Per View):**
    For each camera view $i$ from $0$ to $V_{cam}-1$:
    a.  **Transform:** Transform mesh vertices $v \in V$ to camera space:
        $$ v_{cam} = W2C_i \cdot v_{world} $$
        [S1, S2, unofficial]
    b.  **Project:** Project camera-space vertices to 2D UV coordinates:
        $$ \begin{bmatrix} u' \\ v' \\ z \end{bmatrix} = K \cdot v_{cam} $$
        $$ u = \frac{u'}{z}, \quad v = \frac{v'}{z} $$
        [S1, S2, unofficial]
        Add a small epsilon ($10^{-6}$) to $z$ to prevent division by zero [S1, unofficial].
    c.  **Frustum Check:** Determine if the point is within the image bounds and in front of the camera:
        $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z > 0) $$
        [S2, unofficial]
    d.  **Sample Mask:** Sample the dilated binary mask $M_i$ at the projected coordinates $(u, v)$ using nearest-neighbor interpolation with zero-padding [S1, unofficial].
        $$ m_{sample} = \text{GridSample}(M_i, (u, v), \text{mode}=\text{nearest}, \text{padding}=\text{zeros}) $$
        [S1, unofficial]
    e.  **Sample Depth:** Sample the reference depth map $D_i$ at $(u, v)$ using bilinear interpolation with border padding [S2, unofficial].
        $$ d_{sample} = \text{GridSample}(D_i, (u, v), \text{mode}=\text{bilinear}, \text{padding}=\text{border}) $$
        [S2, unofficial]
    f.  **Depth Test:** Determine if the vertex is in front of the sampled surface. If $d_{sample} > 0$, the vertex is visible if $z < d_{sample} + \epsilon$. If $d_{sample} \le 0$, the vertex is considered front-facing [S2, unofficial].
        $$ \text{is\_front} = \begin{cases} z < d_{sample} + \epsilon & \text{if } d_{sample} > 0 \\ \text{True} & \text{otherwise} \end{cases} $$
        [S2, unofficial]
    g.  **Aggregate:** If $\text{in\_frustum} \land (m_{sample} > 0) \land \text{is\_front}$, increment `valid_counts` for the vertex [S1, S2, unofficial].

5.  **Filter Mesh:**
    a.  Identify vertices where `valid_counts` $\ge N_{min}$ (e.g., $N_{min}=20$) [S2, unofficial].
    b.  Identify faces where all three vertices are valid [S1, unofficial].
    c.  Construct the new mesh with filtered vertices and faces [S1, unofficial].
    d.  Transform vertices back to world space using scale factors if necessary [S1, unofficial].

**Parameters:**
*   $\epsilon$: Depth comparison tolerance. Typical value: $0.005$ [S2, unofficial].
*   $N_{min}$: Minimum number of views a vertex must be visible in. Typical value: $20$ [S2, unofficial].
*   $r$: Mask dilation radius. Typical value: $24$ pixels [S1, unofficial].
*   `points_batch_size`: Max points processed per batch to limit GPU memory. Typical value: $5 \times 10^5$ [S2, unofficial].
*   `remove_small_geometry_threshold`: Minimum area ratio for a connected component to be retained. Typical value: $0.2$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def dilate_mask(mask, radius):
    """Simple binary dilation using max-pooling [G]."""
    kernel_size = 2 * radius + 1
    kernel = torch.ones(1, 1, kernel_size, kernel_size, device=mask.device)
    dilated = F.max_pool2d(mask.unsqueeze(0).unsqueeze(0), kernel_size=kernel_size, stride=1, padding=radius)
    return dilated.squeeze(0).squeeze(0)

def mask_projection_culling(mesh, cameras, masks, depth_maps, W, H, eps=0.005, min_views=20, dilation_radius=24):
    """
    mesh: dict with 'vertices' (N, 3) and 'faces' (F, 3)
    cameras: list of dicts with 'w2c' (4, 4) and 'K' (3, 3)
    masks: list of tensors (H, W) binary masks
    depth_maps: list of tensors (H, W) float depth maps
    """
    V = mesh['vertices']
    N = V.shape[0]
    valid_counts = torch.zeros(N, dtype=torch.int32, device=V.device)
    
    # Dilate masks
    dilated_masks = [dilate_mask(m.float(), dilation_radius) for m in masks]
    
    # Process in batches to save memory
    batch_size = 500000
    for start in range(0, N, batch_size):
        end = min(start + batch_size, N)
        batch_verts = V[start:end]
        B = end - start
        
        for i, cam in enumerate(cameras):
            w2c = cam['w2c']
            K = cam['K']
            
            # Transform to camera space
            verts_h = torch.cat([batch_verts, torch.ones(B, 1, device=V.device)], dim=1)
            cam_verts = (w2c @ verts_h.t()).t()[:, :3]
            
            # Project
            cam_verts_h = torch.cat([cam_verts, torch.ones(B, 1, device=V.device)], dim=1)
            proj = (K @ cam_verts_h.t()).t()
            z = proj[:, 2] + 1e-6
            u = proj[:, 0] / z
            v = proj[:, 1] / z
            
            # Frustum check
            in_frustum = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z > 0)
            
            # Normalize to [-1, 1] for grid_sample
            u_ndc = (2.0 * u / (W - 1)) - 1.0
            v_ndc = (2.0 * v / (H - 1)) - 1.0
            grid = torch.stack([u_ndc, v_ndc], dim=-1).unsqueeze(0) # (1, B, 1, 2)
            
            # Sample Mask
            mask_tensor = dilated_masks[i].unsqueeze(0).unsqueeze(0) # (1, 1, H, W)
            mask_sample = F.grid_sample(mask_tensor, grid, mode='nearest', padding_mode='zeros', align_corners=True).squeeze().squeeze() # (B,)
            
            # Sample Depth
            depth_tensor = depth_maps[i].unsqueeze(0).unsqueeze(0)
            depth_sample = F.grid_sample(depth_tensor, grid, mode='bilinear', padding_mode='border', align_corners=True).squeeze().squeeze() # (B,)
            
            # Depth Test
            is_front = torch.where(depth_sample > 0.0, z < (depth_sample + eps), torch.ones(B, dtype=torch.bool, device=V.device))
            
            # Combine conditions
            visible = in_frustum & (mask_sample > 0.5) & is_front
            
            # Update counts
            valid_counts[start:end] += visible.int()
            
    # Filter vertices
    valid_verts_mask = valid_counts >= min_views
    valid_vert_indices = torch.where(valid_verts_mask)[0]
    
    # Create mapping from old index to new index
    new_vert_map = torch.full((N,), -1, dtype=torch.long, device=V.device)
    new_vert_map[valid_vert_indices] = torch.arange(len(valid_vert_indices), device=V.device)
    
    # Filter faces
    faces = mesh['faces']
    face_valid = valid_verts_mask[faces].all(dim=1)
    valid_faces = faces[face_valid]
    valid_faces = new_vert_map[valid_faces]
    
    # Return new mesh
    new_verts = V[valid_vert_indices]
    return {'vertices': new_verts, 'faces': valid_faces}
```

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5) [S3, unofficial]. Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. This avoids the driver overhead of splitting a chunk into 6 sub-meshes, which would increase overhead 6-fold [S3, unofficial]. It is chosen when rendering performance is critical and the scene can be decomposed into chunks with known orientations [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V_{cam} \cdot B)$, where $N$ is the number of points, $V_{cam}$ is the number of validation frames, and $B$ is the batch size [S2, unofficial].
*   **Memory Complexity:** $O(N \cdot V_{cam})$ for intermediate masks and valid counts [S2, unofficial].
*   **Strengths:** Effective at removing noise and background artifacts in reconstructed meshes from multi-view stereo or NeRF-based pipelines [S1, S2, unofficial]. Robust to small misalignments due to mask dilation [S1, unofficial].
*   **Limitations:** Requires rendering depth maps for all views, which can be expensive for large meshes and many views [S2, unofficial]. Sensitive to coordinate system mismatches if not handled correctly [S2, unofficial]. Performance degrades with a large number of views [S2, unofficial].
*   **When to use:** When reconstructing meshes from multi-view images (e.g., NeRF, 3DGS) and significant background noise or floating artifacts are present [S1, S2, unofficial].
*   **When not to use:** When real-time performance is required for the culling step itself, or when the number of views is very large [S2, unofficial].

## Background
Mask Projection Culling is primarily used in mesh extraction pipelines from multi-view data [S1, S2, unofficial]. It builds on the principles of projection-based culling [16407-projection-based-culling.md]. The technique leverages the fact that valid geometry should be consistently visible across multiple views and should not be occluded by the reference depth [S1, S2, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
