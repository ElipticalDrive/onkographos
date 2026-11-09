# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps or binary masks [S1, S2, unofficial]. It differs from local topology-based methods like [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by relying on global camera viewpoints rather than immediate voxel neighbors, allowing it to remove geometry occluded by distant objects or outside the field of view. It is distinct from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) which typically uses pre-computed coverage masks, whereas this method actively renders depth maps or samples existing masks during the culling process [S1, S2, unofficial].

## How it works
The method filters a 3D mesh by validating the visibility of its vertices and faces against a set of reference camera views. It supports two primary validation modes: sampling pre-existing binary masks (often used in Gaussian Splatting pipelines) [S1, unofficial] or rendering forward depth maps from the mesh itself (used in differentiable rasterization pipelines) [S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Input mesh: Vertices $V \in \mathbb{R}^{N \times 3}$ and Faces $F \in \mathbb{R}^{M \times 3}$ (indices).
    *   Camera Data: A set of $V$ camera poses. For mask-based culling [S1, unofficial], this includes world-to-scale matrices and intrinsics. For depth-based culling [S2, unofficial], this includes camera-to-world matrices ($C2W$) and intrinsics ($K$).
    *   Reference Data: Either a directory of binary masks (`mask_dir/*.png`) [S1, unofficial] or the mesh itself for self-rendering [S2, unofficial].
*   **Outputs**:
    *   Filtered mesh: Vertices and faces where all constituent vertices are visible in at least $T$ views (typically $T=20$ [S2, unofficial]) and lie in front of the reference surface/mask.

### Data Structures
*   **Camera Pose**: A $4 \times 4$ matrix. In [S1, unofficial], computed as $P = \text{world\_mat} \times \text{scale\_mat}$. In [S2, unofficial], stored as $C2W$.
*   **Intrinsics**: $3 \times 3$ matrix $K$.
*   **Mask/Depth Map**: 2D array of size $H \times W$. For [S1, unofficial], binary float tensor. For [S2, unofficial], depth float tensor.
*   **Visibility Count**: Integer array of size $N$ (number of vertices), initialized to 0.

### Parameters
*   $W, H$: Image width and height. Typical values: $W=1600, H=1200$ [S1, unofficial].
*   $\epsilon$: Depth comparison tolerance. Typical value: $0.005$ [S2, unofficial].
*   $B$: Points batch size to limit GPU memory. Typical value: $5 \times 10^5$ [S2, unofficial].
*   $T_{min}$: Minimum number of views a vertex must be visible in. Typical value: $20$ [S2, unofficial].
*   $R_{dilate}$: Mask dilation radius. Typical value: $24$ [S1, unofficial].

### Steps

1.  **Initialize Visibility**: Create an integer array `valid_counts` of size $N$ initialized to zeros.
2.  **Iterate Cameras**: For each camera view $i$ from $0$ to $V-1$:
    a.  **Load Camera Parameters**:
        *   If using masks [S1, unofficial]: Load `world_mat` and `scale_mat`. Compute projection matrix $P = \text{world\_mat} \times \text{scale\_mat}$. Extract intrinsics $K$ and inverse pose $W2C = P^{-1}$ (using `rend_util.load_K_Rt_from_P`).
        *   If using depth [S2, unofficial]: Load $C2W$. Compute $W2C = C2W^{-1}$. Handle coordinate system mismatch by flipping Y and Z axes of the camera pose if necessary: $C2W[:3, 1:3] *= -1$ [S2, unofficial]. Extract $K$.
    b.  **Load Reference Data**:
        *   If using masks [S1, unofficial]: Load binary mask image. Dilate using `binary_dilation` with a disk of radius $R_{dilate}$. Convert to float tensor.
        *   If using depth [S2, unofficial]: Render the mesh from this camera pose using an offscreen renderer with flags `OFFSCREEN | DEPTH_ONLY | SKIP_CULL_FACES` to generate the depth map.
    c.  **Project Vertices**:
        *   Process vertices in batches of size $B$ to manage memory [S2, unofficial].
        *   Transform vertex coordinates to camera space: $P_{cam} = W2C \times V_{batch}$.
        *   Project to 2D:
            *   For masks [S1, unofficial]: $P_{ndc} = K \times P_{cam}$. Normalize to NDC $[-1, 1]$: $u = P_{ndc}[0] / (P_{ndc}[2] + 10^{-6})$, $v = P_{ndc}[1] / (P_{ndc}[2] + 10^{-6})$. Then $u_{ndc} = (u / (W-1) \times 2) - 1$, $v_{ndc} = (v / (H-1) \times 2) - 1$.
            *   For depth [S2, unofficial]: $uv = K \times P_{cam}$. $z = uv[:, 2] + 10^{-8}$. $u = uv[:, 0] / z$, $v = uv[:, 1] / z$. Normalize to $[-1, 1]$ for grid sampling: $u_{grid} = u / (W-1) \times 2 - 1$, $v_{grid} = v / (H-1) \times 2 - 1$.
    d.  **Check Frustum Bounds**:
        *   For masks [S1, unofficial]: Check if $u_{ndc} \in (-1, 1)$ and $v_{ndc} \in (-1, 1)$.
        *   For depth [S2, unofficial]: Check if $u \in [0, W-1]$, $v \in [0, H-1]$, and $z > 0$.
    e.  **Sample Visibility**:
        *   For masks [S1, unofficial]: Sample the dilated mask tensor at $(u_{ndc}, v_{ndc})$ using `grid_sample` with `mode='nearest'`, `padding_mode='zeros'`, `align_corners=True`. A point is valid in this view if the sampled value $> 0$.
        *   For depth [S2, unofficial]: Sample the depth map at $(u_{grid}, v_{grid})$ using `grid_sample` with `mode='bilinear'`, `padding_mode='border'`, `align_corners=True`. Compare camera-space depth $z$ with sampled depth $d_{sample}$. A point is in front of the surface if $d_{sample} > 0$ and $z < d_{sample} + \epsilon$.
    f.  **Update Counts**: Increment `valid_counts` for vertices that passed the frustum and visibility checks in this view.
3.  **Filter Mesh**:
    *   Identify vertices with `valid_counts` $\ge T_{min}$.
    *   Identify faces where all three vertices are in the valid vertex set.
    *   Discard invalid faces and unused vertices.
4.  **Transform and Export**:
    *   If the mesh was scaled during projection [S1, unofficial], transform remaining vertices back to world space using `scale_mat`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(mesh, cameras, mode='mask', dilate_radius=24, min_views=20, eps=0.005):
    """
    Filter mesh vertices and faces based on visibility across multiple camera views.
    
    Args:
        mesh: Object with .vertices (N,3) and .faces (M,3)
        cameras: List of dicts containing 'w2c' (4,4), 'K' (3,3), 'W', 'H', and optionally 'mask_path' or 'render_mesh'
        mode: 'mask' or 'depth'
        dilate_radius: Radius for mask dilation (mask mode only)
        min_views: Minimum number of views a vertex must be visible in
        eps: Depth tolerance (depth mode only)
        
    Returns:
        Filtered mesh vertices and faces
    """
    N = mesh.vertices.shape[0]
    valid_counts = np.zeros(N, dtype=np.int32)
    
    # Process in batches if necessary (simplified here for clarity)
    vertices = mesh.vertices
    
    for cam in cameras:
        w2c = cam['w2c']
        K = cam['K']
        W, H = cam['W'], cam['H']
        
        # Transform vertices to camera space
        # Homogeneous coordinates: (N, 4)
        verts_homo = np.hstack([vertices, np.ones((N, 1))])
        cam_coords = (w2c @ verts_homo.T).T[:, :3] # (N, 3)
        
        # Project to 2D
        proj = K @ cam_coords.T # (3, N)
        u = proj[0, :]
        v = proj[1, :]
        z = proj[2, :]
        
        # Avoid division by zero
        z_safe = np.where(z > 0, z, 1e-8)
        
        # Normalize to pixel coordinates
        u_px = u / z_safe
        v_px = v / z_safe
        
        # Check frustum bounds (pixel space)
        in_frustum = (u_px >= 0) & (u_px <= W - 1) & (v_px >= 0) & (v_px <= H - 1) & (z > 0)
        
        if mode == 'mask':
            # Load and dilate mask
            mask = load_and_dilate_mask(cam['mask_path'], dilate_radius) # Returns (H, W) float
            
            # Normalize to NDC [-1, 1] for grid sampling
            u_ndc = (u_px / (W - 1)) * 2.0 - 1.0
            v_ndc = (v_px / (H - 1)) * 2.0 - 1.0
            
            # Check NDC bounds
            in_ndc = (u_ndc > -1.0) & (u_ndc < 1.0) & (v_ndc > -1.0) & (v_ndc < 1.0)
            
            valid_in_view = in_frustum & in_ndc
            
            # Sample mask (conceptual, using nearest neighbor logic)
            # In implementation, use F.grid_sample or similar
            sampled_mask = sample_at_coords(mask, u_px, v_px, W, H)
            is_in_mask = sampled_mask > 0.0
            
            view_valid = valid_in_view & is_in_mask
            
        elif mode == 'depth':
            # Render depth map
            depth_map = render_depth(cam['render_mesh'], w2c, K, W, H) # (H, W)
            
            # Normalize to [-1, 1] for grid sampling
            u_grid = (u_px / (W - 1)) * 2.0 - 1.0
            v_grid = (v_px / (H - 1)) * 2.0 - 1.0
            
            # Sample depth with border padding to handle out-of-bounds
            sampled_depth = sample_at_coords(depth_map, u_grid, v_grid, W, H, padding='border')
            
            # Check if point is in front of surface
            # sampled_depth might be 0 for background/out-of-bounds if not handled by padding
            # Assuming padding returns a valid depth or we check > 0
            is_front = (sampled_depth > 0.0) & (z < sampled_depth + eps)
            
            view_valid = in_frustum & is_front
            
        else:
            raise ValueError("Unknown mode")
            
        valid_counts += view_valid.astype(np.int32)
        
    # Filter vertices
    valid_vertices_mask = valid_counts >= min_views
    valid_vertex_indices = np.where(valid_vertices_mask)[0]
    
    # Create mapping from old index to new index
    # This is complex if we want to preserve indices, but usually we just rebuild
    # For face culling, we check if all vertices of a face are valid
    
    # Filter faces
    faces = mesh.faces
    # Check if all 3 vertices of each face are in valid_vertices_mask
    face_valid = np.all(valid_vertices_mask[faces], axis=1)
    
    valid_faces = faces[face_valid]
    
    # Rebuild vertex array
    valid_vertices = vertices[valid_vertices_mask]
    
    # Remap face indices
    # Create a map from old index to new index
    index_map = np.full(N, -1, dtype=np.int32)
    index_map[valid_vertex_indices] = np.arange(len(valid_vertex_indices))
    valid_faces = index_map[valid_faces]
    
    return valid_vertices, valid_faces

def load_and_dilate_mask(path, radius):
    # Placeholder for mask loading and dilation logic
    pass

def sample_at_coords(data, u, v, W, H, padding='zeros'):
    # Placeholder for grid sampling logic
    pass
```

### Edge Cases and Pitfalls
*   **Coordinate System Mismatch**: OpenCV and OpenGL conventions differ in Y and Z axes. Failure to flip $C2W[:3, 1:3] *= -1$ can cause incorrect depth comparisons [S2, unofficial].
*   **Out-of-Bounds Sampling**: UV coordinates outside the image bounds can cause NaNs or crashes. Use `padding_mode='border'` in depth sampling [S2, unofficial] or `padding_mode='zeros'` in mask sampling [S1, unofficial].
*   **Precision Errors**: Small projection errors may cause points to fall slightly outside masks. Dilating the mask [S1, unofficial] or using a depth tolerance $\epsilon$ [S2, unofficial] mitigates this.
*   **Batch Size**: Processing all points at once may exceed GPU memory. Use a batch size $B$ (e.g., $5 \times 10^5$) [S2, unofficial].

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5) [S3, unofficial]. Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. It is chosen when driver overhead must be minimized compared to regular chunk meshing, which would require splitting chunks into 6 sub-meshes [S3, unofficial]. The implementation involves clearing a `groups` set, inserting group IDs based on camera position signs (e.g., if `cam::pos.x < 0` insert 0), and calling `vertpool.mask` with a lambda returning `groups.contains(cmd.group)` [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size [S2, unofficial].
*   **Memory Complexity**: $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
*   **Strengths**: Effective at removing geometry occluded by distant objects or outside the field of view, which local culling cannot do. Can leverage pre-existing masks or rendered depth for high accuracy [S1, S2, unofficial].
*   **Limitations**: Requires access to multiple camera views. Computationally expensive for large meshes due to per-vertex projection and sampling. Sensitive to camera pose accuracy and coordinate system conventions [S2, unofficial].
*   **When to Use**: When global visibility is required, such as in NeRF/Gaussian Splatting post-processing or when local topology culling leaves significant occluded geometry.
*   **When Not to Use**: For real-time rendering where per-frame culling overhead is prohibitive, or when camera poses are unavailable.

## Background
Mask Projection Culling originates from the post-processing steps of 3D Gaussian Splatting and differentiable rasterization pipelines [S1, S2, unofficial]. It was developed to clean up noisy geometry reconstructed from sparse or dense views by ensuring that only visible parts of the mesh are retained [S1, S2, unofficial]. The technique is closely related to visibility filtering in NeRF-based mesh extraction methods [S2, unofficial]. The variant "True Back-Face Culling via Group Masking" is described in the context of high-performance voxel engines to reduce driver overhead [S3, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
