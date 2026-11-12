# Physically Based Inverse Rendering with Split-Sum Approximation

Also called: NVDIFFREC-based PBR, Sur2f PBR

## Overview
This method recovers surface materials (albedo, roughness, metallic) and environment lighting from multi-view images by integrating Sur2f with differentiable rendering and a split-sum approximation of the rendering equation [S1]. It distinguishes itself from sibling methods by decoupling geometry and appearance, allowing seamless transfer of different appearances to different geometries by exchanging the shading network, whereas other methods often couple them [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view RGB images, camera poses, and an initial geometry representation (from Sur2f).
*   **Outputs:** Learnable environment map $E$, base color $\mathbf{k}_d$, metallic value $k_m$, and surface roughness $k_r$.

**Data Structures:**
1.  **Environment Map ($E$):** A learnable texture representing incident illumination.
2.  **Environment BRDF Lookup Texture ($\mathbf{K}$):** A 2D texture storing the pre-calculated specular integral. It is indexed by surface roughness $k_r$ and the dot product $\langle \mathbf{n} \cdot -\boldsymbol{\omega} \rangle$ [S1].
3.  **Pre-Filtered Environment Map ($\mathbf{I}_s$):** A texture obtained by convolving $E$ with a multi-level GGX distribution [S1].
4.  **Diffuse Irradiance ($\mathbf{I}_d$):** A texture/vector representing the integration of the upper hemisphere of $E$ [S1].

**Parameters:**
*   Base color $\mathbf{k}_d \in [0, 1]^3$: View-independent diffuse term parameter [S1].
*   Metallic value $k_m \in [0, 1]$: View-independent diffuse term parameter [S1].
*   Surface roughness $k_r \in [0, 1]$: Conditions the microfacet distribution function ($D$), Fresnel reflection ($F$), and geometric shadowing factor ($G$) [S1].

**Formulas:**
The BRDF $B(\mathbf{l}, -\boldsymbol{\omega})$ is modeled as a sum of Lambertian diffuse and Cook-Torrance specular terms [S1]:
$$
B(\mathbf{l}, -\boldsymbol{\omega}) = (1 - k_m) \frac{\mathbf{k}_d}{\pi} + \frac{DFG}{4 \langle \mathbf{n} \cdot \mathbf{l} \rangle \langle \mathbf{n} \cdot -\boldsymbol{\omega} \rangle}
$$
where $\mathbf{n}$ is the surface normal, $\mathbf{l}$ is the light direction, and $\boldsymbol{\omega}$ is the view direction [S1].

The rendering equation is split into diffuse and specular components [S1]:
$$
L(\mathbf{x}_S, \boldsymbol{\omega}) = L_{\text{diffuse}}(\mathbf{x}_S) + L_{\text{specular}}(\mathbf{x}_S, \boldsymbol{\omega})
$$

1.  **Diffuse Component:** Approximated using the environment map $E$ [S1]:
    $$
    L_{\text{diffuse}}(\mathbf{x}_S) \approx (1 - k_m) \frac{\mathbf{k}_d}{\pi} \mathbf{I}_d
    $$
    where $\mathbf{I}_d$ is the diffuse irradiance obtained by integrating the upper hemisphere of $E$ [S1].

2.  **Specular Component:** Uses split-sum approximation to handle the intractable integral [S1]:
    $$
    L_{\text{specular}}(\mathbf{x}_S, \boldsymbol{\omega}) \approx \mathbf{K} \mathbf{I}_s
    $$
    where $\mathbf{K}$ is the Environment BRDF stored in a 2D look-up texture searchable by $k_r$ and $\langle \mathbf{n} \cdot -\boldsymbol{\omega} \rangle$, and $\mathbf{I}_s$ is the Pre-Filtered Environment Map obtained by convolving $E$ with multi-level GGX distribution [S1].

The final rendered color $\mathbf{C}_{\text{pbr}}(\mathbf{r})$ for ray $\mathbf{r}$ is obtained via linear-to-sRGB tone mapping [S1]:
$$
\mathbf{C}_{\text{pbr}}(\mathbf{r}) = \text{ToneMap}(L(\mathbf{x}_S, \boldsymbol{\omega}))
$$

**Optimization:**
The method minimizes the L1 loss between the predicted color $\mathbf{C}(\mathbf{r})$ and the PBR rendered color $\mathbf{C}_{\text{pbr}}(\mathbf{r})$ [S1]:
$$
\mathcal{L}_{\text{pbr}} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \mathbb{L}1 \left( \mathbf{C}(\mathbf{r}), \mathbf{C}_{\text{pbr}}(\mathbf{r}) \right)
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pbr_color(surface_point, normal, view_direction, roughness, metallic, base_color, env_map):
    """
    Computes the physically based rendered color for a surface point.
    
    Args:
        surface_point: 3D position x_S
        normal: Surface normal n
        view_direction: View direction omega
        roughness: Surface roughness k_r
        metallic: Metallic value k_m
        base_color: Base color k_d
        env_map: Environment map E
        
    Returns:
        Rendered color C_pbr
    """
    # 1. Compute Diffuse Irradiance (I_d)
    # Integrate the upper hemisphere of the environment map E
    # I_d = integral_{Omega} E(l) * max(0, n . l) dl
    # Note: In implementation, this is often pre-computed or approximated via spherical harmonics or convolution.
    # Here we assume a function `integrate_diffuse_irradiance` exists.
    I_d = integrate_diffuse_irradiance(env_map, normal)
    
    # 2. Compute Diffuse Term
    # L_diffuse = (1 - k_m) * (k_d / pi) * I_d
    diffuse_term = (1.0 - metallic) * (base_color / math.pi) * I_d
    
    # 3. Compute Specular Term using Split-Sum Approximation
    # a. Compute Pre-Filtered Environment Map (I_s)
    # Convolve env_map E with multi-level GGX distribution based on roughness and normal
    # I_s = integral_{Omega} E(l) * D(l, h, roughness) * max(0, n . l) dl
    # Note: D is the GGX distribution. This is typically done via mip-mapped convolution.
    I_s = prefilter_env_map(env_map, normal, roughness)
    
    # b. Lookup Environment BRDF (K)
    # K is stored in a 2D texture indexed by roughness and <n . -omega>
    # K = integral_{Omega} (DFG / (4 * <n . l> * <n . -omega>)) * <n . l> dl
    dot_nv = dot(normal, -view_direction)
    K = lookup_env_brdf(roughness, dot_nv)
    
    # c. Compute Specular Term
    # L_specular = K * I_s
    specular_term = K * I_s
    
    # 4. Combine and Tone Map
    # L = L_diffuse + L_specular
    L = diffuse_term + specular_term
    
    # Apply linear-to-sRGB tone mapping
    C_pbr = tone_map_srgb(L)
    
    return C_pbr

def train_step(images, poses, surface_points, normals, env_map, materials):
    """
    Single training step for inverse rendering.
    """
    # Integrate I_d and I_s from environment map E at the beginning of each training step
    # This ensures the pre-filtered maps are up-to-date with the current env_map
    I_d = integrate_diffuse_irradiance(env_map, normals)
    I_s = prefilter_env_map(env_map, normals, materials.roughness)
    
    # Render PBR images
    C_pbr = compute_pbr_color(surface_points, normals, view_directions, 
                              materials.roughness, materials.metallic, 
                              materials.base_color, env_map)
    
    # Compute Loss
    # L_pbr = Mean L1 loss between predicted color C(r) and PBR rendered color C_pbr(r)
    loss = mean_l1_loss(images, C_pbr)
    
    # Backpropagate to update env_map and materials
    loss.backward()
```

**Edge Cases and Pitfalls:**
*   **Intractable Integral:** The specular integral is intractable analytically; the split-sum approximation is required to make it computationally feasible [S1].
*   **Lookup Texture Indexing:** The Environment BRDF $\mathbf{K}$ must be correctly indexed by roughness $k_r$ and $\langle \mathbf{n} \cdot -\boldsymbol{\omega} \rangle$. Incorrect indexing leads to visual artifacts [S1].
*   **Tone Mapping:** The final color must undergo linear-to-sRGB tone mapping to match the input image space [S1].

## Subtopics

### [Sur2f](5573-sur2f.md)
A hybrid representation for high-quality and efficient surface reconstruction from multi-view images. It provides the geometry and appearance disentanglement foundation for this PBR method.

### [NVDIFFREC](5573-nvdiffrec.md)
A differentiable rendering framework used to integrate the PBR shading model into the optimization loop.

## Operating characteristics
*   **Time Complexity:** Depends on the cost of integrating the environment map ($\mathbf{I}_d$ and $\mathbf{I}_s$) at each training step. The lookup of $\mathbf{K}$ is $O(1)$ [S1].
*   **Memory Complexity:** Requires storage for the environment map $E$, the pre-filtered environment map $\mathbf{I}_s$, and the Environment BRDF lookup texture $\mathbf{K}$ [S1].
*   **Strengths:** Enables disentanglement of geometry and appearance, allowing seamless transfer of different appearances to different geometries [S1]. Suitable for editing and relighting tasks [S1].
*   **Limitations:** Relies on the accuracy of the split-sum approximation. The intractable specular integral is approximated, which may introduce errors in complex lighting scenarios [S1].
*   **When to use:** Physically based inverse rendering tasks, including editing and relighting, on NeRF synthetic datasets and DTU datasets [S1].

## Background
The method was introduced in the paper "Sur2f: A Hybrid Representation for High-Quality and Efficient Surface Reconstruction from Multi-view Images" [S1]. It builds on the Sur2f representation and integrates it with NVDIFFREC for differentiable rendering [S1]. The split-sum approximation technique is based on the work cited as [16] in the source, which pre-calculates the Environment BRDF and Pre-Filtered Environment Map [S1].

## Sources
[S1] Sur2f: A Hybrid Representation for High-Quality and Efficient Surface Reconstruction from Multi-view Images, https://doi.org/10.1007/978-3-031-73027-6_1, fetched 2026-10-26
