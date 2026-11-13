# BRDF Decoder Strategies

Also called: Physical and Neural BRDF Decoding

## Overview
BRDF Decoder Strategies define how 3D Gaussian Splatting primitives map their stored latent attributes to physically based or neural reflectance values during rendering. While the parent method handles geometry and visibility, these strategies determine the material appearance by either regressing physical parameters (albedo, roughness, metalness) for evaluation via analytical models or directly predicting reflection weights via neural networks. This approach enables relighting and inverse rendering capabilities within the Gaussian splatting framework, distinguishing it from standard appearance-only splatting by decoupling geometry from material properties.

## How it works
The method operates in two distinct modes depending on the availability of physical ground truth or desired interpretability: Physical Attribute Regression and Neural Reflection Weighting.

### Inputs and Outputs
*   **Inputs:**
    *   Gaussian primitive attributes: Position $\mathbf{x}$, Covariance $\Sigma$, Opacity $\alpha$, and learned latent features $\mathbf{z}$ (or physical parameters if pre-defined).
    *   View direction $\mathbf{v}$ and Light direction $\mathbf{l}$ (for relighting).
    *   Surface normal $\mathbf{n}$ (derived from covariance or explicit normal attributes).
*   **Outputs:**
    *   Reflected radiance $L_r$ or BRDF value $f_r(\mathbf{l}, \mathbf{v})$.
    *   Physical parameters: Albedo $\rho$, Roughness $\alpha_r$, Metalness $m$.

### Data Structures
The Gaussian primitive stores attributes in a contiguous buffer. For physical decoding, the structure includes:
*   `base_color`: 3 floats (RGB).
*   `roughness`: 1 float.
*   `metallic`: 1 float.
*   `sh_coefficients`: $N$ floats for indirect illumination (Spherical Harmonics).
*   `latent_vector`: Variable size (e.g., 32 features) for neural decoding [S3, unofficial].

### Step 1: Attribute Regression or Latent Encoding
**Mode A: Physical Attributes (Disney Model)**
For datasets with known physical attributes, multiple Multi-Layer Perceptrons (MLPs) regress physical properties from the Gaussian's latent features or position [S1, unofficial].
1.  Input latent features $\mathbf{z}$ into MLPs.
2.  Output physical parameters: Albedo $\rho \in [0,1]^3$, Roughness $\alpha_r \in [0.09, 1]$, Metalness $m \in [0,1]$ [S2, unofficial].
3.  Clamp values to valid ranges. Roughness is typically shifted to $[0.09, 1]$ to prevent singular specularities [S2, unofficial].

**Mode B: Neural Reflection Weighting**
For datasets lacking physical attributes, MLPs or LSTMs directly analyze the energy distribution of incident light and reflection [S1, unofficial].
1.  Input view $\mathbf{v}$, light $\mathbf{l}$, and normal $\mathbf{n}$ into the network.
2.  Output a reflection weight $W$ representing the BRDF value.
3.  This acts as a degenerate form of light reflection weight without physical constraints [S1, unofficial].

### Step 2: BRDF Evaluation
**Physical Mode:**
Calculate the BRDF using the simplified Disney principle-based model [S1, unofficial].
1.  **Normal Distribution Function (NDF):** Use the formula from NeILF [S1, unofficial].
    $$ D(\mathbf{h}, \alpha_r) = \frac{\alpha_r^2}{\pi ((\mathbf{n} \cdot \mathbf{h})^2 (\alpha_r^2 - 1) + 1)^2} $$
    where $\mathbf{h}$ is the half-vector.
2.  **Fresnel Term:** Calculate using Schlick's approximation or similar, dependent on metalness $m$ and base color $\rho$.
3.  **Geometry Term:** Calculate using the NeILF geometry function $G(\mathbf{l}, \mathbf{v}, \mathbf{n}, \alpha_r)$ [S1, unofficial].
4.  Combine terms to get $f_r$.

**Neural Mode:**
The network output $W$ is used directly as the reflectance scaling factor.
$$ L_r = W \cdot E_i $$
where $E_i$ is the incident irradiance [S1, unofficial].

### Step 3: Relighting and Sampling
For inverse rendering and relighting, light sampling is critical [S2, unofficial].
1.  **Training:** Use Fibonacci sampling for random incoming light directions to ensure uniform coverage on the hemisphere [S2, unofficial].
2.  **Relighting (Inference):** Use Multiple Importance Sampling (MIS) as implemented in NVDiffrecMC [S2, unofficial].
    *   Cast equal amounts of "light importance" rays and "GGX importance" rays.
    *   Combine samples using MIS weights to reduce variance.

### Step 4: Denoising (Deferred Pipeline)
To enable real-time rendering with low sample counts, a non-parametric bilateral denoiser is applied to the PBR result [S2, unofficial].
1.  Render with low Samples Per Pixel (spp), e.g., 64 spp [S2, unofficial].
2.  Apply bilateral filtering using spatial proximity and photometric similarity (based on albedo/normal buffers) to suppress noise while preserving edges.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_brdf(gaussian, view_dir, light_dir, mode='physical'):
    """
    Decodes BRDF for a single Gaussian primitive.
    
    Args:
        gaussian: Object containing attributes (base_color, roughness, metallic, latent)
        view_dir: Normalized view vector v
        light_dir: Normalized light vector l
        mode: 'physical' or 'neural'
        
    Returns:
        reflected_radiance: Float or Vector3
    """
    n = gaussian.normal
    v = view_dir
    l = light_dir
    h = normalize(v + l)
    
    if mode == 'physical':
        # 1. Retrieve physical parameters
        rho = clamp(gaussian.base_color, 0.0, 1.0)
        alpha_r = clamp(gaussian.roughness, 0.09, 1.0) # Prevent extreme speculars [S2, unofficial]
        m = clamp(gaussian.metallic, 0.0, 1.0)
        
        # 2. Calculate NeILF terms [S1, unofficial]
        # NDF
        nh = dot(n, h)
        denom = (nh * nh) * (alpha_r * alpha_r - 1.0) + 1.0
        D = (alpha_r * alpha_r) / (pi * denom * denom)
        
        # Fresnel (Schlick approximation for dielectrics, blended for metals)
        F0 = mix(vec3(0.04), rho, m)
        F = F0 + (1.0 - F0) * pow(1.0 - dot(v, h), 5.0)
        
        # Geometry (NeILF simplified)
        # Note: Exact NeILF geometry formula is complex; simplified approximation often used
        # G = G_Schlick(v, n, alpha_r) * G_Schlick(l, n, alpha_r)
        G = geometry_term(v, l, n, alpha_r) 
        
        # 3. Combine
        numerator = D * F * G
        denominator = 4.0 * max(dot(v, n), 1e-5) * max(dot(l, n), 1e-5)
        brdf = numerator / max(denominator, 1e-5)
        
        return brdf * rho # Diffuse component included in Disney model usually
        
    elif mode == 'neural':
        # 1. Neural Network Forward Pass [S1, unofficial]
        # Input: v, l, n, latent_vector
        reflection_weight = neural_net(v, l, n, gaussian.latent)
        
        # 2. Mitigate highlight loss [S1, unofficial]
        # If training, apply brightening to target images. 
        # During inference, weight is used directly.
        return reflection_weight

def render_with_mis(gaussians, env_map, spp=64):
    """
    Renders using Multiple Importance Sampling for relighting [S2, unofficial].
    """
    color = vec3(0.0)
    
    # Split samples between Light Importance and GGX Importance
    n_light = spp // 2
    n_ggx = spp - n_light
    
    for i in range(n_light):
        l = fibonacci_sample_hemisphere()
        # Cast ray, find intersection/weight
        weight = evaluate_gaussian_weight(gaussians, l)
        Li = sample_env_map(env_map, l)
        color += weight * Li / n_light
        
    for i in range(n_ggx):
        # Sample based on GGX distribution (roughness-dependent)
        l = ggx_sample_hemisphere(gaussians)
        weight = evaluate_gaussian_weight(gaussians, l)
        Li = sample_env_map(env_map, l)
        color += weight * Li / n_ggx
        
    return color
```

### Parameters and Edge Cases
*   **Roughness Clamping:** Must be clamped to $[0.09, 1]$ to prevent numerical instability and excessively strong specular highlights [S2, unofficial].
*   **Highlight Loss:** Direct training of reflection weights without physical constraints leads to loss of highlight information [S1, unofficial]. Mitigation: Apply a brightening operation to training images by multiplying pixels with gray values surpassing a threshold by a brightness coefficient [S1, unofficial].
*   **Mesh Degeneracy:** In deferred pipelines, imperfect surface meshes can cause dramatic position shifts during PBR training in HDR space, impairing visibility [S2, unofficial].

## Subtopics

### Deferred Rendering Pipeline for Phys3DGS
A rendering pipeline specifically designed to train Physically Based Rendering (PBR) parameters (albedo, metalness, roughness, SH coefficients) within 3DGS [S2, unofficial]. It employs multiple importance sampling (MIS) for light rays and post-processing denoising to enable real-time rendering. It uses Fibonacci sampling for training light directions and NVDiffrecMC-style MIS (equal light and GGX rays) for relighting [S2, unofficial]. It applies a non-parametric bilateral denoiser to handle low sample counts (64 spp vs 1,024 spp for voxel methods) [S2, unofficial].

### BRDF-SMAE
A Smooth Material Auto-Encoder (SMAE) that encodes a single BRDF per point in a neural volume to enable spatially varying BRDF decomposition [S3, unofficial]. It uses an encoder, decoder, and discriminator, each consisting of 3 MLP layers with 32 output features [S3, unofficial]. It enforces smoothness in the latent space using loss weighting parameters $\lambda_1 = 0.01$, $\lambda_2 = 0.01$, $\lambda_3 = 0.001$ [S3, unofficial]. It uses 64 interpolation steps in the latent space and Mean Absolute Error (MAE) for reconstruction [S3, unofficial]. Training runs for 1.5 million steps with batch size 256 on an NVIDIA 1080 TI GPU using Adam optimizer (lr $1e-4$) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Real-time rendering is achievable with the deferred pipeline and denoiser, requiring only 64 samples per pixel (spp) compared to 1,024 spp for voxel-based methods [S2, unofficial]. Training takes approximately 30 minutes for PBR parameters (10k steps) on an RTX 4090 [S2, unofficial]. Neural decoding adds inference overhead per Gaussian but avoids complex analytical evaluations.
*   **Memory Complexity:** Increases with the storage of physical attributes (albedo, roughness, metalness, SH coefficients) or larger latent vectors for neural decoding [S3, unofficial]. BRDF-SMAE requires storing a latent vector per point [S3, unofficial].
*   **Strengths:** Enables physically plausible relighting and material decomposition. The deferred pipeline allows real-time performance with low sample counts via denoising [S2, unofficial]. Physical decoding provides interpretable material parameters.
*   **Limitations:** Direct neural decoding may lose highlight information without specific training mitigations (brightening) [S1, unofficial]. Physical decoding requires accurate normal estimation and can suffer from mesh degeneracy issues during optimization [S2, unofficial].

## Background
BRDF decoding in 3D Gaussian Splatting emerged from the need to extend the fast rendering capabilities of 3DGS to inverse rendering and relighting tasks. Early methods focused on novel view synthesis with fixed lighting. The integration of PBR parameters (albedo, roughness, metalness) allowed for scene decomposition [S2, unofficial]. The use of simplified Disney models and NeILF formulas provided a bridge between differentiable rendering and physical optics [S1, unofficial]. The development of deferred pipelines and neural auto-encoders (SMAE) further refined the ability to handle spatially varying materials and real-time relighting [S2, S3, unofficial].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
[S3] Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition, https://doi.org/10.48550/arxiv.2110.14373, fetched 2026-10-26 (unofficial)
