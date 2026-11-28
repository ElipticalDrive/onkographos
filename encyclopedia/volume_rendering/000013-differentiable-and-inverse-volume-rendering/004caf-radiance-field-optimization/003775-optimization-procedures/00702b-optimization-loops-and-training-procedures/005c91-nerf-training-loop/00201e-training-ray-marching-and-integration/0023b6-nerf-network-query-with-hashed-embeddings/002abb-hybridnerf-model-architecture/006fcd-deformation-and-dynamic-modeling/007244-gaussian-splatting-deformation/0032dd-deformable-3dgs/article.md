# Deformable 3DGS

Also called: 4D Gaussian Splatting / Deformable 3DGS, Deformable 3D-GS, Deformable 3DGS / Dynamic 3DGS, Dynamic 3D Gaussian Splatting (Deformation Field)

## Overview
Deformable 3DGS extends static 3D Gaussian Splatting to dynamic scenes by modeling time-dependent geometric and appearance parameters [S1, unofficial]. It solves the problem of reconstructing and rendering moving subjects (e.g., people, fluids) from multi-view video by decomposing the 4D representation into a static canonical set of Gaussians and a deformation function that predicts their state at any time $t$ [S2, unofficial]. It distinguishes itself from sibling methods like [Sparse-Controlled 3D Gaussian Splatting](6145-sparse-controlled-3d-gaussian-splatting.md) by using a direct MLP-based deformation network on per-Gaussian features rather than sparse control points [S2, unofficial]. Builds on [Gaussian Splatting Deformation](22469-gaussian-splatting-deformation.md).

## How it works
**Inputs:** Multi-view video frames with known camera poses and timestamps $t$ [S4, unofficial].
**Outputs:** A set of time-varying 3D Gaussians $\mathcal{G}(t)$ rendered into images [G].

**Data Structures:**
1.  **Canonical Gaussians $\mathcal{G}_{canon}$:** Each Gaussian $i$ stores static base parameters: mean $\boldsymbol{\mu}_i \in \mathbb{R}^3$, rotation $\mathbf{q}_i \in \mathbb{R}^4$ (unit quaternion), scale $\mathbf{s}_i \in \mathbb{R}^3$, opacity $\sigma_i$, and spherical harmonics coefficients $\mathbf{c}_{SH,i}$ [S4, unofficial].
2.  **Deformation Network $D$:** An MLP that takes spatial embeddings of canonical positions and temporal embeddings of time $t$ as input [S4, unofficial].

**Algorithm:**
1.  **Deformation Prediction:** Compute network output: $(\delta\boldsymbol{\mu}, \delta\mathbf{q}, \delta\mathbf{s}) = D(E_p(\boldsymbol{\mu}_g), E_t(t))$ [S4, unofficial].
2.  **Parameter Update:** Apply offsets to canonical parameters: $\boldsymbol{\mu}_g(t) = \boldsymbol{\mu}_g + \delta\boldsymbol{\mu}$, $\mathbf{s}_g(t) = \mathbf{s}_g + \delta\mathbf{s}$, $\mathbf{q}_g(t) = \mathbf{q}_g \cdot \delta\mathbf{q}$ [S4, unofficial].
3.  **Rendering:** Rasterize the updated Gaussians $\mathcal{G}(t)$ using standard differentiable rasterization with camera pose and time $t$ [G].
4.  **Optimization:** Align predicted temporal updates with ground-truth frames to jointly optimize both Gaussian parameters and deformation network parameters [S4, unofficial]. The method uses an Annealing Smooth Training strategy to smooth rendering jitters caused by inconsistent scenes across frames [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deformable_3dgs_forward(canonical_gaussians, time_t, deformation_network):
    """
    canonical_gaussians: list of dicts with keys 'mu', 'q', 's', 'sigma', 'c_SH'
    time_t: float timestamp
    deformation_network: callable D(Ep(mu), Et(t)) returning (delta_mu, delta_q, delta_s)
    """
    updated_gaussians = []
    for g in canonical_gaussians:
        mu_g, q_g, s_g = g['mu'], g['q'], g['s']
        # 1. Deformation Prediction
        delta_mu, delta_q, delta_s = deformation_network(mu_g, time_t)
        # 2. Parameter Update
        mu_t = mu_g + delta_mu
        s_t = s_g + delta_s
        q_t = normalize_quaternion(quat_mul(q_g, delta_q))
        # 3. Assemble time-varying Gaussian
        g_t = {
            'mu': mu_t, 'q': q_t, 's': s_t,
            'sigma': g['sigma'], 'c_SH': g['c_SH']
        }
        updated_gaussians.append(g_t)
    return updated_gaussians
```

**Parameters:**
*   Annealing Smooth Training strategy: Used to smooth rendering jitters caused by inconsistent scenes across frames [S3, unofficial].

**Edge Cases and Pitfalls:**
*   Quaternion normalization prevents drift in rotation updates [G].
*   Scale updates must avoid negative values; clipping or exponentiating deltas is required to maintain valid covariance decomposition $\Sigma = RSS^T R^T$ [G].
*   Precision issues may arise if $\Delta\mathbf{s}_i$ causes negative scales; absolute value or softplus activation on deltas is standard [G].

## Subtopics

### SplineGS
A method that replaces discrete per-frame deformation with a smooth spline modeling the trajectory of each primitive's center and rotation as a function of time, ensuring physically plausible, smooth motion with fewer parameters [S5, unofficial]. It uses B-spline basis functions $\mathbf{B}_k(t)$ and control points $\mathbf{p}_{i,k}$:
$$ \mu_i(t) = \sum_{k=0}^K \mathbf{B}_k(t) \mathbf{p}_{i,k} $$
[S5, unofficial].

### MLP Architecture for Local Deformation
A neural network that takes local body pose and latent codes to predict residual transformations (translation, rotation) and shading for Gaussians [S6]. Input is the concatenation of local body pose vector $\theta_t$ and per-Gaussian learnable latent features $l$ [S6]. Hidden layers: 2 layers, 64 neurons each, ReLU activations [S6]. Output heads: 3 separate heads, each with 2 layers, outputting: 1) Translation vector $t_{mlp}$, 2) Rotation quaternion $q_{mlp}$, 3) Scaling factor $s$ [S6]. Using latent codes $l$ instead of position leads to better performance [S6]. The final observed position and orientation are computed as:
$$ \mathbf {p}_{\text {obs}} = \mathbf {p}_{\text {lbs}} + (\mathbf {R}_{t}\mathbf {t}_{\text {mlp}}) $$
$$ \mathbf {q}_{\text {obs}} = \mathbf {q}_{\text {mlp}} \circ \mathbf {q}_{\text {lbs}} $$
[S6].

### Scale-Adaptive Gaussian Fitting for Dynamic 3DGS
Applying the scale-adaptive Gaussian fitting method as a plug-in to dynamic 3DGS models to solve zoom-out dilation without disturbing the deformation network [S7]. Scaffold-GS and Pixel-GS lack multi-scale deformation, compromising robustness for unseen settings [S7].

### E-D3DGS
Embeds between Gaussian band and MLP for deformation prediction [S3, unofficial]. Assigns each Gaussian a latent learnable embedding for unique characteristics and uses typical temporal embeddings for different frames [S3, unofficial]. MLP takes latent and temporal embeddings to predict changes in position, scale, and rotation [S3, unofficial]. Separately conducts deformation for coarse and fine motions: leverages a 1D feature grid for N frames and interpolates temporally to decode embedding for fine deformation; downsamples feature grid for coarse deformation [S3, unofficial]. Uses Local Smoothness Regularization to encourage nearby Gaussians to conduct similar deformation to capture textures and fine details [S3, unofficial].

### SP-GS
Groups 3D Gaussians into superpoints using a Property Reconstruction Loss [S3, unofficial]. The deformation network takes canonical position of superpoints and time as input to predict deformation, applied to Gaussians within superpoints [S3, unofficial].

## Operating characteristics
*   **Strengths:** Handles dynamic scenes (people moving, fluids) [S1, unofficial].
*   **When to use:** Dynamic scene reconstruction and rendering [S1, unofficial].

## Background
Introduced by Yang et al. in CVPR'24 [S2, unofficial]. It extends 3D-GS to canonical space and leverages a deformation MLP network to learn the offsets of the canonical 3D-GS to the final 3D shape [S3, unofficial]. It is part of the broader effort to extend 3D Gaussian Splatting to 4D (spatiotemporal) representations [S2, unofficial].

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
[S4] Reconstructing and Simulating Dynamic 3D Objects with Mesh-adsorbed Gaussian Splatting, https://doi.org/10.48550/arXiv.2406.01593, fetched 2026-10-30 (unofficial)
[S5] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S6] Human Gaussian Splatting: Real-Time Rendering of Animatable Avatars, https://doi.org/10.1109/cvpr52733.2024.00081, fetched 2026-10-26
[S7] Mipmap-GS: Let Gaussians Deform with Scale-Specific Mipmap for Anti-Aliasing Rendering, https://doi.org/10.1109/3dv66043.2025.00091, fetched 2026-10-27
