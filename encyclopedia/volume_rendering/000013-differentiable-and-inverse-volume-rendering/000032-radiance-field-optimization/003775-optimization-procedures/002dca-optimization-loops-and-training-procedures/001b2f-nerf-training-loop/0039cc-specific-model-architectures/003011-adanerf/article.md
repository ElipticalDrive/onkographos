# AdaNeRF
Also called: Adaptive Sampling for Real-time Rendering of Neural Radiance Fields
## Overview
AdaNeRF is a method employing fixed sample positions along each ray and incrementally introducing sparsity during training to achieve real-time rendering [S1]. It sets itself apart from its siblings by splitting the NeRF network into a lightweight Sampling Network and a heavier Shading Network, where the former predicts sample importance weights in a single evaluation per ray [S1]. This allows the Shading Network to process only the most significant samples, reducing the number of sample points required for rendering [S2, unofficial].
## How it works
**Inputs and Outputs:**
*   Input: Ray parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), number of total candidate samples $N$.
*   Output: Rendered color $\mathbf{C}(\mathbf{r})$ and depth $\alpha$ (accumulated opacity).

**Data Structures:**
1.  **Sampling Network ($\mathcal{N}_s$):** A lightweight MLP.
    *   Input: Ray parameters (or positional encoding thereof).
    *   Output: A vector of $N$ weights $\mathbf{w} \in \mathbb{R}^N$, representing the importance of each fixed sample position along the ray.
2.  **Shading Network ($\mathcal{N}_c$):** A heavier MLP (standard NeRF architecture).
    *   Input: 3D position $\mathbf{x}_i$ and viewing direction $\mathbf{d}$ for selected samples.
    *   Output: Volume density $\sigma_i$ and view-dependent color $\mathbf{c}_i$.

**Numbered Steps:**
1.  **Initialization:** Pre-train or initialize a standard NeRF model.
2.  **Architecture Split:** Decompose the model into $\mathcal{N}_s$ and $\mathcal{N}_c$ [S1].
3.  **Sampling Weight Prediction:** For a given ray $\mathbf{r}$, evaluate $\mathcal{N}_s$ once to predict importance weights $\mathbf{w}$ for $N$ fixed sample positions $\mathbf{x}_1, \dots, \mathbf{x}_N$ along the ray [S1].
4.  **Sparsity Introduction:** During training, incrementally introduce sparsity by selecting only the top-$k$ samples with the highest weights $\mathbf{w}$ [S1].
5.  **Shading:** Evaluate $\mathcal{N}_c$ only for the selected $k$ samples to obtain $\sigma_i$ and $\mathbf{c}_i$ [S1].
6.  **Volume Rendering:** Compute the final color using standard volume rendering integration over the selected samples:
    $$
    \mathbf{C}(\mathbf{r}) = \sum_{i=1}^{k} T_i \alpha_i \mathbf{c}_i
    $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, and $\delta_i$ is the distance between adjacent samples [G].
7.  **Fine-tuning:** Fine-tune the combined model to the desired number of samples $k$ to create a compact neural representation [S1].

**Parameters:**
*   $N$: Total candidate sample positions (fixed along the ray). Value: Not given in the sources.
*   $k$: Number of selected significant samples. Value: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Initialization:** While TermiNeRF requires a pre-trained NeRF to initialize the color network for best results, AdaNeRF's specific initialization requirements are not explicitly detailed beyond the split architecture [S1].
*   **Precision:** The method relies on the accuracy of $\mathcal{N}_s$ to identify significant samples; poor weight prediction may lead to aliasing or missing geometry.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray_ada_nef(ray_origin, ray_direction, N, k, sampling_net, shading_net):
    # Step 1: Predict importance weights for N fixed sample positions
    # Assume fixed_positions is precomputed for the ray
    weights = sampling_net(ray_origin, ray_direction) # Returns vector of size N
    
    # Step 2: Select top-k samples based on weights
    top_k_indices = argsort_descending(weights)[:k]
    selected_positions = fixed_positions[top_k_indices]
    
    # Step 3: Shade only selected samples
    densities = []
    colors = []
    for pos in selected_positions:
        sigma, c = shading_net(pos, ray_direction)
        densities.append(sigma)
        colors.append(c)
        
    # Step 4: Volume rendering integration
    # Calculate deltas between selected positions (and boundaries)
    deltas = compute_deltas(selected_positions, ray_origin, ray_direction)
    
    transmittance = 1.0
    final_color = [0.0, 0.0, 0.0]
    
    for i in range(k):
        alpha = 1.0 - exp(-densities[i] * deltas[i])
        final_color += transmittance * alpha * colors[i]
        transmittance *= (1.0 - alpha)
        
    return final_color
```
## Subtopics

### TermiNeRF
Uses a sampling network conditioned on dense depth maps to localize surfaces. Choosable when depth priors are available. [8668-dasnerf.md] (Note: Source [S1] mentions TermiNeRF in the context of sampling networks).

### SfMNeRF
Uses Structure-from-Motion priors. [8664-sfmnerf.md]

### DASNeRF
Integrates external depth features. [8668-dasnerf.md]

### R2L (Neural Light Field)
Uses 2D pixel coordinates. [8670-r2l-neural-light-field.md]

### AligNeRF
Alignment-focused architecture. [8695-alignerf.md]

### HyperNeRFGAN
Uses hypernetworks for weights. [8765-hypernerfgan.md]

### Mega-NeRF Spatial Partitioning
Decomposes scene into sub-networks. [8776-mega-nerf-spatial-partitioning.md]

### MultiPlaneNeRF
Uses 2D image projections. [8875-multiplanenerf.md]

### Sparse Radiance Field (SRF) Representation
Uses sparse voxel coordinates. [8876-sparse-radiance-field-srf-representation.md]

## Operating characteristics
*   **Time Complexity:** Reduced compared to standard NeRF because the shading network is evaluated only $k$ times per ray instead of $N$ times. The sampling network adds a lightweight overhead per ray.
*   **Memory Complexity:** Requires storing two networks ($\mathcal{N}_s$ and $\mathcal{N}_c$). The compact representation allows for real-time rendering [S1].
*   **Measured Performance:** Capable of real-time rendering after fine-tuning [S1].
*   **Strengths:** Real-time rendering capability; reduces sample count effectively [S2, unofficial].
*   **Limitations:** Requires a two-stage training process (splitting and fine-tuning); relies on the sampling network's ability to accurately predict importance [S1].
*   **When to use:** When real-time rendering is required and computational budget for shading is limited.
*   **When not to use:** When high geometric accuracy in complex, ambiguous regions is critical without pre-training, or when depth priors are unavailable and sparsity might miss details.
## Background
AdaNeRF was introduced by Kurz et al. in 2022 [S1]. It is part of the broader trend of accelerating neural radiance fields by baking or optimizing sampling strategies. It builds on the concept of sampling networks used in methods like TermiNeRF [S1].
## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
