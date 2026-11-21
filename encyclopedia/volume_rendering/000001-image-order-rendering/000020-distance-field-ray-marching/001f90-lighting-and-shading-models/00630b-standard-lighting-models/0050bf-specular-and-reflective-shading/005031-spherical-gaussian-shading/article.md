# Spherical Gaussian Shading

Also called: SG Shading, SG-Based Rendering, Shading Rendering of Spherical Gaussians, Specular BRDF Approximation

## Overview
Spherical Gaussian (SG) Shading is a technique that approximates direct illumination and specular BRDFs using a mixture of Spherical Gaussian basis functions to enable fast, analytic integration and automatic differentiation [S3, unofficial]. It distinguishes itself from Monte Carlo (MC) integration by being less expensive and more robust to roughness variations, though it sacrifices high-frequency detail accuracy [S3, unofficial]. It is primarily used in differentiable rendering and inverse rendering pipelines where lighting and material parameters must be optimized jointly [S2][S4].

## How it works
The method operates on a surface point $\mathbf{p}$ with normal $\mathbf{n}$, view direction $\mathbf{v}$, and incident light direction $\mathbf{l}$ [G]. It approximates the rendering equation by representing both the incident radiance and the BRDF lobes as sums of Spherical Gaussians [S2].

### Data Structures
1. **Spherical Gaussian (SG) Kernel**: Defined by four parameters: $\omega \in S^2$ (input direction), $\xi \in S^2$ (axis/center), $\lambda \in \mathbb{R}^+$ (sharpness), and $\mu \in \mathbb{R}^{3+}$ (amplitude/HDR RGB color) [S3, unofficial].
2. **Lighting Representation**: A mixture of $K$ SGs representing the environment map or spatially variant lights, with parameters $\gamma = \{ \xi_{lk}, \lambda_{lk}, \mu_{lk} \}_{k=1}^K$ [S3, unofficial]. Typical $K$ is 128 for fitting HDR environment maps [S3, unofficial].
3. **Volumetric SG (VSG) Layout**: Stored as an 8-channel tensor $\mathcal{L} \in \mathbb{R}^{8 \times X \times Y \times Z}$ [S5]. Each voxel contains opacity $\alpha \in [0, 1]$, HDR RGB intensity $c \in \mathbb{R}^3$, lobe axis $\mu \in \mathbb{R}^3$, and sharpness $\sigma \in \mathbb{R}^+$ [S5].

### Formulas
The Spherical Gaussian function is defined as [S3, unofficial]:
$$ G(\omega; \xi, \lambda, \mu) = \mu e^{\lambda(\xi \cdot \omega - 1)} $$

Cosine foreshortening is approximated by a single SG [S3, unofficial]:
$$ |\mathbf{n} \cdot \omega_i| \approx G_c(\omega_i; \mathbf{n}, 2.133, 1.17) $$

The specular lobe of the BRDF is fitted to a single monochromatic SG, denoted $f_r^{(SG)}$, forming a sum of diffuse and specular lobes [S3, unofficial].

The final pixel color $S^{(SG)}$ is computed by integrating the product of the BRDF, incident radiance, and cosine term [S3, unofficial]:
$$ S^{(SG)}(\mathbf{x}, \mathbf{n}, \omega_o; \theta, \gamma) = \int_{S^2} f_r^{(SG)}(\mathbf{x}, \omega_i, \omega_o; \theta) L_i^{(SG)}(\omega_i; \gamma) G_c(\omega_i) d\omega_i $$
where $L_i^{(SG)}$ is the incident radiance represented by $K$ SGs [S3, unofficial].

Alternatively, each SG can be rendered as 5 point lights (1 center + 4 sampled around) [S4]. For $k$ point lights with intensity $\mathbf{I}_i$, the shading is [S4]:
$$ \mathbf{S}_{SG}(\mathbf{r}) = \sum_{i=1}^k \frac{\mathbf{N}(\mathbf{r}) \cdot \mathbf{L}_i(\mathbf{r})}{|\mathbf{N}(\mathbf{r})| |\mathbf{L}_i(\mathbf{r})|} \cdot \frac{\mathbf{I}_i}{\gamma \cdot \mathbf{Dis}_i(\mathbf{r})^2} $$
where $\mathbf{L}_i(\mathbf{r})$ is the direction from light to point, $\mathbf{Dis}_i(\mathbf{r})$ is the distance, and $\gamma$ is a hyperparameter controlling luminance [S4].

For VSG, incident radiance $R(\mathbf{p}, \mathbf{l}, \mathcal{L})$ at point $\mathbf{p}$ in direction $\mathbf{l}$ is calculated via alpha compositing along $N$ equi-spaced points [S5]:
$$ R(\mathbf{p}, \mathbf{l}, \mathcal{L}) = \left( \prod_{i=1}^{k-1} (1 - \alpha_i) \right) \sum_{k=1}^{N} \alpha_k G(-\mathbf{l}; c_k, \mu_k, \sigma_k) $$

The total image for ray $\mathbf{r}$ combines diffuse, specular, and SG shading [S4]:
$$ I(\mathbf{r}) = I_d(\mathbf{r}) + I_s(\mathbf{r}) + V(\mathbf{r})\mathcal{R}(\mathbf{r})S_{SG}(\mathbf{r}) $$
where $V$ is shadow visibility and $\mathcal{R}$ is reflectance [S4].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_gaussian_shading(surface_point, normal, view_dir, lighting_params, method="point_light"):
    """
    Compute shading using Spherical Gaussians.
    Args:
        surface_point: 3D position p.
        normal: Surface normal n.
        view_dir: View direction v.
        lighting_params: Dictionary containing 'K', 'SGs' (list of dicts with 'xi', 'lambda', 'mu'),
                         or 'VSG_volume' for volumetric method.
        method: 'analytic' for direct integration, 'point_light' for 5-light approximation.
    Returns:
        Color C.
    """
    if method == "point_light":
        total_shading = np.zeros(3)
        gamma = lighting_params.get('gamma', 1.0)
        for sg in lighting_params['SGs']:
            lights = generate_point_lights_from_sg(sg['xi'])
            for light_dir, light_intensity in lights:
                l_dir = normalize(light_dir)
                dist = 1.0
                cosine = max(dot(normal, l_dir), 0.0)
                attenuation = light_intensity / (gamma * dist**2)
                total_shading += cosine * attenuation
        return total_shading
    elif method == "analytic":
        cosine_sg = {'xi': normal, 'lambda': 2.133, 'mu': 1.17}
        specular_sg = fit_specular_sg(normal, view_dir, roughness=0.1, metalness=0.5)
        color = np.zeros(3)
        for sg in lighting_params['SGs']:
            # The analytic integral of the product of SGs on a sphere is not provided in the sources.
            # Standard closed-form integration applies [G].
            color += integrate_sg_product(cosine_sg, specular_sg, sg)
        return color

def spherical_gaussian(direction, axis, sharpness, amplitude):
    """Evaluate a single Spherical Gaussian: G(w; xi, lambda, mu) = mu * exp(lambda * (xi . w - 1)) [S3, unofficial]."""
    return amplitude * exp(sharpness * (dot(axis, direction) - 1.0))

def generate_point_lights_from_sg(center_axis):
    """Generate 5 light directions around the SG center axis [S4]."""
    lights = [(center_axis, 1.0)]
    for i in range(4):
        offset = orthogonal_vector(center_axis)
        lights.append((normalize(center_axis + 0.5 * offset), 0.5))
    return lights

def fit_specular_sg(normal, view_dir, roughness, metalness):
    """Fit a single monochromatic SG to the specular lobe of the BRDF [S3, unofficial]."""
    half_vec = normalize(normal + view_dir)
    sharpness = 2.0 / (roughness**2 + 1e-6)
    return {'xi': half_vec, 'lambda': sharpness, 'mu': 1.0}

def integrate_sg_product(sg1, sg2, sg3):
    # The analytic integral of the product of SGs on a sphere is not provided in the sources.
    # Standard closed-form integration applies [G].
    return np.zeros(3)
```

### Parameters
* **K**: Number of SGs. Typical value is 128 for fitting HDR environment maps [S3, unofficial].
* **$\gamma$**: Hyperparameter for luminance control in point-light approximation [S4].
* **$\beta$**: Surface roughness. SG shading is robust to variations in $\beta$ [S3, unofficial].
* **$\sigma$**: VSG sharpness. If $\sigma \gg 1$, VSG reduces to RGB$\alpha$ [S5].

### Edge Cases and Pitfalls
* **High Frequency Loss**: SGs smooth out high-frequency details and sharp corners [S3, unofficial]. They are not precise enough for real-time scenarios requiring high fidelity without a very large number of coefficients [S1, unofficial].
* **HDR Lighting**: If the environment is HDR, SGs should be placed far from the scene to assume distant lighting [S4].
* **VSG Degeneracy**: If $\sigma \gg 1$ in VSG, the representation loses directional lighting capability and becomes a standard RGB$\alpha$ volume [S5].
* **Coefficient Count**: Accurate reconstruction may require significantly more coefficients than dense pixel maps (e.g., 896 SGs vs 98,304 pixels for an HDR map) [S3, unofficial].

## Subtopics

### Final NeRF Relighting Equation
A specific application of SG shading within Neural Radiance Fields (NeRF) for relighting. It combines diffuse ($I_d$), specular ($I_s$), and SG shading ($S_{SG}$) modulated by shadow visibility ($V$) and reflectance ($\mathcal{R}$) [S4]. It allows for material editing and scene relighting by optimizing SG parameters [S4].

### Volumetric Spherical Gaussian (VSG)
A voxel-based representation for 3D spatially-varying lighting where each voxel acts as a light source parameterized by SG lobes [S5]. It captures view-dependent effects and directional lighting in 3D space, unlike standard RGB$\alpha$ volumes. It is calculated via alpha compositing along rays [S5].

### PhySG
Assumes full light source visibility, ignoring self-occlusions and assuming all light sources in the upper hemisphere above any surface are fully visible [S6, unofficial]. It accelerates rendering by representing environment lighting and scene BRDFs as mixtures of spherical Gaussians, enabling the hemispherical integral of incoming light multiplied by the BRDF to be computed in closed form [S6, unofficial]. Assuming full light source visibility is unable to simulate effects such as cast shadows due to occlusion of light sources by scene geometry, though it works well for mostly convex objects [S6, unofficial].

## Operating characteristics
* **Time Complexity**: Faster than Monte Carlo (MC) integration because it uses analytic integration of SG products [S3, unofficial]. The cost scales linearly with the number of SGs $K$ [S3, unofficial].
* **Memory Complexity**: Compact. Requires significantly fewer parameters than dense environment maps (e.g., 896 parameters for K=128 SGs vs 98,304 for a dense HDR map) [S3, unofficial]. VSG adds spatial storage overhead ($8 \times X \times Y \times Z$) [S5].
* **Performance**: Robust to roughness variations [S3, unofficial]. MC is better for mirror-like objects (low roughness) where high frequencies are preserved, but SG is more efficient for moderate-to-high roughness [S3, unofficial].
* **Limitations**: Loses high-frequency details [S3, unofficial]. Not suitable for real-time rendering if high precision is required without large $K$ [S1, unofficial].
* **When to Use**: Differentiable rendering, inverse rendering, optimization of lighting/materials, and scenes with moderate roughness [S3, unofficial].
* **When Not to Use**: Real-time rendering requiring high-fidelity specular highlights with sharp features, or scenes with very low roughness (mirrors) where MC is superior [S1, unofficial][S3, unofficial].

## Background
Spherical Gaussian shading was introduced as a compact approximation for direct illumination in differentiable rendering frameworks, specifically within the DIB-R++ pipeline [S3, unofficial]. It builds on the mathematical properties of Spherical Gaussians to allow analytic integration of the rendering equation, facilitating gradient-based optimization of lighting and materials [S3, unofficial]. It has been applied in various contexts including intrinsic decomposition, relighting in NeRFs [S4], and 3D inverse rendering with spatially varying lighting [S5].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Modeling Indirect Illumination for Inverse Rendering, https://doi.org/10.1109/cvpr52688.2022.01809, fetched 2026-10-26
[S3] DIB-R++: Learning to Predict Lighting and Material with a Hybrid Differentiable Renderer, https://doi.org/10.48550/arxiv.2111.00140, fetched 2026-10-26 (unofficial)
[S4] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
[S5] Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting, https://doi.org/10.1109/iccv48922.2021.01231, fetched 2026-10-26
[S6] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
