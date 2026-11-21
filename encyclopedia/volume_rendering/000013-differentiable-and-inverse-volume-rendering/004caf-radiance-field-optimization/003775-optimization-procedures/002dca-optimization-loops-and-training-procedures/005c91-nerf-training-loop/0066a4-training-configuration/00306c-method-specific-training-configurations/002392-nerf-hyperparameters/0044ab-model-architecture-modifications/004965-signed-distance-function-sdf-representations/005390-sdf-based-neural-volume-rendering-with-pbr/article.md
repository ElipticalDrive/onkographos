# SDF-based Neural Volume Rendering with PBR

Also called: Neural volume rendering based on PBR and neural implicit surfaces, SDF-NeRF with PBR

## Overview
This method integrates Signed Distance Function (SDF) representations for geometry with Physically Based Rendering (PBR) material models to achieve smoother geometric structures and accurate specular reflections, overcoming the coarse detail and foggy artifacts typical of standard NeRF [S1]. Unlike sibling methods such as [ProbeSDF](11363-probesdf.md) or [SDF-based Neural Fields](12947-sdf-based-neural-fields.md) which may focus primarily on geometry or general neural fields, this approach specifically decouples geometry (via SDF) from material appearance (via PBR), providing interpretable parameters for reflection and diffuse components [S1]. It builds on [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md) by converting the SDF into a density field for volumetric integration while using the SDF gradient for surface normals required by the PBR model.

## How it works
**Inputs:** Camera poses and pixel information (images).
**Outputs:** Rendered image colors $C(r)$ for each ray $r$.

**Data Structures:**
1.  **Spatial MLP:** An $N$-layer Multi-Layer Perceptron.
    *   Input: Positional Encoded (PE) 3D coordinates $\mathbf{x}$.
    *   Output: Intermediate features used to derive SDF value $f(\mathbf{x})$, normal vector $\mathbf{n}$, volume density $\tau(\mathbf{x})$, and diffuse color $C_d$.
2.  **Directional MLP:** (Implied by "directional MLP" in source [S1] for prefiltering, though details are sparse).
    *   Input: Surface normal $\mathbf{n}$, viewing direction $\omega_o$, roughness $\rho$.
    *   Output: Prefiltered specular color $f_{prefilteredColor}$.
3.  **BRDF LUT:** A precomputed lookup table indexed by $\cos\theta$ and roughness $\rho$.
    *   Fields: `.r` (red channel), `.g` (green channel).
4.  **Ray Sampling Buffer:** Stores sampled points $\mathbf{x}_i$, densities $\sigma_i$, and colors $c_i$ along a ray.

**Parameters:**
*   $\alpha$: Learnable parameter controlling the transition between volume and surface density. Large $\alpha$ implies a surface with low transmittance (e.g., metal); $\alpha \to 0$ implies a homogeneous medium (e.g., fog) [S1].
*   $\beta$: Scale parameter for the Laplace distribution. In VolSDF, it is fixed as $\alpha^{-1}$ [S1].
*   $\rho$: Surface roughness.
*   $F_0$: Reflectance at a perpendicular angle.
*   $\lambda$: Weight for the SDF loss term.

**Steps:**

1.  **Ray Generation:** For each pixel, generate a ray $\mathbf{x} = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin, $\mathbf{d}$ is the direction, and $t$ is incrementally adjusted to sample points along the ray path [S1].
2.  **Positional Encoding:** Apply one round of positional encoding (PE) to the sampled 3D points $\mathbf{x}$ [S1].
3.  **Spatial Feature Extraction:** Feed the encoded points into the Spatial MLP to generate perspective-independent intermediate values. The MLP outputs:
    *   The SDF value $f(\mathbf{x})$.
    *   The volume density $\tau(\mathbf{x})$ derived from $f(\mathbf{x})$ [S1].
    *   The diffuse color $C_d$ [S1].
    *   The normal vector $\mathbf{n}$ is derived from the SDF gradient $\nabla f(\mathbf{x})$ [S1].
4.  **Density Conversion:** Convert the SDF value $f(\mathbf{x})$ to volume density $\sigma(\mathbf{x})$ (denoted as $\tau$ in the source formula context, but used as $\sigma$ in the rendering equation) using the cumulative distribution function $\Psi_\beta$ of a zero-mean Laplace distribution with scale $\beta > 0$:
    $$ \sigma(\mathbf{x}) = \alpha \Psi_\beta(f(\mathbf{x})) $$
    where $\Psi_\beta$ is the CDF of the Laplace distribution [S1].
5.  **Diffuse Color Retrieval:** Obtain $C_d$ directly from the Spatial MLP output. This simulates irradiance with low angular correlation, representing subsurface scattering or air impurities [S1].
6.  **Specular Color Calculation (PBR):**
    a.  **Prefiltering:** Calculate the prefiltered specular color $f_{prefilteredColor}$ using a Directional MLP or sampling. The incident direction $\omega_i$ for the prefiltering is the reflection of the viewing direction $\omega_o$ about the normal $\mathbf{n}$:
        $$ \omega_i = 2 (\omega_o \cdot \mathbf{n}) \mathbf{n} - \omega_o $$
        The prefiltered color depends on roughness $\rho$, position $\mathbf{p}$, and $\omega_i$ [S1].
    b.  **BRDF Integration Approximation:** Use the split sum approximation to separate the integration into a prefiltering texture part and a BRDF LUT part [S1]:
        $$ L_0(\mathbf{p}, \omega_o) \approx \int_{\Omega} L_i(\mathbf{p}, \omega_i) \, d\omega_i * \int_{\Omega} f_r(\mathbf{p}, \omega_i, \omega_o) \cos \theta d\omega_i $$
        The right side (BRDF integration) is approximated using the BRDF LUT:
        $$ \text{BRDF Term} = F_0 * \text{brdfLUT}.r + \text{brdfLUT}.g $$
        where the LUT is indexed by $\cos\theta$ and roughness $\rho$ [S1].
    c.  **Final Specular Term:** Calculate the specular color $C_s$:
        $$ C_s = f_{prefilteredColor} * (F_0 * A + B) $$
        where $A$ and $B$ correspond to the `.r` and `.g` outputs of the BRDF LUT respectively [S1].
7.  **Composite Color:** Compute the total color $c_i$ at each sample point:
    $$ c_i = C_d + C_s $$
8.  **Volume Rendering:** Render the final pixel color $C(r)$ using the discrete volume rendering equation:
    $$ C(r) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
    where $\sigma_i$ is the volume density at sample point $i$, $\delta_i$ is the distance between sample point $i$ and $i+1$, and $T_i$ is the cumulative transmittance:
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    with $T_1 = 1$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def laplace_cdf(x, beta):
    """
    Cumulative distribution function of zero-mean Laplace distribution with scale beta.
    """
    if beta == 0:
        return 0.5 if x == 0 else (1.0 if x > 0 else 0.0)
    return 0.5 * (1.0 + math.copysign(1.0, x) * (1.0 - math.exp(-abs(x) / beta)))

def get_normal_from_sdf(sdf_mlp, x):
    """
    Computes the normal vector as the gradient of the SDF.
    Uses finite differences for gradient estimation.
    """
    eps = 1e-4
    x_vec = np.array(x)
    sdf_val = sdf_mlp(x_vec)
    
    grad = np.zeros(3)
    for i in range(3):
        x_plus = x_vec.copy()
        x_minus = x_vec.copy()
        x_plus[i] += eps
        x_minus[i] -= eps
        grad[i] = (sdf_mlp(x_plus) - sdf_mlp(x_minus)) / (2 * eps)
    
    return grad / np.linalg.norm(grad)

def get_brdf_lut_value(cos_theta, roughness, lut):
    """
    Retrieves A and B values from the precomputed BRDF LUT.
    lut is a 2D array indexed by [cos_theta_bin, roughness_bin].
    """
    # Binarize indices based on resolution of LUT
    # Implementation detail: mapping continuous values to discrete indices
    idx_cos = int(cos_theta * (lut.shape[0] - 1))
    idx_rough = int(roughness * (lut.shape[1] - 1))
    
    # Clamp indices
    idx_cos = max(0, min(idx_cos, lut.shape[0] - 1))
    idx_rough = max(0, min(idx_rough, lut.shape[1] - 1))
    
    A = lut[idx_cos, idx_rough, 0] # .r channel
    B = lut[idx_cos, idx_rough, 1] # .g channel
    return A, B

def render_ray(ray_origin, ray_direction, spatial_mlp, directional_mlp, brdf_lut, alpha, beta, num_samples):
    """
    Renders a single ray using SDF-based Neural Volume Rendering with PBR.
    """
    # 1. Sample points along the ray
    # Assuming a fixed range or adaptive sampling based on SDF zero-crossing
    # For simplicity, assume uniform sampling in a bounding box or near-surface region
    t_values = np.linspace(0.0, 10.0, num_samples) # Placeholder range
    points = ray_origin + ray_direction * t_values[:, np.newaxis]
    
    # 2. Positional Encoding (Placeholder for actual PE function)
    encoded_points = positional_encoding(points)
    
    # 3. Spatial MLP Inference
    # Output: SDF values, Diffuse Color, and potentially other features
    # Note: Source says MLP outputs normal and density. 
    # We compute normal from SDF gradient explicitly as per formula (4).
    
    sdf_values = spatial_mlp(encoded_points) # Shape: (N,)
    diffuse_colors = spatial_mlp_get_diffuse(encoded_points) # Shape: (N, 3)
    
    # 4. Compute Density from SDF
    # tau(x) = alpha * Psi_beta(f(x))
    densities = alpha * np.array([laplace_cdf(f, beta) for f in sdf_values])
    
    # 5. Compute Normals from SDF Gradient
    normals = np.zeros((num_samples, 3))
    for i in range(num_samples):
        normals[i] = get_normal_from_sdf(spatial_mlp, points[i])
        
    # 6. Compute Colors
    final_colors = np.zeros((num_samples, 3))
    
    for i in range(num_samples):
        x = points[i]
        n = normals[i]
        C_d = diffuse_colors[i]
        
        # Specular Component
        # View direction is opposite to ray direction for BRDF
        # However, in NeRF context, ray_direction is from camera to scene.
        # Viewing direction omega_o is from surface to camera = -ray_direction
        omega_o = -ray_direction
        
        # Prefiltering: Incident direction is reflection of view direction
        # omega_i = 2 * (omega_o . n) * n - omega_o
        dot_no = np.dot(omega_o, n)
        omega_i = 2 * dot_no * n - omega_o
        
        # Get roughness (assumed to be a parameter or learned per point)
        # Source implies roughness rho is an input to the directional MLP/LUT
        # For this implementation, we assume a global or per-point roughness
        roughness = 0.5 # Placeholder, typically learned or input
        
        # Directional MLP for prefiltered color
        # f_prefilteredColor = f(rho, p, omega_i)
        # We assume directional_mlp takes encoded normal, view, roughness etc.
        # Simplified: assume it returns the prefiltered color directly
        f_prefiltered = directional_mlp(n, omega_o, roughness) # Shape: (3,)
        
        # BRDF LUT lookup
        # cos_theta is the angle between omega_o and n? 
        # In split sum, the integral is over the hemisphere. 
        # The LUT is indexed by cos(theta) where theta is the angle between 
        # the half-vector and normal? Or viewing angle?
        # Source formula 10 uses cos(theta) in LUT index. 
        # Standard PBR LUTs use cos(theta_v) (view angle) and roughness.
        cos_theta = max(0.0, dot_no) # omega_o . n
        
        A, B = get_brdf_lut_value(cos_theta, roughness, brdf_lut)
        
        # F0 is reflectance at perpendicular angle (input or learned)
        F0 = 0.04 # Default dielectric, or learned
        
        # C_s = f_prefilteredColor * (F0 * A + B)
        C_s = f_prefiltered * (F0 * A + B)
        
        final_colors[i] = C_d + C_s
        
    # 7. Volume Rendering
    # C(r) = sum T_i * (1 - exp(-sigma_i * delta_i)) * c_i
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    
    delta_t = np.diff(t_values)
    delta_t = np.insert(delta_t, 0, 0.0) # Delta for first point is 0 or handled by T_1=1
    
    # Calculate optical depth
    # sigma_i * delta_i
    optical_depths = densities * delta_t
    
    # Cumulative transmittance
    # T_i = exp(-sum_{j=0}^{i-1} optical_depths[j])
    # Note: Source uses 1-based indexing j=1 to i-1. 
    # T_1 = 1 (sum is empty).
    
    T = np.ones(num_samples)
    current_transmittance = 1.0
    rendered_color = np.zeros(3)
    
    for i in range(num_samples):
        # T_i is already computed from previous step
        # Contribution: T_i * (1 - exp(-sigma_i * delta_i)) * c_i
        # Note: delta_i is distance between i and i+1. 
        # For the last point, delta is 0 or handled by boundary.
        
        # If i is the last point, delta might be 0, making contribution 0.
        # Standard NeRF rendering handles this by assuming infinite tail or fixed range.
        
        if i < num_samples - 1:
            sigma_i = densities[i]
            delta_i = t_values[i+1] - t_values[i]
            
            alpha_i = 1.0 - math.exp(-sigma_i * delta_i)
            weight = T[i] * alpha_i
            
            rendered_color += weight * final_colors[i]
            
            # Update transmittance for next point
            T[i+1] = T[i] * (1.0 - alpha_i)
        else:
            # Last point contribution is typically negligible or handled by background
            pass
            
    return rendered_color
```

**Edge Cases and Pitfalls:**
*   **SDF Gradient Noise:** Computing normals via finite differences on noisy SDF outputs can lead to jittery specular highlights. Smoothing or analytical gradients (if the MLP structure allows) are recommended [G].
*   **Density Interpretation:** The parameter $\alpha$ controls the "sharpness" of the surface. If $\alpha$ is too small, the volume appears foggy; if too large, it may become too opaque too quickly, losing volumetric details [S1].
*   **LUT Resolution:** The accuracy of the specular term depends on the resolution of the BRDF LUT. Low resolution can cause banding artifacts in reflections [S1].
*   **Roughness Input:** The source mentions roughness $\rho$ as an input to the prefiltering and LUT lookup but does not specify if it is a global constant, per-pixel input, or learned. In practice, it is often a learnable parameter per object or global [S1].

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the MLP evaluations for each sample point along the ray. The PBR calculation adds overhead for normal computation and LUT lookups compared to standard NeRF's view-dependent radiance MLP.
*   **Memory Complexity:** Requires storing the Spatial MLP, Directional MLP (if used for prefiltering), and the BRDF LUT. The LUT is small (typically $256 \times 64$ or similar).
*   **Performance:** Achieves better reconstruction quality than prior methods, particularly for specular reflections and geometric structures, overcoming the "foggy artifacts" of standard NeRF [S1].
*   **Strengths:** Interpretable parameters (SDF for geometry, PBR for materials), smoother normals, accurate speculars.
*   **Limitations:** Complexity of PBR integration; potential for artifacts if SDF is not well-regularized; requires careful tuning of $\alpha$ and $\beta$.

## Background
This method was proposed by Chenyang Lv et al. in the paper "Volume rendering based on PBR and neural implicit surfaces" [S1]. It addresses the limitations of NeRF, such as coarse detail and foggy artifacts in specular regions, by integrating SDF representation with PBR material models. It contrasts with Ref-NeRF (which uses reflected radiance interpolation) and NeRFReN (which uses an additional MLP for speculars) by providing interpretable parameters and smoother normals through the SDF-PBR combination [S1].

## Sources
[S1] Volume rendering based on PBR and neural implicit surfaces, https://doi.org/10.1145/3650400.3650464, fetched 2026-10-27
