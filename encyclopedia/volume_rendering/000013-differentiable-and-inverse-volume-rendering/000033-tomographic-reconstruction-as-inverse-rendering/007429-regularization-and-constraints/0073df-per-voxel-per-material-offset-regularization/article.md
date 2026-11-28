# Per-voxel Per-material Offset Regularization

Also called: Material-dependent Continuous Offset, Offset Penalty

## Overview
Per-voxel Per-material Offset Regularization is a technique for discrete volume reconstruction that introduces continuous, per-voxel, per-material offset parameters to absorb local density variations and measurement errors in real-world CT data [S1]. It solves the problem of overlapping attenuation ranges caused by fabrication artifacts (e.g., in 3D-printed phantoms), allowing the optimizer to maintain correct discrete material assignments despite these variations [S1]. Unlike standard inverse path tracing with Total Variation (TV) regularization, which struggles to refine boundaries and produces intermediate material rings at non-adjacent attenuation transitions, this method absorbs local density variations without blurring boundaries, enabling crisp transitions between materials with overlapping or non-adjacent attenuation ranges [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $N$ X-ray projections (attenuation measurements); a discrete material label field $L$ where each voxel $i$ is assigned to one of $m$ materials $k \in \{1, \dots, m\}$; nominal attenuation coefficients $\mu_k$ for each material $k$; and hyperparameters $\lambda_\delta$ (penalty weight) and $\sigma_{\delta,k}$ (per-material standard deviation) [S1].
*   **Outputs:** A reconstructed volume with discrete material assignments $L$ and optimized offset parameters $\delta_{i,k}$ for each voxel and material [S1].

**Data Structures**
*   **Voxel Grid:** A 3D grid of voxels indexed by $i$.
*   **Material Labels:** A discrete field $L_i \in \{1, \dots, m\}$ indicating the primary material of voxel $i$ [S1].
*   **Offset Parameters:** A continuous tensor $\delta$ of shape $(N_{voxels}, m)$, where $\delta_{i,k}$ is the offset for voxel $i$ and material $k$ [S1].
*   **Nominal Attenuation:** A vector $\mu$ of size $m$, where $\mu_k$ is the nominal attenuation coefficient for material $k$ [S1].

**Method Steps**
1.  **Initialization:** Initialize the material labels $L$ and offset parameters $\delta_{i,k} = 0$ for all voxels $i$ and materials $k$ [G]. Initialize nominal attenuation coefficients $\mu_k$ based on material properties or initial estimates [G].
2.  **Effective Attenuation Calculation:** For each voxel $i$ and material $k$, compute the effective attenuation coefficient $\mu'_{i,k}$ by adding the offset to the nominal value:
    $$ \mu'_{i,k} = \mu_k + \delta_{i,k} $$
    This allows the actual attenuation to deviate from the nominal value $\mu_k$ by $\delta_{i,k}$ [S1].
3.  **Forward Projection:** Render the projections from the current volume state. The rendering process uses the effective attenuation coefficients $\mu'_{i,k}$ for the voxels assigned to material $k$ (i.e., where $L_i = k$) [S1]. Note: In a discrete categorical formulation, only the offset for the assigned material $L_i$ typically influences the forward model for voxel $i$, or all offsets are computed but only the active one contributes to the ray integral [S1, G].
4.  **Data Fidelity Loss:** Compute the difference between the rendered projections and the measured projections $\mathbf{y}$, typically using a least-squares or negative log-likelihood loss $\mathcal{L}_{data}$ [G].
5.  **Offset Penalty Loss:** Compute the regularization term $\mathcal{L}_{\text{offset}}$ to penalize large deviations from the nominal attenuation:
    $$ \mathcal{L}_{\text{offset}} = \lambda_\delta \sum_i \sum_{k=1}^m \frac{\delta_{i,k}^2}{2\sigma_{\delta,k}^2} $$
    where:
    *   $\lambda_\delta$ is the penalty weight [S1].
    *   $i$ indexes voxels [S1].
    *   $k$ indexes materials ($k=1..m$) [S1].
    *   $\delta_{i,k}$ is the per-voxel per-material offset [S1].
    *   $\sigma_{\delta,k}$ is the per-material standard deviation, controlling the allowed deviation strength for material $k$ [S1].
6.  **Total Loss:** Combine the data fidelity loss and the offset penalty loss:
    $$ \mathcal{L}_{total} = \mathcal{L}_{data} + \mathcal{L}_{\text{offset}} $$
    [S1, G].
7.  **Optimization:** Update the parameters (material labels $L$ and offsets $\delta$) using gradient-based optimization (e.g., Adam) or a discrete optimization strategy, minimizing $\mathcal{L}_{total}$ [S1, G]. The offsets $\delta_{i,k}$ absorb local density variations, while the discrete labels $L$ maintain material boundaries [S1].

**Parameters**
*   $\lambda_\delta$: Weight scaling the offset penalty term. Value not specified in the sources; tuned via Optuna TPESampler on a limited-angle phantom [S1].
*   $\sigma_{\delta,k}$: Per-material standard deviation. Not specified in the sources [S1].

**Pitfalls**
*   **Fabrication Artifacts:** FDM 3D printing artifacts (varying filament density, differential shrinkage, layer bonding) cause actual attenuation values to vary across the volume [S1]. This creates overlapping attenuation ranges between different materials, particularly near boundaries where temperature effects are strongest, making pointwise extinction values insufficient for material discrimination [S1].
*   **Intermediate Material Rings:** Standard TV regularization can favor distributing the gradient over multiple smaller steps rather than one large transition, manifesting as characteristic rings of intermediate materials at boundaries between materials with non-adjacent attenuation ranges (e.g., LW-PLA and S-PLA) [S1]. This method avoids such artifacts by allowing offsets to absorb variations [S1].
*   **No Pixel-Accurate Ground Truth:** Due to printing artifacts, no pixel-accurate ground truth is available for real-world phantoms, making quantitative evaluation difficult [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_offset_penalty_loss(delta, lambda_delta, sigma_delta):
    """
    Computes the per-voxel per-material offset regularization loss.
    
    Args:
        delta: Tensor of shape (num_voxels, num_materials), containing offset values.
        lambda_delta: Float, penalty weight.
        sigma_delta: Tensor of shape (num_materials,), containing per-material standard deviations.
        
    Returns:
        Float, the computed loss value.
    """
    # delta: (V, M)
    # sigma_delta: (M,)
    
    # Compute squared offsets normalized by variance
    # (delta^2) / (2 * sigma^2)
    normalized_squares = (delta ** 2) / (2.0 * (sigma_delta ** 2))
    
    # Sum over all voxels and materials
    loss = torch.sum(normalized_squares)
    
    # Scale by lambda_delta
    loss = lambda_delta * loss
    
    return loss

def update_effective_attenuation(mu, delta, labels):
    """
    Computes effective attenuation coefficients for the forward model.
    
    Args:
        mu: Tensor of shape (num_materials,), nominal attenuation coefficients.
        delta: Tensor of shape (num_voxels, num_materials), offsets.
        labels: Tensor of shape (num_voxels,), discrete material labels.
        
    Returns:
        Tensor of shape (num_voxels,), effective attenuation for each voxel.
    """
    # Gather nominal mu based on labels
    # mu_gathered: (V,)
    mu_gathered = mu[labels]
    
    # Gather offsets based on labels
    # delta_gathered: (V,)
    delta_gathered = delta[range(len(labels)), labels]
    
    # Effective attenuation is nominal + offset for the assigned material
    effective_mu = mu_gathered + delta_gathered
    
    return effective_mu
```

## Subtopics

### Weighted Prior Based Reconstruction
Uses weighted priors to guide reconstruction. [Weighted Prior Based Reconstruction](17313-weighted-prior-based-reconstruction.md)

### FreeNeRF
A differentiable volume rendering approach using neural fields. [FreeNeRF](17644-freenerf.md)

### InfoNeRF
Uses information-theoretic objectives for volume rendering. [InfoNeRF](17849-infonerf.md)

## Operating characteristics
*   **Time and Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Evaluated on real-world CT reconstruction of 3D-printed multi-material phantoms (PLA, LW-PLA, S-PLA, ABS) scanned on a ZEISS Metrotom 800 system [S1]. Sparse-view conditions: 64 projections for stochastic geometry phantom, 128 projections for TPMS phantom [S1]. Limited-angle acquisitions: 90° and 120° angular ranges, keeping same projection counts [S1].
*   **Scaling**: Not given in the sources.
*   **Strengths**: Absorbs local density variations without blurring boundaries; enables crisp transitions between materials with overlapping or non-adjacent attenuation ranges; avoids intermediate material rings at boundaries [S1].
*   **Limitations**: Requires tuning of hyperparameters ($\lambda_\delta$, $\sigma_{\delta,k}$); no pixel-accurate ground truth available for real-world phantoms due to fabrication artifacts [S1].
*   **When to use**: Real-world CT reconstruction of multi-material objects with fabrication-induced attenuation variations (e.g., 3D-printed phantoms) [S1].
*   **When not**: When material attenuation ranges are distinct and non-overlapping, and fabrication artifacts are minimal [G].

## Background
*   **Origin**: Introduced in "Inverse Rendering for Discrete X-Ray Computed Tomography" [S1].
*   **History and Development**: Developed to handle real-world deviations in CT data caused by 3D printing artifacts, such as varying filament density and differential shrinkage [S1].
*   **Usage**: Applied to real-world CT reconstruction of 3D-printed multi-material phantoms (PLA, LW-PLA, S-PLA, ABS) scanned on a ZEISS Metrotom 800 system [S1].

## Sources
[S1] Inverse Rendering for Discrete X-Ray Computed Tomography, https://doi.org/10.1145/3811391, fetched 2026-11-03
