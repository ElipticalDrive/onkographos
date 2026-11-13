# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing their depth against rendered depth maps or binary masks [S1, S2, unofficial]. It differs from neighbor-based culling (e.g., [Face Culling](8735-face-culling.md)) by relying on external camera viewpoints rather than local voxel topology, making it suitable for sparse or non-voxelized meshes where occlusion from specific viewpoints must be resolved [S1, S2, unofficial].

## How it works
**Inputs:**
*   A mesh defined by vertices $V \in \mathbb{R}^{N \times 3}$ and faces $F \in \mathbb{R}^{M \times 3}$.
*   A set of $V$ camera poses, each defined by a world-to-camera matrix $W2C$ and an intrinsic matrix $K$.
*   Optional: Binary masks $M_i$ for each view $i$, representing valid regions (e.g., foreground masks) [S1, unofficial].
*   Optional: Pre-rendered depth maps $D_i$ for each view $i$ [S2, unofficial].

**Outputs:**
*   A filtered mesh containing only vertices and faces visible in at least $T$ views (typically $T=20$) [S2, unofficial].

**Data Structures:**
*   `valid_counts`: Array of size $N$ (number of vertices), initialized to 0, storing the number of views in which each vertex is visible.
*   `face_mask`: Boolean array of size $M$, indicating if a face is kept.

**Parameters:**
*   `eps`: Depth comparison tolerance. Typical value: $0.005$ [S2, unofficial].
*   `points_batch_size`: Maximum points processed per batch to limit GPU memory. Typical value: $5 \times 10^5$ [S2, unofficial].
*   `remove_small_geometry_threshold`: Minimum area ratio for connected components (post-processing). Typical value: $0.2$ [S2, unofficial].
*   `mask_dilation_radius`: Radius for morphological dilation of binary masks. Typical value: $24$ [S1, unofficial].

**Steps:**

1.  **Camera Setup:**
    *   Load camera parameters. If using scale matrices, compute $P = \text{world\_mat} \times \text{scale\_mat}$ and extract intrinsics $K$ and pose $W2C$ [S1, unofficial].
    *   Handle coordinate system mismatches (e.g., OpenCV vs. OpenGL) by flipping Y and Z axes of the camera pose if necessary: $C2W[:3, 1:3] *= -1$ [S2, unofficial].
    *   Compute $W2C = C2W^{-1}$ [S1, S2, unofficial].

2.  **Depth Map Rendering (Optional but recommended for robustness):**
    *   Render forward depth maps $D_i$ for each camera pose using an offscreen renderer with flags `DEPTH_ONLY` and `SKIP_CULL_FACES` to ensure all geometry is rendered regardless of backface orientation [S2, unofficial].

3.  **Mask Preparation (Optional):**
    *   Load binary masks $M_i$ from image files.
    *   Dilate masks using morphology binary dilation with a disk of radius 24 to account for projection inaccuracies [S1, unofficial].
    *   Convert masks to float tensors [S1, unofficial].

4.  **Projection and Visibility Check:**
    For each view $i$ from $0$ to $V-1$:
    a.  **Transform to Camera Space:**
        Transform vertices $V$ to camera space:
        $$ V_{cam} = W2C_i \times V_{homogeneous} $$
        Where $V_{homogeneous}$ is $V$ with a column of 1s appended [S1, S2, unofficial].
    
    b.  **Project to 2D:**
        Project to pixel coordinates using intrinsics $K_i$:
        $$ uv_{hom} = K_i \times V_{cam} $$
        $$ z = uv_{hom}[:, 2] + 10^{-8} $$
        $$ u = uv_{hom}[:, 0] / z $$
        $$ v = uv_{hom}[:, 1] / z $$
        [S2, unofficial]
        
        *Alternative NDC normalization (if using grid sampling directly):*
        Normalize to $[-1, 1]$ range:
        $$ u_{ndc} = \frac{u}{W-1} \times 2 - 1 $$
        $$ v_{ndc} = \frac{v}{H-1} \times 2 - 1 $$
        [S1, S2, unofficial]

    c.  **Frustum Culling:**
        Identify points within the image bounds:
        $$ \text{in\_frustum} = (u \ge 0) \land (u \le W-1) \land (v \ge 0) \land (v \le H-1) \land (z > 0) $$
        [S2, unofficial]
        Or using NDC:
        $$ \text{valid} = ((u_{ndc} > -1) \land (u_{ndc} < 1) \land (v_{ndc} > -1) \land (v_{ndc} < 1)).\text{all}() $$
        [S1, unofficial]

    d.  **Depth/Mask Sampling:**
        *   **If using Depth Maps:** Sample depth at $(u, v)$ using bilinear interpolation with `padding_mode='border'` to handle out-of-bounds queries [S2, unofficial].
            $$ D_{sample} = \text{grid\_sample}(D_i, [u_{ndc}, v_{ndc}], \text{mode}='bilinear', \text{padding\_mode}='border') $$
            Determine if point is in front of surface:
            $$ \text{is\_front} = (D_{sample} > 0) \land (z < D_{sample} + \text{eps}) $$
            [S2, unofficial]
        *   **If using Binary Masks:** Sample the dilated mask at $(u_{ndc}, v_{ndc})$ using nearest neighbor interpolation [S1, unofficial].
            $$ M_{sample} = \text{grid\_sample}(M_i, [u_{ndc}, v_{ndc}], \text{mode}='nearest', \text{padding\_mode}='zeros') $$
            $$ \text{is\_valid} = M_{sample} > 0 $$
            [S1, unofficial]

    e.  **Accumulate Visibility:**
        For each vertex $j$, if $\text{is\_front}_j$ (or $\text{is\_valid}_j$) is true, increment `valid_counts[j]`.
        [S2, unofficial]

5.  **Filtering:**
    *   Identify vertices visible in at least $T$ views:
        $$ \text{keep\_vertices} = \text{valid\_counts} \ge T $$
        [S2, unofficial]
    *   Identify faces where all vertices are kept:
        $$ \text{keep\_faces} = \text{keep\_vertices}[F].\text{all}(\text{axis}=1) $$
        [S1, unofficial]
    *   Filter mesh vertices and faces.
    *   Transform remaining vertices back to world space using scale matrices if applicable [S1, unofficial].

6.  **Post-Processing (Optional):**
    *   Remove small disconnected components based on area ratio threshold [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(vertices, faces, cameras, depth_maps=None, masks=None, 
                            eps=0.005, threshold=20, batch_size=500000):
    """
    vertices: Tensor (N, 3)
    faces: Tensor (M, 3)
    cameras: List of dicts with 'K' (3,3), 'W2C' (4,4), 'H', 'W'
    depth_maps: List of Tensors (H, W) or None
    masks: List of Tensors (H, W) or None
    """
    N = vertices.shape[0]
    valid_counts = torch.zeros(N, dtype=torch.int32, device=vertices.device)
    
    # Ensure vertices are homogeneous
    vertices_h = torch.cat([vertices, torch.ones(N, 1, device=vertices.device)], dim=1)
    
    for i, cam in enumerate(cameras):
        K = cam['K'].to(vertices.device)
        W2C = cam['W2C'].to(vertices.device)
        H, W = cam['H'], cam['W']
        
        # Process in batches to limit memory
        for start in range(0, N, batch_size):
            end = min(start + batch_size, N)
            v_batch = vertices_h[start:end]
            
            # 1. Transform to camera space
            # v_cam = W2C @ v_h
            v_cam = torch.matmul(W2C, v_batch.t()).t() # (B, 4)
            
            # 2. Project to 2D
            uv_hom = torch.matmul(K, v_cam[:, :3].t()).t() # (B, 3)
            z = uv_hom[:, 2] + 1e-8
            u = uv_hom[:, 0] / z
            v = uv_hom[:, 1] / z
            
            # 3. Normalize to NDC [-1, 1] for grid_sample
            u_ndc = (u / (W - 1)) * 2.0 - 1.0
            v_ndc = (v / (H - 1)) * 2.0 - 1.0
            
            # 4. Frustum check
            in_frustum = (u >= 0) & (u <= W - 1) & (v >= 0) & (v <= H - 1) & (z > 0)
            
            # 5. Sample Depth or Mask
            is_visible = torch.zeros(end - start, dtype=torch.bool, device=vertices.device)
            
            if depth_maps is not None:
                # Sample depth
                vgrid = torch.stack([u_ndc, v_ndc], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, B, 2)
                # grid_sample expects (N, C, H, W) for input, so depth_maps[i] is (1, 1, H, W)
                depth_sample = torch.nn.functional.grid_sample(
                    depth_maps[i].unsqueeze(0).unsqueeze(0), 
                    vgrid, 
                    mode='bilinear', 
                    padding_mode='border', 
                    align_corners=True
                ).squeeze() # (B,)
                
                # Check if point is in front of surface
                is_visible = (depth_sample > 0) & (z < depth_sample + eps)
            elif masks is not None:
                # Sample mask
                vgrid = torch.stack([u_ndc, v_ndc], dim=-1).unsqueeze(0).unsqueeze(0)
                mask_sample = torch.nn.functional.grid_sample(
                    masks[i].unsqueeze(0).unsqueeze(0), 
                    vgrid, 
                    mode='nearest', 
                    padding_mode='zeros', 
                    align_corners=True
                ).squeeze() # (B,)
                
                is_visible = mask_sample > 0
            
            # Combine frustum and sampling visibility
            is_visible = is_visible & in_frustum
            
            # Accumulate counts
            valid_counts[start:end] += is_visible.int()
            
    # Filter vertices and faces
    keep_vertices = valid_counts >= threshold
    keep_faces = keep_vertices[faces].all(dim=1)
    
    filtered_vertices = vertices[keep_vertices]
    filtered_faces = faces[keep_faces]
    
    return filtered_vertices, filtered_faces
```

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5). Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader. This avoids the 6-fold driver overhead increase associated with splitting chunks into 6 sub-meshes [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size factor (related to grid sampling overhead) [S2, unofficial].
*   **Memory Complexity:** $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
*   **Strengths:** Effective for removing geometry occluded by other objects in the scene from specific viewpoints; robust to non-voxelized meshes; handles complex occlusions better than simple neighbor checks.
*   **Limitations:** Requires pre-existing camera poses and depth maps/masks; computationally expensive for large meshes and many views; sensitive to coordinate system mismatches and projection errors [S2, unofficial].
*   **When to use:** When post-processing a mesh reconstructed from multi-view images (e.g., NeRF, Gaussian Splatting) to remove background noise or occluded artifacts.
*   **When not to use:** For real-time rendering where per-frame culling is needed without pre-computation; for meshes where camera poses are unknown.

## Background
Mask Projection Culling is commonly used in the post-processing pipeline of 3D reconstruction methods such as 3D Gaussian Splatting and NeRF-based mesh extraction. It leverages the multi-view consistency of the reconstruction process to filter out spurious geometry. The technique is implemented in projects like `2d-gaussian-splatting` and `svraster-win10` to clean up extracted meshes [S1, S2, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
