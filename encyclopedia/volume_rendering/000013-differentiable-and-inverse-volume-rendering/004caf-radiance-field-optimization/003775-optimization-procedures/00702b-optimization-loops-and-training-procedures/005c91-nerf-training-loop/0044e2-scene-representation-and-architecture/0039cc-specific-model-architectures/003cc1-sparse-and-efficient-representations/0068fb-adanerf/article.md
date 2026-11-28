# AdaNeRF

Also called: Adaptive NeRF

## Overview
AdaNeRF is a compact dual-network neural representation for real-time rendering of neural radiance fields that reduces the number of required sample points per ray by learning to predict sample importance [S1]. It consists of a sampling network, evaluated once per view ray to predict density weights, and a shading network, evaluated only for the most significant samples [S1]. Unlike sibling methods such as [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md) or [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md) which rely on explicit spatial data structures, AdaNeRF supports variable sample counts and single-sample-per-ray rendering without spatial data structures [S1]. It distinguishes itself from [TermiNeRF](8776-mega-nerf-spatial-partitioning.md) by not requiring a pretrained NeRF for initialization, and from [AutoInt](8776-mega-nerf-spatial-partitioning.md) by using fixed sample locations and a simpler training scheme [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   Ray origin $p$ and ray direction $d$.
*   Fixed discrete sample positions along the ray (one per cell in the discretized space).

**Outputs:**
*   Accumulated color $\hat{C}(r)$ for the ray.

### Data Structures
*   **Sampling Network ($S$):** An MLP with 8 layers and 256 units [S1].
    *   Input: Ray origin $p$ and direction $d$.
    *   Output: Vector $\delta \in \mathbb{R}^N$, where $N$ is the total number of fixed sample positions along the ray. Each $\delta_i$ represents the predicted importance/density weight for sample $i$.
*   **Shading Network ($T$):** An MLP with 8 layers and 256 units [S1].
    *   Input: Positional encoding of the selected sample positions.
    *   Output: Density $\sigma_i$ and color $c_i$ for each selected sample $i$.

### Numbered Steps
1.  **Sampling Network Evaluation:** Input ray origin $p$ and direction $d$ into the sampling network $S$. Obtain the vector of predicted densities $\delta = [\delta_1, \dots, \delta_N]$.
2.  **Adaptive Selection:** Determine the subset of indices $I$ to evaluate the shading network based on $\delta$ and a threshold $\tau$ and maximum samples $N_{max}$:
    *   Count $N_s$, the number of $\delta_i$ values exceeding $\tau$.
    *   If $N_s = 0$: Select the index $i$ corresponding to the largest $\delta_i$.
    *   If $N_s \le N_{max}$: Select all indices $i$ where $\delta_i > \tau$.
    *   If $N_s > N_{max}$: Select the indices corresponding to the $N_{max}$ largest $\delta_i$ values.
3.  **Shading Network Evaluation:** For each selected index $i \in I$, evaluate the shading network $T$ to obtain density $\sigma_i$ and color $c_i$.
4.  **Ray Accumulation:** Compute the accumulated color $\hat{C}(r)$ using the modified volume rendering equation:
    $$ \hat{C}(r) = \sum_{i \in I} T_i (1 - \exp(-\delta_i \sigma_i t_i)) c_i $$
    where $T_i = \exp(-\sum_{j \in I, j < i} \delta_j \sigma_j t_j)$ is the accumulated transmittance, and $t_i$ is the distance between adjacent samples [S1]. Note that the summation and transmittance calculation only include samples in the selected subset $I$, but the transmittance term $T_i$ implicitly accounts for the culling by using the product of $\delta_j \sigma_j$ for previous samples. *Correction based on source logic:* The source defines the accumulation over $N$ samples but implies culling. The formula provided in [S1] Eq 1 sums over $i=1$ to $N$. However, the text states "evaluate the shading network T only for the remaining subset". The implementation detail implies that for $i \notin I$, $\sigma_i$ is effectively 0 or not computed. The formula $\hat{C}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\delta_i \sigma_i t_i)) c_i$ is used, where if sample $i$ is culled, it is treated as having no contribution (effectively $\sigma_i=0$ or skipped in the sum). The transmittance $T_i$ must still account for the density of previous samples to maintain correct occlusion. The source states: "multiply the density $\sigma$ predicted by the shading network with the density $\delta$ predicted by the sampling network". This multiplication is key for gradient flow.
5.  **Training:** The networks are trained end-to-end using a 4-phase scheme.

### Formulas
**Modified Ray Accumulation:**
$$ \hat{C}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\delta_i \sigma_i t_i)) c_i $$
$$ T_i = \exp\left(-\sum_{j=1}^{i-1} \delta_j \sigma_j t_j\right) $$
Where:
*   $\hat{C}(r)$: Estimated accumulated color.
*   $N$: Total number of fixed sample positions.
*   $T_i$: Accumulated transmittance up to sample $i$.
*   $t_i$: Distance between adjacent samples.
*   $\sigma_i$: Density from the shading network.
*   $\delta_i$: Density/weight from the sampling network.
*   $c_i$: Color from the shading network [S1].

**Sampling Network Loss:**
$$ l_{sampling}(\delta, \sigma, c) = \lambda_0 \cdot l_{mse}(\delta, \sigma, c) + \lambda_1 \cdot l_{sparsity}(\delta, \sigma) $$
With $\lambda_0 = 0.001$ and $\lambda_1 = 1.0$ [S1].

**Sparsity Loss:**
$$ l_{sparsity}(\delta, \sigma) = \lambda \cdot \frac{1}{N} \sum_{i=1}^{N} (|\delta_i - 0| + |\sigma_i - \delta_i|) + (1 - \lambda) \cdot \frac{1}{N} \sum_{i=1}^{N} |\delta_i - 1| $$
Where $\lambda = \frac{t_c - t_0}{t_d}$, with $t_0$ being the start iteration, $t_d$ the duration, and $t_c$ the current iteration [S1].

**MSE Loss for Sampling Network:**
The source mentions $l_{mse}(\delta, \sigma, c)$ but does not provide the explicit formula for how $\delta$ is compared to $\sigma$ and $c$ in the MSE term, other than implying it helps stabilize the network. The ablation study notes that removing shading density supervision degrades quality, suggesting the MSE term involves $\sigma$. *Note: The exact formulation of $l_{mse}$ is not explicitly defined in the provided text snippets beyond the loss combination equation.* [S1].

### Parameters
*   **$\tau$ (tau):** Adaptive sampling threshold. Typical range swept $[0.05, 0.40]$ [S1].
*   **$N_{max}$:** Maximum number of allowed samples per ray. Typical values: 2, 4, 8, 16 [S1].
*   **Learning Rate:** Adam optimizer with value $5 \times 10^{-4}$ [S1].
*   **Network Architecture:** 8 layers, 256 units for both networks [S1].

### Edge Cases and Pitfalls
*   **Collapse to Constant:** Without density multiplication in ray accumulation (Equation 1), the sampling network collapses to a constant output because the L1-loss alone is insufficient to stabilize it [S1].
*   **Abrupt Sparsification:** If sparsification blending is skipped, the sampling network is abruptly forced sparse, losing potentially important samples [S1].
*   **No Preprocessing Required:** Unlike previous methods, AdaNeRF does not require ground truth depth, pretraining, or special input parametrizations like Plücker coordinates [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_adanerf(ray_origin, ray_direction, sampling_network, shading_network, 
                     fixed_sample_positions, tau, n_max):
    """
    Renders a single ray using AdaNeRF.
    
    Args:
        ray_origin: (3,) array, origin of the ray.
        ray_direction: (3,) array, direction of the ray.
        sampling_network: Function taking (ray_origin, ray_direction) -> (N,) array of deltas.
        shading_network: Function taking (positional_encoded_sample_pos) -> (sigma, color).
        fixed_sample_positions: (N, 3) array, fixed locations along the ray.
        tau: float, threshold for culling.
        n_max: int, maximum number of samples to evaluate.
        
    Returns:
        accumulated_color: (3,) array.
    """
    n_samples = len(fixed_sample_positions)
    
    # Step 1: Evaluate Sampling Network
    # S takes ray origin and direction, outputs delta vector
    deltas = sampling_network(ray_origin, ray_direction)
    
    # Step 2: Adaptive Selection
    # Count samples exceeding threshold
    above_threshold = deltas > tau
    n_s = sum(above_threshold)
    
    selected_indices = []
    
    if n_s == 0:
        # Case 1: No samples exceed threshold. Pick the one with largest delta.
        max_idx = argmax(deltas)
        selected_indices = [max_idx]
    elif n_s <= n_max:
        # Case 2: Number of significant samples is within limit.
        selected_indices = [i for i, d in enumerate(deltas) if d > tau]
    else:
        # Case 3: Too many significant samples. Pick top n_max.
        # Get indices sorted by delta descending
        sorted_indices = argsort(deltas, descending=True)
        selected_indices = sorted_indices[:n_max]
        
    # Step 3: Evaluate Shading Network and Accumulate
    accumulated_color = zeros(3)
    accumulated_transmittance = 1.0
    
    # Sort selected indices to maintain ray order for transmittance calculation
    selected_indices.sort()
    
    for i in selected_indices:
        pos = fixed_sample_positions[i]
        
        # Evaluate Shading Network T
        # T takes positional encoding of sample position
        sigma, color = shading_network(pos)
        
        # Calculate distance to previous sample
        # Note: t_i is distance between adjacent samples. 
        # For the first selected sample, we need distance from previous sample in the sequence.
        # If i is the first selected sample, we need t_i relative to the previous sample index i-1?
        # The formula uses t_i as distance between adjacent samples. 
        # In fixed discretization, t_i is constant or precomputed.
        # Let's assume t_i is precomputed for each position i.
        t_i = distance_between_samples(i) 
        
        # Update transmittance
        # T_i = exp(- sum_{j < i} delta_j * sigma_j * t_j)
        # We maintain accumulated_transmittance which is T_i
        # But we must account for the density of samples that were CULLED but are before i.
        # The source formula sums over j=1 to i-1. 
        # If a sample j < i was culled, its sigma is effectively 0? 
        # Or does the culling happen *before* accumulation?
        # The text says: "evaluate the shading network T only for the remaining subset".
        # And "multiply the density sigma ... with the density delta".
        # If we don't evaluate T for a culled sample, we don't have sigma.
        # However, the transmittance T_i depends on previous densities.
        # If a previous sample was culled, it contributes 0 to the color, but does it contribute to transmittance?
        # The formula is: T_i = exp(- sum_{j=1}^{i-1} delta_j sigma_j t_j).
        # If sample j is culled, we didn't compute sigma_j.
        # Pitfall: "Without density multiplication... sampling network collapses".
        # The multiplication delta * sigma is in the exponent.
        # If we skip evaluating T for culled samples, we must assume sigma_j = 0 for those j.
        # Thus, delta_j * sigma_j = 0.
        # So, we only need to update transmittance for selected samples?
        # No, we need to update transmittance for ALL previous samples j < i.
        # For j not in selected_indices, sigma_j is effectively 0.
        # So the term delta_j * sigma_j * t_j is 0.
        # Therefore, we only need to update accumulated_transmittance when we process a selected sample?
        # Let's check the formula again.
        # T_i is the transmittance BEFORE sample i.
        # T_i = exp(- sum_{j=1}^{i-1} delta_j sigma_j t_j).
        # If we process samples in order, we can maintain a running sum of (delta * sigma * t).
        # For culled samples, sigma=0, so they add 0 to the sum.
        # So we only need to update the sum when we evaluate a selected sample.
        
        # However, we need to know the transmittance T_i for the current sample i.
        # T_i is the transmittance accumulated from all previous samples (selected or not).
        # Since non-selected samples contribute 0 to the density product, 
        # the accumulated transmittance only changes when a selected sample is processed.
        
        # Calculate contribution of current sample
        # Color contribution: T_i * (1 - exp(-delta_i * sigma_i * t_i)) * c_i
        weight = 1.0 - math.exp(-deltas[i] * sigma * t_i)
        accumulated_color += accumulated_transmittance * weight * color
        
        # Update transmittance for NEXT sample
        # T_{i+1} = T_i * exp(-delta_i * sigma_i * t_i)
        accumulated_transmittance *= math.exp(-deltas[i] * sigma * t_i)
        
    return accumulated_color

def train_adanerf(sampling_network, shading_network, optimizer, dataset, phases):
    """
    Trains AdaNeRF using the 4-phase scheme.
    """
    for phase in phases:
        if phase == "Dense Training":
            # 25k iterations
            # Encourage delta -> 1
            # Loss: L1(delta, 1)
            pass
        elif phase == "Sparsification":
            # 50k iterations
            # Linearly blend forcing delta -> 0 and 1
            # Add L1 loss |sigma - delta|
            # Lambda increases from 0 to 1
            pass
        elif phase == "Sparse Training":
            # 225k iterations
            # Lock sampling network weights
            # Train shading network via MSE loss on accumulated RGB
            pass
        elif phase == "Fine-tuning":
            # 300k iterations
            # Fine-tune shading network for specific N_max
            # Removes samples with negligible contribution
            pass
```

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of selected samples per ray, which is typically much less than the total fixed samples $N$ due to adaptive culling. The sampling network is evaluated once per ray. The shading network is evaluated $N_s$ times, where $N_s \le N_{max}$.
*   **Memory Complexity:** The memory footprint is determined by the two MLPs (8 layers, 256 units each). No spatial data structures are required [S1].
*   **Measured Performance:** On the Pavillon scene (DONeRF), with $N_{max}=2$, AdaNeRF achieves 2.00 samples/ray and 28.25 PSNR. With $N_{max}=4$, it achieves 3.99 samples/ray and 29.33 PSNR [S1]. Adaptive sampling with $\tau=0.15$ yields 11.63 samples/ray and 31.68 PSNR, compared to fixed $N=16$ yielding 16.00 samples/ray and 30.89 PSNR [S1].
*   **Hardware:** Evaluated on Nvidia RTX 3090 using a custom real-time renderer based on CUDA and TensorRT [S1].
*   **Strengths:** Real-time rendering capability; no need for spatial data structures; no preprocessing or pretraining required; supports variable sample counts; high quality at low sample counts.
*   **Limitations:** Requires a 4-phase training scheme; performance depends on the quality of the sampling network's importance prediction; fixed sample positions may not be optimal for all scenes compared to learned positions (though this avoids the complexity of integral networks) [S1].

## Background
AdaNeRF was introduced by Kurz et al. in 2022 [S1]. It addresses the need for real-time rendering of neural radiance fields by reducing the number of network evaluations per ray. It builds on [Specific Model Architectures](8585-specific-model-architectures.md) [G]. It is distinct from [TermiNeRF](8776-mega-nerf-spatial-partitioning.md) which requires a pretrained NeRF, and [AutoInt](8776-mega-nerf-spatial-partitioning.md) which uses integral networks with complex training [S1]. It is the first neural representation relying on volume integration that can go down to a single sample per ray and supports variable sample counts without spatial data structures [S1].

## Sources
[S1] AdaNeRF: Adaptive Sampling for Real-Time Rendering of Neural Radiance Fields, https://doi.org/10.1007/978-3-031-19790-1_16, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
