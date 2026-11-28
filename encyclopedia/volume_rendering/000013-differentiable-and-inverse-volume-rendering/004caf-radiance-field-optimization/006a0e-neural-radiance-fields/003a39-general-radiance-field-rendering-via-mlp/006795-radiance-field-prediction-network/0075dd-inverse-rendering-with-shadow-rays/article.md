# Inverse Rendering with Shadow Rays

Also called: Radiance field with explicit direct illumination

## Overview
Inverse Rendering with Shadow Rays is a technique that augments a global illumination radiance field with explicit direct illumination terms from known point light sources, computed via shadow rays to improve mesh reconstruction and novel view synthesis accuracy [S1]. It sets itself apart from sibling methods like the base Radiance Field Prediction Network by decoupling direct lighting from the learned global illumination residual, allowing for physically accurate shadow casting that rasterization-based point-based methods cannot achieve [S1].

## How it works
This method extends the radiance field representation by explicitly modeling direct illumination from known light sources while retaining a neural network to predict the residual appearance (indirect illumination and material properties).

**Inputs and Outputs**
*   **Inputs:**
    *   Scene geometry initialization: Dense point cloud, surface normals, and camera poses (typically extracted via COLMAP) [S1].
    *   Light source data: Positions of point light sources, which may vary per image in the experimental setup [S1].
    *   Canonicalized 3D points $x$ and view directions $\omega$ [S1].
*   **Outputs:**
    *   Radiance $L_{\text{sh.rays}}(x, \omega)$ for a given point and view direction [S1].
    *   Albedo $\alpha(x)$ as an additional output of the MLP [S1].

**Data Structures**
The method relies on the underlying Fast Dipole Sums representation. Key data structures include:
*   **Dipole Parameters:** Stored per point, including the mean position and dipole strength.
*   **Kernel Outputs:** $\bar{\ell}_\epsilon^k(x)$, the outputs of the Poisson kernel functions for $K$ kernels [S1].
*   **MLP:** A neural network taking inputs $(x, \omega, n_{\text{imp}}(x), \bar{\ell}_\epsilon^1(x), \dots, \bar{\ell}_\epsilon^K(x))$ [S1].
*   **Normals:** $n_{\text{imp}}(x)$, the normal at point $x$ [S1].

**Numbered Steps**
1.  **Initialization:** Initialize the scene with a dense point cloud, normals, and camera poses using COLMAP. Use ground truth point light positions for each view [S1].
2.  **Optimization Setup:** Prepare two optimization runs: one using the standard radiance field representation (global illumination only) and one augmented with shadow rays [S1].
3.  **Global Illumination Baseline:** For the baseline, use the radiance field representation defined in Equation 21 of the source, which models global (direct and indirect) illumination implicitly through the MLP and kernel outputs [S1].
4.  **Direct Illumination Augmentation:** For the shadow ray method, augment the radiance field equation to include explicit direct illumination terms for each point light source [S1].
5.  **Radiance Computation:** For each ray passing through point $x$ with direction $\omega$, compute the total radiance by summing the direct illumination from all light sources and the residual predicted by the MLP [S1].
6.  **Albedo and Transmittance:** Compute albedo $\alpha(x)$ as an additional output of the MLP. Compute transmittance $T(x, l_i)$ using quadrature and fast queries [S1].
7.  **Regularization Guard:** To prevent numerical errors and undefined values during training, apply a small cutoff (desingularization) in the denominator of the Poisson kernel. Do not set the kernel width parameter $\epsilon$ to 0 [S1].

**Formulas**
The total radiance with shadow rays is defined as [S1]:

$$
L_{\text{sh.rays}}(x, \omega) \equiv \sum_{i=1,2} L_d^i(x, \omega) + \text{MLP}(x, \omega, n_{\text{imp}}(x), \bar{\ell}_\epsilon^1(x), \dots, \bar{\ell}_\epsilon^K(x))
$$

Where $L_d^i(x, \omega)$ is the direct illumination from the $i$-th light source [S1]:

$$
L_d^i(x, \omega) \equiv \alpha(x) T(x, l_i) \frac{n_{\text{imp}}(x) \cdot (l_i - x)}{\|l_i - x\|^3}
$$

**Symbol Definitions:**
*   $L_{\text{sh.rays}}(x, \omega)$: Total radiance at point $x$ in direction $\omega$ with shadow rays.
*   $L_d^i(x, \omega)$: Direct illumination contribution from light source $i$.
*   $\alpha(x)$: Albedo at point $x$, predicted by the MLP [S1].
*   $T(x, l_i)$: Exponential transmittance between point $x$ and light source $l_i$, computed via quadrature [S1].
*   $n_{\text{imp}}(x)$: Normal vector at point $x$ [S1].
*   $l_i$: Position of the $i$-th light source [S1].
*   $x$: 3D position of the point on the surface [S1].
*   $\text{MLP}(\dots)$: Neural network predicting the residual appearance based on position, view direction, normal, and kernel outputs [S1].
*   $\bar{\ell}_\epsilon^k(x)$: Kernel outputs for $k=1 \dots K$ [S1].

**Parameters**
*   **Light Sources:** Two point light sources are used in the primary experiment [S1].
*   **Kernel Regularization:** A small cutoff is applied to the Poisson kernel denominator to avoid numerical instability; $\epsilon \neq 0$ [S1].
*   **Quadrature:** Used for computing transmittance $T(x, l_i)$ [S1].

**Edge Cases and Pitfalls**
*   **Numerical Instability:** Setting the kernel width parameter $\epsilon$ to 0 (removing regularization) causes training failures due to numerical errors and undefined values [S1].
*   **Mesh Artifacts:** Using desingularization (a small cutoff in the denominator) without proper optimization can result in extracted meshes with strong artifacts [S1].
*   **Boundary Conditions:** The method assumes known light positions. If light positions are unknown or dynamic, the explicit shadow ray formulation requires adaptation [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_radiance_with_shadow_rays(x, omega, normals, light_positions, mlp, kernel_outputs):
    """
    Computes radiance at point x for view direction omega using shadow rays.
    
    Args:
        x: 3D point position (vector)
        omega: View direction (vector)
        normals: Surface normal at x (vector)
        light_positions: List of light source positions (list of vectors)
        mlp: Neural network function taking (x, omega, normal, kernel_outputs)
        kernel_outputs: List of kernel outputs [l_1, ..., l_K]
        
    Returns:
        radiance: Scalar radiance value
        albedo: Scalar albedo value
    """
    # 1. Get albedo and residual from MLP
    # The MLP takes position, view direction, normal, and kernel outputs
    residual = mlp(x, omega, normals, kernel_outputs)
    albedo = residual[-1]  # Assuming albedo is the last output or extracted separately
    residual_appearance = residual[:-1]  # Remaining outputs for indirect illumination
    
    # 2. Compute direct illumination from each light source
    direct_illumination = 0.0
    
    for light_pos in light_positions:
        # Vector from point to light
        light_vec = light_pos - x
        
        # Distance squared for normalization and transmittance
        dist_sq = dot(light_vec, light_vec)
        dist = sqrt(dist_sq)
        
        # Normalized light direction
        light_dir = light_vec / dist
        
        # Compute transmittance T(x, l_i) using quadrature
        # Note: In implementation, this requires sampling along the ray from x to light_pos
        # and integrating the density. Here we assume a function compute_transmittance exists.
        transmittance = compute_transmittance_quadrature(x, light_pos)
        
        # Lambertian shading term: n . l / dist^2
        # Note: The formula uses dist^3 in denominator but dot product has one dist factor in numerator if l is not normalized?
        # Formula: (n . (l - x)) / ||l - x||^3
        # Let's stick strictly to the source formula:
        # n_imp(x) . (l_i - x) / ||l_i - x||^3
        # This is equivalent to (n . l_dir) / dist^2 if l_dir is normalized.
        # Source formula: n . (l-x) / ||l-x||^3
        # (l-x) has length dist. So (n . (l-x)) / dist^3 = (n . (l_dir * dist)) / dist^3 = (n . l_dir) / dist^2.
        
        dot_product = dot(normals, light_vec)
        direct_term = (albedo * transmittance * dot_product) / (dist_sq * dist)
        
        direct_illumination += direct_term
        
    # 3. Sum direct and residual
    total_radiance = direct_illumination + sum(residual_appearance)
    
    return total_radiance, albedo

def compute_transmittance_quadrature(start_point, light_point):
    """
    Computes exponential transmittance between start_point and light_point using quadrature.
    This requires access to the density field sigma(x) from the underlying radiance field.
    """
    # Sample points along the segment from start_point to light_point
    num_samples = 10  # Not given in sources, typical value for quadrature
    transmittance_integral = 0.0
    
    for i in range(num_samples):
        t = i / num_samples
        sample_point = lerp(start_point, light_point, t)
        density = query_density(sample_point)  # Query from underlying radiance field
        
        # Simple Riemann sum approximation
        step_size = distance(start_point, light_point) / num_samples
        transmittance_integral += density * step_size
        
    # Exponential transmittance
    return exp(-transmittance_integral)
```

## Subtopics

### Global Illumination Radiance Field (Base Method)
The underlying representation that models indirect and direct illumination implicitly through the MLP and kernel outputs. Choose this when light sources are unknown or when computational simplicity is prioritized over explicit shadow accuracy. See [Radiance Field Prediction Network](19916-radiance-field-prediction-network.md).

### Rasterization-Based Point Rendering
Methods like Gaussian surfels that use rasterization instead of ray tracing. Choose these for real-time performance where explicit shadow casting is not required. See [Gaussian Surfels](https://doi.org/10.1145/3687914).

## Operating characteristics
*   **Time Complexity:** Depends on the number of light sources and the cost of transmittance quadrature. Each ray requires querying the MLP and computing transmittance for each light source. Transmittance computation via quadrature adds a linear factor with respect to the number of quadrature samples [S1].
*   **Memory Complexity:** Similar to the base radiance field, plus storage for light source positions. The MLP and kernel outputs dominate memory usage [S1].
*   **Measured Performance:** In the Lego scene experiment, rendering 200 images from random viewpoints with two point lights showed that optimizing with shadow rays resulted in extracted meshes with fewer artifacts and finer details, and images with more accurate shadows compared to the method without shadow rays [S1].
*   **Strengths:** Enables rendering of direct illumination from known light sources, which rasterization methods cannot do. Leads to fewer mesh artifacts and more accurate shadows in novel views [S1].
*   **Limitations:** Requires known light source positions. Numerical instability can occur if regularization is removed. Desingularization can introduce mesh artifacts if not handled carefully [S1].
*   **When to Use:** When reconstructing scenes with known illumination and high-fidelity shadow casting is required.
*   **When Not to Use:** When light sources are unknown, or when real-time performance is critical and rasterization is sufficient.

## Background
The method originates from the paper "3D Reconstruction with Fast Dipole Sums" [S1]. It builds upon the Fast Dipole Sums radiance field representation, which uses dipole sums to model global illumination. The specific augmentation with shadow rays was introduced to address the limitations of rasterization-based point-based methods in rendering explicit direct illumination and shadows [S1]. The technique was demonstrated on the NeRF Realistic Synthetic Lego scene, highlighting its ability to improve mesh reconstruction quality and novel view synthesis accuracy under varying lighting conditions [S1].

## Sources
[S1] 3D Reconstruction with Fast Dipole Sums, https://doi.org/10.1145/3687914, fetched 2026-10-28
