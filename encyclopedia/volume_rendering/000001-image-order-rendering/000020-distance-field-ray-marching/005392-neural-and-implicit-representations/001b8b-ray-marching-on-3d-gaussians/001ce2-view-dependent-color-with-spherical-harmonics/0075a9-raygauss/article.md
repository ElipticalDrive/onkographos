# RayGauss

Also called: Volumetric Gaussian-Based Ray Casting

## Overview
RayGauss is a differentiable volume ray casting algorithm that approximates radiance and density fields using ray casting of irregularly distributed anisotropic Gaussian primitives to generate photorealistic novel views [S1, unofficial]. It treats 3D Gaussians as volumetric primitives and ray marches them in a NeRF-like manner, introducing a slab-by-slab integration scheme and augmenting spherical harmonics with spherical Gaussian encoding to better capture view-dependent color [S2]. Unlike 3D Gaussian Splatting, which uses a splatting algorithm that inherently produces rendering artifacts such as flickering, RayGauss uses volume ray casting which promises to produce minimal artifacts [S1, unofficial]. Unlike grid-based methods (e.g., Instant NGP, TetraNeRF) which are limited by the resolution of the underlying structure, RayGauss uses irregularly distributed points/primitives that efficiently adapt to scene geometry [S1, unofficial].

## How it works
**Inputs:** A set of $n$ anisotropic Gaussian primitives, each defined by a mean position $\mathbf{\mu}$, a covariance matrix $\mathbf{\Sigma}$ (or rotation and scale parameters), and view-dependent color coefficients. A set of camera viewpoints with corresponding images for optimization. Ray origins and directions for rendering.

**Outputs:** Rendered images (radiance) for given camera viewpoints. Optimized primitive parameters (position, covariance, color coefficients) after training.

**Data Structures:**
1.  **Primitive:** An anisotropic Gaussian function.
    *   $\mathbf{\mu} \in \mathbb{R}^3$: Mean position.
    *   $\mathbf{\Sigma} \in \mathbb{R}^{3 \times 3}$: Covariance matrix, ensuring elliptical shape.
    *   $\alpha \in \mathbb{R}$: Scalar opacity/density amplitude.
    *   $\mathbf{c}_{SH} \in \mathbb{R}^{k}$: Spherical Harmonics coefficients for view-dependent color.
    *   $\mathbf{c}_{SG} \in \mathbb{R}^{m}$: Spherical Gaussian coefficients for view-dependent color [S2].
2.  **Bounding Volume Hierarchy (BVH):** Acceleration structure storing Axis-Aligned Bounding Boxes (AABBs) for each primitive.
    *   AABB: Defined by min/max coordinates $[\mathbf{b}_{min}, \mathbf{b}_{max}]$.
    *   Construction: Uses a dedicated API (e.g., OptiX) [S1, unofficial].

**Formulas:**
The global density $\sigma(\mathbf{x})$ and radiance $L(\mathbf{x}, \mathbf{v})$ fields are derived from the radiative transfer equation for a medium containing $n$ independent entities [S1, unofficial].

The individual density of a primitive $i$ is:
$$ \sigma_i(\mathbf{x}) = \alpha_i \cdot \mathcal{N}(\mathbf{x}; \mathbf{\mu}_i, \mathbf{\Sigma}_i) $$
where $\mathcal{N}$ is the anisotropic Gaussian function:
$$ \mathcal{N}(\mathbf{x}; \mathbf{\mu}, \mathbf{\Sigma}) = \exp\left(-\frac{1}{2}(\mathbf{x} - \mathbf{\mu})^T \mathbf{\Sigma}^{-1} (\mathbf{x} - \mathbf{\mu})\right) $$

The radiance emitted by primitive $i$ in direction $\mathbf{v}$ is parameterized by a combination of spherical harmonics (SH) and spherical Gaussians (SG) [S1, unofficial][S2]:
$$ L_i(\mathbf{x}, \mathbf{v}) = \text{SH}(\mathbf{v}, \mathbf{c}_{SH,i}) + \text{SG}(\mathbf{v}, \mathbf{c}_{SG,i}) $$

The global density and radiance are the superposition of individual primitives:
$$ \sigma(\mathbf{x}) = \sum_{i=1}^{n} \sigma_i(\mathbf{x}) $$
$$ L(\mathbf{x}, \mathbf{v}) = \sum_{i=1}^{n} L_i(\mathbf{x}, \mathbf{v}) $$

**Rendering Steps:**
1.  **Bounding Primitives:** Since anisotropic Gaussian functions have unbounded support, restrict evaluation to the region where density exceeds a threshold $\tau$. Compute the AABB for each primitive based on this threshold [S1, unofficial].
2.  **BVH Construction:** Calculate AABBs and build the BVH using an API like OptiX [S1, unofficial].
3.  **Ray Traversal:** For each pixel, cast a ray. Traverse the BVH to find intersected primitives.
4.  **Slab-by-Slab Integration:** Evaluate the discretized rendering equation over successive segments (slabs) containing multiple samples. Compute intersections of multiple samples with primitives directly to reduce intersection costs [S1, unofficial][S2].
5.  **Volume Rendering:** Integrate density and radiance along the ray using standard volume rendering accumulation (alpha blending) [S1, unofficial].

**Optimization:**
Optimize primitive parameters (position, covariance, SH/SG coefficients) using gradient descent on the difference between rendered images and ground truth images from known viewpoints, similar to NeRF [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_density(x, mu, Sigma_inv):
    """
    Compute anisotropic Gaussian density.
    x: (3,) point
    mu: (3,) mean
    Sigma_inv: (3,3) inverse covariance
    """
    diff = x - mu
    exponent = -0.5 * np.dot(diff, np.dot(Sigma_inv, diff))
    return np.exp(exponent)

def view_dependent_color(v, sh_coeffs, sg_coeffs):
    """
    Compute radiance from view direction using SH and SG.
    v: (3,) normalized view direction
    sh_coeffs: array of SH coefficients
    sg_coeffs: array of SG coefficients
    """
    # Placeholder for actual SH and SG evaluation functions
    # In practice, these are standard mathematical functions
    sh_val = evaluate_spherical_harmonics(v, sh_coeffs)
    sg_val = evaluate_spherical_gaussians(v, sg_coeffs)
    return sh_val + sg_val

def render_ray(ray_origin, ray_direction, primitives, threshold=1e-4):
    """
    Render a single ray using volume ray casting.
    primitives: list of dicts with keys 'mu', 'Sigma_inv', 'alpha', 'sh_coeffs', 'sg_coeffs', 'aabb'
    """
    # 1. Traverse BVH to get relevant primitives
    # This step is abstracted as finding primitives whose AABB intersects the ray
    relevant_primitives = traverse_bvh(ray_origin, ray_direction, primitives)
    
    # 2. Collect intersection intervals and sort by distance
    intervals = []
    for prim in relevant_primitives:
        # Compute intersection of ray with primitive's AABB
        t_min, t_max = intersect_aabb(ray_origin, ray_direction, prim['aabb'])
        if t_max > 0:
            intervals.append((t_min, t_max, prim))
    
    # Sort intervals by t_min
    intervals.sort(key=lambda x: x[0])
    
    # 3. Slab-by-slab integration
    total_transmittance = 1.0
    accumulated_color = np.zeros(3)
    
    # Define ray segment length for sampling
    step_size = 0.01 # Not given in sources, typical value for ray marching
    
    current_t = 0.0
    
    for t_min, t_max, prim in intervals:
        # Sample points along the ray within the slab
        t_start = max(current_t, t_min)
        t_end = t_max
        
        # Number of samples in this slab
        num_samples = int((t_end - t_start) / step_size)
        if num_samples == 0:
            num_samples = 1
            
        for i in range(num_samples):
            t = t_start + (i + 0.5) * step_size
            if t > t_end:
                break
                
            # Sample point on ray
            x = ray_origin + t * ray_direction
            
            # Compute density
            density = prim['alpha'] * gaussian_density(x, prim['mu'], prim['Sigma_inv'])
            
            # Compute radiance
            # View direction is opposite to ray direction for outgoing light
            v = -ray_direction
            radiance = view_dependent_color(v, prim['sh_coeffs'], prim['sg_coeffs'])
            
            # Volume rendering equation: dL = L * (1 - exp(-sigma * dt))
            # Approximation for small dt: dL = L * sigma * dt
            alpha = 1.0 - np.exp(-density * step_size)
            
            accumulated_color += total_transmittance * alpha * radiance
            total_transmittance *= (1.0 - alpha)
            
            if total_transmittance < 1e-4:
                break
                
        current_t = t_end
        
    return accumulated_color

def optimize_primitives(primitives, images, cameras, lr=0.01):
    """
    Optimize primitive parameters using gradient descent.
    """
    for epoch in range(1000): # Not given in sources, typical training loop
        for image, camera in zip(images, cameras):
            # Render image
            rendered_image = render_image(camera, primitives)
            
            # Compute loss (e.g., MSE)
            loss = np.mean((rendered_image - image) ** 2)
            
            # Backpropagate and update parameters
            # This requires differentiable rendering of the Gaussian primitives
            gradients = compute_gradients(loss, primitives)
            update_parameters(primitives, gradients, lr)
```

**Parameters:**
*   **Threshold ($\tau$):** Density threshold for bounding primitives. Not given in the sources.
*   **Step Size:** Distance between samples along the ray. Not given in the sources.
*   **Spherical Harmonics Degree:** Determines the number of SH coefficients. Not given in the sources.
*   **Spherical Gaussian Order:** Determines the number of SG coefficients. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Unbounded Support:** Anisotropic Gaussians have infinite support. Must be truncated using a threshold to compute AABBs for BVH construction [S1, unofficial].
*   **Precision:** High precision required for density and radiance calculations to avoid artifacts [S1, unofficial].
*   **Degenerate Input:** If primitives are too small or too large, rendering quality may suffer. Optimization must handle this [S1, unofficial].

## Subtopics

### [Progressive View-Dependent Color Baking](2834-progressive-view-dependent-color-baking.md)
A technique to bake view-dependent color into textures over time. Choose when real-time performance is critical and offline baking is acceptable.

### [Spherical Beta View-Dependent Color](3823-spherical-beta-view-dependent-color.md)
Uses spherical beta functions for view-dependent color. Choose when smooth, physically plausible view-dependent effects are needed.

### [Spherical Harmonics Per Point](7172-spherical-harmonics-per-point.md)
Assigns SH coefficients to each point. Choose when simple view-dependent effects are sufficient.

### [Spherical Beta Function](12574-spherical-beta-function.md)
A mathematical function for view-dependent color. Choose as a building block for other methods.

## Operating characteristics
*   **Time Complexity:** Slower than 3D Gaussian Splatting due to volume ray marching. Training is 6× slower than 3DGS [S2]. Frame rates are approximately 25× lower than 3DGS [S2].
*   **Memory Complexity:** Depends on the number of primitives. BVH construction adds overhead [S1, unofficial].
*   **Measured Performance:** Yields higher PSNR than 3DGS [S2].
*   **Strengths:** High quality, minimal artifacts compared to splatting, adapts to scene geometry [S1, unofficial][S2].
*   **Limitations:** Slow performance, high training time [S2].
*   **When to use:** When high quality is more important than speed, and artifacts from splatting are unacceptable [S1, unofficial][S2].
*   **When not to use:** When real-time performance is critical [S2].

## Background
RayGauss was introduced by Blanc et al. in 2025 [S2]. It is inspired by point cloud inspired representations and aims to develop a differentiable volume ray casting algorithm capable of rendering primitives represented by elliptical basis functions [S1, unofficial]. The work validates modeling choices through empirical tests, demonstrating that anisotropic Gaussians combined with spherical harmonics and spherical Gaussians achieve accurate rendering [S1, unofficial].

## Sources
[S1] RayGauss: Volumetric Gaussian-Based Ray Casting for Photorealistic Novel View Synthesis, https://www.youtube.com/watch?v=F8tg0CTBiCo, fetched 2026-10-27 (unofficial)
[S2] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
