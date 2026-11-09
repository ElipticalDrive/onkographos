# Deformable 3DGS

Also called: Dynamic 3DGS

## Overview
Deformable 3DGS is a method for rendering dynamic scenes by applying a Multi-Layer Perceptron (MLP) deformation network to per-Gaussian features over time [S1, unofficial]. It solves the problem of real-time rendering of non-rigid motion by decomposing the scene into a static canonical representation and a time-dependent deformation field, building on [Deformation and Dynamic Modeling](11474-deformation-and-dynamic-modeling.md). It is distinguished from sibling methods like [Sparse-Controlled 3D Gaussian Splatting](6145-sparse-controlled-3d-gaussian-splatting.md) by using a dense MLP predictor for all Gaussians rather than sparse control points, and from [4D Gaussian Splatting](https://arxiv.org/abs/2404.03414) (which uses HexPlane/k-Plane decoders) by its reliance on a standard MLP architecture [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of canonical 3D Gaussians $\mathcal{G}_{canon}$, each defined by position $\boldsymbol{\mu}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_{SH,i}$. A time parameter $t$ normalized to $[0, 1]$.
*   **Outputs:** Time-varying Gaussian parameters $\mathbf{c}_i(t)$ for rendering at time $t$.

**Data Structures:**
The canonical Gaussian $i$ stores a feature vector $\mathbf{f}_i$ (typically derived from the SH coefficients or a learned latent code) and the base geometric parameters. The deformation network $\mathcal{F}_{\theta}$ is an MLP.

**Algorithm:**
1.  **Feature Extraction:** For each Gaussian $i$, extract or retrieve its canonical feature vector $\mathbf{f}_i$.
2.  **Deformation Prediction:** Input the concatenation of $\mathbf{f}_i$ and the normalized time $t$ into the MLP deformation network $\mathcal{F}_{\theta}$ to predict deformation deltas:
    $$ \Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i = \mathcal{F}_{\theta}(\mathbf{f}_i, t) $$
3.  **Parameter Update:** Compute the time-varying parameters using the shared principles of the parent family [Deformation and Dynamic Modeling](11474-deformation-and-dynamic-modeling.md):
    $$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i $$
    $$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i \otimes \Delta\mathbf{q}_i) $$
    $$ \mathbf{s}_i(t) = \mathbf{s}_i + \Delta\mathbf{s}_i $$
    where $\otimes$ denotes quaternion multiplication [G].
4.  **Rendering:** Render the updated Gaussians $\mathbf{c}_i(t)$ using standard differentiable rasterization.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deformable_3dgs_render(gaussians, time_t, deformation_mlp):
    """
    Render a dynamic scene at time t using Deformable 3DGS.
    
    Args:
        gaussians: List of canonical Gaussian objects.
                   Each has fields: mu (3,), q (4,), s (3,), alpha (1,), 
                   sh_coeffs (array), and feature f (vector).
        time_t: Float, normalized time in [0, 1].
        deformation_mlp: A callable MLP function taking (feature, time) -> (d_mu, d_q, d_s).
    
    Returns:
        Image tensor.
    """
    # 1. Predict deformations for all Gaussians
    # In practice, this is batched.
    deformations = []
    for g in gaussians:
        # Input to MLP: canonical feature vector concatenated with time
        input_vec = concatenate(g.feature, [time_t])
        d_mu, d_q, d_s = deformation_mlp(input_vec)
        deformations.append((d_mu, d_q, d_s))
    
    # 2. Update Gaussian parameters
    dynamic_gaussians = []
    for i, g in enumerate(gaussians):
        d_mu, d_q, d_s = deformations[i]
        
        # Update position
        new_mu = g.mu + d_mu
        
        # Update rotation (quaternion multiplication and normalization)
        # q_new = q_canon * q_delta
        new_q = quaternion_multiply(g.q, d_q)
        new_q = normalize(new_q)
        
        # Update scale
        new_s = g.s + d_s
        
        # Create dynamic Gaussian instance
        dyn_g = Gaussian(
            mu=new_mu,
            q=new_q,
            s=new_s,
            alpha=g.alpha,
            sh_coeffs=g.sh_coeffs
        )
        dynamic_gaussians.append(dyn_g)
    
    # 3. Rasterize
    image = rasterize(dynamic_gaussians)
    return image
```

**Edge Cases and Pitfalls:**
*   **Quaternion Normalization:** The output $\Delta\mathbf{q}_i$ from the MLP is not guaranteed to be a unit quaternion. It must be normalized before multiplication [G].
*   **Temporal Continuity:** Without regularization, the MLP may produce jittery predictions between adjacent frames.
*   **Degenerate Input:** If $t$ is outside $[0, 1]$, extrapolation behavior is undefined and may cause artifacts.

## Subtopics

### SplineGS
A method that replaces the discrete per-frame MLP deformation with a smooth spline modeling the trajectory of each primitive's center and rotation as a function of time, ensuring physically plausible, smooth motion with fewer parameters [S2, unofficial]. It uses B-spline basis functions $\mathbf{B}_k$ and control points $\mathbf{p}_{i,k}$:
$$ \mu_i(t) = \sum_{k=0}^K \mathbf{B}_k(t) \mathbf{p}_{i,k} $$
[S2, unofficial]

### VolSegGS
A method for segmenting and tracking structures across timesteps in dynamic volumes by representing scenes as deformable 3D Gaussians, sidestepping grid memory constraints [S3, unofficial]. It performs coarse-level segmentation by clustering Gaussians by view-independent colors, refines clusters into fine-level segments using an affinity field network guided by SAM 2D prompts, and maintains segments across timesteps through the deformation field [S3, unofficial]. It achieved PSNR values of 25.76–29.02 dB at 88–89 FPS on datasets up to 960x1440x240 [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the MLP evaluation per Gaussian and the standard 3DGS rasterization. The MLP adds a constant overhead per Gaussian per frame.
*   **Memory Complexity:** Requires storing the canonical Gaussians and the parameters of the deformation MLP. The MLP size is typically small compared to the number of Gaussians.
*   **Performance:** Real-time rendering is achievable on modern GPUs, though the MLP inference adds latency compared to static 3DGS.
*   **Strengths:** Handles complex non-rigid deformations without explicit skeletal structures.
*   **Limitations:** The MLP may struggle with high-frequency temporal details or require significant regularization to avoid flickering. It is less compact than sparse-control methods for articulated objects.

## Background
Deformable 3DGS was introduced by Yang et al. in CVPR'24 [S1, unofficial]. It emerged as a response to the need for real-time rendering of dynamic scenes, extending the static 3D Gaussian Splatting framework. It is part of the broader family of 4D Gaussian Splatting methods, distinct from those using grid-based encoders like HexPlane [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
