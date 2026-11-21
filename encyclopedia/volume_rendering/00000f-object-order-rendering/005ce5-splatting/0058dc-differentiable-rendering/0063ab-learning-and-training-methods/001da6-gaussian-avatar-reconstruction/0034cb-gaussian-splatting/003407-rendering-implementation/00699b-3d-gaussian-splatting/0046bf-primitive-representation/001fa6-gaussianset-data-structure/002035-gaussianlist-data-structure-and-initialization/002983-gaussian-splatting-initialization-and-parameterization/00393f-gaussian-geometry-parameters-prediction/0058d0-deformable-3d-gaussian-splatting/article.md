# Deformable 3D Gaussian Splatting

Also called: Canonical Space and Deformation Field, Deformable-GS, Deformable3DGS (Yang et al.), Dynamic 3D Gaussian Splatting (Canonical Space + Deformation Field), Implicit Deformation Modeling, Yang et al

## Overview
Deformable 3D Gaussian Splatting is a method for dynamic scene reconstruction that learns 3D Gaussians in a canonical static space and models spatial-temporal deformation using an implicit deformation field implemented as a Multi-Layer Perceptron (MLP) [S1, S2]. Unlike static 3D Gaussian Splatting, which assumes a fixed geometry, this method makes Gaussian parameters time-dependent, enabling the reconstruction and real-time rendering of 4D scenes [S3][S4, unofficial]. It distinguishes itself from sibling methods like [GSGEN](8985-gsgen.md) or [Fourier Splatting Primitive](11094-fourier-splatting-primitive.md) by explicitly predicting temporal offsets for geometric attributes (position, rotation, scale) via an MLP conditioned on time, rather than relying on explicit per-frame storage or texture-based extensions [S2][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of canonical 3D Gaussians $G$ (learned from static initialization), a set of training images with known camera poses and timestamps $t$, and a deformation field network $F$ [S1, S2].
*   **Outputs:** Deformed Gaussians $G_t$ at any timestep $t$, and rendered images $\hat{I}$ that minimize the difference from ground truth images $I$ [S3].

**Data Structures:**
The method extends the [GaussianList Data Structure and Initialization](10060-gaussian-list-data-structure-and-initialization.md) by adding a deformation field $F$ [S2].
*   **Canonical Gaussian $G$:** Defined by learnable attributes per Gaussian $i$:
    *   $\mu_i$: Spatial mean (3D vector) [S3].
    *   $r_i$: Rotation quaternion (4D vector) [S3].
    *   $s_i$: Scaling vector (3D vector) [S3].
    *   $c_i$: View-dependent color (Spherical Harmonic coefficients) [S3].
    *   $o_i$: Opacity (scalar) [S3].
*   **Deformation Field $F$:** An MLP that takes positional-encoded Gaussian position $\mu$ and timestep $t$ as input, and outputs offsets $\Delta G_t = (\Delta \mu, \Delta r, \Delta s, \Delta o)$ [S2].

**Numbered Steps:**

1.  **Canonical Initialization:** Initialize a static set of 3D Gaussians $G$ using standard 3DGS initialization techniques (e.g., from SfM points) [S2][S4, unofficial]. Optimize these Gaussians on static frames to establish a canonical representation [S2].
2.  **Deformation Field Construction:** Define the deformation $\Delta G_t$ for a canonical Gaussian $G$ at time $t$ as:
    $$ \Delta G_t = G_t - G $$
    where $G_t$ is the deformed Gaussian at timestep $t$ [S3]. The deformation captures changes in mean ($\Delta \mu$), rotation ($\Delta r$), scaling ($\Delta s$), and opacity ($\Delta o$) [S3]. Color $c$ remains unchanged to ensure consistent tracking [S3].
3.  **Temporal Smoothing (Noise Injection):** To improve temporal smoothing without additional computational overhead, a linearly decaying Gaussian noise is added to the encoded time vector input to the MLP [S2]. This mitigates artifacts caused by inaccurate poses, which are amplified in discrete 3DGS compared to continuous NeRF, especially during time interpolation [S2].
4.  **Deformation Prediction:** For a query time $t$ and canonical Gaussian $i$:
    *   Encode $\mu_i$ and $t$ (with noise) [S2].
    *   Pass through MLP $F$ to predict offsets $\Delta \mu_i, \Delta r_i, \Delta s_i, \Delta o_i$ [S2].
    *   Compute deformed attributes:
        $$ \mu_{i,t} = \mu_i + \Delta \mu_i $$
        $$ r_{i,t} = \text{Normalize}(r_i + \Delta r_i) $$
        $$ s_{i,t} = \text{Exp}(s_i + \Delta s_i) $$
        $$ o_{i,t} = o_i + \Delta o_i $$
    *   Note: Some variants apply exponential and normalization operations to scaling and rotation respectively after adding delta values to ensure valid ranges [S2].
5.  **Rendering:**
    *   Project the deformed 3D covariance matrix $\Sigma$ (derived from $r_{i,t}$ and $s_{i,t}$) to 2D using the viewing transformation matrix $W$ and the Jacobian $J$ of the affine approximation of the projective transformation:
        $$ \Sigma' = J W \Sigma W^T J^T $$
        [S3]
    *   Compute opacity $\alpha_i$ as a weighted product of $\Sigma'$ and intrinsic opacity $o_{i,t}$ [S3].
    *   Composite colors using alpha blending in screen space order:
        $$ C = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $c_i$ is the view-dependent color and $N$ is the set of splats [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(x, min_deg, max_deg):
    """Standard positional encoding for MLP input."""
    scales = np.array([2**i for i in range(min_deg, max_deg)])
    xb = np.reshape((x[..., None, :] * scales[:, None]), (x.shape[0], -1))
    return np.concatenate([np.sin(xb), np.cos(xb)], axis=-1)

def project_and_composite(gaussians, W, J):
    """Projects Gaussians to 2D and composites them per [S3]."""
    # Compute 2D covariance and opacity for each Gaussian
    projected = []
    for g in gaussians:
        # Construct 3D covariance from quaternion r and scale s [G]
        R = quaternion_to_rotation_matrix(g['r'])
        S = np.diag(g['s'])
        Sigma = R @ S @ S.T @ R.T
        # Project to 2D [S3]
        Sigma_prime = J @ W @ Sigma @ W.T @ J.T
        # Compute opacity alpha from Sigma' and intrinsic o [S3]
        alpha = compute_gaussian_alpha(Sigma_prime, g['o'])
        projected.append({'mu_2d': g['mu'][:2], 'alpha': alpha, 'c': g['c']})
    
    # Sort by depth (z-coordinate) for correct alpha blending [G]
    projected.sort(key=lambda x: x['mu_2d'][2])
    
    # Composite colors [S3]
    img_h, img_w = 1080, 1920  # Placeholder resolution [G]
    image = np.zeros((img_h, img_w, 3))
    for p in projected:
        # Rasterize Gaussian to screen pixels [G]
        screen_pixels = rasterize_gaussian(p['mu_2d'], p['alpha'], p['c'])
        for (u, v), color in screen_pixels.items():
            image[v, u] += color * p['alpha']
    return image

def deformable_gaussian_splatting(canonical_gaussians, time, deformation_mlp, camera_params):
    """
    canonical_gaussians: List of dicts with keys 'mu', 'r', 's', 'c', 'o'
    time: float, current timestep
    deformation_mlp: Function taking (pos_enc_mu, pos_enc_time) -> (d_mu, d_r, d_s, d_o)
    camera_params: Dict with 'W' (view matrix), 'J' (Jacobian)
    """
    deformed_gaussians = []
    
    for g in canonical_gaussians:
        mu = g['mu']
        r = g['r']
        s = g['s']
        c = g['c']
        o = g['o']
        
        # 1. Encode inputs
        # Add linearly decaying Gaussian noise to time encoding for smoothing [S2]
        decay = 1.0 / (1.0 + time)  # Linear decay approximation [G]
        time_noise = np.random.normal(0, decay)
        enc_mu = positional_encoding(mu, 0, 4)
        enc_time = positional_encoding(np.array([time + time_noise]), 0, 4)
        
        # 2. Predict offsets via MLP
        d_mu, d_r, d_s, d_o = deformation_mlp(enc_mu, enc_time)
        
        # 3. Apply offsets
        mu_t = mu + d_mu
        r_t = r + d_r
        s_t = s + d_s
        o_t = o + d_o
        
        # 4. Ensure valid ranges [G]
        r_t = r_t / np.linalg.norm(r_t)
        s_t = np.exp(s_t)
        o_t = np.clip(o_t, 0.0, 1.0)
        
        deformed_gaussians.append({
            'mu': mu_t,
            'r': r_t,
            's': s_t,
            'c': c, # Color remains unchanged [S3]
            'o': o_t
        })
        
    # 5. Render using deformed Gaussians
    rendered_image = project_and_composite(deformed_gaussians, camera_params['W'], camera_params['J'])
    
    return rendered_image, deformed_gaussians
```

**Parameters:**
*   $\lambda_1, \lambda_2$: Weights for Total Variation (TV) and Dissimilarity (DSSIM) losses. Typical values are not explicitly given in the primary source for Yang et al., but VolSegGS uses them to balance regularization [S3].
*   MLP Depth/Width: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Inaccurate Poses:** Inaccurate camera poses affect rendering quality significantly more in discrete 3DGS than in continuous NeRF, especially during time interpolation [S2]. The noise injection in the time vector is a specific mitigation for this [S2].
*   **Temporal Coherence:** Without regularization, the MLP may produce jittery deformations [G]. Total Variation (TV) loss is used to smooth learned features [S3].
*   **Opacity Drift:** Opacity deformation $\Delta o$ captures appearance and disappearance of scene regions over time [S3].

## Subtopics

### [Shape-Shifting Splats](10708-shape-shifting-splats.md)
A dynamic reconstruction method that predicts all 3D Gaussian parameters via shallow, fully-connected MLPs conditioned on a control parameter, allowing all attributes to be changed at render time. It builds on Deformable 3D Gaussian Splatting and [Gaussian Splatting Initialization and Parameterization](52-gaussian-splatting-initialization-and-parameterization.md) by replacing explicit storage of scale, rotation, offset, and opacity with learned functions, enabling context translation (e.g., RGB to thermal) and stylization without retraining the scene geometry.

### [4D-GS](10060-deformable-3d-gaussian-splatting.md)
(Note: The parent article itself is often referred to as 4D-GS in broader literature, but specific variants like the HexPlane-based 4D-GS exist. This section is empty as no distinct child algorithm details are provided in the source tree for a separate "4D-GS" child distinct from the main method's variants).

## Operating characteristics
*   **Time Complexity:** Rendering time is comparable to static 3DGS, as the deformation is computed per-Gaussian per-frame (or per-query time) before rasterization [G]. The method adds noise to the time vector without additional computational overhead [S2].
*   **Memory Complexity:** The deformation field reduces memory complexity from $O(N^3 T)$ to $O(R_s N^2 + R_s N + R_t T)$, where $N$ is spatial resolution and $T$ is temporal resolution, using low-rank decomposition or implicit MLP storage [S3].
*   **Measured Performance:** On the D-NeRF dataset, Yang et al. achieves PSNR 39.51, SSIM 0.990, and LPIPS 0.012 [S2].
*   **Strengths:** Real-time visualization and tracking of segments in time-varying data; compact representation compared to per-frame storage; handles unseen intermediate timesteps [S3].
*   **Limitations:** Sensitive to inaccurate camera poses; requires careful regularization (TV, DSSIM) to avoid artifacts; optimization is harder than static 3DGS due to the coupled deformation field [S2, S3].
*   **When to use:** Dynamic scene reconstruction where temporal coherence is critical and real-time rendering of novel views at arbitrary times is required [G].
*   **When not to use:** Static scenes (use standard 3DGS); scenes with extremely high-frequency temporal changes that an MLP cannot capture smoothly without overfitting [G].

## Background
Deformable 3D Gaussian Splatting was proposed by Yang et al. [19] (cited as Yang et al. [291] in [S1] and [19] in [S2]) as an extension of 3D Gaussian Splatting to dynamic scenes [S1, S2]. It follows the idea of modeling deformation fields implicitly using MLPs, similar to dynamic NeRF [66] [S2]. The method was introduced to enable 4D scene reconstruction and rendering, allowing Gaussians to adapt over time [S3][S4, unofficial]. It has been adopted in subsequent works such as VolSegGS for segmentation and tracking in dynamic volumetric scenes [S3].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] VolSegGS: Segmentation and Tracking in Dynamic Volumetric Scenes via Deformable 3D Gaussians, https://doi.org/10.1109/tvcg.2025.3642516, fetched 2026-10-26
[S4] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
