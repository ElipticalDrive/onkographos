# Material Smoothness Loss

Also called: Lmat

## Overview
Material Smoothness Loss is a regularization term applied to material fields to enforce spatial smoothness by penalizing differences in material parameters between neighboring surface points [S1]. A specific variant, the BRDF Smoothness Loss, enforces smoothness of BRDF parameters using a normalized difference weighted by pseudo-albedo gradients [S2]. It differs from standard smoothness losses by incorporating a bilateral weight derived from rendered color and irradiance to modulate the penalty [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Material parameters $\boldsymbol{\beta}$ (e.g., roughness, specular tint) defined at surface points $\mathbf{x}_{\text{pos}}$; perturbation vector $\boldsymbol{\xi}$; pseudo-albedo $\mathbf{a}_{\text{pseudo}}$ derived from rendered color and irradiance; weighting coefficient $C_{\text{BRDF}}$; sample weights $w_k$ [S1, S2].
*   **Outputs:** Scalar loss value $\mathcal{L}_{\text{BRDF}}$ used for gradient-based optimization [S2].

**Data Structures:**
*   Material parameters $\boldsymbol{\beta}$: Tensor of shape $(N, D)$ representing BRDF parameters at $N$ sampled points [S2].
*   Surface points $\mathbf{x}_{\text{pos}}$: Tensor of shape $(N, 3)$ containing 3D coordinates [S2].
*   Perturbation $\boldsymbol{\xi}$: Tensor of shape $(N, 3)$ sampled from a Gaussian distribution [S2].

**Steps:**
1.  **Sample Perturbations:** For each surface point $\mathbf{x}_{\text{pos}_k}$, sample a perturbation vector $\boldsymbol{\xi}$ from a normal distribution $\mathcal{N}(\mathbf{0}, \epsilon \mathrm{I})$ [S2].
2.  **Evaluate Material Difference:** Compute the normalized difference in material parameters between the point and its perturbed neighbor:
    $$
    \Delta \boldsymbol{\beta}_k = \left| \frac{\boldsymbol{\beta}(\mathbf{x}_{\text{pos}_k}) - \boldsymbol{\beta}(\mathbf{x}_{\text{pos}_k} + \boldsymbol{\xi})}{\operatorname{max}(\boldsymbol{\beta}(\mathbf{x}_{\text{pos}_k}), \boldsymbol{\beta}(\mathbf{x}_{\text{pos}_k} + \boldsymbol{\xi}))} \right|
    $$
    where the division, absolute value, and max are element-wise operations [S2].
3.  **Compute Pseudo-Albedo:** Calculate the pseudo-albedo $\mathbf{a}_{\text{pseudo}}$ at both the original point and the perturbed point to serve as the bilateral weight [S2]:
    $$
    \mathbf{a}_{\text{pseudo}}(\mathbf{x}) = \frac{\mathbf{c}(\mathbf{x}, \mathbf{d}_{\text{out}})}{\mathbf{c}_{\text{irradiance}}(\mathbf{x}, \mathbf{d}_{\text{out}})}
    $$
    where $\mathbf{c}$ is the rendered color, $\mathbf{d}_{\text{out}}$ is the outgoing view direction, and $\mathbf{c}_{\text{irradiance}}$ is the irradiance from the physically-based model (computed by setting the BRDF to be perfectly Lambertian with albedo $a=1$) [S2].
4.  **Compute Bilateral Weight:** Calculate the weight $\lambda$ based on the difference in pseudo-albedo:
    $$
    \lambda(\mathbf{x}, \mathbf{x} + \boldsymbol{\xi}) = | \mathbf{a}_{\text{pseudo}}(\mathbf{x}_{\text{pos}_k}) - \mathbf{a}_{\text{pseudo}}(\mathbf{x}_{\text{pos}_k} + \boldsymbol{\xi}) |
    $$
    [S2].
5.  **Aggregate Loss:** Sum the weighted material differences over all sampled points $k$ with weights $w_k$:
    $$
    \mathcal{L}_{\text{BRDF}} = C_{\text{BRDF}} \sum_{k} w_k \Delta \boldsymbol{\beta}_k \lambda(\mathbf{x}, \mathbf{x} + \boldsymbol{\xi})
    $$
    [S2].

**Parameters:**
*   $\epsilon$: Standard deviation of the perturbation noise. Set to 0.01 for the TensoIR dataset, and 0.005 for other datasets [S2].
*   $C_{\text{BRDF}}$: Scaling coefficient for the loss. Set to 0.05 for the TensoIR dataset, and 0.001 for other datasets [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def material_smoothness_loss(beta_orig, beta_pert, pseudo_albedo_orig, pseudo_albedo_pert, weights, epsilon, coefficient):
    """
    Computes the Material Smoothness Loss (BRDF variant).
    
    Args:
        beta_orig: Tensor of shape (N, D) containing BRDF parameters beta at N points.
        beta_pert: Tensor of shape (N, D) containing BRDF parameters at (x_pos + xi).
        pseudo_albedo_orig: Tensor of shape (N, 3) containing pseudo-albedo at N points.
        pseudo_albedo_pert: Tensor of shape (N, 3) containing pseudo-albedo at (x_pos + xi).
        weights: Tensor of shape (N,) containing sample weights w_k.
        epsilon: Float, standard deviation for perturbation noise (used for sampling, not in forward pass).
        coefficient: Float, scaling factor C_BRDF.
        
    Returns:
        Scalar loss value.
    """
    import torch
    
    # [G] beta_pert and pseudo_albedo_pert are obtained by querying the material/irradiance fields 
    # at positions (surface_points + xi) using the reparameterization trick for differentiability.
    
    # Step 2: Normalized material difference
    max_beta = torch.maximum(beta_orig, beta_pert)
    max_beta = torch.clamp(max_beta, min=1e-6) # Avoid division by zero
    diff_beta = torch.abs(beta_orig - beta_pert)
    normalized_diff = diff_beta / max_beta
    
    # Step 4: Bilateral weight
    lambda_weight = torch.abs(pseudo_albedo_orig - pseudo_albedo_pert)
    
    # Step 5: Aggregate loss
    loss_per_point = normalized_diff * lambda_weight * weights.unsqueeze(1)
    loss = coefficient * torch.sum(loss_per_point)
    
    return loss
```

## Subtopics

### [Bilateral Smoothness Regularization](10915-bilateral-smoothness-regularization.md)
Bilateral Smoothness Regularization is a geometric and normal alignment loss applied to material fields to enforce spatial smoothness of BRDF parameters (such as roughness and specular tint) weighted by image gradients. It alleviates the ambiguity between material and illumination by constraining material gradients to correspond with image edges, preventing noisy, inconsistent material reconstructions. It differs from standard smoothness losses by using the image gradient as a bilateral weight, preserving material boundaries where the image shows sharp transitions.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sampled surface points $N$, as it involves a summation over $k$ points [S2].
*   **Memory Complexity:** Linear with respect to $N$ for storing material parameters and pseudo-albedo values at sampled points [G].
*   **Strengths:** Effectively reduces noise in reconstructed material fields by enforcing spatial smoothness [S1, S2].
*   **Limitations:** Depends on accurate pseudo-albedo estimation; hyperparameters $\epsilon$ and $C_{\text{BRDF}}$ are dataset-dependent [S2].
*   **When to use:** When reconstructing materials from multi-view images and spatial smoothness of BRDF parameters is desired [S1, S2].
*   **When not to use:** When the scene contains highly heterogeneous materials where smoothness constraints would blur distinct material boundaries [G].

## Background
The loss was introduced to regularize material fields in inverse rendering from multi-view images [S1]. A variant enforcing BRDF smoothness using pseudo-albedo weighting was later detailed for radiance cache-based inverse rendering [S2].

## Sources
[S1] IRCasTRF: Inverse Rendering by Optimizing Cascaded Tensorial Radiance Fields, Lighting, and Materials From Multi-view Images, https://doi.org/10.1145/3581783.3612010, fetched 2026-10-27
[S2] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
