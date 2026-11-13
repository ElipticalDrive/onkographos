# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes faces and vertices not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing their depth against rendered depth maps or binary masks [S1, unofficial][S2, unofficial]. It solves the problem of retaining only the geometry visible from a set of training or reference views, effectively pruning occluded or background artifacts from the extracted mesh. Unlike topological culling which relies on neighbor occupancy, this method uses explicit view-dependent projection and depth testing [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- A triangular mesh (vertices $V$, faces $F$).
- A set of $V$ camera poses (world-to-camera matrices $W_i$) and intrinsic matrices $K_i$ for $i \in \{0, \dots, V-1\}$.
- Optional: Binary masks $M_i$ for each view, indicating valid regions (e.g., foreground).
- Optional: Rendered depth maps $D_i$ for each view.
- Parameters: `eps` (depth tolerance, typically 0.005 [S2, unofficial]), `points_batch_size` (typically $5 \times 10^5$ [S2, unofficial]), `remove_small_geometry_threshold` (typically 0.2 [S2, unofficial]).

**Outputs:**
- A filtered mesh containing only vertices and faces deemed visible in at least $N_{min}$ views (e.g., 20 [S2, unofficial]).

**Data Structures:**
- `valid_counts`: Array of size $|V|$, initialized to 0, storing the number of views each vertex is visible in.
- `face_mask`: Boolean array of size $|F|$, indicating if a face should be retained.

**Steps:**

1. **Initialize Visibility Counts:**
   Create an integer array `valid_counts` of length equal to the number of vertices in the mesh, initialized to zero.

2. **Iterate Over Views:**
   For each camera view $i$ from $0$ to $V-1$:
   
   a. **Coordinate Transformation:**
      Compute the world-to-camera matrix $W_i$ (inverse of the camera-to-world matrix).
      *Note: Handle coordinate system mismatches (e.g., OpenCV vs OpenGL) by flipping Y and Z axes of the camera pose if necessary [S2, unofficial].*
      
   b. **Project Vertices:**
      For each vertex $v \in V$ with world coordinates $\mathbf{p}_w$:
      1. Transform to camera space: $\mathbf{p}_c = W_i \cdot [\mathbf{p}_w, 1]^T$.
      2. Project to 2D using intrinsic matrix $K_i$: $\mathbf{p}_{uv} = K_i \cdot \mathbf{p}_c$.
      3. Perform perspective division to get pixel coordinates $(u, v)$:
         $$ u = \frac{\mathbf{p}_{uv}[0]}{\mathbf{p}_{uv}[2] + \epsilon} $$
         $$ v = \frac{\mathbf{p}_{uv}[1]}{\mathbf{p}_{uv}[2] + \epsilon} $$
         where $\epsilon = 10^{-6}$ to prevent division by zero.
      4. Normalize to NDC space $[-1, 1]$ for grid sampling (assuming image dimensions $W, H$):
         $$ u_{ndc} = \frac{u}{W - 1} \cdot 2 - 1 $$
         $$ v_{ndc} = \frac{v}{H - 1} \cdot 2 - 1 $$
      
   c. **Frustum and Mask Check:**
      Determine if the vertex is within the image bounds and in front of the camera:
      $$ \text{in\_frustum} = (0 \le u < W) \land (0 \le v < H) \land (\mathbf{p}_{uv}[2] > 0) $$
      
      If binary masks $M_i$ are provided:
      1. Dilate the binary mask $M_i$ using a morphological operation (e.g., disk of radius 24 [S1, unofficial]).
      2. Sample the dilated mask at $(u_{ndc}, v_{ndc})$ using nearest-neighbor interpolation.
      3. The vertex is considered "masked" if the sampled value is $> 0$.
      
      If depth maps $D_i$ are provided:
      1. Sample the depth map $D_i$ at $(u_{ndc}, v_{ndc})$ using bilinear interpolation with border padding to handle out-of-bounds queries [S2, unofficial].
      2. Compare the vertex's camera-space depth $z = \mathbf{p}_{uv}[2]$ with the sampled depth $z_{sampled}$:
         $$ \text{is\_front} = (z_{sampled} > 0) \land (z < z_{sampled} + \text{eps}) $$
         where `eps` is the depth tolerance (typically 0.005 [S2, unofficial]).

   d. **Update Counts:**
      For each vertex $v$, if it is within the frustum and passes the mask/depth check, increment `valid_counts[v]`.

3. **Filter Vertices:**
   Define a threshold $N_{min}$ (e.g., 20 [S2, unofficial]).
   Create a boolean array `vertex_valid` where `vertex_valid[v]` is true if `valid_counts[v] >= N_min`.

4. **Filter Faces:**
   Create a boolean array `face_valid` of size $|F|$.
   For each face $f$ with vertex indices $(v_1, v_2, v_3)$:
   $$ \text{face\_valid}[f] = \text{vertex\_valid}[v_1] \land \text{vertex\_valid}[v_2] \land \text{vertex\_valid}[v_3] $$

5. **Post-Processing (Optional):**
   Identify connected components in the filtered mesh. Remove components whose area is less than `remove_small_geometry_threshold` times the total mesh area [S2, unofficial].

6. **Export:**
   Construct the new mesh using only vertices where `vertex_valid` is true and faces where `face_valid` is true. Transform vertices back to world space if they were scaled during projection (e.g., using scale matrices [S1, unofficial]).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(mesh, cameras, masks=None, depth_maps=None, 
                            eps=0.005, n_min=20, batch_size=500000):
    """
    mesh: dict with 'vertices' (N, 3) and 'faces' (M, 3)
    cameras: list of dicts with 'w2c' (4, 4) and 'K' (3, 3)
    masks: list of tensors (H, W) or None
    depth_maps: list of tensors (H, W) or None
    """
    vertices = mesh['vertices']
    faces = mesh['faces']
    num_vertices = vertices.shape[0]
    num_views = len(cameras)
    
    # Initialize visibility counts
    valid_counts = np.zeros(num_vertices, dtype=np.int32)
    
    # Process in batches to limit GPU memory
    for batch_start in range(0, num_vertices, batch_size):
        batch_end = min(batch_start + batch_size, num_vertices)
        batch_vertices = vertices[batch_start:batch_end]
        
        # Initialize batch counts
        batch_valid_counts = np.zeros(batch_end - batch_start, dtype=np.int32)
        
        for i, cam in enumerate(cameras):
            w2c = cam['w2c']
            K = cam['K']
            
            # Transform vertices to camera space
            # Add homogeneous coordinate
            verts_hom = np.hstack([batch_vertices, np.ones((batch_vertices.shape[0], 1))])
            cam_coords = (w2c @ verts_hom.T).T # (N, 4)
            
            # Project to 2D
            uv_hom = (K @ cam_coords[:, :3].T).T # (N, 3)
            z = uv_hom[:, 2] + 1e-6
            
            u = uv_hom[:, 0] / z
            v = uv_hom[:, 1] / z
            
            # Check frustum bounds
            W, H = depth_maps[i].shape if depth_maps is not None else (1600, 1200) # Default from S1
            in_frustum = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z > 0)
            
            # Check mask or depth
            view_valid = np.zeros(batch_end - batch_start, dtype=bool)
            
            if masks is not None:
                # Sample mask (simplified, assuming mask is already dilated and on GPU/CPU as needed)
                # In practice, use grid_sample or similar for nearest neighbor
                # Here we assume a helper function for sampling
                mask_vals = sample_mask(masks[i], u, v, W, H)
                view_valid = in_frustum & (mask_vals > 0)
            elif depth_maps is not None:
                # Sample depth
                depth_vals = sample_depth(depth_maps[i], u, v, W, H)
                # Check if vertex is in front of surface
                is_front = (depth_vals > 0) & (z < depth_vals + eps)
                view_valid = in_frustum & is_front
            
            # Update counts for this view
            batch_valid_counts += view_valid.astype(np.int32)
        
        # Accumulate to global counts
        valid_counts[batch_start:batch_end] += batch_valid_counts
        
    # Filter vertices
    vertex_valid = valid_counts >= n_min
    
    # Filter faces
    face_valid = vertex_valid[faces].all(axis=1)
    
    # Create new mesh
    new_vertices = vertices[vertex_valid]
    # Remap face indices
    vertex_map = np.zeros(num_vertices, dtype=np.int64) - 1
    vertex_map[vertex_valid] = np.arange(vertex_valid.sum())
    new_faces = vertex_map[faces[face_valid]]
    
    return {'vertices': new_vertices, 'faces': new_faces}

def sample_mask(mask, u, v, W, H):
    # Normalize to NDC
    u_ndc = (u / (W - 1)) * 2 - 1
    v_ndc = (v / (H - 1)) * 2 - 1
    # Stack for grid_sample
    grid = torch.stack([u_ndc, v_ndc], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, N, 2)
    # Assuming mask is a tensor on the same device
    # Use nearest interpolation
    # Note: This is a conceptual representation; actual implementation uses F.grid_sample
    # sampled = F.grid_sample(mask.unsqueeze(0).unsqueeze(0), grid, mode='nearest', padding_mode='zeros', align_corners=True)
    # return sampled.squeeze()
    pass

def sample_depth(depth, u, v, W, H):
    # Normalize to NDC
    u_ndc = (u / (W - 1)) * 2 - 1
    v_ndc = (v / (H - 1)) * 2 - 1
    grid = torch.stack([u_ndc, v_ndc], dim=-1).unsqueeze(0).unsqueeze(0)
    # Use bilinear interpolation with border padding
    # sampled = F.grid_sample(depth.unsqueeze(0).unsqueeze(0), grid, mode='bilinear', padding_mode='border', align_corners=True)
    # return sampled.squeeze()
    pass
```

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5) [S3, unofficial]. Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader [S3, unofficial]. Unlike regular chunk meshing which would require splitting a chunk into 6 sub-meshes (increasing driver overhead 6-fold), this method uses existing vertex pool bucketing [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size [S2, unofficial].
- **Memory Complexity:** $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
- **Strengths:** Effectively removes occluded geometry and artifacts not visible from training views. Can handle partial occlusions via mask dilation.
- **Limitations:** Requires pre-rendered depth maps or masks. Sensitive to coordinate system mismatches. Performance depends on the number of views and resolution of depth maps.
- **When to use:** When a set of reference views is available and the goal is to clean up a mesh generated from those views (e.g., in NeRF-based mesh extraction).
- **When not to use:** When real-time culling is required without pre-computation, or when no reference views are available.

## Background
Mask Projection Culling is derived from techniques used in 3D Gaussian Splatting and NeRF-based mesh extraction pipelines, where visibility from training views is used to filter noisy or occluded geometry [S1, unofficial][S2, unofficial]. The method leverages standard computer graphics projection and depth testing operations to determine visibility.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
