# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting points into camera space and comparing their depth against rendered depth maps or binary masks [S1, S2, unofficial]. It differs from neighbor-occupancy checks [Face Culling](8735-face-culling.md) by relying on external camera viewpoints rather than local voxel topology, and from simple back-face culling [Voxel Backface Culling and Internal Face Masking](4569-voxel-backface-culling.md) by handling complex occlusions and non-convex geometries through multi-view consistency.

## How it works
**Inputs:**
- A mesh (vertices $V$, faces $F$).
- A set of $V$ camera poses (world-to-camera matrices $W_i$) and intrinsic matrices $K_i$.
- Optional: Binary masks $M_i$ for each view (e.g., from segmentation or depth maps).
- Parameters: `eps` (depth tolerance, typically 0.005 [S2, unofficial]), `points_batch_size` (GPU memory limit, typically $5 \times 10^5$ [S2, unofficial]), `min_visibility_frames` (minimum views a point must be visible in, typically 20 [S2, unofficial]).

**Data Structures:**
- `valid_counts`: Array of size $N$ (number of vertices), initialized to 0.
- `face_valid`: Boolean array of size $|F|$, initialized to false.

**Steps:**
1. **Load Camera Parameters:** Extract world matrices and intrinsics. For each camera $i$, compute the world-to-camera matrix $W_i$ and intrinsic $K_i$. If using OpenCV/OpenGL mixed conventions, flip Y and Z axes of the camera pose to ensure correct depth comparison [S2, unofficial].
2. **Initialize Visibility Counts:** Create an integer array `valid_counts` of length $N$ (number of vertices), initialized to zero.
3. **Process Each Camera View:** For each camera $i$ from $0$ to $V-1$:
    a. **Transform Vertices to Camera Space:**
       For each vertex $v \in V$ with world coordinates $\mathbf{p}_w$:
       $$ \mathbf{p}_c = W_i \cdot \begin{bmatrix} \mathbf{p}_w \\ 1 \end{bmatrix} $$
       Let $\mathbf{p}_c = (x_c, y_c, z_c, w_c)$.
    b. **Project to 2D:**
       Compute homogeneous coordinates:
       $$ \mathbf{p}_{homo} = K_i \cdot \begin{bmatrix} x_c/w_c \\ y_c/w_c \\ 1 \end{bmatrix} $$
       Or directly using the provided code logic which often assumes pre-homogenized or uses a specific pipeline. Based on [S1, unofficial] and [S2, unofficial], the projection is:
       $$ \mathbf{p}_{cam} = K_i \cdot W_i \cdot \mathbf{p}_w^{homog} $$
       $$ u = \frac{\mathbf{p}_{cam}[0]}{\mathbf{p}_{cam}[2] + \epsilon} $$
       $$ v = \frac{\mathbf{p}_{cam}[1]}{\mathbf{p}_{cam}[2] + \epsilon} $$
       where $\epsilon = 10^{-6}$ to avoid division by zero [S1, unofficial].
    c. **Check Frustum Bounds:**
       Determine if the point is within the image bounds $[0, W-1] \times [0, H-1]$ and in front of the camera ($z_c > 0$):
       $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (z_c > 0) $$
    d. **Sample Visibility:**
       - **Option A (Mask-based [S1, unofficial]):** If binary masks $M_i$ are available, dilate $M_i$ (e.g., disk radius 24) and sample at $(u, v)$ using nearest-neighbor interpolation. If the sampled value is $>0$, the point is visible in this view.
       - **Option B (Depth-based [S2, unofficial]):** Render or load a depth map $D_i$ for the scene from camera $i$. Sample the depth at $(u, v)$ using bilinear interpolation with border padding to handle out-of-bounds queries (preventing NaNs) [S2, unofficial]. Let $d_{sampled}$ be the sampled depth.
         Compare the vertex's camera-space depth $z_c$ with $d_{sampled}$:
         $$ \text{is\_front} = (d_{sampled} > 0) \land (z_c < d_{sampled} + \text{eps}) $$
         If `is_front` is true, the vertex is considered visible in this view.
    e. **Update Counts:**
       If the vertex is visible in view $i$, increment `valid_counts[j]` for vertex $j$.
4. **Filter Vertices:**
   Identify vertices that are visible in at least `min_visibility_frames` views:
   $$ \text{valid\_vertices} = \{ j \mid \text{valid\_counts}[j] \ge \text{min\_visibility\_frames} \} $$
5. **Filter Faces:**
   A face is valid if all its constituent vertices are in `valid_vertices`:
   $$ \text{face\_mask}[f] = \bigwedge_{v \in f} (v \in \text{valid\_vertices}) $$
6. **Export:**
   Create a new mesh containing only valid faces and their vertices. Transform vertices back to world space if necessary (e.g., applying scale matrices) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(mesh, cameras, depth_maps=None, masks=None, eps=0.005, min_visibility=20, batch_size=500000):
    """
    mesh: dict with 'vertices' (N, 3) and 'faces' (F, 3)
    cameras: list of dicts with 'w2c' (4, 4), 'K' (3, 3), 'W' (int), 'H' (int)
    depth_maps: list of tensors (V, H, W) or None
    masks: list of tensors (V, H, W) or None
    """
    N = mesh['vertices'].shape[0]
    valid_counts = np.zeros(N, dtype=np.int32)
    
    # Process in batches to limit GPU memory
    for batch_start in range(0, N, batch_size):
        batch_end = min(batch_start + batch_size, N)
        batch_vertices = mesh['vertices'][batch_start:batch_end] # (B, 3)
        
        # Initialize visibility for this batch across all views
        batch_valid = np.zeros(batch_end - batch_start, dtype=bool)
        
        for i, cam in enumerate(cameras):
            w2c = cam['w2c']
            K = cam['K']
            W_img = cam['W']
            H_img = cam['H']
            
            # Transform to camera space
            # Add homogeneous coordinate
            verts_homog = np.hstack([batch_vertices, np.ones((batch_vertices.shape[0], 1))]) # (B, 4)
            cam_coords = (w2c @ verts_homog.T).T # (B, 4)
            z_c = cam_coords[:, 2]
            
            # Project to 2D
            # K @ (x/z, y/z, 1)
            # Handle division by zero with epsilon
            z_safe = z_c + 1e-6
            uv_homog = K @ (cam_coords[:, :3] / z_safe[:, np.newaxis]).T # (3, B)
            u = uv_homog[0, :]
            v = uv_homog[1, :]
            
            # Check frustum
            in_frustum = (u >= 0) & (u < W_img) & (v >= 0) & (v < H_img) & (z_c > 0)
            
            is_visible = np.zeros(batch_vertices.shape[0], dtype=bool)
            
            if masks is not None:
                # Mask-based culling [S1, unofficial]
                # Note: In practice, this requires sampling the mask at (u, v)
                # Here we assume a helper function for grid sampling
                mask = masks[i]
                # Normalize to [-1, 1] for grid_sample
                u_norm = (u / (W_img - 1)) * 2.0 - 1.0
                v_norm = (v / (H_img - 1)) * 2.0 - 1.0
                vgrid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, B, 2)
                mask_tensor = torch.from_numpy(mask).unsqueeze(0).unsqueeze(0).float() # (1, 1, H, W)
                # Sample
                sampled = torch.nn.functional.grid_sample(mask_tensor, vgrid, mode='nearest', padding_mode='zeros', align_corners=True)
                sampled_mask = sampled.squeeze().cpu().numpy() # (B,)
                is_visible = sampled_mask > 0.0
            else:
                # Depth-based culling [S2, unofficial]
                if depth_maps is None:
                    continue # Cannot cull without depth or mask
                
                depth_map = depth_maps[i]
                # Normalize to [-1, 1]
                u_norm = (u / (W_img - 1)) * 2.0 - 1.0
                v_norm = (v / (H_img - 1)) * 2.0 - 1.0
                vgrid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(0).unsqueeze(0)
                depth_tensor = torch.from_numpy(depth_map).unsqueeze(0).unsqueeze(0).float()
                sampled_depth = torch.nn.functional.grid_sample(depth_tensor, vgrid, mode='bilinear', padding_mode='border', align_corners=True)
                sampled_depth = sampled_depth.squeeze().cpu().numpy() # (B,)
                
                # Compare depth
                # is_front_face = (sampled_depth > 0) & (z_c < sampled_depth + eps)
                is_visible = (sampled_depth > 0) & (z_c < sampled_depth + eps)
            
            # Combine with frustum check
            is_visible = is_visible & in_frustum
            
            # Update counts
            valid_counts[batch_start:batch_end] += is_visible.astype(np.int32)
            
    # Filter vertices
    valid_vertex_indices = np.where(valid_counts >= min_visibility)[0]
    valid_vertex_set = set(valid_vertex_indices)
    
    # Filter faces
    valid_faces = []
    for face in mesh['faces']:
        if all(v in valid_vertex_set for v in face):
            valid_faces.append(face)
            
    # Re-index vertices
    vertex_map = {old: new for new, old in enumerate(valid_vertex_indices)}
    new_vertices = mesh['vertices'][valid_vertex_indices]
    new_faces = np.array([[vertex_map[v] for v in f] for f in valid_faces])
    
    return {'vertices': new_vertices, 'faces': new_faces}
```

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5) [S3, unofficial]. Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. This avoids the 6-fold driver overhead increase associated with splitting chunks into 6 sub-meshes [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V \cdot B)$, where $N$ is the number of points/vertices, $V$ is the number of validation frames (cameras), and $B$ is the batch size [S2, unofficial].
- **Memory Complexity:** $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
- **Strengths:** Effective for complex, non-convex geometries where local neighbor checks fail; robust against self-occlusion; leverages GPU parallelism for projection and sampling.
- **Limitations:** Requires pre-computed depth maps or masks for all views; sensitive to camera calibration errors; coordinate system mismatches (e.g., OpenCV vs OpenGL) can cause incorrect depth comparisons if not handled [S2, unofficial].
- **When to use:** When generating static meshes from multi-view reconstructions (e.g., NeRF, Gaussian Splatting) where visibility is defined by the capture setup.
- **When not to use:** For dynamic scenes with changing viewpoints where pre-computed masks are not feasible; for simple convex voxel structures where neighbor checks [Face Culling](8735-face-culling.md) are sufficient.

## Background
Mask Projection Culling originates from the post-processing steps of neural rendering pipelines, specifically 3D Gaussian Splatting and NeRF-based mesh extraction, where high-density point clouds or Gaussians need to be converted to manifold meshes [S1, S2, unofficial]. The technique was adapted to filter out "ghost" geometry and noise by ensuring that only geometry consistently visible across multiple training views is retained [S1, S2, unofficial]. The variant "True Back-Face Culling via Group Masking" is described in the context of high-performance voxel engines to optimize draw calls by leveraging vertex bucketing [S3, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
