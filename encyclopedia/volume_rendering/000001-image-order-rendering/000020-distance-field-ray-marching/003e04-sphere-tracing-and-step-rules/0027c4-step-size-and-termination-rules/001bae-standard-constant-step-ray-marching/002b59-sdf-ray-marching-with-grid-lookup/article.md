# SDF Ray Marching with Grid Lookup

Also called: Intersect kernel

## Overview
A GPU-based ray marching algorithm that samples a 3D Signed Distance Function (SDF) grid to find the first intersection point of a ray with the zero-level set (surface) [S1, unofficial]. It uses linear interpolation to evaluate the SDF at arbitrary points within the grid bounds. It differs from [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by utilizing a pre-computed grid structure and specific termination logic based on pixel-wise distance thresholds rather than simple fixed-step traversal to the first root.

## How it works
**Inputs and Outputs:**
*   `grid`: Float tensor, 3D SDF values.
*   `origins`: Float tensor, shape $(N, 3)$, ray origins.
*   `directions`: Float tensor, shape $(N, 3)$, ray directions.
*   `origin_image_distances`: Float tensor, shape $(N)$, distance from camera origin to image plane.
*   `pixel_distances`: Float tensor, shape $(N)$, distance from camera origin to the pixel's target depth or reference plane.
*   `voxel_position`: Float tensor, shape $(N, 3)$, output voxel coordinates of the intersection.
*   `intersection_pos`: Float tensor, shape $(N, 3)$, output world coordinates of the intersection.

**Data Structures:**
The SDF grid is defined by resolution `grid_res_x`, `grid_res_y`, `grid_res_z` and an axis-aligned bounding box defined by `bounding_box_min` and `bounding_box_max` (coordinates for x, y, z).

**Parameters:**
*   `max_steps`: Maximum number of marching steps per ray. Value: 1000 [S1, unofficial].

**Algorithm Steps:**
1.  Initialize state variables for each ray: `depth = 0`, `first_time = true`, `gotten_result = 0`.
2.  Loop up to `max_steps`:
    a.  Calculate current world point: $P = \text{origins}[i] + \text{depth} \times \text{directions}[i]$.
    b.  Evaluate SDF distance $d = \text{ValueAt}(P)$ using trilinear interpolation.
    c.  Check for exit: If $d == -1$, set `voxel_position` and `intersection_pos` to -1, set `gotten_result = 1`, and break.
    d.  Check for intersection: If $d < \frac{\text{pixel\_distances}[i]}{\text{origin\_image\_distances}[i]} \times \text{depth}$ AND $d \neq 0$, record intersection. Set `voxel_position` to $\lfloor P \rfloor$ (clamped to `grid_res - 1`). Set `intersection_pos` to $P$. Set `gotten_result = 1` and break.
    e.  Increment `depth`.

**Trilinear Interpolation (`ValueAt`):**
Given point $P$ with fractional coordinates $t_x, t_y, t_z$ relative to the grid cell:
1.  Interpolate along X to get $c_{01}, c_{23}, c_{45}, c_{67}$.
2.  Interpolate along Y to get $c_0, c_1$.
3.  Interpolate along Z for final result.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_ray_marching_with_grid_lookup(grid, origins, directions, origin_image_distances, pixel_distances, grid_res, bounding_box_min, bounding_box_max, max_steps=1000):
    N = origins.shape[0]
    voxel_position = torch.full((N, 3), -1.0)
    intersection_pos = torch.full((N, 3), -1.0)
    gotten_result = torch.zeros(N, dtype=torch.bool)
    
    # Precompute grid bounds for clamping
    grid_res_tensor = torch.tensor([grid_res[0], grid_res[1], grid_res[2]], dtype=torch.float32)
    
    for step in range(max_steps):
        # Calculate current point
        # depth is implicitly step * step_size, but here depth is accumulated distance
        # Note: The source implies depth is accumulated distance, not step count.
        # However, the loop structure suggests step count. 
        # Re-reading source: "depth = 0 ... depth * directions". 
        # If directions are normalized, depth is distance. 
        # If not, depth is a scalar multiplier. 
        # Standard ray marching uses distance. Let's assume depth is distance traveled.
        # But the loop is fixed steps. This implies a fixed step size is missing or depth is step index.
        # Source says: "reached_point_x = origins[i] + depth * directions[i]".
        # And "max_steps (1000)".
        # If depth is just step index, directions must be scaled by step size.
        # The source does not specify step size. 
        # Pitfall: Without step size, this is ambiguous. 
        # However, source also uses "depth" in the intersection check: 
        # "distance < pixel_distances / origin_image_distances * depth".
        # This suggests depth is the distance along the ray.
        # We assume a fixed step size 'dt' is used to increment depth, but it is not given.
        # [G] Assuming a fixed step size dt is applied to depth each iteration.
        # Since dt is not given, we cannot implement the increment. 
        # We will assume depth represents the accumulated distance and is incremented by a constant dt.
        # To make it implementable, we define dt as a parameter or assume it's 1.0 if directions are not normalized?
        # No, standard ray marching uses normalized directions.
        # Let's look at the intersection condition again.
        # It compares SDF distance to a projected depth.
        
        # Let's rewrite the loop to be self-contained with an assumed dt.
        # Since dt is not in sources, we mark it.
        dt = 0.01 # [G] Typical step size, not given in source.
        
        current_depth = step * dt # [G] Assuming uniform step size.
        
        reached_point = origins + current_depth.unsqueeze(1) * directions
        
        # Evaluate SDF
        distances = value_at_sdf_grid(reached_point, grid, grid_res, bounding_box_min, bounding_box_max)
        
        # Check for exit
        exit_mask = (distances == -1)
        if exit_mask.any():
            voxel_position[exit_mask] = -1
            intersection_pos[exit_mask] = -1
            gotten_result[exit_mask] = True
            
        # Check for intersection
        # Formula: distance < (pixel_distances / origin_image_distances) * depth
        threshold = (pixel_distances / origin_image_distances) * current_depth
        intersect_mask = (distances < threshold) & (distances != 0) & (~gotten_result)
        
        if intersect_mask.any():
            # Record intersection
            voxel_position[intersect_mask] = torch.floor(reached_point[intersect_mask]).clamp(max=grid_res_tensor - 1)
            intersection_pos[intersect_mask] = reached_point[intersect_mask]
            gotten_result[intersect_mask] = True
            
        # Update gotten_result for all rays that have finished
        gotten_result = gotten_result | intersect_mask | exit_mask
        
        if gotten_result.all():
            break
            
    return voxel_position, intersection_pos

def value_at_sdf_grid(points, grid, grid_res, bbox_min, bbox_max):
    # Normalize points to [0, 1] range relative to bbox
    # [G] Implementation of normalization and clamping to grid bounds.
    # If point is outside bbox, return -1.
    
    # Normalize
    points_norm = (points - bbox_min) / (bbox_max - bbox_min)
    
    # Check bounds
    if (points_norm < 0).any() or (points_norm > 1).any():
        return torch.full(points.shape[0], -1.0)
        
    # Convert to grid coordinates
    grid_coords = points_norm * (torch.tensor(grid_res) - 1)
    
    # Extract integer and fractional parts
    x = grid_coords[:, 0]
    y = grid_coords[:, 1]
    z = grid_coords[:, 2]
    
    x0 = torch.floor(x)
    y0 = torch.floor(y)
    z0 = torch.floor(z)
    
    x1 = x0 + 1
    y1 = y0 + 1
    z1 = z0 + 1
    
    # Clamp indices to grid bounds
    x0 = x0.clamp(0, grid_res[0]-1)
    x1 = x1.clamp(0, grid_res[0]-1)
    y0 = y0.clamp(0, grid_res[1]-1)
    y1 = y1.clamp(0, grid_res[1]-1)
    z0 = z0.clamp(0, grid_res[2]-1)
    z1 = z1.clamp(0, grid_res[2]-1)
    
    # Get fractional parts
    fx = x - x0
    fy = y - y0
    fz = z - z0
    
    # Fetch 8 corners
    # Grid is 3D tensor. Accessing with batched indices requires gather or advanced indexing.
    # For simplicity in pseudocode, we assume a function get_voxel(x,y,z)
    
    c000 = get_voxel(grid, x0, y0, z0)
    c100 = get_voxel(grid, x1, y0, z0)
    c010 = get_voxel(grid, x0, y1, z0)
    c110 = get_voxel(grid, x1, y1, z0)
    c001 = get_voxel(grid, x0, y0, z1)
    c101 = get_voxel(grid, x1, y0, z1)
    c011 = get_voxel(grid, x0, y1, z1)
    c111 = get_voxel(grid, x1, y1, z1)
    
    # Interpolate along X
    c00 = c000 * (1 - fx) + c100 * fx
    c10 = c010 * (1 - fx) + c110 * fx
    c01 = c001 * (1 - fx) + c101 * fx
    c11 = c011 * (1 - fx) + c111 * fx
    
    # Interpolate along Y
    c0 = c00 * (1 - fy) + c10 * fy
    c1 = c01 * (1 - fy) + c11 * fy
    
    # Interpolate along Z
    result = c0 * (1 - fz) + c1 * fz
    
    return result

def get_voxel(grid, x, y, z):
    # Helper to access 3D grid with batched indices
    # This is a placeholder for the actual tensor indexing logic
    # In PyTorch: grid[x.long(), y.long(), z.long()]
    return grid[x.long(), y.long(), z.long()]
```

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(N \times S)$, where $N$ is the number of rays and $S$ is the number of steps (up to `max_steps`).
*   **Memory Complexity:** $O(N)$ for output tensors and intermediate state.
*   **Performance:** Performance depends on `max_steps` and the density of the SDF grid.
*   **Strengths:** Efficient GPU implementation using tensor operations.
*   **Limitations:** Fixed step size may lead to aliasing or missed intersections if not tuned. Precision issues with floating point comparisons.

## Background
Originated in the context of differentiable rendering and SDF optimization, specifically in the CVPR 2020 paper "SDFDiff: Differentiable Rendering of Signed Distance Fields" [S1, unofficial]. It is used in neural rendering pipelines to compute surface intersections for gradient-based optimization of SDF representations.

## Sources
[S1] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
