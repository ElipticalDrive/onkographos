# Clipping Distance Computation via Volume Corners

Also called: IJK to RAS Corner Projection, Volume Bounding Box Clipping

## Overview
Computes the near and far clipping distances for ray marching by projecting the eight corners of the volume bounding box onto the camera view direction, then taking the minimum and maximum of these projected distances [S1, unofficial]. This method ensures that all sample points generated along a ray fall within the volume's spatial extent, preventing out-of-bounds memory access and unnecessary computation outside the volume [S1, unofficial]. It differs from sibling methods like [Ray Point Sampling](2382-ray-point-sampling.md) by explicitly handling the coordinate transformation from volume-local (IJK) to world/camera (RAS) space before determining the clipping bounds, whereas siblings may assume pre-computed bounds or different sampling strategies.

## How it works
**Inputs:**
- Camera matrix $M_{cam}$ (4x4).
- Volume bounding box dimensions in IJK space: $I, J, K$ (number of voxels along each axis).
- IJK-to-RAS transformation matrix $M_{ijk2ras}$ (4x4).
- Margin parameter $\delta$ (default 30.0 mm) [S1, unofficial].

**Outputs:**
- Near clipping distance $d_{near}$.
- Far clipping distance $d_{far}$.

**Data Structures:**
- Camera matrix $M_{cam}$: 4x4 matrix. If the norm of the translation column (last column, first 3 rows) is less than 1.0, it is treated as a world-to-camera matrix and inverted to obtain the camera-to-world matrix $M_{world}$. Otherwise, it is used directly as $M_{world}$ [S1, unofficial].
- Volume corners in IJK space: 8 points $(i, j, k)$ where $i \in \{0, I-1\}$, $j \in \{0, J-1\}$, $k \in \{0, K-1\}$ [S1, unofficial].
- Volume corners in RAS space: 8 points $\mathbf{p}_{ras}^{(n)}$ for $n=0..7$ [S1, unofficial].

**Steps:**
1. **Extract Camera Pose:** Determine the camera position $\mathbf{c}$ and view direction $\mathbf{v}$ from $M_{cam}$. If $\|M_{cam}[0:3, 3]\| < 1.0$, invert $M_{cam}$ to get $M_{world}$. Extract $\mathbf{c} = M_{world}[0:3, 3]$ and $\mathbf{v} = -M_{world}[0:3, 2]$ (assuming standard OpenGL-style camera where -Z is view direction) or appropriate axis based on convention. Normalize $\mathbf{v}$ [S1, unofficial].
2. **Generate IJK Corners:** Create the 8 corners of the volume bounding box in IJK space. The indices are $0$ and $I-1$ for the first dimension, $0$ and $J-1$ for the second, and $0$ and $K-1$ for the third [S1, unofficial].
3. **Transform to RAS Space:** For each corner $\mathbf{p}_{ijk} = (i, j, k, 1)^T$, compute $\mathbf{p}_{ras} = M_{ijk2ras} \cdot \mathbf{p}_{ijk}$. Take the first three components of the result to get the 3D position in RAS space [S1, unofficial].
4. **Project Corners:** For each RAS corner $\mathbf{p}_{ras}^{(n)}$, compute the vector $\mathbf{d}^{(n)} = \mathbf{p}_{ras}^{(n)} - \mathbf{c}$. Project this vector onto the normalized view direction $\mathbf{v}$ to get the scalar distance $t^{(n)} = \mathbf{d}^{(n)} \cdot \mathbf{v}$ [S1, unofficial].
5. **Compute Clipping Distances:** Set $t_{min} = \min_n(t^{(n)})$ and $t_{max} = \max_n(t^{(n)})$. Set $d_{near} = t_{min} - \delta$ and $d_{far} = t_{max} + \delta$. Clamp $d_{near}$ to a minimum of 0.1 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_clipping_distances(camera_matrix, ijk2ras, volume_dims, margin=30.0):
    """
    Compute near and far clipping distances for volume ray marching.
    
    Args:
        camera_matrix: (4, 4) tensor, camera matrix.
        ijk2ras: (4, 4) tensor, IJK to RAS transformation matrix.
        volume_dims: tuple (I, J, K), number of voxels in each dimension.
        margin: float, safety margin in mm.
        
    Returns:
        d_near: float, near clipping distance.
        d_far: float, far clipping distance.
    """
    I, J, K = volume_dims
    
    # Step 1: Extract Camera Pose
    # Check if translation norm is less than 1.0 to determine if inversion is needed
    translation = camera_matrix[:3, 3]
    translation_norm = torch.norm(translation)
    
    # Use torch.where to avoid branching
    # If norm < 1.0, invert camera_matrix; else use as is
    # Note: Inverting a 4x4 matrix in PyTorch can be done via torch.inverse
    # However, for efficiency, we might only need the relevant parts. 
    # Assuming full inversion for correctness as per source description.
    is_world_to_cam = translation_norm < 1.0
    camera_matrix_world = torch.where(
        is_world_to_cam,
        torch.inverse(camera_matrix),
        camera_matrix
    )
    
    # Extract camera position (translation part of world matrix)
    cam_pos = camera_matrix_world[:3, 3]
    
    # Extract view direction (negative Z axis of world matrix, assuming standard convention)
    # Source does not specify exact axis, but standard is -Z. 
    # We assume the third column (index 2) is the view direction vector in world space.
    view_dir = -camera_matrix_world[:3, 2]
    view_dir = view_dir / torch.norm(view_dir)
    
    # Step 2: Generate IJK Corners
    # Indices: 0 and I-1, 0 and J-1, 0 and K-1
    i_vals = torch.tensor([0, I - 1], dtype=torch.float32)
    j_vals = torch.tensor([0, J - 1], dtype=torch.float32)
    k_vals = torch.tensor([0, K - 1], dtype=torch.float32)
    
    # Create meshgrid for all 8 combinations
    # Shape: (2, 2, 2) for each dimension
    i_grid, j_grid, k_grid = torch.meshgrid(i_vals, j_vals, k_vals, indexing='ij')
    
    # Flatten to (8,)
    i_flat = i_grid.flatten()
    j_flat = j_grid.flatten()
    k_flat = k_grid.flatten()
    
    # Stack to form (8, 3)
    corners_ijk = torch.stack([i_flat, j_flat, k_flat], dim=1)
    
    # Append homogeneous coordinate 1.0 to make (8, 4)
    corners_ijk_homo = torch.cat([corners_ijk, torch.ones(8, 1)], dim=1)
    
    # Step 3: Transform to RAS Space
    # p_ras = ijk2ras @ p_ijk_homo^T for each point
    # Matrix multiplication: (4, 4) @ (8, 4)^T -> (4, 8) -> transpose to (8, 4)
    corners_ras_homo = (ijk2ras @ corners_ijk_homo.t()).t()
    
    # Take first 3 components
    corners_ras = corners_ras_homo[:, :3]
    
    # Step 4: Project Corners
    # Vector from camera to corner
    vec_to_corner = corners_ras - cam_pos
    # Project onto view direction
    distances = torch.sum(vec_to_corner * view_dir, dim=1)
    
    # Step 5: Compute Clipping Distances
    t_min = torch.min(distances)
    t_max = torch.max(distances)
    
    d_near = t_min - margin
    d_far = t_max + margin
    
    # Clamp near to minimum of 0.1
    d_near = torch.clamp(d_near, min=0.1)
    
    return d_near.item(), d_far.item()
```

**Parameters:**
- `margin`: Safety margin added to near/far distances in mm. Default value is 30.0. Positive values expand the clipping range [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the camera is inside the volume, $t_{min}$ may be negative. The margin and clamping handle this, but $d_{near}$ is clamped to 0.1 to avoid zero or negative distances which could cause division by zero or sampling errors [S1, unofficial].
- Precision issues may arise if the volume is very large or the camera is very far. Using double precision for intermediate calculations may be necessary [G].
- The assumption that the view direction is the negative Z-axis of the world matrix is standard but should be verified against the specific camera convention used [G].

## Subtopics

### Volume Bounding Box Projection for Ray Marching
A vertex shader implementation that projects the world-space axis-aligned bounding box (AABB) of a fire volume to enable fragment shader ray marching [S2, unofficial]. It transforms volume-space corners (unit cube $[0,1]^3$) to world space using a `volume_to_world` matrix, then to clip space using view and projection matrices [S2, unofficial]. It outputs the world position of the corners for use in the fragment shader [S2, unofficial]. This variant is specific to GPU-based rendering pipelines using Vulkan/DirectX-style push constants and shaders, unlike the CPU-based PyTorch implementation in the main method [S2, unofficial]. It is chosen when real-time rendering of volumetric effects like fire is required [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ with respect to volume resolution, as it only processes 8 corners. The cost is dominated by matrix multiplications and vector operations, which are constant time for a fixed number of corners.
- **Memory Complexity:** $O(1)$, storing only 8 corners and a few scalar values.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of rays if applied per-ray, but since the volume bounds are global, it can be computed once per volume and reused for all rays [G].
- **Strengths:** Simple, robust, and efficient. Ensures all samples are within the volume bounds.
- **Limitations:** Assumes the volume is axis-aligned in IJK space and convex. Does not account for internal structures or non-convex shapes. The margin parameter needs to be tuned to avoid excessive empty space sampling or clipping errors.
- **When to use:** When a quick and reliable method to determine ray clipping bounds is needed for volume rendering, especially in differentiable rendering pipelines where gradients are required [S1, unofficial].
- **When not to use:** For non-convex volumes or when more precise ray-box intersection is required (e.g., for sparse volumes with complex shapes) [G].

## Background
This method is used in differentiable volume raycasting for digitally reconstructed radiographs (DRRs) [S1, unofficial]. It was implemented in the `flashdrr` library by pcarnah [S1, unofficial]. The approach is based on projecting the volume bounding box corners onto the camera view direction to determine the integration range for ray marching [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] migaelstrydom/VoxelPhase, https://github.com/migaelstrydom/VoxelPhase, fetched 2026-10-28 (unofficial)
