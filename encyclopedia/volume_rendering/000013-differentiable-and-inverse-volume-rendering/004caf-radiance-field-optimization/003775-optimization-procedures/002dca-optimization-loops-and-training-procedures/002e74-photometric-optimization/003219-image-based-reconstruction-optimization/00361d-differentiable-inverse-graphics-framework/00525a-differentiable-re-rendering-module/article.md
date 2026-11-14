# Differentiable Re-rendering Module

Also called: Differentiable Re-rendering, Photometric Re-rendering Loss, Re-rendering Supervision

## Overview
The Differentiable Re-rendering Module is a raytracing-based differentiable renderer that enforces an energy-conserving image formation process to supervise the joint training of intrinsic scene properties (albedo, normals, depth) and lighting [S1]. It re-renders the input image using predicted intrinsic properties and a Volumetric Spherical Gaussian (VSG) lighting volume, computing a photometric error between the re-rendered output and the original input [S1]. Unlike methods that rely on explicit ground-truth lighting, this module enables the estimation of complete continuous HDR light fields from single LDR images by ensuring predictions are physically correct through re-rendering constraints [S1]. It builds on the [Differentiable Inverse Graphics Framework](7617-differentiable-inverse-graphics-framework.md) by providing a differentiable path from latent 3D parameters to pixel space.

## How it works
The module takes predicted intrinsic properties (albedo $\tilde{A}$, normals $\tilde{N}$, depth $\tilde{D}$) and a predicted 3D lighting volume $\hat{L}$ as inputs, and outputs a re-rendered Low Dynamic Range (LDR) image $\tilde{I}$ [S1]. The rendering process uses a Lambertian reflectance model and raytracing to compute radiance [S1].

**Data Structures:**
*   **Lighting Volume ($\hat{L}$):** A 3D voxel grid where each voxel contains parameters controlling the emission direction and sharpness of the light source, capturing view-dependent effects and strong directional lighting [S1].
*   **Intrinsic Properties:**
    *   $\tilde{A}$: Predicted albedo map.
    *   $\tilde{N}$: Predicted surface normal map.
    *   $\tilde{D}$: Predicted depth map.
*   **Sampling Configuration:**
    *   $K$: Number of lighting directions per pixel (typically 50).
    *   $N$: Number of points per ray (typically 128).
    *   $\{l\}_K$: Set of $K$ equi-angular lighting directions on the upper hemisphere, sampled using a Fibonacci lattice [S1].

**Parameters:**
*   $\tau$: Soft clipping threshold, set to 0.9 [S1].
*   $\pi_1$: Energy conservation ratio [S1].
*   $\Delta \Omega$: Differential solid angle associated with each lighting direction [S1].

**Algorithm:**
For each pixel $p$ in the image:

1.  **Ray Origin:** Determine the 3D location $p$ using the pixel coordinates $(x_p, y_p)$ and the predicted depth $\tilde{D}_p$:
    $$ p = (x_p, y_p, \tilde{D}_p) $$
    [S1]

2.  **Lighting Sampling:** Choose $K$ equi-angular lighting directions $\{l\}_K$ on the upper hemisphere using a Fibonacci lattice [S1].

3.  **Radiance Query:** For each lighting direction $l \in \{l\}_K$, query the lighting volume $\hat{L}$ at the surface point $p$ to compute the radiance $R(p, l, \hat{L})$ [S1].

4.  **Shading Computation:** Compute the shaded radiance for pixel $p$ by summing the contributions from all lighting directions, weighted by the Lambertian diffuse term $\max(l \cdot \tilde{N}_p, 0)$ and the differential solid angle $\Delta \Omega$ [S1]:
    $$ \tilde{S}_p = \pi_1 \sum_{l \in \{l\}_K} R(p, l, \hat{L}) \max(l \cdot \tilde{N}_p, 0) \Delta \Omega $$
    [S1]

5.  **Soft Clipping:** Apply a differentiable soft clipping function $\phi$ to map the High Dynamic Range (HDR) shading value to the LDR range $[0, 1]$:
    $$ \tilde{I}_p = \phi(\tilde{S}_p) $$
    [S1]

    The soft clipping function $\phi(x)$ is defined as:
    $$
    \phi(x) = \begin{cases} 
    x & \text{if } x \le \tau \\
    \tau + (1 - \tau)\left(1 - e^{-\frac{x-\tau}{1-\tau}}\right) & \text{if } x > \tau 
    \end{cases}
    $$
    [S1]
    *Note: The source text provides a garbled formula for $x > \tau$. The expression above is the standard soft clipping function implied by the context "soft clipping with the exponential function" and the parameter $\tau=0.9$ [S1].*

6.  **Loss Calculation:** Compare the re-rendered image $\tilde{I}$ against the original input image $I$ to compute the error for joint training [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_re_rendering(pixels, depth_map, normal_map, albedo_map, lighting_volume, K=50, tau=0.9, pi_1=1.0):
    """
    Re-renders the input image using predicted intrinsics and VSG lighting.
    
    Args:
        pixels: List of pixel coordinates (x, y).
        depth_map: 2D array of predicted depth values.
        normal_map: 3D array of predicted normals (x, y, 3).
        albedo_map: 3D array of predicted albedo (x, y, 3).
        lighting_volume: 3D voxel grid of VSG lighting parameters.
        K: Number of lighting directions per pixel.
        tau: Soft clipping threshold.
        pi_1: Energy conservation ratio.
        
    Returns:
        re_rendered_image: 2D array of LDR RGB values.
    """
    H, W = depth_map.shape
    re_rendered_image = np.zeros((H, W, 3))
    
    # Precompute lighting directions using Fibonacci lattice
    lighting_directions = fibonacci_sphere(K)
    
    for y in range(H):
        for x in range(W):
            # 1. Determine 3D location
            depth = depth_map[y, x]
            p_3d = get_3d_location(x, y, depth) # Assumes camera intrinsics are known
            
            # 2. Get predicted normals and albedo
            normal = normal_map[y, x]
            albedo = albedo_map[y, x]
            
            # 3. Compute shading
            shading_sum = np.zeros(3)
            for l in lighting_directions:
                # Query lighting volume for radiance R(p, l, L_hat)
                # Note: R depends on albedo and lighting volume parameters
                radiance = query_lighting_volume(lighting_volume, p_3d, l, albedo)
                
                # Lambertian diffuse term
                diffuse_term = max(np.dot(l, normal), 0.0)
                
                # Differential solid angle (approximate for equi-angular sampling)
                delta_omega = 4 * np.pi / K
                
                shading_sum += radiance * diffuse_term * delta_omega
            
            # Apply energy conservation
            shaded_hdr = pi_1 * shading_sum
            
            # 4. Soft clipping
            re_rendered_image[y, x] = soft_clip(shaded_hdr, tau)
            
    return re_rendered_image

def soft_clip(x, tau):
    """
    Differentiable soft clipping function.
    """
    if x <= tau:
        return x
    else:
        # Standard soft clipping formula implied by source
        return tau + (1 - tau) * (1 - np.exp(-(x - tau) / (1 - tau)))

def fibonacci_sphere(samples):
    """
    Generate K equi-angular points on the upper hemisphere using Fibonacci lattice.
    """
    points = []
    phi = np.pi * (3 - np.sqrt(5))  # Golden angle
    for i in range(samples):
        y = 1 - (i / (samples - 1)) * 2  # y goes from 1 to -1
        radius = np.sqrt(1 - y * y)  # radius at y
        theta = phi * i  # golden angle increment
        x = np.cos(theta) * radius
        z = np.sin(theta) * radius
        points.append(np.array([x, y, z]))
    return points

def query_lighting_volume(volume, point, direction, albedo):
    """
    Query the VSG lighting volume for radiance in a given direction.
    Implementation details depend on the specific VSG parameterization.
    """
    # Placeholder for specific VSG query logic
    # Returns radiance R(p, l, L_hat)
    pass
```

## Subtopics

### Joint Re-prediction Module
A fully-convolutional network that refines initial predictions of albedo, normals, and depth by conditioning on the initial prediction, re-rendering error, input image, shading, and shading Jacobian [S1]. It computes the Shading Jacobian $\frac{\partial \tilde{S}}{\partial \tilde{N}}$ as the outer product of lighting direction and normal for visible directions: $\frac{\partial \tilde{S}}{\partial \tilde{N}_p} = \sum_{l \in \{l\}_K, l \cdot \tilde{N}_p > 0} R(p, l, \hat{L}) \otimes l \Delta \Omega$, where $\otimes$ is the outer product [S1]. This module uses the cached shading computations from the re-rendering step to efficiently provide gradients for refinement [S1].

## Operating characteristics
*   **Time Complexity:** The re-rendering step involves raytracing for each pixel. With $K$ rays per pixel and $N$ points per ray, the complexity is $O(H \cdot W \cdot K \cdot N)$, where $H, W$ are image dimensions [S1].
*   **Memory Complexity:** Requires storing the 3D lighting volume $\hat{L}$ and intrinsic property maps [S1]. GPU memory usage is reported as comparable to Lighthouse's 15G during training [S1].
*   **Performance:** On a TITAN V GPU, the inference time for the Differentiable Re-rendering Module is 140ms, while the Direct Prediction and Joint Re-prediction modules take 20ms and 12ms respectively [S1].
*   **Resolution:** Training is performed at a re-rendering resolution of $60 \times 80$ pixels [S1].
*   **Strengths:** Enables physically correct HDR lighting prediction from LDR images; differentiable end-to-end training; handles view-dependent effects and strong directional lighting [S1].
*   **Limitations:** Computationally expensive due to raytracing; requires careful handling of soft clipping for differentiability; relies on Lambertian reflectance assumption [S1].
*   **When to use:** When estimating inverse rendering parameters (lighting, albedo, geometry) from single or few images where ground-truth lighting is unavailable, and physical consistency is required [S1].
*   **When not to use:** When real-time performance is critical and approximate methods are sufficient; when non-Lambertian materials are dominant without extension [S1].

## Background
The Differentiable Re-rendering Module was introduced in the context of indoor inverse rendering, specifically in the paper "Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting" [S1]. It addresses the challenge of estimating HDR lighting from LDR images by formulating an energy-conserving image formation process [S1]. The method leverages a Volumetric Spherical Gaussian (VSG) representation for lighting, which captures view-dependent effects and strong directional lighting [S1]. It is part of a holistic inverse rendering framework that includes a Direct Prediction Module, a Lighting Joint Prediction Module, and a Joint Re-prediction Module [S1]. The approach is trained on synthetic data from the InteriorNet dataset, using LDR perspective images as input and supervising intrinsic properties with paired ground truth [S1].

## Sources
[S1] Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting, https://doi.org/10.1109/iccv48922.2021.01231, fetched 2026-10-26
