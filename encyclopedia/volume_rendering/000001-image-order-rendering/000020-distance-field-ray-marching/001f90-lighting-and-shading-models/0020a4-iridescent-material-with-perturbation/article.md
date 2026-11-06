# Iridescent Material with Perturbation

Also called: Iridescent Material (Cosine Palette), Iridescent Material with Specular and Shadow

## Overview
This method renders iridescent surfaces by modulating a cosine-based palette function with a perturbed surface normal, simulating thin-film interference colors, while combining this base color with multi-component specular highlights and directional shadowing [S1, unofficial]. It differs from standard diffuse or simple specular models by generating color variation through geometric perturbation rather than wavelength-dependent physics, and it differs from physically accurate thin-film models (like Airy summation) by using a computationally cheaper, artistic approximation driven by position and normal noise [S1, S2, unofficial].

## How it works
**Inputs:**
*   Surface point position $\mathbf{p}$ (world space).
*   Surface normal $\mathbf{n}$ (unit vector, derived from SDF gradient).
*   Ray direction $\mathbf{v}_{ray}$ (from camera to $\mathbf{p}$, or eye direction $\mathbf{v} = \mathbf{p} - \mathbf{rayOrigin}$ normalized).
*   Light direction $\mathbf{l}$ (from $\mathbf{p}$ to light source, normalized).
*   Gamma value $\gamma$ (default 2.2).
*   Optional blend factor $\alpha \in [0,1]$ for mixing with Blinn-Phong (default 0).
*   Optional tint color $\mathbf{c}_{tint}$ (default white).

**Outputs:**
*   Final linear RGB color $\mathbf{c}_{final}$.

**Data Structures:**
*   Vectors are 3-component floating-point vectors (`vec3`).
*   Scalars are floating-point numbers.

**Parameters:**
*   `GAMMA`: Gamma correction exponent. Typical value: 2.2 [S1, unofficial].
*   `perturb_scale`: Normal perturbation magnitude. Value: 0.05 [S1, unofficial].
*   `perturb_freq`: Frequency of position-based perturbation. Value: 10.0 [S1, unofficial].
*   `specular_mod_freq`: Frequency of specular modulation sine wave. Value: 20.0 [S1, unofficial].
*   `specular_mod_phase`: Phase offset for specular modulation. Value: -3.0 [S1, unofficial].
*   `specular_mod_offset`: Offset added to specular modulation. Value: 0.1 [S1, unofficial].
*   `specular_mod_power`: Power applied to modulated specular. Value: 32.0 [S1, unofficial].
*   `specular_primary_scale`: Scale factor for primary specular term. Value: 0.1 [S1, unofficial].
*   `specular_secondary_offset`: Offset for secondary specular term. Value: 0.3 [S1, unofficial].
*   `specular_secondary_power`: Power applied to secondary specular term. Value: 8.0 [S1, unofficial].
*   `specular_secondary_scale`: Scale factor for secondary specular term. Value: 0.1 [S1, unofficial].
*   `shadow_dome`: Direction vector for shadow calculation. Value: $(0, 1, 0)$ [S1, unofficial].
*   `shadow_scale`: Scale factor for normal-dome dot product in shadow. Value: 0.5 [S1, unofficial].
*   `shadow_offset`: Offset added in shadow calculation. Value: 1.2 [S1, unofficial].
*   `shadow_power`: Power applied to shadow term. Value: 3.0 [S1, unofficial].

**Steps:**

1.  **Calculate Reflection Vector:**
    Compute the reflection of the view direction off the surface normal:
    $$ \mathbf{r} = \text{reflect}(-\mathbf{v}, \mathbf{n}) $$
    where $\mathbf{v}$ is the normalized eye direction (from surface to camera). Note: In the source code, `reflect(rayDirection, nor)` is used where `rayDirection` points from camera to surface, implying the standard GLSL `reflect(I, N)` where $I$ is incident vector. We define $\mathbf{v}$ as the vector from surface to camera for clarity in dot products with light. If $\mathbf{v}_{eye}$ is from surface to camera, the incident vector is $-\mathbf{v}_{eye}$.
    $$ \mathbf{r} = 2(\mathbf{n} \cdot (-\mathbf{v}_{eye}))\mathbf{n} - (-\mathbf{v}_{eye}) = 2(\mathbf{n} \cdot \mathbf{v}_{eye})\mathbf{n} + \mathbf{v}_{eye} $$
    *Correction based on source:* The source uses `reflect(rayDirection, nor)` where `rayDirection` is from camera to surface. Let $\mathbf{d}$ be the ray direction (camera to surface). Then $\mathbf{r} = \text{reflect}(\mathbf{d}, \mathbf{n})$.

2.  **Calculate Perturbed Normal for Iridescence:**
    Compute a perturbation vector based on the surface position:
    $$ \mathbf{p}_{ert} = \sin(\mathbf{p} \cdot 10.0) $$
    Add this perturbation to the geometric normal, scaled by 0.05:
    $$ \mathbf{n}_{pert} = \mathbf{n} + \mathbf{p}_{ert} \cdot 0.05 $$

3.  **Calculate Base Iridescent Color:**
    Compute the dot product between the perturbed normal and the eye direction (normalized):
    $$ t = \mathbf{n}_{pert} \cdot \mathbf{v}_{eye} $$
    Pass this scalar through the spectrum palette function:
    $$ \mathbf{c}_{base} = \text{spectrum}(t \cdot 2.0) $$
    The `spectrum` function uses the cosine palette:
    $$ \text{pal}(t, \mathbf{a}, \mathbf{b}, \mathbf{c}, \mathbf{d}) = \mathbf{a} + \mathbf{b} \cdot \cos(2\pi (\mathbf{c} \cdot t + \mathbf{d})) $$
    With parameters:
    $$ \mathbf{a} = (0.5, 0.5, 0.5), \quad \mathbf{b} = (0.5, 0.5, 0.5) $$
    $$ \mathbf{c} = (1.0, 1.0, 1.0), \quad \mathbf{d} = (0.0, 0.33, 0.67) $$

4.  **Calculate Specular Highlights:**
    Compute the primary specular term:
    $$ s_1 = \text{clamp}(\mathbf{r} \cdot \mathbf{l}, 0.0, 1.0) $$
    Modulate with a sine wave:
    $$ s_1 = \left( \sin(s_1 \cdot 20.0 - 3.0) \cdot 0.5 + 0.5 + 0.1 \right)^{32.0} \cdot s_1 $$
    Scale by 0.1:
    $$ s_1 = s_1 \cdot 0.1 $$
    
    Compute the secondary specular term:
    $$ s_2 = \text{clamp}(\mathbf{r} \cdot \mathbf{l}, 0.0, 1.0) $$
    $$ s_2 = (s_2 + 0.3)^{8.0} \cdot 0.1 $$
    
    Combine specular terms:
    $$ \mathbf{c}_{spec} = s_1 + s_2 $$

5.  **Calculate Shadow:**
    Compute the dot product between the geometric normal and the dome direction $\mathbf{d}_{ome} = (0, 1, 0)$:
    $$ sh = \text{clamp}(\mathbf{n} \cdot \mathbf{d}_{ome} \cdot 0.5 + 1.2, 0.0, 1.0) $$
    Apply power:
    $$ \mathbf{c}_{shadow} = sh^{3.0} $$

6.  **Combine Colors:**
    $$ \mathbf{c}_{iridescent} = \mathbf{c}_{base} \cdot \mathbf{c}_{shadow} + \mathbf{c}_{spec} $$

7.  **Optional: Mix with Blinn-Phong:**
    If blending is enabled ($\alpha > 0$):
    Calculate Blinn-Phong specular:
    $$ h = \text{normalize}(\mathbf{v}_{eye} + \mathbf{l}) $$
    $$ power = (\mathbf{n} \cdot \mathbf{h})^{0.5} \quad \text{(Note: Source uses exponent 0.5 in blinnPhongSpec call, likely representing roughness or a simplified model)} $$
    *Correction:* The source code snippet says `blinnPhongSpec(lightDirection, eyeDirection, nor, 0.5)`. Standard Blinn-Phong specular is $(\mathbf{n} \cdot \mathbf{h})^{\text{shininess}}$. The parameter 0.5 is unusual for shininess (usually >1). It might be a roughness parameter or a simplified power. Assuming the function returns a scalar intensity.
    $$ \mathbf{c}_{phong} = power \cdot \mathbf{c}_{tint} $$
    Blend:
    $$ \mathbf{c}_{final\_linear} = \text{mix}(\mathbf{c}_{phong}, \mathbf{c}_{iridescent}, \alpha) $$
    If $\alpha = 0$, $\mathbf{c}_{final\_linear} = \mathbf{c}_{iridescent}$.

8.  **Gamma Correction:**
    Convert linear color to screen space:
    $$ \mathbf{c}_{final} = \mathbf{c}_{final\_linear}^{\frac{1}{\gamma}} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def pal(t: np.ndarray, a: np.ndarray, b: np.ndarray, c: np.ndarray, d: np.ndarray) -> np.ndarray:
    """Cosine palette function."""
    # t is scalar or array, a,b,c,d are vec3
    return a + b * np.cos(2 * math.pi * (c * t + d))

def spectrum(n: float) -> np.ndarray:
    """Generates iridescent color from scalar input."""
    a = np.array([0.5, 0.5, 0.5])
    b = np.array([0.5, 0.5, 0.5])
    c = np.array([1.0, 1.0, 1.0])
    d = np.array([0.0, 0.33, 0.67])
    return pal(n, a, b, c, d)

def iridescent_material(p: np.ndarray, n: np.ndarray, v_eye: np.ndarray, l: np.ndarray, 
                        gamma: float = 2.2, mix_alpha: float = 0.0, tint: np.ndarray = None) -> np.ndarray:
    """
    Computes the iridescent material color.
    
    Args:
        p: Surface position (world space).
        n: Surface normal (unit vector).
        v_eye: Normalized eye direction (from surface to camera).
        l: Normalized light direction (from surface to light).
        gamma: Gamma value.
        mix_alpha: Blend factor between Blinn-Phong and Iridescence [0,1].
        tint: Tint color for Blinn-Phong component.
        
    Returns:
        Final screen-space RGB color.
    """
    if tint is None:
        tint = np.array([1.0, 1.0, 1.0])
        
    # 1. Reflection vector
    # reflect(I, N) where I is incident vector (camera to surface = -v_eye)
    # r = 2 * dot(n, -v_eye) * n - (-v_eye)
    # r = 2 * dot(n, v_eye) * n + v_eye  <-- Wait, standard reflect(I,N) = I - 2*dot(I,N)*N
    # If I = -v_eye, r = -v_eye - 2*dot(-v_eye, n)*n = -v_eye + 2*dot(v_eye, n)*n
    # Let's stick to the source logic: reflect(rayDirection, nor)
    # rayDirection is from camera to surface. Let d = -v_eye.
    # r = d - 2 * dot(d, n) * n
    d = -v_eye
    r = d - 2 * np.dot(d, n) * n
    
    # 2. Perturbed Normal
    perturb = np.sin(p * 10.0)
    n_pert = n + perturb * 0.05
    
    # 3. Base Iridescent Color
    t_val = np.dot(n_pert, v_eye) * 2.0
    c_base = spectrum(t_val)
    
    # 4. Specular
    spec_dot = np.clip(np.dot(r, l), 0.0, 1.0)
    
    # Primary specular
    s1 = (np.sin(spec_dot * 20.0 - 3.0) * 0.5 + 0.5 + 0.1) ** 32.0 * spec_dot
    s1 *= 0.1
    
    # Secondary specular
    s2 = (spec_dot + 0.3) ** 8.0 * 0.1
    
    c_spec = s1 + s2
    
    # 5. Shadow
    dome = np.array([0.0, 1.0, 0.0])
    sh = np.clip(np.dot(n, dome) * 0.5 + 1.2, 0.0, 1.0)
    c_shadow = sh ** 3.0
    
    # 6. Combine
    c_iridescent = c_base * c_shadow + c_spec
    
    # 7. Mix with Blinn-Phong
    if mix_alpha > 0.0:
        # Blinn-Phong Specular
        h = np.linalg.norm(v_eye + l)
        if h > 1e-6:
            h = (v_eye + l) / h
        else:
            h = v_eye # Fallback
            
        # Source uses exponent 0.5 in blinnPhongSpec call. 
        # Assuming this is the power for the specular highlight.
        power = np.dot(n, h) ** 0.5 
        
        c_phong = power * tint
        c_final_linear = np.where(mix_alpha > 0, 
                                  c_phong * (1.0 - mix_alpha) + c_iridescent * mix_alpha, 
                                  c_iridescent)
    else:
        c_final_linear = c_iridescent
        
    # 8. Gamma Correction
    c_final = c_final_linear ** (1.0 / gamma)
    
    return c_final
```

## Subtopics

### Thin-Film Interference via Airy Summation
A physically based technique for rendering iridescence in thin films (e.g., soap bubbles, oil films) using Airy summation for s- and p-polarization components over multiple wavelengths [S2, unofficial]. It samples 9 wavelengths to compute the interference pattern and can be applied over a dielectric base or a metal base (where the complex Index of Refraction is derived from F0 and edge tint using Gulbrandsen's inversion) [S2, unofficial]. A key pitfall is handling thin films at zero thickness to avoid numerical instability [S2, unofficial].

### Mixing Phong and Iridescence
A technique to blend Blinn-Phong lighting with the iridescent lighting model to create tinted iridescent objects [S1, unofficial]. It calculates the Blinn-Phong specular intensity, multiplies it by a tint color to get a base color, and then blends this with the iridescent color (base + shadow + specular) using a mix factor `mixBaseAndIridescent` in the range [0,1] [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per surface point. The computation involves a fixed number of trigonometric functions, dot products, and power operations.
*   **Memory Complexity:** $O(1)$. No additional data structures are required beyond standard shading inputs.
*   **Performance:** Very fast, suitable for real-time ray marching. The cosine palette avoids expensive spectral calculations.
*   **Strengths:** Visually striking iridescent effect with minimal computational cost; easy to tune via palette parameters; supports blending with standard lighting models.
*   **Limitations:** Not physically accurate; color variation is driven by geometry perturbation rather than film thickness or wavelength; may exhibit banding if the palette function is not smooth enough or if precision is low.
*   **When to use:** Real-time applications, artistic rendering, crystal-like materials, soap bubbles, oil slicks where physical accuracy is not required.
*   **When not to use:** Physically based rendering requiring accurate thin-film interference, scientific visualization of optical phenomena.

## Background
This method originates from the ray marching and SDF community, popularized by shaders on platforms like ShaderToy [S1, unofficial]. It was notably used in Thomas Hooper's "Crystals" shader and adapted by Varun for an article on ray marching SDFs [S1, unofficial]. The cosine palette technique for iridescence is a common artistic shortcut in procedural shading. The variant mixing Blinn-Phong lighting was introduced to allow for tinted iridescent objects [S1, unofficial]. Physically accurate thin-film interference via Airy summation is a more rigorous approach found in advanced rendering engines like the Photon Engine [S2, unofficial].

## Sources
[S1] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
