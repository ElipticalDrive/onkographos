# Frustum Culling Integration

Also called: Base+Frustum culling

## Overview
Frustum Culling Integration is a visibility optimization technique applied during the progressive loading and rendering of 3D Gaussian Splatting scenes [S1]. It filters out splats that lie outside the user's current viewing frustum to prioritize the transmission and rendering of visible geometry [S1]. It differs from its sibling [Bounding-Sphere Tree Traversal with Frame-Rate Control](7227-bounding-sphere-tree-traversal-with-frame-rate-control.md) by focusing on spatial visibility rather than frame-rate stability, and from [Frustum-Occlusion Culling for Octree Traversal](12324-frustum-occlusion-culling-for-octree-traversal.md) by operating on a pre-ordered list of splats rather than a hierarchical tree structure during the culling phase [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of 3D Gaussians (splats) with their mean positions, the current camera view and projection matrices, and a frustum margin parameter [S1].
*   **Output:** A filtered subset of splats that are considered visible (inside the frustum) [S1].

**Data Structures:**
*   **Splat List:** An array of Gaussian objects containing mean positions and screen-space projections [S1].

**Steps:**
1.  **Global Ordering (Pre-requisite):** Frustum culling is applied *after* a global ordering of splats has been determined [S1]. This means the culling step filters the list rather than re-ordering it [S1].
2.  **Frustum Margin Application:** Define the viewing frustum boundaries (left, right, bottom, top, near, far) and increase them by 30% [S1].
3.  **Projection and Filtering:** Project all splats onto the image plane [S1]. Filter the splats that fall outside the expanded frustum boundaries [S1].
4.  **Selection:** Retain only those splats where the projected center lies within the expanded frustum [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frustum_cull_splats(splats, view_matrix, proj_matrix, margin_factor=0.3):
    """
    Filters a list of splats based on the viewing frustum with a margin.
    
    Args:
        splats: List of Gaussian objects with 'mean' (vec3).
        view_matrix: 4x4 view matrix.
        proj_matrix: 4x4 projection matrix.
        margin_factor: Percentage to expand frustum boundaries (e.g., 0.3 for 30%).
        
    Returns:
        List of splats inside the expanded frustum.
    """
    visible_splats = []
    vp_matrix = proj_matrix @ view_matrix
    margin = margin_factor
    # [G] Standard NDC frustum bounds are [-1, 1] for x and y.
    # Increasing boundaries by margin_factor expands the visible region.
    x_min = -1.0 * (1.0 + margin)
    x_max = 1.0 * (1.0 + margin)
    y_min = -1.0 * (1.0 + margin)
    y_max = 1.0 * (1.0 + margin)
    
    for gaussian in splats:
        # [G] Standard perspective projection and NDC conversion
        pos_homogeneous = vec4(gaussian.mean, 1.0)
        clip_pos = vp_matrix @ pos_homogeneous
        if clip_pos.w <= 0.0:
            continue
        ndc_x = clip_pos.x / clip_pos.w
        ndc_y = clip_pos.y / clip_pos.w
        if x_min <= ndc_x <= x_max and y_min <= ndc_y <= y_max:
            visible_splats.append(gaussian)
    return visible_splats
```

**Edge Cases and Pitfalls:**
*   **Occlusion Ignorance:** Frustum culling occurs after global ordering and does not account for occlusions from the current viewpoint [S1].
*   **Small Background Splats:** If more than 10% of splats are visible within the frustum, small background splats may still not be selected for high-quality loading because the culling happens after the initial priority list is formed [S1].
*   **Gaussian Extent:** Since Gaussians cover areas rather than points on the image plane, a Gaussian with a mean outside the frustum might still contribute to the image [S1]. The margin mitigates this by counting Gaussians inside the expanded frustum [S1].

## Subtopics

### In-frustum prioritization
A technique to fine-tune the ordering of splats based on the user's initial pose, selecting a percentage of splats from within the viewing frustum to allow for rapid movement while focusing on visible content [S1]. It involves projecting all splats onto the image plane and filtering those outside the frustum [S1]. A typical parameter is the frustum margin, set to 30% to increase frustum boundaries [S1]. This variant is chosen when user-specific view information is available to improve perceived quality during camera movement [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The complete approach (octree + frustum culling) performs better when user-specific view information is available [S1]. However, if more than 10% of splats are visible, the benefit is reduced for background details [S1].
*   **Strengths:** Reduces the number of splats processed by excluding clearly invisible geometry [S1].
*   **Limitations:** Does not handle occlusion; relies on pre-computed ordering; margin parameter requires tuning [S1].
*   **When to use:** When user view direction is known and the scene has significant geometry outside the current view [S1].
*   **When not to use:** When occlusion is the primary bottleneck or when the view is fully contained within the scene (high visibility percentage) [S1].

## Background
The method originates from the PRoGS (Progressive Rendering of Gaussian Splats) paper [S1]. It was developed to address the bandwidth and rendering costs of progressive 3D Gaussian Splatting by leveraging user view information to prioritize visible splats [S1]. The approach builds on the base progressive rendering method by adding a visibility filter [S1].

## Sources
[S1] PRoGS: Progressive Rendering of Gaussian Splats, https://doi.org/10.1109/wacv61041.2025.00308, fetched 2026-10-26
