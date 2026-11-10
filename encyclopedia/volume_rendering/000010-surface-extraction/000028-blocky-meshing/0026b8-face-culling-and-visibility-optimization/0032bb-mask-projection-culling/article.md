# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps or binary masks [S1, unofficial][S2, unofficial]. It differs from adjacency-based culling (e.g., [Face Culling](8735-face-culling.md)) by relying on explicit geometric visibility tests against known camera viewpoints rather than local voxel occupancy, making it suitable for reconstructing clean meshes from sparse or noisy input data where local topology is ambiguous.

## How it works
The method filters a mesh by determining which vertices and faces are visible in a set of reference views. It involves projecting 3D geometry into 2D camera space, sampling depth or mask information, and aggregating visibility counts across multiple frames.

### Inputs and Outputs
*   **Input:** A mesh (vertices $V$, faces $F$), a set of camera poses (world-to-camera matrices $W_i$, intrinsic matrices $K_i$), and optionally binary masks $M_i$ for each view.
*   **Output:** A filtered mesh containing only vertices and faces that are visible in at least $T$ views (where $T$ is a threshold, e.g., 20).

### Data Structures
*   **Camera Parameters:**
    *   `world_mat`: $4 \times 4$ matrix transforming world coordinates to camera coordinates.
    *   `scale_mat`: $4 \times 4$ matrix containing scale and translation for world-space normalization.
    *   `K`: $3 \times 3$ intrinsic matrix.
*   **Visibility Counters:**
    *   `valid_num`: Array of size $N$ (number of vertices) initialized to 0, storing the count of views in which each vertex is visible.
*   **Depth/Mask Maps:**
    *   `depth_map`: Image of size $H \times W$, storing depth values for each pixel.
    *   `mask`: Binary image of size $H \times W$, storing visibility masks (0 or 1).

### Steps

1.  **Initialize Visibility Counters:**
    Create an integer array `valid_num` of size $N$ (number of vertices in the mesh) and initialize all elements to 0.

2.  **Iterate Over Camera Views:**
    For each camera view $i$ from $0$ to $V-1$ (where $V$ is the number of validation frames):
    
    a. **Load Camera Parameters:**
    Extract the world-to-camera matrix $W_i$ and intrinsic matrix $K_i$. If the source provides world-to-scale matrices, compute $P = \text{world\_mat} \times \text{scale\_mat}$ and decompose to get $W_i$ and $K_i$ [S1, unofficial].
    
    b. **Render/Load Reference Data:**
    *   *Option A (Depth):* Render a forward depth map from the mesh for the current camera pose using an offscreen renderer with flags `DEPTH_ONLY` [S2, unofficial].
    *   *Option B (Mask):* Load a pre-computed binary mask $M_i$ from disk (e.g., `mask_dir/*.png`) with dimensions $W \times H$ (e.g., $1600 \times 1200$) [S1, unofficial].
    
    c. **Project Vertices to Camera Space:**
    For each vertex $v_j$ in the mesh:
    1.  Transform vertex to camera space:
        $$ \mathbf{p}_{cam} = W_i \times \begin{bmatrix} v_j \\ 1 \end{bmatrix} $$
        Let $\mathbf{p}_{cam} = [x_c, y_c, z_c, w_c]^T$.
    2.  Project to 2D pixel coordinates:
        $$ \mathbf{uv}_{raw} = K_i \times \begin{bmatrix} x_c/w_c \\ y_c/w_c \\ 1 \end{bmatrix} $$
        $$ u = \mathbf{uv}_{raw}[0], \quad v = \mathbf{uv}_{raw}[1], \quad z_{proj} = \mathbf{uv}_{raw}[2] $$
    3.  Normalize to NDC space $[-1, 1]$ for grid sampling:
        $$ u_{ndc} = \frac{u}{W - 1} \times 2.0 - 1.0 $$
        $$ v_{ndc} = \frac{v}{H - 1} \times 2.0 - 1.0 $$
    
    d. **Check Frustum Bounds:**
    Determine if the point is within the image bounds and in front of the camera:
    $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z_{proj} > 0) $$
    If not in frustum, skip this vertex for this view.

    e. **Sample Visibility:**
    *   *If using Masks:*
        1.  Dilate the binary mask $M_i$ using a morphological operation (e.g., disk of radius 24) to account for projection inaccuracies [S1, unofficial].
        2.  Sample the dilated mask at $(u_{ndc}, v_{ndc})$ using nearest-neighbor interpolation with zero padding:
            $$ \text{sampled\_mask} = \text{grid\_sample}(M_i^{\text{dilated}}, [u_{ndc}, v_{ndc}], \text{mode}=\text{'nearest'}, \text{padding\_mode}=\text{'zeros'}) $$
        3.  The vertex is visible in this view if $\text{sampled\_mask} > 0$.
    
    *   *If using Depth:*
        1.  Sample the depth map at $(u_{ndc}, v_{ndc})$ using bilinear interpolation with border padding to handle out-of-bounds queries:
            $$ d_{sampled} = \text{grid\_sample}(D_i, [u_{ndc}, v_{ndc}], \text{mode}=\text{'bilinear'}, \text{padding\_mode}=\text{'border'}) $$
        2.  Determine if the vertex is in front of the surface:
            $$ \text{is\_front} = (z_{proj} < d_{sampled} + \epsilon) $$
            where $\epsilon$ is a small tolerance (e.g., 0.005) [S2, unofficial].
        3.  The vertex is visible if $\text{is\_front}$ is true.

    f. **Update Counters:**
    If the vertex is visible in view $i$, increment `valid_num[j]`.

3.  **Filter Vertices:**
    After iterating over all views, identify vertices that are visible in at least $T$ views (e.g., $T=20$):
    $$ \text{valid\_vertices} = \{ j \mid \text{valid\_num}[j] \ge T \} $$

4.  **Filter Faces:**
    A face is retained if all its constituent vertices are in `valid_vertices`:
    $$ \text{valid\_faces} = \{ f \in F \mid \forall v \in f, v \in \text{valid\_vertices} \} $$

5.  **Reconstruct Mesh:**
    Create a new mesh using `valid_vertices` and `valid_faces`. Transform vertices back to world space if necessary using the inverse of the scale/translation matrices [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(mesh, cameras, masks=None, depth_maps=None, threshold=20, eps=0.005):
    """
    Filter mesh vertices and faces based on visibility across multiple camera views.
    
    Args:
        mesh: Object with vertices (N, 3) and faces (M, 3).
        cameras: List of dicts with 'K' (3,3), 'w2c' (4,4), 'scale_mat' (4,4).
        masks: List of binary masks (H, W) or None.
        depth_maps: List of depth maps (H, W) or None.
        threshold: Minimum number of views a vertex must be visible in.
        eps: Depth comparison tolerance.
        
    Returns:
        Filtered mesh vertices and faces.
    """
    N = mesh.vertices.shape[0]
    valid_num = np.zeros(N, dtype=np.int32)
    
    H, W = 1200, 1600  # Typical dimensions from sources [S1, unofficial]
    
    for i, cam in enumerate(cameras):
        K = cam['K'].float()
        w2c = cam['w2c'].float()
        
        # Handle coordinate system mismatch if necessary [S2, unofficial]
        # c2w = torch.inverse(w2c)
        # c2w[:3, 1:3] *= -1
        # w2c = torch.inverse(c2w)
        
        # Get depth or mask for this view
        if depth_maps is not None:
            depth_map = depth_maps[i].float()
            use_depth = True
        elif masks is not None:
            mask = masks[i].float()
            # Dilate mask if using masks [S1, unofficial]
            # mask_dilated = binary_dilation(mask, disk(24))
            use_depth = False
        else:
            raise ValueError("Either masks or depth_maps must be provided")
            
        # Process vertices in batches to limit GPU memory [S2, unofficial]
        batch_size = 500000
        for start in range(0, N, batch_size):
            end = min(start + batch_size, N)
            batch_verts = mesh.vertices[start:end] # (B, 3)
            
            # Homogeneous coordinates
            verts_homo = torch.cat([batch_verts, torch.ones(end-start, 1, device=batch_verts.device)], dim=1) # (B, 4)
            
            # Transform to camera space
            cam_verts = (w2c @ verts_homo.t()).t() # (B, 4)
            x_c, y_c, z_c, w_c = cam_verts[:, 0], cam_verts[:, 1], cam_verts[:, 2], cam_verts[:, 3]
            
            # Project to 2D
            uv_raw = K @ (cam_verts[:, :3] / w_c.unsqueeze(1)).t() # (3, B)
            u = uv_raw[0, :]
            v = uv_raw[1, :]
            z_proj = uv_raw[2, :]
            
            # Normalize to NDC [-1, 1]
            u_ndc = (u / (W - 1)) * 2.0 - 1.0
            v_ndc = (v / (H - 1)) * 2.0 - 1.0
            
            # Check frustum bounds
            in_frustum = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z_proj > 0)
            
            if use_depth:
                # Sample depth
                vgrid = torch.stack([u_ndc, v_ndc], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, B, 2)
                # depth_map needs to be (1, 1, H, W)
                depth_sampled = torch.nn.functional.grid_sample(
                    depth_map.unsqueeze(0).unsqueeze(0), 
                    vgrid, 
                    mode='bilinear', 
                    padding_mode='border', 
                    align_corners=True
                ).squeeze() # (B,)
                
                is_front = z_proj < (depth_sampled + eps)
                visible = in_frustum & is_front
            else:
                # Sample mask
                vgrid = torch.stack([u_ndc, v_ndc], dim=-1).unsqueeze(0).unsqueeze(0)
                # mask_dilated needs to be (1, 1, H, W)
                mask_sampled = torch.nn.functional.grid_sample(
                    mask_dilated.unsqueeze(0).unsqueeze(0), 
                    vgrid, 
                    mode='nearest', 
                    padding_mode='zeros', 
                    align_corners=True
                ).squeeze() # (B,)
                
                visible = in_frustum & (mask_sampled > 0.0)
            
            # Update valid counts
            valid_num[start:end] += visible.int().cpu().numpy()
            
    # Filter vertices
    valid_vertex_indices = np.where(valid_num >= threshold)[0]
    valid_vertex_set = set(valid_vertex_indices)
    
    # Filter faces
    valid_faces = []
    for face in mesh.faces:
        if all(v in valid_vertex_set for v in face):
            valid_faces.append(face)
            
    # Reconstruct mesh
    # Map old indices to new indices
    index_map = {old: new for new, old in enumerate(valid_vertex_indices)}
    new_vertices = mesh.vertices[valid_vertex_indices]
    new_faces = np.array([[index_map[v] for v in f] for f in valid_faces])
    
    return new_vertices, new_faces
```

### Parameters
*   `threshold`: Minimum number of views a vertex must be visible in. Typical value: 20 [S2, unofficial].
*   `eps`: Depth comparison tolerance. Typical value: 0.005 [S2, unofficial].
*   `points_batch_size`: Maximum number of points processed per batch. Typical value: 500,000 [S2, unofficial].
*   `mask_dilation_radius`: Radius for morphological dilation of binary masks. Typical value: 24 [S1, unofficial].

### Pitfalls
*   **Coordinate System Mismatch:** OpenCV and OpenGL conventions differ in Y and Z axes. Ensure camera poses are correctly transformed (e.g., flipping Y and Z) before projection [S2, unofficial].
*   **Out-of-Bounds Sampling:** UV coordinates outside $[0, W-1] \times [0, H-1]$ can cause NaNs. Use `padding_mode='border'` for depth and `padding_mode='zeros'` for masks [S2, unofficial][S1, unofficial].
*   **Precision Errors:** Projection and interpolation errors may cause visible vertices to be incorrectly culled. Use mask dilation or depth tolerance to mitigate [S1, unofficial][S2, unofficial].

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5) [S3, unofficial]. Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. This avoids the 6-fold increase in driver overhead that would result from splitting a chunk into 6 sub-meshes [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size [S2, unofficial].
*   **Memory Complexity:** $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
*   **Strengths:** Effective for removing noise and artifacts from reconstructed meshes; robust to local topology errors; leverages existing camera data.
*   **Limitations:** Requires a set of reference views; computationally expensive for large meshes and many views; sensitive to camera pose accuracy.
*   **When to Use:** When reconstructing meshes from multi-view stereo or NeRF-based methods where explicit visibility information is available.
*   **When Not to Use:** When real-time performance is critical and reference views are not available; when local topology is well-defined and adjacency culling is sufficient.

## Background
Mask Projection Culling is commonly used in 3D reconstruction pipelines, particularly those involving Gaussian Splatting or NeRF-based methods, to clean up extracted meshes [S1, unofficial][S2, unofficial]. It builds on the principles of visibility testing and depth comparison, adapting them for mesh filtering.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
