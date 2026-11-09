# BRDF Decoder Strategies

Also called: Physical and Neural BRDF Decoding

## Overview
BRDF Decoder Strategies define how 3D Gaussian Splatting primitives store and interpret surface reflectance properties to enable relighting and physically-based rendering. While standard 3D Gaussian Splatting [3D Gaussian Splatting](52-3d-gaussian-splatting.md) encodes color as a view-dependent spherical harmonic expansion, BRDF decoders map geometric attributes (position, covariance) and material parameters (albedo, roughness, metalness) to a reflectance function. This approach solves the problem of view-independent material representation, allowing for dynamic illumination changes. It differs from [3D Gaussian Splatting with BRDF Embedding](1174-3d-gaussian-splatting-with-brdf-embedding.md) by explicitly regressing physical PBR parameters rather than learning implicit embeddings, and from [3D Gaussian Splatting with Reflection and Relighting](5873-3d-gaussian-splatting-with-reflection-and-relighting.md) by focusing on the specific architectural choices (MLP vs. analytical models) for decoding these parameters.

## How it works
The method involves regressing physical or neural BRDF parameters per Gaussian primitive and evaluating them during the deferred rendering pass.

**Inputs:**
*   Gaussian primitive attributes: mean position $\mu$, covariance $\Sigma$, opacity $\alpha$, and base color/albedo $\mathbf{c}_{base}$.
*   Incident light direction $\omega_i$ and view direction $\omega_o$.
*   Optional: Scene lighting environment map.

**Outputs:**
*   Reflected radiance $L_o$ for each pixel.

**Data Structures:**
*   **Physical Attributes:** Stored as scalar or vector fields per Gaussian.
    *   Albedo $\mathbf{c}_{base}$: RGB vector, clamped $[0, 1]$.
    *   Roughness $\alpha_{rough}$: Scalar, scaled/shifted to $[0.09, 1]$ to prevent extreme specular highlights [S2, unofficial].
    *   Metalness: Scalar, clamped $[0, 1]$ [S2, unofficial].
    *   Indirect Illumination: Spherical Harmonic (SH) coefficients [S2, unofficial].
*   **Neural Attributes:** If physical attributes are unavailable, MLP weights or LSTM states encode the BRDF as a light reflection weight [S1, unofficial].

**Steps:**
1.  **Attribute Regression:**
    *   *Physical Path:* Use multiple Multi-Layer Perceptrons (MLPs) to regress albedo, roughness, and metalness from the Gaussian's spatial coordinates or latent features [S1, unofficial].
    *   *Neural Path:* Use MLPs or LSTMs to analyze the energy distribution of incident light and reflection directly, representing the BRDF as a reflection weight [S1, unofficial].
2.  **BRDF Evaluation:**
    *   *Simplified Disney Model:* Calculate the Normal Distribution Function (NDF), Fresnel term, and Geometry term using Neilf's formula [S1, unofficial].
    *   *Neural Model:* Evaluate the neural network with inputs $\omega_i, \omega_o$ to output the reflection weight [S1, unofficial].
3.  **Lighting Integration:**
    *   Sample incident light directions $\omega_i$. For training, use Fibonacci sampling for random incoming light [S2, unofficial]. For relighting, use Multiple Importance Sampling (MIS) combining light importance rays and GGX importance rays [S2, unofficial].
    *   Compute reflected radiance $L_o = f_r(\omega_i, \omega_o) L_i(\omega_i) \cos \theta_i$.
4.  **Denoising (Deferred Pipeline):**
    *   Apply a non-parametric bilateral denoiser to the PBR result to handle low samples per point (spp) [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_brdf_decoder(gaussian, omega_o, omega_i, lighting_env):
    """
    Evaluates the BRDF for a single Gaussian primitive.
    gaussian: object containing albedo, roughness, metalness, or neural weights.
    omega_o: view direction.
    omega_i: incident light direction.
    lighting_env: environment map or light source data.
    """
    # 1. Retrieve or Regress Physical Parameters
    if gaussian.has_physical_params:
        albedo = clamp(gaussian.albedo, 0, 1)
        roughness = scale_shift(gaussian.roughness, min=0.09, max=1.0)
        metalness = clamp(gaussian.metalness, 0, 1)
        
        # 2. Calculate BRDF Terms (Simplified Disney/Neilf)
        # NDF, Fresnel, Geometry terms calculated via Neilf's formula
        ndf = calculate_ndf(roughness, dot_product(normal, omega_i))
        fresnel = calculate_fresnel(metalness, albedo, dot_product(normal, omega_i))
        geometry = calculate_geometry(roughness, dot_product(normal, omega_i), dot_product(normal, omega_o))
        
        brdf_value = (ndf * fresnel * geometry) / (4.0 * dot_product(normal, omega_i) * dot_product(normal, omega_o))
    else:
        # Neural Decoder Path
        # MLP/LSTM analyzes energy distribution of incident light and reflection
        brdf_value = neural_network_forward(gaussian.neural_weights, omega_i, omega_o)
        
    # 3. Apply Brightening if using Neural/Reflection Weight directly
    # Mitigates highlight loss by boosting high-intensity pixels
    if not gaussian.has_physical_params:
        pixel_gray = luminance(brdf_value)
        if pixel_gray > threshold:
            brdf_value *= brightness_coefficient
            
    # 4. Integrate with Lighting
    incident_radiance = sample_lighting(lighting_env, omega_i)
    reflected_radiance = brdf_value * incident_radiance * dot_product(normal, omega_i)
    
    return reflected_radiance
```

**Parameters:**
*   **Roughness Bounds:** $[0.09, 1]$ [S2, unofficial].
*   **Brightness Coefficient:** Multiplier for pixels exceeding a gray-value threshold [S1, unofficial].
*   **Samples Per Pixel (spp):** 64 spp used with denoising for real-time rendering; 1,024 spp for comparison with voxel grid methods [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Highlight Loss:** Direct training of reflection weights without physical constraints leads to loss of highlight information [S1, unofficial]. Mitigated by brightening operations [S1, unofficial].
*   **Degenerate Geometry:** Imperfect surface meshes and large gradients in HDR space during PBR training can cause dramatic position shifts of vertices, impairing visibility [S2, unofficial].
*   **Overfitting:** Methods may overfit to the lighting conditions of the training dataset [S2, unofficial].

## Subtopics

### Deferred Rendering Pipeline for Phys3DGS
A specific implementation of BRDF decoding used to train Physically-Based Rendering (PBR) parameters in Phys3DGS. It distinguishes itself by employing a deferred rendering pipeline that separates the geometry pass from the material/lighting pass. It uses multiple importance sampling (MIS) for light rays during training and applies a non-parametric bilateral denoiser to the PBR result. This allows for real-time rendering with only 64 samples per pixel (spp), compared to 1,024 spp required by voxel grid-based methods [S2, unofficial]. It is chosen when real-time relighting quality is prioritized over raw sample accuracy without denoising.

## Operating characteristics
*   **Time Complexity:** Training involves iterative regression of MLPs or SH coefficients. Stage 2 (PBR training) takes approximately 10k steps, taking up to 30 minutes on an NVIDIA RTX 4090 [S2, unofficial].
*   **Memory Complexity:** Requires storing additional attributes per Gaussian (albedo, roughness, metalness, SH coefficients) compared to standard 3DGS.
*   **Performance:** The deferred pipeline with denoising achieves real-time rendering speeds (64 spp) with quality comparable to high-sample (1,024 spp) voxel methods [S2, unofficial].
*   **Strengths:** Enables physically plausible relighting; separates material from lighting; efficient real-time rendering via denoising.
*   **Limitations:** Sensitive to mesh quality; prone to highlight loss if physical constraints are not used; potential overfitting to training lighting [S1, unofficial][S2, unofficial].
*   **When to use:** When dynamic lighting and physically-based material interaction are required.
*   **When not to use:** When static lighting is sufficient (use standard 3DGS) or when hardware lacks support for efficient denoising.

## Background
BRDF decoding in 3D Gaussian Splatting emerged from the need to extend the static, view-dependent color representation of 3DGS [3D Gaussian Splatting](52-3d-gaussian-splatting.md) to support inverse rendering and relighting. Early approaches used neural networks to directly predict reflection weights, which suffered from highlight loss [S1, unofficial]. Subsequent work introduced physically-based models (Disney BRDF) and deferred rendering pipelines to stabilize training and enable real-time performance [S2, unofficial]. Key developments include the use of Neilf's formula for BRDF terms [S1, unofficial] and the integration of bilateral denoisers to reduce sample counts [S2, unofficial].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
