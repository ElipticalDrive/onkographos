# Mask Projection Culling

Also called: Depth-Projected Mesh Culling, Mesh Culling via Mask Projection, point mask culling

## Overview
Mask Projection Culling is a visibility-based mesh filtering technique that removes vertices and faces not consistently observed across multiple camera poses by projecting 3D points into camera space and comparing their depth against rendered depth maps or binary masks [S1, unofficial][S2, unofficial]. It solves the problem of retaining only the geometry visible from the capture viewpoints, effectively removing floating artifacts and internal structures that do not contribute to the final rendered images. Unlike neighbor-based culling methods like [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) which rely on local voxel occupancy, this method relies on global camera visibility, making it suitable for sparse or unstructured capture setups [S1, unofficial][S2, unofficial].

## How it works
The method iterates over a set of 3D points (vertices or point cloud samples) and a set of camera poses. For each point, it determines visibility by checking if the point projects within the image bounds of a camera and lies in front of the surface defined by the depth map or mask at that projection. A point is retained if it is visible in a sufficient number of views.

### Inputs and Outputs
*   **Inputs**:
    *   `mesh`: A mesh object containing vertices $V$ and faces $F$.
    *   `cameras`: A list of camera poses (world-to-camera or camera-to-world matrices) and intrinsic parameters.
    *   `depth_maps`: Rendered depth maps for each camera pose, or binary masks from `mask_dir/*.png` [S1, unofficial][S2, unofficial].
    *   `image_dimensions`: Width $W$ and Height $H$ of the images (e.g., $W=1600, H=1200$) [S1, unofficial].
    *   `visibility_threshold`: Minimum number of views a point must be visible in to be retained (e.g., 20) [S2, unofficial].
    *   `depth_tolerance` ($\epsilon$): Tolerance for depth comparison (e.g., 0.005) [S2, unofficial].
*   **Outputs**:
    *   `filtered_mesh`: A mesh containing only vertices and faces where all constituent vertices are visible in at least `visibility_threshold` views.

### Data Structures
*   `valid_counts`: An array of size $N$ (number of vertices), initialized to 0, storing the number of views each vertex is visible in.
*   `depth_maps`: A tensor or array of shape $(V, H, W)$, where $V$ is the number of views, containing depth values or binary masks.
*   `camera_poses`: A list of $4 \times 4$ matrices representing the transformation from world space to camera space ($W2C$) or vice versa.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mask_projection_culling(mesh, cameras, depth_maps, W, H, visibility_threshold, eps=0.005, batch_size=500000):
    """
    Filters mesh vertices and faces based on visibility across multiple camera views.
    
    Args:
        mesh: Object with .vertices (N, 3) and .faces (M, 3).
        cameras: List of dicts or objects containing 'c2w' (4x4) and 'K' (3x3) matrices.
        depth_maps: Tensor of shape (V, H, W) containing depth or mask values.
        W: Image width.
        H: Image height.
        visibility_threshold: Minimum number of views a vertex must be visible in.
        eps: Depth comparison tolerance.
        batch_size: Number of points to process per batch to limit memory.
        
    Returns:
        filtered_mesh: A new mesh with filtered vertices and faces.
    """
    vertices = mesh.vertices # Shape (N, 3)
    faces = mesh.faces       # Shape (M, 3)
    N = vertices.shape[0]
    
    # Initialize visibility count for each vertex
    valid_counts = np.zeros(N, dtype=np.int32)
    
    # Process vertices in batches to manage GPU memory
    for start in range(0, N, batch_size):
        end = min(start + batch_size, N)
        batch_vertices = vertices[start:end] # Shape (B, 3)
        
        # Initialize visibility count for this batch
        batch_valid_counts = np.zeros(end - start, dtype=np.int32)
        
        # Convert batch vertices to tensor for GPU processing
        batch_vertices_tensor = torch.tensor(batch_vertices, dtype=torch.float32, device="cuda")
        
        for i, cam in enumerate(cameras):
            # Get camera matrices
            # Handle coordinate system mismatch: flip Y and Z axes of c2w if necessary
            c2w = cam['c2w'].clone()
            c2w[:3, 1:3] *= -1 
            
            # Compute inverse (World to Camera)
            w2c = torch.inverse(c2w).cuda()
            
            # Get intrinsics
            K = cam['K'].cuda()
            
            # Transform vertices to camera space
            # Homogeneous coordinates: (B, 4)
            vertices_homo = torch.cat([batch_vertices_tensor, torch.ones(end - start, 1, device="cuda")], dim=1)
            cam_coords = (w2c @ vertices_homo.t()).t() # Shape (B, 4)
            
            # Project to 2D
            # uv = K @ cam_coords
            uv = torch.matmul(K, cam_coords[:, :3].t()).t() # Shape (B, 3)
            
            # Homogeneous division
            z = uv[:, 2:] + 1e-8
            uv_xy = uv[:, :2] / z # Shape (B, 2)
            
            # Normalize to [0, W-1] and [0, H-1] for grid sampling
            # Or to [-1, 1] for grid_sample depending on implementation
            # Using [-1, 1] range for grid_sample
            u_norm = (uv_xy[:, 0] / (W - 1)) * 2.0 - 1.0
            v_norm = (uv_xy[:, 1] / (H - 1)) * 2.0 - 1.0
            
            # Create grid for sampling
            # vgrid shape (B, 1, 1, 2) for grid_sample
            vgrid = torch.stack([u_norm, v_norm], dim=-1).unsqueeze(1).unsqueeze(2) # Shape (B, 1, 1, 2)
            
            # Sample depth/mask from depth_maps
            # depth_maps shape (V, H, W) -> (V, 1, H, W) for grid_sample
            depth_map_tensor = torch.tensor(depth_maps[i], dtype=torch.float32, device="cuda").unsqueeze(0) # Shape (1, H, W)
            
            # Sample using bilinear interpolation with border padding
            # depth_sample shape (B, 1, 1, 1)
            depth_sample = torch.nn.functional.grid_sample(
                depth_map_tensor.unsqueeze(0), # Add batch dim if not present, or ensure shape (1, 1, H, W)
                vgrid,
                padding_mode='border',
                align_corners=True
            ).squeeze(-1).squeeze(-1) # Shape (B,)
            
            # Check if point is in front of the surface
            # z is the depth of the point in camera space (positive if in front)
            # depth_sample is the depth of the surface at that pixel
            # Point is visible if z < depth_sample + eps
            # Also check if point is within frustum (handled by padding_mode='border' and depth_sample > 0)
            
            # In some implementations, depth_maps are masks (0 or 1). 
            # If mask: valid if sampled_mask > 0.
            # If depth: valid if z < depth_sample + eps AND depth_sample > 0
            
            # Assuming depth_maps contains depth values
            is_front = (depth_sample > 0.0) & (z.squeeze() < (depth_sample + eps))
            
            # Accumulate visibility
            batch_valid_counts += is_front.cpu().numpy().astype(np.int32)
        
        valid_counts[start:end] = batch_valid_counts
        
    # Filter vertices
    valid_vertices_mask = valid_counts >= visibility_threshold
    valid_vertices = vertices[valid_vertices_mask]
    
    # Map old vertex indices to new indices
    vertex_index_map = np.full(N, -1, dtype=np.int32)
    vertex_index_map[valid_vertices_mask] = np.arange(valid_vertices_mask.sum())
    
    # Filter faces
    # A face is valid if all its vertices are valid
    face_vertices_valid = valid_vertices_mask[faces] # Shape (M, 3)
    valid_faces_mask = face_vertices_valid.all(axis=1) # Shape (M,)
    valid_faces = faces[valid_faces_mask]
    
    # Remap face indices
    valid_faces = vertex_index_map[valid_faces]
    
    # Create new mesh
    filtered_mesh = Mesh(vertices=valid_vertices, faces=valid_faces)
    
    return filtered_mesh
```

### Parameters
*   `eps`: Depth comparison tolerance. Typical value: 0.005 [S2, unofficial].
*   `visibility_threshold`: Minimum number of views a point must be visible in. Typical value: 20 [S2, unofficial].
*   `batch_size`: Maximum number of points processed per batch. Typical value: $5 \times 10^5$ [S2, unofficial].
*   `image_dimensions`: $W=1600, H=1200$ are common defaults [S1, unofficial].

### Edge Cases and Pitfalls
*   **Coordinate System Mismatch**: OpenCV and OpenGL conventions differ in Y and Z axes. Camera poses may need flipping (`c2w[:3, 1:3] *= -1`) to ensure correct depth comparisons [S2, unofficial].
*   **Out-of-Bounds UV Coordinates**: Sampling depth at UV coordinates outside the image bounds can cause NaNs or crashes. Using `padding_mode='border'` in grid sampling handles this gracefully [S2, unofficial].
*   **Depth Discretization**: Projection errors and depth map discretization require a tolerance $\epsilon$ when comparing point depth to surface depth [S2, unofficial].
*   **Small Geometry Removal**: Connected components with small area ratios may be retained if they pass the visibility threshold. A post-processing step may be needed to remove small artifacts using a threshold like 0.2 [S2, unofficial].

## Subtopics

### True Back-Face Culling via Group Masking
A back-face culling technique that splits chunk meshes into 6 vertex pool buckets based on face orientation (groups 0-5). Draw calls are masked based on the camera position relative to the chunk, preventing the draw call from being issued entirely rather than culling in the fragment shader. This avoids the 6-fold driver overhead of splitting chunks into sub-meshes [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot V \cdot B)$, where $N$ is the number of points, $V$ is the number of validation frames, and $B$ is the batch size (processing overhead) [S2, unofficial].
*   **Memory Complexity**: $O(N \cdot V)$ for intermediate masks and valid counts [S2, unofficial].
*   **Strengths**: Effective at removing non-visible geometry in sparse capture setups; robust to complex scene structures where local voxel occupancy is insufficient.
*   **Limitations**: Requires pre-rendered depth maps or masks; sensitive to camera pose accuracy; computationally expensive for large point clouds due to per-point projection and sampling.
*   **When to Use**: When a set of camera poses and corresponding depth/mask images are available, and the goal is to clean up geometry reconstructed from these views (e.g., in NeRF or Gaussian Splatting pipelines).
*   **When Not to Use**: For real-time rendering where pre-computation is not feasible, or when camera poses are unknown or inaccurate.

## Background
Mask Projection Culling is commonly used in 3D reconstruction pipelines, particularly those involving NeRF (Neural Radiance Fields) and 3D Gaussian Splatting, to filter out noise and artifacts from the reconstructed geometry. It was popularized by implementations such as 2D Gaussian Splatting [S1, unofficial] and SVRaster [S2, unofficial], which use visibility checks to ensure that the final mesh or point cloud only contains geometry visible from the training cameras. The technique leverages the fact that valid geometry should be consistently visible across multiple viewpoints, while artifacts often are not.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
