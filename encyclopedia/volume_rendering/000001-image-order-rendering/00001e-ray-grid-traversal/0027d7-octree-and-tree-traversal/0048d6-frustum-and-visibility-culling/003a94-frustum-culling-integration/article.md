# Frustum Culling Integration

Also called: Base+Frustum culling

## Overview
Frustum Culling Integration is a visibility optimization technique applied during the progressive loading of volumetric splats (specifically 3D Gaussian Splatting) to prioritize data transmission and rendering of elements within the user's current view [S1]. It operates by filtering out splats that lie outside the viewing frustum, thereby reducing the computational load on background elements [S1]. Unlike the base octree-only approach, which provides a static scene representation without user-specific knowledge, this method leverages the user's initial pose to improve performance when such information is available [S1].

## How it works
**Inputs:** A list of splats (Gaussians) with their spatial parameters (mean position, covariance) and the current camera view matrix defining the viewing frustum.
**Outputs:** A filtered subset of splats marked as "visible" for immediate processing/loading.

**Data Structures:**
*   **Splat List:** Array of splat objects containing 3D mean position $\mathbf{\mu}$ and projection parameters.
*   **Frustum Planes:** Six planes (left, right, bottom, top, near, far) defining the viewing volume.

**Steps:**
1.  **Global Ordering (Pre-requisite):** The method is applied *after* a global ordering of splats has been determined (e.g., by depth or contribution) [S1]. This means the culling does not account for occlusions from the current viewpoint, only geometric visibility [S1].
2.  **Projection:** Project the mean position $\mathbf{\mu}$ of each splat onto the image plane using the current view-projection matrix [S1].
3.  **Margin Application:** Expand the boundaries of the viewing frustum by a specified margin percentage to account for the spatial extent of Gaussians, which cover areas rather than single points [S1].
4.  **Filtering:** Determine if the projected splat (including its margin-adjusted extent) falls within the expanded frustum boundaries [S1].
    *   If the splat is outside the expanded frustum, it is excluded from the current batch.
    *   If inside, it is included.
5.  **Selection:** Select a percentage of the filtered splats for rapid movement and high-priority loading [S1].

**Parameters:**
*   **Frustum Margin:** The percentage by which frustum boundaries are increased. Typical value: 30% [S1].

**Pitfalls:**
*   **Late Application:** Because frustum culling occurs after global ordering, it does not account for occlusions from the current viewpoint [S1].
*   **Background Noise:** If more than 10% of splats are visible within the current frustum, small background splats may still not be selected efficiently, as the ordering was not optimized for the specific view [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frustum_cull_splats(splats, view_projection_matrix, margin_percent=0.3):
    """
    Filters splats based on visibility in the viewing frustum.
    Assumes splats are already globally ordered.
    """
    visible_splats = []
    
    # Define frustum boundaries from view_projection_matrix
    # This step extracts the 6 planes (left, right, bottom, top, near, far)
    frustum_planes = extract_frustum_planes(view_projection_matrix)
    
    # Apply margin to frustum boundaries
    # Note: Exact expansion logic depends on projection space; 
    # typically involves scaling the near/far or extending clip-space bounds.
    expanded_planes = expand_frustum(frustum_planes, margin_percent)
    
    for splat in splats:
        # Project splat mean to clip space
        clip_pos = view_projection_matrix @ splat.mean_position
        
        # Check if splat is inside expanded frustum
        # For Gaussians, we check if the mean is inside, 
        # potentially expanded by the margin to cover the Gaussian's area
        if is_inside_frustum(clip_pos, expanded_planes):
            visible_splats.append(splat)
            
    return visible_splats

def is_inside_frustum(clip_pos, planes):
    """
    Checks if a point in clip space is inside all 6 frustum planes.
    """
    for plane in planes:
        # Plane equation: Ax + By + Cz + D <= 0 for inside
        if plane.a * clip_pos.x + plane.b * clip_pos.y + \
           plane.c * clip_pos.z + plane.d > 0:
            return False
    return True
```

## Subtopics

### In-frustum prioritization
A technique to fine-tune ordering based on the user's initial pose, selecting a percentage of splats from within the viewing frustum to allow rapid movement [S1]. It involves projecting all splats onto the image plane and filtering those that fall outside the frustum [S1]. The frustum margin is typically set to 30% to ensure Gaussians covering areas are not incorrectly culled [S1].

## Operating characteristics
*   **Performance:** Improves foreground quality marginally by focusing resources on visible splats [S1].
*   **Limitations:** Does not improve background representation if the visible splat count is high (>10%), as small background splats may still be deprioritized due to the pre-existing global ordering [S1].
*   **Comparison:** The complete approach (with octree and frustum culling) performs better when user-specific view information is available, whereas the octree-only approach provides good scene representation without user-specific knowledge [S1].

## Background
This method is part of the PRoGS (Progressive Rendering of Gaussian Splats) framework, which brings progressive rendering of 3D Gaussian Splats into an academic context [S1]. It builds upon base ordering techniques by adding view-dependent culling to optimize bandwidth and rendering load for dynamic user viewpoints [S1].

## Sources
[S1] PRoGS: Progressive Rendering of Gaussian Splats, https://doi.org/10.1109/wacv61041.2025.00308, fetched 2026-10-26
