# Full-body Avatar Modeling

Also called: Hybrid Deformation Strategy for 3D Gaussians

## Overview
Full-body Avatar Modeling anchors 3D Gaussians in a canonical space and deforms them using parametric body models (e.g., SMPL) to reconstruct dynamic human avatars from multi-view images [S1]. It distinguishes itself from head-centric modeling by focusing on full-body skeletal rigidity and secondary non-rigid effects (e.g., muscle motion) rather than fine-grained facial expressions [S1]. It builds on [SMPL-based rigid/non-rigid deformation](8385-smpl-based-rigid-non-rigid-deformation.md) by applying a hybrid deformation strategy specifically optimized for full-body dynamics.

## How it works
**Inputs:** Multi-view images of a human subject in various poses; a parametric body model (e.g., SMPL) providing skeleton hierarchy and rest-pose mesh.
**Outputs:** A set of 3D Gaussians $\{g_i\}$ defined in canonical space, each with position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and appearance coefficients (e.g., Spherical Harmonics $\mathbf{c}_i$).

**Data Structures:**
*   **Gaussian Set:** List of $N$ Gaussians. Each $g_i$ stores:
    *   $\mathbf{p}_i \in \mathbb{R}^3$: Canonical position.
    *   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Canonical covariance matrix.
    *   $\alpha_i \in [0, 1]$: Opacity.
    *   $\mathbf{c}_i$: Appearance coefficients.
*   **SMPL Model:** Provides bone transformation matrices $\mathbf{T}_j(\theta)$ for pose $\theta$ and skinning weights $w_{i,j}$ for each Gaussian $i$ relative to bone $j$.

**Steps:**
1.  **Initialization:** Initialize Gaussians $\{g_i\}$ on the surface of the canonical SMPL mesh in a rest pose [S1, 1477-gaussian-avatars.md, 1480-3dgs-avatar.md]. Assign initial skinning weights $w_{i,j}$ based on proximity to SMPL vertices or learned optimization [S1, 1480-3dgs-avatar.md].
2.  **Rigid Deformation (LBS):** For a target pose $\theta$, compute the rigidly deformed position $\mathbf{p}_{rigid}$ using Linear Blend Skinning [S1, 1480-3dgs-avatar.md]:
    $$ \mathbf{p}_{rigid} = \sum_{j=1}^{N_{bones}} w_{i,j} \mathbf{T}_j(\theta) \mathbf{p}_i $$
    where $N_{bones}$ is the number of bones in the SMPL skeleton [S1, 1480-3dgs-avatar.md].
3.  **Non-Rigid Deformation:** Apply a pose-conditioned deformation field to predict a residual offset $\Delta \mathbf{p}$ [S1]. This field accounts for secondary non-rigid effects like muscle jiggles that LBS cannot capture [S1]. The deformed position is:
    $$ \mathbf{p}'_i = \mathbf{p}_{rigid} + \Delta \mathbf{p}(\mathbf{p}_i, \theta) $$
    The deformation field is typically implemented as a small MLP conditioned on canonical position and pose parameters [S1, 1480-3dgs-avatar.md].
4.  **Covariance Update:** Transform the covariance matrix $\Sigma_i$ using the Jacobian $J$ of the total deformation map (LBS + non-rigid field) with respect to $\mathbf{p}_i$ [G]:
    $$ \Sigma'_i = J \Sigma_i J^T + \sigma^2 I $$
    where $\sigma^2 I$ is a small regularization term to ensure positive definiteness [G]. The Jacobian $J$ is computed via automatic differentiation of the deformation pipeline [G, 11805-full-body-avatar-modeling.md].
5.  **Rendering:** Render the deformed Gaussians $\{g'_i\}$ using differentiable 3D Gaussian Splatting [S1, 1477-gaussian-avatars.md]. Compute loss against target images using photometric ($\mathcal{L}_{L1}$) and perceptual ($\mathcal{L}_{MS-SSIM}$) metrics [6136-instantsplat.md][6464-pixelsplat.md].
6.  **Optimization:** Backpropagate gradients through the deformation pipeline to update Gaussian parameters and deformation network weights [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deform_gaussian(gaussian, smpl_model, pose, deformation_net):
    """
    Deforms a single Gaussian from canonical to posed space.
    gaussian: dict with keys 'p', 'Sigma', 'alpha', 'c'
    smpl_model: object providing get_skinning_weights(p) and get_bone_transforms(pose)
    pose: pose parameters for SMPL
    deformation_net: MLP for non-rigid offset
    """
    # 1. Rigid Deformation via LBS
    weights = smpl_model.get_skinning_weights(gaussian['p'])
    bone_transforms = smpl_model.get_bone_transforms(pose)
    p_rigid = sum(weights[j] * bone_transforms[j] @ gaussian['p'] for j in range(len(weights)))
    
    # 2. Non-Rigid Deformation
    delta_p = deformation_net(gaussian['p'], pose)
    p_deformed = p_rigid + delta_p
    
    # 3. Covariance Update via Jacobian
    # Compute Jacobian J of deformation map w.r.t canonical position
    # This requires differentiating the sum of LBS and deformation_net
    J = compute_jacobian(p_rigid, delta_p, gaussian['p'])
    Sigma_deformed = J @ gaussian['Sigma'] @ J.T + 1e-6 * np.eye(3)
    
    return {
        'p': p_deformed,
        'Sigma': Sigma_deformed,
        'alpha': gaussian['alpha'],
        'c': gaussian['c']
    }

def compute_jacobian(p_rigid, delta_p, p_canonical):
    """
    Computes the Jacobian of the total deformation with respect to canonical position.
    """
    # Jacobian of LBS part
    # J_lbs = sum(w_j * d(T_j @ p)/dp) = sum(w_j * T_j)
    # Note: T_j are constant for fixed pose, so d(T_j @ p)/dp = T_j
    # However, weights w_j might depend on p, so full derivative is complex.
    # In practice, often approximated or computed via autograd.
    pass # Implementation depends on specific autograd framework
```

**Parameters:**
*   $\sigma^2$: Regularization strength for covariance. Typical value: $10^{-6}$ [G].
*   Deformation network depth/width: Not given in the sources.

**Edge Cases:**
*   **Degenerate Poses:** Extreme poses may cause skinning weights to produce invalid transformations. Guard against negative determinants in covariance update [G].
*   **Precision:** Jacobian computation must be numerically stable. Use automatic differentiation rather than finite differences [G].

## Subtopics

### [Linear Blend Skinning for Gaussian Splatting](16258-linear-blend-skinning-for-gaussian-splatting.md)
A specific deformation technique using skeleton-driven rigid joint transformations and learnable per-Gaussian skinning weights. It avoids pose blend shapes and template vertex deformation, relying on an MLP for pose-dependent deformations to handle shape estimation inaccuracies.

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ per pixel, where $N$ is the number of Gaussians. Deformation is $O(N \cdot N_{bones})$ for LBS plus $O(N)$ for the deformation network.
*   **Memory Complexity:** $O(N)$ for Gaussian parameters. Additional memory for SMPL model and deformation network weights.
*   **Performance:** Fast training and rendering compared to NeRF-based methods due to explicit Gaussian representation [S1].
*   **Strengths:** High fidelity for dynamic full-body avatars; real-time rendering capability; editability of Gaussians [S1].
*   **Limitations:** Requires accurate SMPL fitting; non-rigid deformation field may struggle with extreme articulations or clothing changes [S1].
*   **When to use:** When real-time or near-real-time dynamic full-body avatars are needed from multi-view video.
*   **When not to use:** For head-centric avatars with fine facial details (use [Gaussian Avatar](1479-gaussian-avatar.md)) or static scenes (use standard [3D Gaussian Splatting](52-3d-gaussian-splatting.md)).

## Background
Full-body Avatar Modeling using 3D Gaussian Splatting emerged as a key direction in 3D GS applications, alongside head-centric modeling [S1]. It leverages the speed and editability of GS to enable fast training and rendering of dynamic humans [S1]. The approach typically anchors Gaussians in canonical space and uses a hybrid deformation strategy combining LBS for rigid skeletal transformations and pose-conditioned deformation fields for non-rigid effects [S1]. This method is part of the broader trend of using parametric body models (like SMPL) to guide Gaussian deformation for animatable avatars [S1, 1477-gaussian-avatars.md].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
