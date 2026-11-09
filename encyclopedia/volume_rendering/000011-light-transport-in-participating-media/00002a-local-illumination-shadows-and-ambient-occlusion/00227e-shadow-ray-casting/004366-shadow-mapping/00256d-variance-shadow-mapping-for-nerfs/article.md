# Variance Shadow Mapping for NeRFs

Also called: NeRF shadow mapping, VSM for NeRFs

## Overview
Variance Shadow Mapping for NeRFs is a shadow rendering technique that adapts the variance shadow mapping algorithm from traditional computer graphics to neural radiance fields. It solves the problem of computing soft shadows in NeRFs without performing expensive cumulative transmittance integrations along shadow rays [S1]. By comparing depth inconsistencies between the camera view and a virtual light view, it produces continuous visibility values in the range $[0, 1]$, avoiding the hard-edged artifacts typical of naive shadow mapping [S1].

## How it works
The method computes visibility $V(\mathbf{r})$ for a camera ray $\mathbf{r}$ by projecting the sampled 3D points into a light-space depth buffer and applying statistical filtering.

**Inputs:**
*   Camera ray $\mathbf{r}$ and its associated 3D sample points $\mathbf{x}$ obtained from the NeRF volume rendering process.
*   Camera intrinsics and extrinsics.
*   Light source position (defining the virtual camera).
*   Pre-rendered depth maps from the light view: mean depth map $X$ and squared depth map $X^2$.

**Outputs:**
*   Visibility factor $V(\mathbf{r}) \in [0, 1]$.

**Data Structures:**
*   **Depth Map ($X$):** A 2D array where each element stores the depth $z$ of the nearest surface point along the ray from the light source.
*   **Squared Depth Map ($X^2$):** A 2D array where each element stores the square of the depth $z^2$.
*   **Filtered Maps:** $\mu(X)$ and $\mu(X^2)$, resulting from box filtering the input maps.

**Steps:**

1.  **Reconstruct 3D Coordinates:**
    For each sample point $\mathbf{x}$ on the camera ray $\mathbf{r}$, determine its 3D coordinates using the camera parameters and the depth value associated with the pixel $\mathbf{r}$ in the camera's depth map [S1].

2.  **Project to Light View:**
    Set a virtual camera at the light source position. Transform the 3D coordinates of the sample points into the light camera's coordinate system. Determine the corresponding pixel $\mathbf{r}_l$ in the light view's depth buffer [S1].

3.  **Retrieve Light-Space Depth:**
    Let $z(\mathbf{r})$ be the depth of the sample point $\mathbf{x}$ as measured from the light source (i.e., the distance from the light camera origin to $\mathbf{x}$ along the light ray) [S1].

4.  **Retrieve Filtered Statistics:**
    Access the pre-computed filtered mean depth $\mu(\mathbf{X})(\mathbf{r}_l)$ and filtered mean squared depth $\mu(\mathbf{X}^2)(\mathbf{r}_l)$ at the corresponding pixel $\mathbf{r}_l$ in the light view [S1].

5.  **Calculate Variance:**
    Compute the variance $\sigma^2(\mathbf{r}_l)$ of the depth distribution in the local neighborhood defined by the filter [S1]:
    $$ \sigma^2(\mathbf{r}_l) = \mu(\mathbf{X}^2)(\mathbf{r}_l) - \mu^2(\mathbf{X})(\mathbf{r}_l) $$

6.  **Compute Visibility:**
    Calculate the visibility $V(\mathbf{r})$ using the Chebyshev inequality-based bound [S1]:
    $$ V(\mathbf{r}) = \begin{cases} 1, & z(\mathbf{r}) \leq \mu(\mathbf{X})(\mathbf{r}_l) \\ \frac{\sigma^2(\mathbf{r}_l)}{\sigma^2(\mathbf{r}_l) + (z(\mathbf{r}) - \mu(\mathbf{X})(\mathbf{r}_l))^2}, & z(\mathbf{r}) > \mu(\mathbf{X})(\mathbf{r}_l) \end{cases} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_vsm_visibility(sample_points, camera_intrinsics, light_camera_intrinsics, 
                           light_depth_map, light_depth_sq_map, filter_size):
    """
    Computes visibility for a set of sample points on a camera ray using Variance Shadow Mapping.
    
    Args:
        sample_points: List of 3D points (x, y, z) on the camera ray.
        camera_intrinsics: Camera matrix K.
        light_camera_intrinsics: Light camera matrix K_l.
        light_depth_map: 2D array of depth values Z from light view.
        light_depth_sq_map: 2D array of squared depth values Z^2 from light view.
        filter_size: Integer size of the box filter kernel.
        
    Returns:
        visibility: Float value in [0, 1].
    """
    # Step 1 & 2: Project sample points to light view
    # Note: In practice, this is done per-pixel during rendering. 
    # Here we assume we are processing a specific pixel r and its corresponding 3D point x.
    
    # Assume we have the 3D point x corresponding to the surface point on the camera ray
    # and we know its pixel coordinate r in the camera image.
    # We need the 3D coordinate x.
    
    # Reconstruct 3D point from camera ray and depth (simplified)
    # x_3d = unproject(r, depth_r, camera_intrinsics)
    # For this method, the 3D point x is already available from the volume rendering samples.
    
    # Transform 3D point to light camera space
    # x_light = light_camera_extrinsics * x_3d
    # z_light = x_light.z  # Depth in light view
    
    # Determine pixel coordinate r_l in light view
    # r_l = project(x_light, light_camera_intrinsics)
    
    # Step 3: Get depth from light view for this point
    # z_r = z_light
    
    # Step 4: Get filtered statistics from light view maps
    # The maps must be pre-filtered with a box filter of size filter_size x filter_size
    mu_X = box_filter(light_depth_map, filter_size)[r_l.y, r_l.x]
    mu_X2 = box_filter(light_depth_sq_map, filter_size)[r_l.y, r_l.x]
    
    # Step 5: Calculate Variance
    variance = mu_X2 - (mu_X ** 2)
    
    # Step 6: Compute Visibility
    if z_r <= mu_X:
        visibility = 1.0
    else:
        # Chebyshev inequality bound
        numerator = variance
        denominator = variance + (z_r - mu_X) ** 2
        if denominator == 0:
            visibility = 1.0 # Avoid division by zero; if variance is 0 and z > mu, it's an edge case
        else:
            visibility = numerator / denominator
            
    return visibility

def box_filter(input_map, kernel_size):
    """
    Applies a simple box filter to a 2D map.
    """
    height, width = input_map.shape
    output_map = np.zeros_like(input_map)
    half_k = kernel_size // 2
    
    # Efficient box filter implementation (summing over kernel)
    # For simplicity, we use a direct convolution
    for y in range(height):
        for x in range(width):
            sum_val = 0.0
            count = 0
            for ky in range(-half_k, half_k + 1):
                for kx in range(-half_k, half_k + 1):
                    ny = y + ky
                    nx = x + kx
                    if 0 <= ny < height and 0 <= nx < width:
                        sum_val += input_map[ny, nx]
                        count += 1
            output_map[y, x] = sum_val / count if count > 0 else 0.0
            
    return output_map
```

**Parameters:**
*   **Filter Size:** The size of the box filter applied to the depth and squared depth maps. Not given in the sources. Larger sizes produce softer shadows but may introduce blur artifacts.

**Edge Cases and Pitfalls:**
*   **Negative Variance:** Due to numerical precision errors, $\mu(X^2) - \mu^2(X)$ might be slightly negative. Clamp variance to 0 if negative [G].
*   **Division by Zero:** If variance is 0 and $z(\mathbf{r}) > \mu(\mathbf{X})$, the denominator becomes 0. This implies a sharp edge. The visibility should be handled carefully, potentially defaulting to 0 or 1 depending on the desired behavior [G].
*   **Depth Precision:** Depth values must be consistent between the camera and light views. Mismatches in coordinate systems or depth encoding can lead to incorrect visibility calculations [S1].

## Operating characteristics
*   **Time Complexity:** The visibility calculation per sample is $O(1)$ after the depth maps are pre-filtered. The pre-filtering step is $O(N \cdot K^2)$ where $N$ is the number of pixels and $K$ is the filter size.
*   **Memory Complexity:** Requires storing two additional depth maps (depth and squared depth) for the light view.
*   **Strengths:** Avoids the expensive integration of transmittance along shadow rays, making it efficient for NeRFs [S1]. Produces soft shadows naturally [S1].
*   **Limitations:** Relies on accurate depth maps from the light view. If the NeRF does not provide reliable depth information, the method may fail. The quality of shadows depends on the filter size and the accuracy of the depth maps [S1].
*   **When to use:** When soft shadows are desired in NeRF rendering and computational efficiency is a priority over physically accurate shadow integration.

## Background
This method was proposed in the context of relighting scenes with object insertions in Neural Radiance Fields [S1]. It draws inspiration from variance shadow mapping in traditional computer graphics [S1]. The key insight is that variance shadow mapping transforms discontinuous depth from the light perspective into continuous depth, leading to soft shadows with visibility in $[0, 1]$ [S1].

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
