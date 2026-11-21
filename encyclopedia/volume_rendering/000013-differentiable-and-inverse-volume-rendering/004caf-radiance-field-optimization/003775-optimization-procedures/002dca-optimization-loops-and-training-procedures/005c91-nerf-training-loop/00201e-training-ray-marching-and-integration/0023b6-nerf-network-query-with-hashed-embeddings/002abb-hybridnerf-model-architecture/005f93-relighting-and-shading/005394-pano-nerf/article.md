# Pano-NeRF

Also called: irradiance fields

## Overview
Pano-NeRF is a method for geometry recovery and High Dynamic Range (HDR) novel view synthesis from sparse Low Dynamic Range (LDR) panoramic images [S1]. It models the scene using irradiance fields, which treat radiance as both a signal conveying lighting information and a light source illuminating other pixels, thereby increasing observation counts for geometry and leveraging irradiance-radiance attenuation for HDR reconstruction [S1]. It differs from standard NeRF by integrating surface rendering principles into the volumetric framework to explicitly model inter-reflection, and it differs from sibling methods like [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md) by deriving shading through volumetric irradiance integration rather than explicit light source estimation or normal correlation [S1, 6346-shading-replacement-for-relighting.md].

## How it works
**Inputs:** Sparse LDR panoramic images; initial scene geometry or implicit representation parameters.
**Outputs:** HDR radiance field, geometry (density $\sigma$), and spatially varying lighting estimates.

**Data Structures:**
The method extends the standard NeRF representation.
1.  **Radiance Field:** Standard MLP or grid-based representation outputting density $\sigma(x)$ and outgoing radiance $c_r(x, \mathbf{d})$ for a viewing direction $\mathbf{d}$.
2.  **Irradiance Field:** Implicitly defined by the integration of incoming radiance over the sphere. No separate storage is required; it is computed on-the-fly during the shading step.

**Numbered Steps:**

1.  **Scene Representation:** Model the scene as a combination of volumetric particles (for standard NeRF rendering) and surface pieces with reflection effects [S1]. The scene is represented by an irradiance field integrated into the existing radiance field framework [S1].

2.  **Incoming Radiance Calculation ($c_i$):** For a point $x$ on a surface, consider incoming light directions $\omega_i$ distributed on a sphere $\Omega$ centered at $x$ [S1]. For each direction $\omega_i$, calculate the incoming radiance $c_i(x, \omega_i)$ [S1]. This value is equal to the outgoing radiance $C_r$ of the incident ray traveling in direction $-\omega_i$, computed via standard volume rendering along that ray [S1].
    *   Standard volume rendering formula for $C_r$:
        $$ C_r = \int_{t_n}^{t_f} T(t) \sigma(t) c_r(t, \mathbf{d}) dt $$
        where $T(t) = \exp(-\int_{t_n}^t \sigma(s) ds)$ is the transmittance [S1, implied standard NeRF].

3.  **Outgoing Radiance Calculation ($C_i$):** Compute the outgoing radiance $C_i(x)$ at position $x$ through surface rendering by integrating irradiance from incoming light directions $\omega_i$ [S1]. The formula is:
    $$ C_i(x) = \int_{\omega_i \in \Omega} f_r(x, \omega_o, \omega_i) \cdot \max(\omega_i \cdot \mathbf{n}, 0) \cdot c_i(x, \omega_i) \, d\omega_i $$
    where:
    *   $f_r(x, \omega_o, \omega_i)$ is the Bidirectional Reflection Distribution Function (BRDF) [S1].
    *   $\mathbf{n}$ is the surface normal at point $x$ [S1].
    *   $\omega_o$ is the outgoing viewing direction [S1].
    *   $\max(\omega_i \cdot \mathbf{n}, 0)$ handles back-face culling, replacing the dot product $\omega_i \cdot \mathbf{n}$ when it is negative [S1].

4.  **Joint Optimization:** Integrate the irradiance fields into the existing radiance fields framework and perform joint optimization to recover geometry and HDR radiance [S1]. Although $c_i$ is not directly supervised, its integral is constrained through the training error between the observed ground-truth radiance $C_{gt}$ and the computed outgoing radiance $C_i$ [S1]. This facilitates the optimization of density $\sigma$ and radiance $c_r$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_outgoing_radiance(x, normal, view_direction, brdf, nerf_model):
    """
    Computes outgoing radiance C_i(x) using irradiance field integration.
    
    Args:
        x: 3D position vector.
        normal: Surface normal vector n at x.
        view_direction: Outgoing viewing direction omega_o.
        brdf: Function f_r(x, omega_o, omega_i).
        nerf_model: The NeRF model capable of volume rendering.
        
    Returns:
        C_i: Scalar outgoing radiance.
    """
    # Sample incoming directions omega_i on the unit sphere Omega
    # In practice, this is approximated via Monte Carlo integration
    num_samples = 1024 # Typical value for Monte Carlo estimation
    omega_i_samples = sample_sphere_uniform(num_samples)
    
    total_irradiance = 0.0
    
    for omega_i in omega_i_samples:
        # 1. Calculate incoming radiance c_i(x, omega_i)
        # This is the radiance Cr of the incident ray traveling in direction -omega_i
        # Computed via standard volume rendering along that ray
        ray_origin = x
        ray_direction = -omega_i
        
        # Perform volume rendering to get Cr
        # Standard NeRF volume rendering:
        # T(t) = exp(-integral(sigma))
        # C_r = integral(T(t) * sigma(t) * c_r(t, direction))
        c_i = nerf_model.volume_render(ray_origin, ray_direction)
        
        # 2. Calculate BRDF term
        # f_r(x, omega_o, omega_i)
        fr = brdf(x, view_direction, omega_i)
        
        # 3. Calculate cosine term with back-face culling
        # max(omega_i . n, 0)
        dot_product = dot(omega_i, normal)
        cosine_term = max(dot_product, 0.0)
        
        # 4. Accumulate contribution
        # Integrand: f_r * max(omega_i . n, 0) * c_i
        contribution = fr * cosine_term * c_i
        
        # Monte Carlo integration: sum / N * 4*pi (if uniform sampling on sphere)
        # Or simply average if normalized weights are used
        total_irradiance += contribution
        
    # Average over samples
    C_i = total_irradiance / num_samples
    
    return C_i

def training_step(pano_images, nerf_model, brdf, geometry_params):
    """
    Joint optimization step.
    """
    # For each pixel in panoramic image:
    # 1. Cast ray to get x and normal n.
    # 2. Compute C_i(x) using compute_outgoing_radiance.
    # 3. Compare C_i with ground truth C_gt from LDR image.
    # 4. Compute loss = ||C_i - C_gt||^2.
    # 5. Backpropagate to update nerf_model (sigma, c_r) and geometry_params.
    pass
```

**Parameters:**
*   **Number of Monte Carlo samples:** Not given in the sources.
*   **BRDF:** Not specified; assumed to be a standard model (e.g., Lambertian or Phong) [S1].
*   **Sphere Sampling:** Uniform distribution on $\Omega$ [S1].

**Edge Cases and Pitfalls:**
*   **Back-facing surfaces:** The dot product $\omega_i \cdot \mathbf{n}$ can be negative for back-facing surfaces. This is handled by replacing $\omega_i \cdot \mathbf{n}$ with $\max(\omega_i \cdot \mathbf{n}, 0)$ in the implementation [S1].
*   **Sparse Inputs:** Sparse inputs bring a small number of observation counts, which might cause unconstrained geometry [S1]. The irradiance fields mitigate this by increasing observation counts through inter-reflection between irradiance and outgoing radiance [S1].
*   **LDR to HDR:** The method leverages irradiance-radiance attenuation to reconstruct HDR radiance from LDR inputs [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The method requires additional volume rendering passes for each incoming direction $\omega_i$ during the shading step. If $N$ samples are used for the sphere integration, the shading cost is $O(N \cdot R)$, where $R$ is the number of ray marching steps. This is significantly more expensive than standard NeRF which is $O(R)$ per pixel.
*   **Memory:** Memory usage is similar to standard NeRF, as the irradiance field is not stored explicitly but computed on-the-fly [S1].

**Measured Performance:**
*   The method outperforms state-of-the-art methods on both geometry recovery and HDR reconstruction [S1].
*   It achieves faithful geometry recovery even with sparse inputs [S1].

**Strengths and Limitations:**
*   **Strengths:** Increases observation counts for geometry recovery; leverages irradiance-radiance attenuation for HDR reconstruction; provides spatially varying lighting estimation as a byproduct [S1].
*   **Limitations:** Computationally expensive due to the integration over the sphere and multiple volume rendering passes; relies on sparse LDR inputs which may limit the quality of the initial geometry [S1].

**When to Use:**
*   Use when reconstructing HDR novel views and geometry from sparse LDR panoramic images.
*   Use when explicit light source estimation is not desired, but spatially varying lighting is needed [S1].

**When Not to Use:**
*   Use standard NeRF if inputs are dense and HDR is not required.
*   Use [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md) if the task is object insertion and shading correlation is sufficient.

## Background
Pano-NeRF was proposed by Lu Zhan et al. in 2023 [S1]. It addresses the problem of geometry recovery and HDR novel view synthesis from sparse LDR panoramic images [S1]. The method builds on the concept of irradiance fields, observing that radiance from each pixel in panoramic images can be modeled as both a signal to convey scene lighting information and a light source to illuminate other pixels [S1]. This approach increases observation counts for faithful geometry recovery and leverages irradiance-radiance attenuation for HDR reconstruction [S1]. The code is available at https://github.com/Lu-Zhan/Pano-NeRF [S1].

## Sources
[S1] Pano-NeRF: Synthesizing High Dynamic Range Novel Views with Geometry from Sparse Low Dynamic Range Panoramic Images, https://doi.org/10.48550/arXiv.2312.15942, fetched 2026-10-29
