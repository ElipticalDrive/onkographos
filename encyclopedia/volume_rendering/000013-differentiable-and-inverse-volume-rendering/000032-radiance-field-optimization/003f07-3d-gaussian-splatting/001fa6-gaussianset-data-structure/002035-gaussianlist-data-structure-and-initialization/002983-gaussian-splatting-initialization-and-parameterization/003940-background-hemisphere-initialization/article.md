# Background Hemisphere Initialization

Also called: Background Hemisphere Initialization and Optimization, Hemisphere Background Model

## Overview
Background Hemisphere Initialization addresses the problem of rendering distant background regions (typically hundreds of meters away) in large-scale urban or outdoor scenes where standard foreground Gaussian splatting fails due to sparsity and occlusion [S1]. It models the background as a fixed-radius hemisphere that moves with the camera to maintain a constant relative distance, allowing for efficient regression of Gaussian parameters from projected 2D colors [S1]. It differs from sibling methods like [Gaussian Model Initialization](2349-gaussian-model-initialization.md) and [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by explicitly structuring the background geometry as a moving hemisphere rather than initializing points within the foreground volume or randomly in space [S1].

## How it works
**Inputs:**
- Foreground volume midpoint (used to center the hemisphere).
- $K$ reference images with known camera poses.
- A lightweight two-layer MLP $\mathcal{M}_{bg}$ [S1].

**Outputs:**
- A set of background Gaussian primitives $G_{bg}$.
- For each Gaussian: position (on the hemisphere), color $\mathbf{c}_{bg}$, scale $\mathbf{s}_{bg}$, fixed rotation, and fixed opacity [S1].

**Data Structures:**
- **Background Hemisphere:** A geometric surface defined by a center $\mathbf{c}_{center}$ and radius $r_{bg}$. Points are sampled uniformly on this surface [S1].
- **Gaussian Primitives:** Each primitive $i$ has:
  - Position $\boldsymbol{\mu}_i$: Sampled on the hemisphere surface [S1].
  - Color $\mathbf{c}_{bg}$: Regressed by $\mathcal{M}_{bg}$ [S1].
  - Scale $\mathbf{s}_{bg}$: Sum of initial scale and network residual [S1].
  - Rotation: Fixed unit quaternion $[1, 0, 0, 0]$ [S1].
  - Opacity: Fixed to $1$ [S1].

**Parameters:**
- $r_{bg}$: Fixed radius of the background hemisphere. Value: $100$ meters [S1].
- $K$: Number of reference images used for color projection [S1].

**Steps:**
1. **Hemisphere Placement:** Position the center of the background hemisphere at the midpoint of the foreground volume. Fix the radius to $r_{bg} = 100$ m [S1].
2. **Camera Tracking:** Move the hemisphere along with the vehicle/camera motion such that the relative distance between the hemisphere and the target camera remains fixed [S1].
3. **Point Sampling:** Uniformly sample points over the hemisphere surface [S1].
4. **Color Projection:** Project the sampled 3D points onto the $K$ reference images to retrieve their 2D colors $\{\mathbf{c}_k\}_{k=1}^K$ [S1].
5. **Scale Initialization:** Initialize the base Gaussian scales $\mathbf{s}_{bg}^{init}$ using the K-Nearest Neighbors (KNN) algorithm on the projected points [S1].
6. **Parameter Regression:** Input the projected colors $\{\mathbf{c}_k\}_{k=1}^K$ into the network $\mathcal{M}_{bg}$ to predict:
   - Color residuals/features $\mathbf{c}_{bg}$ [S1].
   - Scale residuals $\Delta \mathbf{s}_{bg}$ [S1].
7. **Final Scale Calculation:** Compute the final scale $\mathbf{s}_{bg}$ by adding the initial scale and the predicted residual [S1].
8. **Fixed Attributes:** Explicitly set the rotation to the unit quaternion $[1, 0, 0, 0]$ and opacity to $1$. These parameters are not optimized by the network [S1].

**Formulas:**
The final scale is computed as:
$$
\mathbf{s}_{bg} = \Delta \mathbf{s}_{bg} + \mathbf{s}_{bg}^{init}
$$
where the scale residual is:
$$
\Delta \mathbf{s}_{bg} = \mathcal{M}_{bg}(\{\mathbf{c}_k\}_{k=1}^K)
$$
The color is directly regressed as:
$$
\mathbf{c}_{bg} = \mathcal{M}_{bg}(\{\mathbf{c}_k\}_{k=1}^K)
$$
Note: In the source, $\mathcal{M}_{bg}$ outputs both color and scale from the same input colors [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def background_hemisphere_initialization(foreground_midpoint, reference_images, K, r_bg=100.0):
    """
    Initialize background Gaussians using the hemisphere model.
    
    Args:
        foreground_midpoint: 3D vector, center of the foreground volume.
        reference_images: List of K images with known camera poses.
        K: Number of reference images.
        r_bg: Fixed radius of the hemisphere (default 100m).
        
    Returns:
        background_gaussians: List of Gaussian primitives.
    """
    # 1. Define Hemisphere Geometry
    hemisphere_center = foreground_midpoint
    hemisphere_radius = r_bg
    
    # 2. Sample Points on Hemisphere
    # Uniformly sample points on the hemisphere surface
    # Note: The specific sampling distribution (e.g., latitude/longitude) 
    # is not detailed beyond "uniform sampling" [S1].
    hemisphere_points = sample_uniform_hemisphere_points(hemisphere_center, hemisphere_radius)
    
    # 3. Project to Reference Images to Get Colors
    projected_colors = []
    for img in reference_images:
        # Project 3D points to 2D image plane
        # Retrieve color at projected coordinates
        colors_k = project_and_query_colors(hemisphere_points, img.camera_pose, img.image_data)
        projected_colors.append(colors_k)
    
    # Combine colors from all K images
    # The source uses {c_k}_{k=1}^K as input to M_bg [S1]
    all_projected_colors = concatenate_colors(projected_colors)
    
    # 4. Initialize Scales using KNN
    # Initialize base scales based on local density/neighbors
    s_bg_init = initialize_scales_knn(hemisphere_points)
    
    # 5. Regress Parameters using MLP M_bg
    # M_bg is a lightweight two-layer MLP [S1]
    # Input: Projected colors {c_k}
    # Output: Color c_bg and Scale residual Delta s_bg
    c_bg, delta_s_bg = M_bg(all_projected_colors)
    
    # 6. Compute Final Scale
    s_bg = s_bg_init + delta_s_bg
    
    # 7. Set Fixed Attributes
    rotation = [1, 0, 0, 0] # Unit quaternion
    opacity = 1.0
    
    # 8. Construct Gaussians
    background_gaussians = []
    for i, point in enumerate(hemisphere_points):
        gaussian = {
            'position': point,
            'color': c_bg[i],
            'scale': s_bg[i],
            'rotation': rotation,
            'opacity': opacity
        }
        background_gaussians.append(gaussian)
        
    return background_gaussians
```

**Edge Cases and Pitfalls:**
- **Occlusion:** Projected points may be occluded in reference images, leading to inaccurate color retrieval. The source mentions using geometric priors (like visibility maps based on depth differences) to mitigate this, but the initialization step itself relies on the projected colors [S1].
- **Fixed Radius:** The radius $r_{bg}$ is fixed at 100m. If the scene scale differs significantly, this may lead to poor background representation [S1].
- **Fixed Rotation/Opacity:** Since rotation and opacity are not learned, the model assumes a specific orientation and full transparency for background Gaussians, which may not hold for all background structures [S1].

## Subtopics

### Generalizable Background Model
This variant uses the same hemisphere structure but emphasizes generalizability across scenes. It uniformly samples points over a hemisphere with fixed radius $r_{bg}$ that moves with camera motion. A lightweight two-layer MLP $\mathcal{M}_{bg}$ learns color $\mathbf{c}_{bg}$ and scale $\mathbf{s}_{bg}$ directly from projected colors $\{\mathbf{c}_k\}_{k=1}^K$ [S1]. Rotation and opacity remain fixed parameters [S1]. The formula is $\mathbf{c}_{bg}, \mathbf{s}_{bg} = \mathcal{M}_{bg}(\{\mathbf{c}_k\}_{k=1}^K)$ [S1].

## Operating characteristics
- **Time Complexity:** Depends on the number of sampled points on the hemisphere and the number of reference images $K$. The MLP $\mathcal{M}_{bg}$ is lightweight (two-layer) [S1].
- **Memory Complexity:** Proportional to the number of background Gaussians, which is determined by the resolution of the hemisphere sampling [S1].
- **Performance:** Allows efficient 3DGS rasterization for background rendering while keeping desirable background quality [S1].
- **Strengths:** Handles distant backgrounds (hundreds of meters) effectively; moves with the camera to maintain relative distance; simple to implement [S1].
- **Limitations:** Fixed radius may not adapt to varying scene scales; fixed rotation and opacity limit flexibility; relies on accurate color projection from reference images [S1].
- **When to Use:** Large-scale urban or outdoor scenes with distant backgrounds where foreground Gaussians are sparse or occluded [S1].
- **When Not to Use:** Indoor scenes or scenes with close-range backgrounds where the hemisphere model is inappropriate [S1].

## Background
Originated in EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis (2025) [S1]. The method was developed to handle the challenge of reconstructing full images via efficient 3DGS rasterization while maintaining desirable background rendering in urban environments [S1]. It builds on the generalizable hemisphere Gaussian model concept to represent backgrounds that lie hundreds of meters away from the vehicle [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
