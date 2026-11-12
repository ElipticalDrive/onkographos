# Deformable 3DGS

Also called: Dynamic 3DGS

## Overview
Deformable 3DGS is a method for representing and rendering dynamic 3D scenes by applying a learned deformation network to a set of canonical 3D Gaussians over time [S1, unofficial]. It solves the problem of real-time rendering of non-rigid motion by predicting per-Gaussian deformation deltas based on a time parameter, allowing the static 3DGS pipeline to handle temporal variations without explicit 4D primitives [S1, unofficial]. It differs from sibling methods like Sparse-Controlled 3D Gaussian Splatting by using a dense MLP deformation network on per-Gaussian features rather than sparse control points, and from 4DGS by using an MLP instead of a HexPlane decoder [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of canonical 3D Gaussians $\mathcal{G}_{canon}$ (position $\boldsymbol{\mu}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, SH coefficients $\mathbf{c}_{SH,i}$), a time parameter $t \in [0, 1]$, and camera poses.
*   **Output:** Rendered image for the scene at time $t$.

**Data Structures:**
*   **Canonical Gaussians:** Standard 3DGS layout.
*   **Deformation Network:** An MLP that takes per-Gaussian features and time $t$ as input and outputs deformation deltas $(\Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i)$ [S1, unofficial].

**Steps:**
1.  **Feature Extraction:** For each Gaussian $i$, extract its canonical features (e.g., position, scale, or learned embeddings) [S1, unofficial].
2.  **Deformation Prediction:** Pass the features and time $t$ through the MLP deformation network to predict deltas $\Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i$ [S1, unofficial].
3.  **Time-Varying Parameter Update:** Compute the current Gaussian parameters using the shared principles of the parent method [G]:
    $$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i $$
    $$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i \otimes \Delta\mathbf{q}_i) $$
    $$ \mathbf{s}_i(t) = \mathbf{s}_i + \Delta\mathbf{s}_i $$
    where $\otimes$ is quaternion multiplication and normalization prevents drift [G].
4.  **Rendering:** Rasterize the updated Gaussians $\mathcal{G}(t)$ using standard differentiable rasterization for the given camera pose and time $t$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deformable_3dgs_render(canonical_gaussians, time_t, camera_pose, deformation_mlp):
    """
    Renders a dynamic scene at time_t using Deformable 3DGS.
    
    Args:
        canonical_gaussians: List of Gaussian objects with attributes 
                             mu, q, s, alpha, c_SH.
        time_t: Float in [0, 1] representing the current time.
        camera_pose: Camera extrinsics and intrinsics.
        deformation_mlp: MLP network predicting deformation deltas.
        
    Returns:
        Rendered image tensor.
    """
    current_gaussians = []
    
    for g in canonical_gaussians:
        # 1. Predict deformation deltas
        # Input to MLP: g's features (e.g., mu, s) and time_t
        # Output: delta_mu, delta_q, delta_s
        delta_mu, delta_q, delta_s = deformation_mlp(g.mu, g.s, time_t)
        
        # 2. Update parameters
        # Position update
        mu_t = g.mu + delta_mu
        
        # Rotation update (quaternion multiplication and normalization)
        # q is stored as [w, x, y, z] or [x, y, z, w]; assume standard format
        q_t = normalize(quaternion_multiply(g.q, delta_q))
        
        # Scale update
        s_t = g.s + delta_s
        
        # Opacity and SH are typically static in this base method
        alpha_t = g.alpha
        c_SH_t = g.c_SH
        
        # Create new Gaussian with updated parameters
        current_gaussians.append(Gaussian(mu=mu_t, q=q_t, s=s_t, alpha=alpha_t, c_SH=c_SH_t))
        
    # 3. Rasterize
    image = rasterize(current_gaussians, camera_pose)
    return image
```

**Parameters:**
*   **Time $t$:** Normalized to $[0, 1]$ [S1, unofficial].
*   **MLP Architecture:** Not specified in sources [S1, unofficial].

**Edge Cases:**
*   **Rotation Drift:** Quaternion normalization is required after multiplication to maintain unit length [G].
*   **Boundary Conditions:** Time $t$ must be within the training range; extrapolation may fail [G].

## Subtopics

### SplineGS
A method that replaces discrete per-frame deformation with a smooth spline modeling the trajectory of each primitive's center and rotation as a function of time, ensuring physically plausible, smooth motion with fewer parameters [S2, unofficial]. It uses B-spline basis functions $\mathbf{B}_k$ and control points $\mathbf{p}_{i,k}$:
$$ \mu_i(t) = \sum_{k=0}^K \mathbf{B}_k(t) \mathbf{p}_{i,k} $$
[S2, unofficial]. Choose this when smooth, continuous motion is prioritized over per-frame flexibility.

### VolSegGS
A method for segmenting and tracking structures across timesteps in dynamic volumes by representing scenes as deformable 3D Gaussians, sidestepping grid memory constraints [S3, unofficial]. It performs coarse segmentation by clustering Gaussians by view-independent colors, refines clusters using an affinity field network guided by SAM 2D prompts, and maintains segments across timesteps through the deformation field [S3, unofficial]. It achieved PSNR values of 25.76–29.02 dB at 88–89 FPS on datasets up to 960x1440x240 [S3, unofficial]. Choose this for dynamic volumetric segmentation tasks.

## Operating characteristics
*   **Time Complexity:** Depends on the MLP inference cost per Gaussian and the rasterization cost. Real-time rendering is claimed [S3, unofficial].
*   **Memory Complexity:** Stores canonical Gaussians plus MLP weights. No explicit temporal storage for frames [S1, unofficial].
*   **Performance:** Real-time rendering speeds reported in related dynamic methods (e.g., VolSegGS at 88–89 FPS) [S3, unofficial].
*   **Strengths:** Real-time rendering of dynamic scenes; compact representation compared to explicit 4D grids.
*   **Limitations:** MLP inference can be slower than lookup-based methods (e.g., HexPlane) [S1, unofficial]; deformation quality depends on MLP capacity.

## Background
Deformable 3DGS was introduced by Yang et al. in CVPR'24 [S1, unofficial]. It extends the static 3D Gaussian Splatting framework to handle dynamic scenes by learning a deformation field, contrasting with 4DGS which uses a HexPlane decoder [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
