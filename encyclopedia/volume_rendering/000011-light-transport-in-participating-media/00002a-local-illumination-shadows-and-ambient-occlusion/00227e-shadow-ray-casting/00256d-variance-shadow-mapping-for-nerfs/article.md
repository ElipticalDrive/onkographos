# Variance Shadow Mapping for NeRFs

Also called: NeRF shadow mapping, VSM for NeRFs

## Overview
Variance Shadow Mapping for NeRFs is a shadow rendering method that applies variance shadow mapping principles to neural radiance fields to generate soft shadows efficiently. It solves the problem of computing visibility for cast shadows without performing expensive cumulative transmittance integrations along shadow rays. Unlike naive shadow mapping which produces hard edges, this method transforms discontinuous depth information from the light's perspective into continuous visibility values in the range $[0, 1]$, enhancing realism by simulating soft shadow penumbras.

## How it works
The method computes visibility $V(\mathbf{r})$ for a camera ray $\mathbf{r}$ by comparing the depth of the rendered point against a precomputed depth distribution from the light source's viewpoint.

**Inputs and Outputs:**
*   **Inputs:**
    *   Camera ray $\mathbf{r}$ and its corresponding 3D surface point depth $z(\mathbf{r})$ (obtained from the NeRF rendering or depth map).
    *   Light source position (defining the virtual camera).
    *   Precomputed depth maps from the light view: mean depth map $X$ and squared depth map $X^2$.
*   **Outputs:**
    *   Visibility factor $V(\mathbf{r}) \in [0, 1]$.

**Data Structures:**
*   **Depth Map ($X$):** A 2D grid storing the depth value $z$ for each pixel from the light's perspective.
*   **Squared Depth Map ($X^2$):** A 2D grid storing the squared depth value $z^2$ for each pixel from the light's perspective.
*   **Filtered Maps:** $\mu(X)$ and $\mu(X^2)$, which are the box-filtered versions of $X$ and $X^2$ respectively.

**Steps:**
1.  **Reconstruct 3D Point:** Determine the 3D coordinates of the surface point corresponding to the camera ray $\mathbf{r}$ using the NeRF's depth output and camera parameters.
2.  **Project to Light View:** Transform the 3D point into the coordinate system of a virtual camera positioned at the light source. Identify the corresponding pixel location $\mathbf{r}_l$ in the light's depth map.
3.  **Retrieve Depth Statistics:** Retrieve the filtered mean depth $\mu(\mathbf{X})(\mathbf{r}_l)$ and filtered mean squared depth $\mu(\mathbf{X}^2)(\mathbf{r}_l)$ from the precomputed maps at pixel $\mathbf{r}_l$.
4.  **Calculate Variance:** Compute the variance $\sigma^2(\mathbf{r}_l)$ of the depth distribution at $\mathbf{r}_l$ using the formula:
    $$ \sigma^2(\mathbf{r}_l) = \mu(\mathbf{X}^2)(\mathbf{r}_l) - \mu^2(\mathbf{X})(\mathbf{r}_l) $$
5.  **Compute Visibility:** Calculate the visibility $V(\mathbf{r})$ based on the relationship between the point's depth $z(\mathbf{r})$ and the filtered mean depth $\mu(\mathbf{X})(\mathbf{r}_l)$:
    $$ V(\mathbf{r}) = \begin{cases} 1, & z(\mathbf{r}) \leq \mu(\mathbf{X})(\mathbf{r}_l) \\ \frac{\sigma^2(\mathbf{r}_l)}{\sigma^2(\mathbf{r}_l) + (z(\mathbf{r}) - \mu(\mathbf{X})(\mathbf{r}_l))^2}, & z(\mathbf{r}) > \mu(\mathbf{X})(\mathbf{r}_l) \end{cases} $$
    If the point is closer to the light than the mean depth of the occluder, it is fully visible. Otherwise, it is partially occluded based on the Chebyshev inequality bound.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_visibility_neRF_VSM(z_point, light_view_coords, filtered_mean_depth_map, filtered_mean_sq_depth_map):
    """
    Computes visibility for a NeRF point using Variance Shadow Mapping.
    
    Args:
        z_point (float): Depth of the point from the camera view (projected to light view distance).
        light_view_coords (tuple): (u, v) coordinates in the light's depth map.
        filtered_mean_depth_map (array): 2D array of mu(X).
        filtered_mean_sq_depth_map (array): 2D array of mu(X^2).
        
    Returns:
        float: Visibility V in [0, 1].
    """
    u, v = light_view_coords
    
    # Retrieve precomputed statistics from light view maps
    mu_X = filtered_mean_depth_map[u, v]
    mu_X2 = filtered_mean_sq_depth_map[u, v]
    
    # Calculate variance: sigma^2 = E[X^2] - (E[X])^2
    variance = mu_X2 - (mu_X ** 2)
    
    # Handle numerical instability where variance might be slightly negative due to filtering precision
    if variance < 0:
        variance = 0.0
        
    # Apply Chebyshev inequality for visibility
    if z_point <= mu_X:
        # Point is in front of the occluder's mean depth -> fully visible
        return 1.0
    else:
        # Point is behind the occluder's mean depth -> partially occluded
        # V = sigma^2 / (sigma^2 + (z - mu)^2)
        depth_diff_sq = (z_point - mu_X) ** 2
        visibility = variance / (variance + depth_diff_sq)
        return visibility
```

**Parameters:**
*   **Box Filter Size:** The size of the box filter applied to $X$ and $X^2$ determines the softness of the shadow. Larger filters produce softer shadows but may introduce blurring artifacts. Not given in the sources.
*   **Light View Resolution:** The resolution of the depth maps $X$ and $X^2$. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Negative Variance:** Due to floating-point precision errors during filtering, $\mu(X^2) - \mu^2(X)$ may become slightly negative. It must be clamped to 0. [G]
*   **Depth Discontinuities:** The method relies on the assumption that depth changes are smooth within the filter window. Sharp depth edges may result in inaccurate variance estimates.
*   **Occlusion Ambiguity:** The visibility is an upper bound derived from Chebyshev's inequality. It may overestimate visibility in complex occlusion scenarios where the depth distribution is multi-modal.

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for visibility lookup, assuming precomputed depth maps. The cost is dominated by the generation of the light-view depth maps.
*   **Memory Complexity:** Requires storing two depth maps ($X$ and $X^2$) at the light view resolution.
*   **Performance:** Efficient as it avoids ray marching for shadow calculations. It requires generating depth maps from two views (camera and light).
*   **Strengths:** Produces soft shadows naturally; computationally efficient compared to ray casting; integrates easily with NeRF rendering pipelines.
*   **Limitations:** Approximate visibility (upper bound); quality depends on light view resolution and filter size; may suffer from shadow acne or peter-panning if not handled carefully.
*   **When to use:** When soft shadows are desired and computational efficiency is prioritized over physically accurate shadow integration.
*   **When not to use:** When hard shadows are required or when high-precision shadow boundaries are critical.

## Background
The method is inspired by variance shadow mapping [14] from traditional computer graphics and was proposed for NeRFs in "Relighting Scenes With Object Insertions in Neural Radiance Fields" [S1]. It addresses the need for realistic lighting effects in NeRFs by simulating occlusion without the computational cost of cumulative transmittance integration along shadow rays.

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
