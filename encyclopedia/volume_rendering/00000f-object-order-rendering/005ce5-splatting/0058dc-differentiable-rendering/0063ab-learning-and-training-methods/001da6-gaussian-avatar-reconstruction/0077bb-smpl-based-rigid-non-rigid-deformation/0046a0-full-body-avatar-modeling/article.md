# Full-body Avatar Modeling

Also called: Hybrid Deformation Strategy for 3D Gaussians

## Overview
Full-body Avatar Modeling anchors 3D Gaussians in a canonical space and deforms them using parametric body models (e.g., SMPL) or cage-based rigging to reconstruct dynamic human avatars [S1]. It distinguishes itself from head-centric modeling by focusing on full-body skeletal rigidity and secondary non-rigid effects (e.g., muscle motion) rather than fine-grained facial expressions [S1]. It builds on [SMPL-based rigid/non-rigid deformation](8385-smpl-based-rigid-non-rigid-deformation.md) by applying a hybrid deformation strategy that combines rigid skeletal transformations with pose-conditioned non-rigid fields [S1].

## How it works
**Inputs:** Multi-view images and pose parameters [G].
**Outputs:** Deformed 3D Gaussians ready for differentiable rasterization [G].
**Data Structures:** Gaussian set storing canonical position, covariance, opacity, and appearance coefficients [G]. Parametric body model providing bone transformations [G]. Not given in the sources: exact initialization placement, skinning weight assignment method, and deformation network architecture.

**Steps:**
1.  **Initialization:** Place Gaussians in canonical space [S1]. Optimize skinning weights and deformation network parameters during training [G].
2.  **Rigid Deformation (LBS):** Compute rigidly deformed position $\mathbf{p}_{\text{rigid}}$ using Linear Blend Skinning [S1]:
    $$ \mathbf{p}_{\text{rigid}} = \sum_{j} w_{i,j} \mathbf{T}_j(\theta) \mathbf{p}_i $$
    where $w_{i,j}$ are skinning weights, $\mathbf{T}_j(\theta)$ are bone transformation matrices for pose $\theta$, and the sum runs over all bones [S1].
3.  **Non-Rigid Deformation:** Apply a pose-conditioned deformation field to predict a residual offset $\Delta \mathbf{p}$ [S1]. This accounts for secondary non-rigid effects like muscle jiggles [S1]. The deformed position is:
    $$ \mathbf{p}'_i = \mathbf{p}_{\text{rigid}} + \Delta \mathbf{p}(\mathbf{p}_i, \theta) $$
4.  **Covariance Update:** Transform the covariance matrix $\Sigma_i$ using the Jacobian $J$ of the total deformation map with respect to $\mathbf{p}_i$ [G]:
    $$ \Sigma'_i = J \Sigma_i J^\top + \sigma^2 I $$
    where $\sigma^2 I$ is a small regularization term to ensure positive definiteness [G]. The Jacobian $J$ is computed via automatic differentiation of the deformation pipeline [G].
5.  **Rendering:** Render the deformed Gaussians using differentiable 3D Gaussian Splatting [S1].
6.  **Optimization:** Update Gaussian parameters and deformation network weights by backpropagating through the deformation and rendering pipeline [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mat_mul(A, B):
    return [[sum(A[i][k] * B[k][j] for k in range(len(A[0]))) for j in range(len(B[0]))] for i in range(len(A))]

def transpose(M):
    return [[M[j][i] for j in range(len(M))] for i in range(len(M[0]))]

def deform_gaussian(gaussian, smpl_model, pose, deformation_net):
    # gaussian: dict with keys 'p', 'Sigma', 'alpha', 'c'
    # smpl_model: object providing get_skinning_weights(p) and get_bone_transforms(pose)
    # pose: pose parameters for SMPL
    # deformation_net: callable for non-rigid offset
    
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
    Sigma_deformed = mat_mul(J, mat_mul(gaussian['Sigma'], transpose(J)))
    for i in range(3):
        Sigma_deformed[i][i] += 1e-6
        
    return {
        'p': p_deformed,
        'Sigma': Sigma_deformed,
        'alpha': gaussian['alpha'],
        'c': gaussian['c']
    }

def compute_jacobian(p_rigid, delta_p, p_canonical):
    # Jacobian computed via automatic differentiation of the deformation pipeline [G].
    raise NotImplementedError("Jacobian computed via automatic differentiation of the deformation pipeline [G].")
```

**Parameters:**
*   $\sigma^2$: Regularization strength for covariance. Typical value: $10^{-6}$ [G].

**Edge Cases:**
*   **Degenerate Poses:** Extreme poses may cause skinning weights to produce invalid transformations. Guard against negative determinants in covariance update [G].
*   **Precision:** Jacobian computation must be numerically stable. Use automatic differentiation rather than finite differences [G].

## Subtopics

### [Linear Blend Skinning for Gaussian Splatting](16258-linear-blend-skinning-for-gaussian-splatting.md)
A specific deformation technique using skeleton-driven rigid joint transformations and learnable per-Gaussian skinning weights. It avoids pose blend shapes and template vertex deformation, relying on an MLP for pose-dependent deformations to handle shape estimation inaccuracies.

## Operating characteristics
*   **Time Complexity:** Rendering leverages the $O(N)$ splatting speed of 3D Gaussian Splatting [S1]. Deformation and network evaluation complexity are Not given in the sources.
*   **Memory Complexity:** $O(N)$ for Gaussian parameters [G]. Additional memory for SMPL model and deformation network weights [G].
*   **Performance:** Exploits the speed advantage and editability of Gaussian Splatting for fast training and rendering [S1].
*   **Strengths:** High fidelity for dynamic full-body avatars; real-time rendering capability; editability of Gaussians [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Full-body Avatar Modeling using 3D Gaussian Splatting emerged as a key direction in 3D GS applications, alongside head-centric modeling [S1]. It leverages the speed and editability of GS to enable fast training and rendering of dynamic humans [S1]. The approach typically anchors Gaussians in canonical space and uses a hybrid deformation strategy combining LBS for rigid skeletal transformations and pose-conditioned deformation fields for non-rigid effects [S1]. This method is part of the broader trend of using parametric body models (like SMPL) to guide Gaussian deformation for animatable avatars [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
